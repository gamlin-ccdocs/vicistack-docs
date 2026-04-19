# VoIP Mos Score Testing Tools

**Last updated: March 2026 | Reading time: ~22 minutes**

Your agents are complaining about [call quality](/blog/vicidial-carrier-selection/). Customers sound robotic. There's echo. Words get clipped. Your manager wants a number that proves the calls are bad, and your carrier wants a number that proves they're not responsible.

That number is the [MOS score](/blog/voip-mos-score-guide/) — Mean Opinion Score — and it's been the standard metric for voice quality since 1996. The problem is that most people who talk about MOS scores have never measured one. They parrot the "4.0+ is good, below 3.5 is bad" line without knowing how to produce the number in the first place.

I'm going to show you exactly how to measure MOS scores on your VoIP/VICIdial system using tools that range from free (Wireshark, Asterisk CLI) to expensive (POLQA hardware probes). You'll leave here with actual testing procedures, not marketing copy.

---

## MOS Score Basics (The 60-Second Version)

MOS was originally defined by ITU-T P.800 as a subjective test: you play audio samples to a panel of listeners, they rate quality from 1 to 5, and you average the results. That's the "Mean Opinion Score" — literally the average opinion of human listeners.

| MOS Score | Quality | What You Hear |
|-----------|---------|---------------|
| 4.3-5.0 | Excellent | Toll-quality, indistinguishable from landline |
| 4.0-4.3 | Good | Minor imperfections, still natural |
| 3.6-4.0 | Fair | Noticeable degradation, still intelligible |
| 3.1-3.5 | Poor | Significant distortion, effort to understand |
| 2.6-3.0 | Bad | distorted, barely usable |
| 1.0-2.5 | Unusable | You'd rather communicate by smoke signal |

For a call center, **4.0 is your minimum target**. Anything below 3.6 and your agents will struggle to understand customers, call times increase, customer satisfaction drops, and conversion rates tank. We've measured the correlation across our client base: a 0.5-point MOS drop corresponds to roughly 8-12% longer average handle times.

### The G.711 Ceiling

No VoIP call can ever reach a perfect 5.0 MOS. Even under laboratory conditions with zero packet loss and zero jitter, the codec itself introduces some degradation:

| Codec | Maximum MOS | Bandwidth |
|-------|-------------|-----------|
| G.711 (ulaw/alaw) | 4.4 | 87.2 kbps |
| G.722 (wideband) | 4.5 | 87.2 kbps |
| G.729 | 3.92 | 31.2 kbps |
| Opus | 4.5+ | 6-510 kbps (variable) |
| GSM | 3.5 | 13.2 kbps |
| iLBC | 4.14 | 15.2 kbps |

If you're using G.729 to save bandwidth, your MOS ceiling is 3.92 before any network impairment. Add even 1% packet loss and you're below 3.5. For call center quality, G.711 ulaw is the standard because it gives you the most headroom before quality degrades.

---

## Method 1: Wireshark RTP Stream Analysis (Free)

Wireshark is the most practical tool for diagnosing VoIP quality issues. It captures the actual RTP packets and calculates jitter, packet loss, and estimated MOS for every stream in the capture.

### Setup

1. Install Wireshark on a machine that can see VoIP traffic. This could be:
 - The Asterisk server itself (captures all call traffic)
 - A network tap or mirror port on the switch connecting your Asterisk server
 - An admin workstation with port mirroring enabled

2. Start a capture with an RTP-focused filter:

```
udp portrange 10000-20000
```

(Adjust the port range to match your Asterisk RTP port configuration in `rtp.conf`:)

```ini
# /etc/asterisk/rtp.conf
[general]
rtpstart=10000
rtpend=20000
```

3. Let it run during a few calls. A 3-minute call generates about 9,000 RTP packets (50 packets/second x 180 seconds per direction).

### Analysis

After capturing, go to **Telephony > RTP > RTP Streams** in Wireshark. You'll see every RTP stream in the capture with:

- **Source/Destination IP and Port**: Identifies the call legs
- **SSRC**: Synchronization Source — unique per stream
- **Packets**: Total RTP packets received
- **Lost**: Packets that never arrived (gap in sequence numbers)
- **Max Jitter**: Maximum inter-packet arrival variation in ms
- **Mean Jitter**: Average jitter across the stream
- **Max Delta**: Maximum time between consecutive packets

Select a stream and click **Analyze**. The detailed view shows:

- Per-packet jitter values graphed over time
- Sequence number gaps (packet loss events)
- Timestamp issues

### Estimating MOS from Wireshark Data

Wireshark doesn't directly display MOS scores, but you can calculate an estimate using the E-model (ITU-T G.107) from the captured metrics:

**Step 1**: Get your jitter and packet loss numbers from the RTP stream analysis.

