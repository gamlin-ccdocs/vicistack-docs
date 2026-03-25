# VICIdial Kamailio Load Balancing for 100+ Agent Call Centers

When your VICIdial deployment crosses the 100-agent threshold, a single Asterisk server stops being viable. Not because Asterisk cannot handle the call volume -- a well-tuned Asterisk box can manage 300+ concurrent calls -- but because you lose all fault tolerance. One kernel panic, one runaway process, one failed disk, and your entire operation goes silent.

The answer is horizontal scaling: multiple Asterisk servers behind a SIP load balancer. And in the VICIdial world, that load balancer is Kamailio.

This guide covers why Kamailio is the right tool, how to configure its dispatcher module for VICIdial, how to implement health probing and automatic server removal, and how to scale from a single server to a 10-server cluster. Every configuration example has been tested in production VICIdial environments running 100-500 agents.

## Why You Need Kamailio at Scale

VICIdial's built-in multi-server support handles database replication and web interface distribution well. But for SIP traffic -- the actual calls -- VICIdial relies on Asterisk's native capabilities, which were not designed for clustered operation.

Here is what breaks without a proper SIP load balancer:

### Problem 1: Uneven Call Distribution

VICIdial assigns calls to servers based on the campaign's server configuration. If Server A and Server B are both assigned to a campaign, VICIdial's hopper tries to distribute leads across servers, but the distribution is based on lead assignment, not real-time server load. Server A might have 180 concurrent calls while Server B sits at 60.

### Problem 2: No Graceful Failure Handling

If Asterisk crashes on Server A, the calls in progress are lost. There is no mechanism to redirect new calls to Server B until VICIdial's keepalive process detects the failure -- which can take 30-60 seconds. During that window, new calls to Server A fail.

### Problem 3: Carrier-to-Server Routing

When inbound calls arrive from your SIP carrier, they hit a single IP address. Without a load balancer, that IP belongs to one Asterisk server. If that server is overloaded or down, the carrier gets a SIP error and the call is lost.

### Problem 4: Codec Transcoding Bottleneck

If different carriers use different codecs, Asterisk must transcode. Transcoding is CPU-intensive -- a single G.729 to G.711 transcoding operation uses roughly 10x the CPU of a native G.711 call. Under heavy load, transcoding on a single server can cause audio quality issues across all calls on that server.

Kamailio solves all of these problems. It sits in front of your Asterisk servers as a SIP proxy, distributing calls based on configurable algorithms, monitoring server health, and transparently rerouting traffic when servers fail.

## Architecture Overview

Here is the target architecture:

```
                    +-----------+
  SIP Carriers ---->| Kamailio  |
  (inbound/        | SIP Proxy |
   outbound)       | Port 5060 |
                    +-----+-----+
                          |
              +-----------+-----------+
              |           |           |
        +-----+---+ +----+----+ +----+----+
        |Asterisk | |Asterisk | |Asterisk |
        |Server 1 | |Server 2 | |Server 3 |
        |Port 5080| |Port 5080| |Port 5080|
        +----+----+ +----+----+ +----+----+
              |           |           |
              +-----------+-----------+
                          |
                    +-----+-----+
                    | VICIdial  |
                    | Database  |
                    | (MySQL)   |
                    +-----------+
```

Kamailio listens on port 5060 (the standard SIP port) and proxies calls to Asterisk servers listening on port 5080. Each Asterisk server connects to the shared VICIdial MySQL database for call routing, agent assignment, and CDR logging.

## Dispatcher Module Configuration

The Kamailio `dispatcher` module is the core of SIP load balancing. It maintains a list of destinations (your Asterisk servers), monitors their health, and distributes calls across them.

### Step 1: Install Kamailio with Dispatcher

On CentOS/RHEL (common for VICIdial):

```bash
# Add Kamailio repository
cat > /etc/yum.repos.d/kamailio.repo << 'REPO'
[kamailio]
name=Kamailio packages
baseurl=https://rpm.kamailio.org/stable/el9/
gpgcheck=0
enabled=1
REPO

# Install Kamailio with required modules
yum install -y kamailio kamailio-mysql kamailio-utils kamailio-tls
```

### Step 2: Configure the Dispatcher List

The dispatcher list defines your Asterisk servers, their weights, and their group assignments. Create the dispatcher list file:

```
# /etc/kamailio/dispatcher.list
# Format: setid destination flags priority attributes

# Set 1: Outbound Asterisk servers
1 sip:10.0.0.11:5080 0 10 weight=50;duid=ast1
1 sip:10.0.0.12:5080 0 5  weight=30;duid=ast2
1 sip:10.0.0.13:5080 0 3  weight=20;duid=ast3

# Set 2: Inbound Asterisk servers (may be the same or different)
2 sip:10.0.0.11:5080 0 10 weight=40;duid=ast1
2 sip:10.0.0.12:5080 0 10 weight=40;duid=ast2
2 sip:10.0.0.13:5080 0 10 weight=20;duid=ast3
```

Each line defines:
- **setid** -- A group identifier. Use different sets for different routing purposes (outbound vs inbound, or by campaign).
- **destination** -- The SIP URI of the Asterisk server. Use the internal IP and the port Asterisk listens on (5080 in our setup).
- **flags** -- 0 for normal operation. Set to 1 to mark a destination as inactive (maintenance mode).
- **priority** -- Higher values mean higher priority. Used with priority-based routing algorithms.
- **attributes** -- Key-value pairs. The `weight` attribute controls proportional distribution. The `duid` is a unique identifier for logging.

### Step 3: Core Kamailio Configuration

Here is a production-tested `kamailio.cfg` for VICIdial load balancing:

```
#!KAMAILIO
#
# VICIdial SIP Load Balancer Configuration
# Kamailio 5.7+

####### Global Parameters #########

debug=2
log_stderror=no
log_facility=LOG_LOCAL0
fork=yes
children=8
auto_aliases=no
listen=udp:0.0.0.0:5060
listen=tcp:0.0.0.0:5060
server_header="Server: ViciStack-LB"
user_agent_header="User-Agent: ViciStack-LB"

####### Modules Section ########

loadmodule "tm.so"
loadmodule "sl.so"
loadmodule "rr.so"
loadmodule "pv.so"
loadmodule "maxfwd.so"
loadmodule "textops.so"
loadmodule "siputils.so"
loadmodule "xlog.so"
loadmodule "sanity.so"
loadmodule "nathelper.so"
loadmodule "rtpengine.so"
loadmodule "dispatcher.so"

# ----- tm params -----
modparam("tm", "fr_timer", 5000)        # 5 second final response timeout
modparam("tm", "fr_inv_timer", 30000)    # 30 second INVITE timeout
modparam("tm", "restart_fr_on_each_reply", 1)

# ----- rr params -----
modparam("rr", "enable_full_lr", 1)
modparam("rr", "append_fromtag", 1)

# ----- nathelper params -----
modparam("nathelper", "received_avp", "$avp(RECEIVED)")
modparam("nathelper", "sipping_bflag", 7)

# ----- rtpengine params -----
modparam("rtpengine", "rtpengine_sock", "udp:127.0.0.1:2223")

# ----- dispatcher params -----
modparam("dispatcher", "list_file", "/etc/kamailio/dispatcher.list")
modparam("dispatcher", "ds_probing_mode", 1)        # Enable probing
modparam("dispatcher", "ds_ping_interval", 15)       # Probe every 15 seconds
modparam("dispatcher", "ds_probing_threshold", 3)    # 3 failures = inactive
modparam("dispatcher", "ds_ping_reply_codes", "class2;class3;class4")
modparam("dispatcher", "ds_ping_from", "sip:loadbalancer@vicistack.local")

####### Routing Logic ########

request_route {
    # Per-request initial checks
    if (!mf_process_maxfwd_header("10")) {
        sl_send_reply("483", "Too Many Hops");
        exit;
    }

    if (!sanity_check("1511", "7")) {
        xlog("L_WARN", "Malformed SIP message from $si:$sp\n");
        exit;
    }

    # CANCEL processing
    if (is_method("CANCEL")) {
        if (t_check_trans()) {
            t_relay();
        }
        exit;
    }

    # Handle retransmissions
    if (!is_method("ACK")) {
        if (t_precheck_trans()) {
            t_check_trans();
            exit;
        }
        t_check_trans();
    }

    # Record-Route for mid-dialog requests
    if (is_method("INVITE|SUBSCRIBE")) {
        record_route();
    }

    # Handle sequential requests (in-dialog)
    if (has_totag()) {
        if (loose_route()) {
            if (is_method("INVITE")) {
                record_route();
            }
            route(RELAY);
            exit;
        }

        if (is_method("ACK")) {
            if (t_check_trans()) {
                t_relay();
                exit;
            }
            exit;
        }

        sl_send_reply("404", "Not Here");
        exit;
    }

    # Handle REGISTER - not typically needed for VICIdial
    if (is_method("REGISTER")) {
        sl_send_reply("404", "No registrar");
        exit;
    }

    # Handle OPTIONS for health checks from carriers
    if (is_method("OPTIONS") && uri==myself) {
        sl_send_reply("200", "OK");
        exit;
    }

    # Route INVITE requests through dispatcher
    if (is_method("INVITE")) {
        route(DISPATCH);
        exit;
    }

    route(RELAY);
}

# Dispatch route - load balance across Asterisk servers
route[DISPATCH] {
    # Use set 1 for outbound, set 2 for inbound
    # Detect direction based on source: carrier IPs = inbound

    $var(ds_set) = 1;  # Default: outbound

    # If source is a known carrier IP, use inbound set
    if ($si == "203.0.113.10" || $si == "203.0.113.20" || $si == "198.51.100.5") {
        $var(ds_set) = 2;  # Inbound from carrier
    }

    # Algorithm 4 = weighted round-robin
    # Algorithm 10 = weighted based on load
    if (!ds_select_dst("$var(ds_set)", "4")) {
        # All servers down - return 503
        xlog("L_ERR", "CRITICAL: No Asterisk servers available for call from $fu to $ru\n");
        sl_send_reply("503", "Service Unavailable");
        exit;
    }

    xlog("L_INFO", "Dispatching $rm from $fu to $du (set=$var(ds_set))\n");

    # Set failure route for automatic failover
    t_on_failure("DISPATCH_FAILURE");

    route(RELAY);
}

# Relay route
route[RELAY] {
    if (!t_relay()) {
        sl_reply_error();
    }
    exit;
}

# Failure route - try next server if current one fails
failure_route[DISPATCH_FAILURE] {
    if (t_is_canceled()) {
        exit;
    }

    # If we get a server error, try the next server
    if (t_check_status("500|503")) {
        xlog("L_WARN", "Server $du returned error, trying next server\n");

        # Mark current destination as probing (temporarily suspect)
        ds_mark_dst("p");

        # Try next destination in the set
        if (ds_next_dst()) {
            xlog("L_INFO", "Failing over to $du\n");
            t_on_failure("DISPATCH_FAILURE");
            t_relay();
            exit;
        }

        xlog("L_ERR", "CRITICAL: All servers exhausted for call from $fu\n");
    }

    # For other failure codes (busy, no answer), don't failover
    # These are called-party issues, not server issues
}
```