**Step 2**: Calculate the R-factor:

```
R = 93.2 - (packet_loss * 2.5) - (jitter_ms * 0.04)
```

This is a simplified version of the E-model formula. The full E-model has dozens of parameters, but for practical call center diagnostics, the simplified version gets you within 0.2 points of the full calculation.

**Step 3**: Convert R-factor to MOS:

```
If R < 0: MOS = 1.0
If R > 100: MOS = 4.5
Otherwise: MOS = 1 + 0.035*R + R*(R-60)*(100-R)*7e-6
```

**Example**: You measure 0.5% packet loss and 12ms average jitter:
```
R = 93.2 - (0.5 * 2.5) - (12 * 0.04)
R = 93.2 - 1.25 - 0.48
R = 91.47
MOS = 1 + 0.035*91.47 + 91.47*(91.47-60)*(100-91.47)*7e-6
MOS ≈ 4.35
```

That's excellent quality. Contrast with 3% packet loss and 40ms jitter:
```
R = 93.2 - (3 * 2.5) - (40 * 0.04)
R = 93.2 - 7.5 - 1.6
R = 84.1
MOS ≈ 4.09
```

Still acceptable but noticeably degraded.

### What "Normal" Looks Like

From our deployments, here's what healthy VICIdial systems show in Wireshark:

| Metric | Healthy | Warning | Critical |
|--------|---------|---------|----------|
| Packet Loss | <0.5% | 0.5-2% | >2% |
| Avg Jitter | <20ms | 20-40ms | >40ms |
| Max Jitter | <50ms | 50-100ms | >100ms |
| R-Factor | >85 | 75-85 | <75 |
| Est. MOS | >4.0 | 3.6-4.0 | <3.6 |

---

## Method 2: Asterisk CLI (Free)

Asterisk provides real-time quality metrics for active calls through the CLI. This won't give you a MOS score directly, but it shows the underlying metrics (jitter, loss, RTT) that you can plug into the E-model formula.

### Checking Active Channel Quality

While a call is in progress:

```bash
asterisk -rx "pjsip show channel PJSIP/carrier-00000042"
```

Look for the RTP statistics section in the output. You'll see:

- `Rxcount` / `Txcount` — packets received and transmitted
- `Rxploss` / `Txploss` — receive and transmit packet loss
- `RTT` — round-trip time in milliseconds
- `Jitter` — measured jitter in ms

For chan_sip (legacy):

```bash
asterisk -rx "sip show channel SIP/carrier-00000042"
```

### Enabling RTP Statistics Logging

You can configure Asterisk to log RTP quality metrics for every call. In `rtp.conf`:

```ini
[general]
rtpstart=10000
rtpend=20000
rtpchecksums=no
strictrtp=yes
icesupport=yes
```

Then in Asterisk CLI:

```bash
asterisk -rx "rtp set debug on"
```

This outputs per-packet RTP statistics to the console, including jitter calculations. It's verbose — don't leave it on in production. Use it for diagnostic sessions.

### Using CDR Quality Fields

Asterisk CDRs (Call Detail Records) include quality data if your system is configured to log it. The `userfield` or custom CDR variables can capture end-of-call RTP statistics.

You can query these from the VICIdial database:

```sql
SELECT calldate, src, dst, duration, userfield
FROM cdr
WHERE calldate > DATE_SUB(NOW(), INTERVAL 1 HOUR)
ORDER BY calldate DESC
LIMIT 20;
```

If quality data isn't showing up in your CDRs, you may need to add AGI or dialplan logic to capture the channel's RTP stats before hangup and write them to the CDR.

---

## Method 3: Command-Line Network Testing (Free)

Before you blame the carrier for bad audio, verify that your network can handle VoIP traffic. These tools test the network path between your Asterisk server and the carrier without involving actual calls.

### iperf3 — Bandwidth and Jitter Test

Install on your Asterisk server and a remote endpoint:

```bash
# On the remote endpoint (or use a public iperf3 server)
iperf3 -s

# On your Asterisk server
iperf3 -c REMOTE_IP -u -b 1M -t 30 -i 1
```

The `-u` flag uses UDP (like RTP), `-b 1M` sets bandwidth to 1 Mbps (roughly 12 concurrent G.711 calls), `-t 30` runs for 30 seconds.

Look for:
- **Jitter**: Should be under 20ms
- **Lost/Total**: Packet loss should be under 0.5%
- **Bandwidth**: Should be consistent across all intervals

### mtr — Latency and Path Analysis

```bash
mtr -rwc 100 CARRIER_SIP_IP
```