This configuration handles the complete SIP flow: initial request routing, mid-dialog routing (so audio continues to flow correctly), health monitoring, and automatic failover when a server returns an error.

## Weighted Round-Robin vs Least-Connections

The dispatcher module supports multiple load balancing algorithms. The two most relevant for VICIdial are weighted round-robin and call-load based.

### Algorithm 4: Weighted Round-Robin

Distributes calls proportionally based on the weight assigned to each server. A server with weight=50 receives roughly 50% of calls, weight=30 gets 30%, and weight=20 gets 20%.

**Best for:** Heterogeneous server hardware. If Server 1 has 32 CPU cores and Server 2 has 16 cores, weight them 2:1.

```
# dispatcher.list with weighted round-robin
1 sip:10.0.0.11:5080 0 10 weight=50;duid=ast1-32core
1 sip:10.0.0.12:5080 0 5  weight=30;duid=ast2-16core
1 sip:10.0.0.13:5080 0 3  weight=20;duid=ast3-8core
```

### Algorithm 10: Call-Load Based (Weight + Active Calls)

Distributes calls based on both weight and the current number of active calls on each server. Kamailio tracks how many calls it has proxied to each destination and routes new calls to the least-loaded server, adjusted by weight.

**Best for:** Homogeneous hardware where you want even distribution based on actual load rather than static weights.

To use algorithm 10, change the dispatch line in the config:

```
# In route[DISPATCH]:
if (!ds_select_dst("$var(ds_set)", "10")) {
```

### Which Should You Use?

For most VICIdial deployments, start with algorithm 4 (weighted round-robin). It is simpler, more predictable, and easier to debug. Switch to algorithm 10 if you observe uneven load distribution due to varying call durations or mixed inbound/outbound traffic patterns.

## Health Probing and Automatic Removal

The dispatcher module's probing system is critical for production reliability. Here is how it works and how to tune it.

### How Probing Works