This sends 100 probes and reports per-hop latency and loss. Look for:
- **Last-mile loss**: Loss at the final hop (carrier's edge) points to carrier issues
- **Mid-path loss**: Loss at intermediate hops points to transit network issues
- **High jitter at any hop**: Indicates congestion or buffering

### ping — Basic Latency Check

```bash
ping -c 100 -i 0.02 CARRIER_SIP_IP
```

Fast-paced ping (50 per second) to simulate real-time traffic timing. Check:
- **Average RTT**: Should be under 80ms for good call quality (one-way latency = RTT/2, target <40ms)
- **Packet loss**: Should be 0%
- **Jitter** (stddev in output): Should be under 10ms

---

## Method 4: PESQ and POLQA (Commercial)

PESQ (ITU-T P.862) and its successor POLQA (ITU-T P.863) are the gold standard for objective voice quality measurement. They work by comparing a reference audio signal to the degraded signal after it's traveled through your VoIP system.

### How They Work

1. You inject a known reference audio file into one end of the call
2. The audio travels through your VoIP system (Asterisk, SIP trunks, carrier network)
3. You capture the received audio at the other end
4. The algorithm compares the reference to the received audio and produces a MOS-LQO (Listening Quality Objective) score

### PESQ vs POLQA

| Feature | PESQ (P.862) | POLQA (P.863) |
|---------|-------------|---------------|
| Year | 2001 | 2011 |
| Status | Superseded | Current |
| Narrowband (8kHz) | Yes | Yes |
| Wideband (16kHz) | Yes (P.862.2) | Yes |
| Super-wideband (32kHz) | No | Yes |
| Modern codecs (Opus, EVS) | Poor | Good |
| Cost | $5,000-15,000 | $15,000-50,000 |

For VICIdial deployments using G.711, PESQ is fine and much cheaper. POLQA is needed only if you're running wideband codecs or need to validate modern codec quality.

### Free/Open-Source PESQ Alternatives

The ITU published the PESQ algorithm source code (P.862 reference implementation) for research purposes. You can find it referenced in academic papers and some open-source projects. It's not commercially licensed for production use, but it's useful for internal testing.

There's also the ViSQOL (Virtual Speech Quality Objective Listener) project from Google, which is open source and available on GitHub. It provides MOS predictions using a different algorithm than PESQ/POLQA but with reasonable accuracy.

### Practical PESQ Testing Setup

For a VICIdial system, here's a practical test workflow:

1. Create a test campaign with a DID that routes to an extension playing back a reference audio file
2. Call the DID from a softphone or another SIP endpoint
3. Record the audio at the receiving end
4. Run the reference and degraded audio through a PESQ tool
5. The output MOS-LQO tells you the end-to-end voice quality

This tests the full path: local network → Asterisk → MeetMe bridge → SIP trunk → carrier → return path.

---

## Method 5: Continuous Monitoring (Free + Commercial)

One-time tests are useful for diagnostics, but VoIP quality changes throughout the day as network conditions fluctuate. Continuous monitoring catches degradation before agents start complaining.

### VoIPmonitor (Free/Open Source)

VoIPmonitor is an open-source network packet sniffer that captures and analyzes SIP/RTP traffic. It calculates MOS scores for every call and stores them in a database for trending.

Install on your Asterisk server (or a mirror port):

```bash
# On CentOS/VICIbox
yum install voipmonitor
# Or compile from source for the latest version
```

Configuration in `/etc/voipmonitor.conf`:

```ini
[general]
interface = eth0
ringbuffer = 200
packetbuffer_enable = yes
packetbuffer_total_maxheap = 2000

[database]
sqldriver = mysql
mysqlhost = localhost
mysqldb = voipmonitor
mysqluser = voipmonitor
mysqlusername = voipmonitor
mysqlpassword = YOUR_PASSWORD

[sip]
cdr_sipport = 5060
```

VoIPmonitor produces per-call quality reports with:
- MOS score (both directions)
- Jitter statistics
- Packet loss percentage
- Codec information
- Call flow visualization

### Grafana + Asterisk CDR (Free)

If you're already running [Grafana dashboards for VICIdial](/blog/vicidial-grafana-dashboards/), you can add VoIP quality panels. The process:

1. Log RTP quality metrics to a table (via dialplan or AMI events)
2. Create Grafana queries against that table
3. Build panels showing MOS trend over time, worst-quality calls, per-carrier quality comparison

This gives operations managers a visual dashboard showing call quality trends without needing to dig through packet captures.

### Commercial Monitoring Tools

- **Obkio**: Cloud-based, deploys monitoring agents that generate synthetic VoIP traffic and measure MOS continuously. $30-50/agent/month.
- **PRTG**: Network monitoring platform with VoIP quality sensors. Calculates MOS from SNMP data. Starts at ~$1,800 for 500 sensors.
- **SolarWinds VNQM**: VoIP quality monitoring. $2,000+ per installation.

For most VICIdial call centers, VoIPmonitor (free) plus Wireshark (for deep dives) covers 90% of quality monitoring needs.

---

## Interpreting Results: What Bad MOS Scores Mean

You've measured your MOS scores. They're bad. Now what? The root cause maps to specific metrics:

### High Packet Loss (>1%)

**Symptoms**: Choppy audio, missing syllables, robot-voice effect.

**Common causes**:
- Network congestion (check with iperf3/mtr)
- QoS not configured — VoIP traffic competing with bulk data transfers
- Carrier congestion during peak hours
- UDP packets being dropped by firewalls

**Fixes**:
- Enable QoS on your router/switch. DSCP marking EF (Expedited Forwarding, value 46) for RTP traffic
- Ensure your internet circuit has dedicated bandwidth for voice or a separate VLAN
- If carrier-side: ask your carrier about their network utilization during your peak hours
- Check for interface errors: `ethtool -S eth0 | grep -i err`

### High Jitter (>30ms)

**Symptoms**: Inconsistent audio quality — sometimes fine, sometimes garbled. Words arrive out of order or with gaps.

**Common causes**:
- Network congestion causing variable queueing delays
- WiFi instead of wired connections (if agents use WiFi)
- VPN tunnels adding variable encryption overhead
- Undersized jitter buffer

**Fixes**:
- Increase the jitter buffer in Asterisk. In `rtp.conf`: set a larger `rtpstart` to `rtpend` range
- Wire your connections — never run production VoIP over WiFi
- If using a VPN, test without it to isolate the variable
- Apply QoS to prioritize RTP over TCP bulk traffic

### High Latency (>150ms one-way)

**Symptoms**: Conversation feels unnatural — people talk over each other, awkward pauses. Not distorted, just delayed.

**Common causes**:
- Geographic distance (US West Coast to India = 200-300ms minimum)
- Too many network hops
- Carrier routing inefficiency
- SBC or [NAT traversal](/blog/vicidial-remote-agent-setup/) adding processing delay

**Fixes**:
- Choose carriers with PoPs (Points of Presence) near your Asterisk server
- Reduce the number of network devices in the voice path
- If your agents are remote, consider deploying Asterisk servers closer to agent clusters
- Use G.711 instead of G.729 — G.729 encoding adds ~15ms of algorithmic delay

---

## MOS Score Testing Checklist for VICIdial

Before you blame the carrier, run through this checklist:

1. **Test during peak hours** — Quality problems often only appear when your network and the carrier's network are both busy. Testing at 3 AM tells you nothing about 10 AM quality.

2. **Test both directions** — Jitter and loss can be asymmetric. The customer-to-agent path might be fine while the agent-to-customer path is degraded.

3. **Test multiple carriers** — If you have multiple SIP trunks, compare MOS scores across carriers. If one carrier consistently scores lower, you have your answer.

4. **Test at your call volume** — Quality at 5 concurrent calls and quality at 100 concurrent calls are different. Bandwidth contention, Asterisk CPU load, and carrier congestion all increase with scale.

5. **Document your baseline** — Capture MOS scores during a period of known good quality. When problems arise, you can compare against the baseline to quantify the degradation.

---

---

**Related reading:**
- [VoIP MOS Score Testing: The Tools and Methods That Actually Work](https://vicistack.com/blog/voip-mos-score-testing-tools/)
- [VoIP MOS Score: What It Means and How to Fix Bad Call Quality](https://vicistack.com/blog/voip-mos-score-guide/)
- [Migrating from GoAutoDial to VICIdial: Step-by-Step](https://vicistack.com/blog/goautodial-to-vicidial-migration/)
- [Speed to Lead: Why Response Time Is the #1 Factor in Call Center Conversions](https://vicistack.com/blog/speed-to-lead-response-time/)
## When MOS Testing Reveals the Real Problem

We audited a 75-agent VICIdial center that was convinced they needed a carrier change because agents reported terrible audio quality. We deployed VoIPmonitor for a week and collected MOS data for every call.

Results:
- Average MOS on the carrier trunk: 4.28 (excellent)
- Average MOS on agent-side legs: 3.21 (poor)

The carrier was fine. The problem was the agents' network — they were [remote agents](/blog/vicidial-webrtc-setup/) using consumer-grade ISPs with no QoS, WiFi connections with interference, and shared household bandwidth. Three agents were on satellite internet with 600ms latency.

The fix wasn't a carrier change. It was QoS configuration at agent home routers, requiring wired Ethernet for agent workstations, and replacing the satellite-internet agents' connections with terrestrial broadband.

Total cost: about $2,000 in router upgrades versus the $3,000/month carrier switch they were planning.

That's the value of measuring MOS instead of guessing. And it's the kind of analysis we do as part of every [ViciStack engagement](https://vicistack.com/) — diagnose the real problem with real data before spending money on the wrong fix.

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/voip-mos-score-testing-tools).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