When `ds_probing_mode=1`, Kamailio sends SIP OPTIONS requests to each destination at the interval specified by `ds_ping_interval`. If a destination fails to respond to `ds_probing_threshold` consecutive probes, Kamailio marks it as inactive and stops sending calls to it.

When the destination starts responding again, Kamailio automatically re-activates it.

### Tuning Probe Parameters

```
# Aggressive probing for fast failover
modparam("dispatcher", "ds_ping_interval", 10)       # Probe every 10 seconds
modparam("dispatcher", "ds_probing_threshold", 2)     # 2 failures = inactive
# Failover time: 10s * 2 = 20 seconds worst case

# Conservative probing to avoid false positives
modparam("dispatcher", "ds_ping_interval", 30)       # Probe every 30 seconds
modparam("dispatcher", "ds_probing_threshold", 3)     # 3 failures = inactive
# Failover time: 30s * 3 = 90 seconds worst case
```

For a 100+ agent center, we recommend 15-second intervals with a threshold of 3. This gives you failover within 45 seconds while avoiding false positives from temporary network hiccups.

### Monitoring Dispatcher State

Check the current state of your dispatch destinations from the Kamailio CLI:

```bash
# Show all dispatcher destinations and their status
kamcmd dispatcher.list

# Output example:
# SET: 1
#   URI: sip:10.0.0.11:5080 FLAGS:AP PRIORITY:10 ATTRS:weight=50;duid=ast1
#   URI: sip:10.0.0.12:5080 FLAGS:AP PRIORITY:5  ATTRS:weight=30;duid=ast2
#   URI: sip:10.0.0.13:5080 FLAGS:IP PRIORITY:3  ATTRS:weight=20;duid=ast3

# FLAGS meaning:
# A = Active
# P = Probing enabled
# I = Inactive (failed probes)
# D = Disabled (manually)
```

### Manual Server Maintenance

To take a server out of rotation for maintenance without affecting active calls:

```bash
# Mark server as inactive (stops new calls, existing calls continue)
kamcmd dispatcher.set_state i 1 sip:10.0.0.13:5080

# Perform maintenance on the server...

# Re-activate the server
kamcmd dispatcher.set_state a 1 sip:10.0.0.13:5080
```

## Scaling from 1 to 10 Asterisk Servers

Here is a practical scaling roadmap for growing your VICIdial cluster.

### Stage 1: Single Server (Up to 50 Agents)

No Kamailio needed. Single Asterisk server handles all traffic. Focus on optimizing that server: tuning AMD, carrier configuration, and agent settings.

### Stage 2: Dual Server with Kamailio (50-100 Agents)

Add a second Asterisk server and deploy Kamailio as the SIP proxy. At this stage, Kamailio can run on one of the Asterisk servers or on a separate lightweight VM.

```
# dispatcher.list for dual-server setup
1 sip:10.0.0.11:5080 0 10 weight=50;duid=ast1
1 sip:10.0.0.12:5080 0 10 weight=50;duid=ast2
```

Equal weights because both servers are identical. Active-active with automatic failover.

### Stage 3: Three Servers with Dedicated Kamailio (100-200 Agents)

Move Kamailio to its own dedicated server (it needs minimal resources: 2 CPU cores, 2GB RAM). Add a third Asterisk server.

```
# dispatcher.list for three-server setup
1 sip:10.0.0.11:5080 0 10 weight=34;duid=ast1
1 sip:10.0.0.12:5080 0 10 weight=33;duid=ast2
1 sip:10.0.0.13:5080 0 10 weight=33;duid=ast3
```

### Stage 4: Five+ Servers with Redundant Kamailio (200-500 Agents)

Deploy two Kamailio instances behind a virtual IP (using keepalived/VRRP) for load balancer redundancy. Five or more Asterisk servers in the dispatch pool.

```
# dispatcher.list for five-server setup
1 sip:10.0.0.11:5080 0 10 weight=20;duid=ast1
1 sip:10.0.0.12:5080 0 10 weight=20;duid=ast2
1 sip:10.0.0.13:5080 0 10 weight=20;duid=ast3
1 sip:10.0.0.14:5080 0 10 weight=20;duid=ast4
1 sip:10.0.0.15:5080 0 10 weight=20;duid=ast5
```

Keepalived configuration for Kamailio redundancy:

```
# /etc/keepalived/keepalived.conf on Kamailio Primary
vrrp_script check_kamailio {
    script "/usr/bin/kamcmd dispatcher.list > /dev/null 2>&1"
    interval 5
    weight -20
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass vicistacklb
    }

    virtual_ipaddress {
        10.0.0.100/24
    }

    track_script {
        check_kamailio
    }
}
```

### Stage 5: 10 Servers and Beyond (500+ Agents)

At this scale, consider:
- Separate dispatcher sets for inbound vs outbound traffic
- Geographic distribution with DNS-based failover between data centers
- Dedicated media servers (running rtpengine) separate from signaling
- Database read replicas for VICIdial reporting
- Dedicated recording storage servers

```
# dispatcher.list for 10-server setup with role separation
# Set 1: Outbound dialing servers
1 sip:10.0.0.11:5080 0 10 weight=10;duid=out1
1 sip:10.0.0.12:5080 0 10 weight=10;duid=out2
1 sip:10.0.0.13:5080 0 10 weight=10;duid=out3
1 sip:10.0.0.14:5080 0 10 weight=10;duid=out4
1 sip:10.0.0.15:5080 0 10 weight=10;duid=out5
1 sip:10.0.0.16:5080 0 10 weight=10;duid=out6
1 sip:10.0.0.17:5080 0 10 weight=10;duid=out7

# Set 2: Inbound/IVR servers (separate pool)
2 sip:10.0.0.18:5080 0 10 weight=34;duid=in1
2 sip:10.0.0.19:5080 0 10 weight=33;duid=in2
2 sip:10.0.0.20:5080 0 10 weight=33;duid=in3
```

## Performance Considerations

### Kamailio Resource Requirements

Kamailio is extremely efficient as a SIP proxy. It does not handle media (audio), only SIP signaling. Resource requirements scale linearly with calls per second (CPS), not concurrent calls.

| Call Volume | CPU | RAM | Network |
|-------------|-----|-----|---------|
| Up to 50 CPS | 2 cores | 2 GB | 100 Mbps |
| 50-200 CPS | 4 cores | 4 GB | 1 Gbps |
| 200-500 CPS | 8 cores | 8 GB | 1 Gbps |
| 500+ CPS | 16 cores | 16 GB | 10 Gbps |

A 100-agent center doing 200 dials/hour/agent generates approximately 5.5 CPS. Even a minimal Kamailio deployment handles this with negligible CPU usage.

### Network Latency

Kamailio adds minimal latency to the SIP signaling path -- typically under 1ms. However, ensure that Kamailio and Asterisk servers are on the same LAN segment (or at least the same data center) to avoid introducing latency that affects call setup time and AMD detection accuracy.

### RTP Media Path

By default, the RTP (audio) media flows directly between the carrier and Asterisk, bypassing Kamailio entirely. This is optimal for performance. If you need Kamailio to proxy RTP as well (for NAT traversal or media recording at the proxy level), deploy rtpengine alongside Kamailio.

## How ViciStack Helps

Designing, deploying, and maintaining a Kamailio load-balanced VICIdial cluster is nontrivial. Configuration mistakes can cause one-way audio, dropped calls, or incorrect CDR logging. ViciStack has deployed Kamailio-based architectures for call centers ranging from 50 to 500+ agents.

Our managed VICIdial optimization includes:

- **Architecture design** tailored to your agent count and growth plans
- **Kamailio deployment and configuration** with health monitoring and automatic failover
- **Ongoing monitoring** of dispatcher health, server load, and call distribution
- **Scaling assistance** when you add servers to the cluster
- **All at $150/agent/month flat** -- infrastructure optimization is included, not an add-on

**[Get your free infrastructure analysis -- we will review your current VICIdial architecture and provide a scaling roadmap.](https://vicistack.com/proof/)**

We respond within 5 minutes. No contracts, no commitment. Just a clear picture of where your infrastructure stands and where it needs to go.

## Further Reading

- [VICIdial SIP Trunk Failover and Redundancy: Complete Setup Guide](/blog/vicidial-sip-trunk-failover) -- carrier-level failover that complements Kamailio server-level balancing
- [How to Reduce VICIdial AMD False Positives from 20% to Under 5%](/blog/vicidial-amd-false-positive-reduction) -- AMD tuning considerations in multi-server environments
- [VICIdial Answering Machine Detection vs AI-Based AMD: Which Is Better?](/blog/vicidial-amd-vs-ai-amd) -- AI AMD deployment in load-balanced clusters

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-kamailio-load-balancing).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
