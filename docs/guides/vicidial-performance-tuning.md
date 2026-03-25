# VICIdial Performance Tuning: Server Optimization for 500+ Agents

*You've got the cluster built. Four telephony servers, two web servers, a dedicated database box. VICIdial installs cleanly, agents log in, calls start flowing. Everything looks great at 50 agents. At 150 agents, real-time reports start lagging. At 300 agents, calls drop during peak hours. At 400, the telephony servers hit 100% CPU at 2 PM every day and you're playing whack-a-mole with screen sessions that silently die. The hardware is there. The architecture is right. But nobody tuned the operating system, nobody sized the Asterisk channels, nobody touched the default Apache config, and the kernel is still running with settings designed for a general-purpose web server — not a machine that needs to manage 2,000 concurrent SIP channels while a dozen Perl daemons hammer the database every second.*

*This is the guide that turns good hardware into a 500+ agent VICIdial deployment that actually holds under load.*

VICIdial's performance ceiling is almost never about the software itself. Matt Florell's code handles scale. The bottleneck is the infrastructure underneath: the OS kernel parameters that throttle file descriptors and network connections, the Asterisk configuration that wastes memory on unused codecs, the MySQL instance running with default buffer sizes, the web server that spawns too many children and OOM-kills itself, and the process management that lets critical screen sessions die without anyone noticing.

We've tuned VICIdial deployments from 20-agent single-server setups to 800+ agent clusters spanning a dozen machines. The pattern is always the same: the installation guide gets you running, but it doesn't get you running well at scale. This guide covers every layer of the stack — OS kernel, Asterisk, MySQL/MariaDB, web server, PHP, process management, capacity planning, and monitoring. Every recommendation comes from production deployments, not theory. Every config example has been load-tested.

If you're running under 100 agents on a single server and things feel fine, bookmark this for later. If you're scaling beyond 200 agents or hitting unexplained performance walls, start reading now.

---

## OS Kernel Tuning

The Linux kernel ships with defaults tuned for a general-purpose server. VICIdial is not a general-purpose workload. It opens thousands of file descriptors (one per SIP channel, one per recording file, one per Perl process, one per database connection), maintains thousands of concurrent network sockets (SIP connections, HTTP agent sessions, database connections, AMI connections), and moves a massive volume of small packets through the network stack. Default kernel parameters throttle all of this.

### sysctl Settings

Create `/etc/sysctl.d/99-vicidial.conf` and apply these settings. Every parameter here solves a specific problem that manifests at scale:

```ini
# ============================================================
# FILE DESCRIPTOR AND INOTIFY LIMITS
# ============================================================
# VICIdial opens a LOT of files. Each SIP channel = 1-2 fds.
# Each recording = 1 fd. Each Perl daemon = dozens of fds.
# Each MySQL connection = 1 fd. Default 1024 per process is
# laughably insufficient for a 500-agent telephony server.
fs.file-max = 524288
fs.inotify.max_user_watches = 131072

# ============================================================
# NETWORK STACK: CONNECTION TRACKING AND BUFFERS
# ============================================================
# Increase the connection tracking table for SIP/RTP traffic.
# Default 65536 gets exhausted quickly with high call volumes.
net.netfilter.nf_conntrack_max = 262144
net.nf_conntrack_max = 262144

# Increase socket backlog — prevents SIP connection drops
# during traffic spikes when Asterisk can't accept() fast enough.
net.core.somaxconn = 8192
net.core.netdev_max_backlog = 16384

# Receive and send buffer sizes for UDP (SIP/RTP).
# Defaults are 128K-256K. At 500 agents with G.711 codec,
# RTP alone pushes ~400 Mbps of small UDP packets.
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.core.rmem_default = 1048576
net.core.wmem_default = 1048576

# ============================================================
# TCP TUNING (agent web sessions, database connections, AMI)
# ============================================================
# Increase TCP buffer auto-tuning range
net.ipv4.tcp_rmem = 4096 1048576 16777216
net.ipv4.tcp_wmem = 4096 1048576 16777216

# Allow more TIME_WAIT sockets. VICIdial's Perl daemons
# create and destroy TCP connections to MySQL constantly.
# Default 8192 causes connection failures at scale.
net.ipv4.tcp_max_tw_buckets = 65536

# Reuse TIME_WAIT sockets for new connections.
# Without this, MySQL connection exhaustion is common
# in clusters where multiple servers hit the database.
net.ipv4.tcp_tw_reuse = 1

# Reduce TIME_WAIT duration from 60s to 30s
net.ipv4.tcp_fin_timeout = 30

# Faster keepalive detection for dead MySQL connections
net.ipv4.tcp_keepalive_time = 300
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 5

# SYN backlog — prevents dropped connections during
# agent login storms (shift change, everyone logs in at once)
net.ipv4.tcp_max_syn_backlog = 8192

# ============================================================
# MEMORY MANAGEMENT
# ============================================================
# Prefer keeping application data in memory over filesystem cache.
# VICIdial's real-time processes need low-latency memory access
# more than they need disk caching.
vm.swappiness = 10

# Don't let the system swap aggressively — swap kills VICIdial
# real-time performance. A swapping telephony server = dropped calls.
vm.dirty_ratio = 20
vm.dirty_background_ratio = 5

# ============================================================
# SHARED MEMORY (used by Asterisk and MySQL)
# ============================================================
kernel.shmmax = 4294967296
kernel.shmall = 4294967296
```

Apply without reboot:

```bash
sysctl --system
```

**The `vm.swappiness` story:** We've seen more VICIdial outages caused by swap than by actual hardware failure. Here's what happens: the database server is running well, using 28 GB of 32 GB RAM. The OS decides to start swapping to make room for filesystem cache. MyISAM's data file reads now compete with swap I/O. MySQL slows down. Perl daemons queue up waiting for database responses. Memory usage spikes because of queued processes. More swapping. Within minutes, the system is in a swap death spiral, agents are frozen, and calls are dropping. Setting `vm.swappiness = 10` tells the kernel to strongly prefer dropping filesystem cache over swapping application memory. On a dedicated VICIdial server, you can set this to `1`. On a shared database+telephony server, `10` is the safe choice.

### ulimits Configuration

The kernel-level `fs.file-max` sets the system-wide ceiling, but each process also has its own limits. VICIdial's Asterisk and MySQL processes need high per-process limits.

Edit `/etc/security/limits.conf`:

```
# VICIdial process limits
*               soft    nofile          65536
*               hard    nofile          131072
*               soft    nproc           65536
*               hard    nproc           65536
root            soft    nofile          65536
root            hard    nofile          131072
mysql           soft    nofile          65536
mysql           hard    nofile          131072
asterisk        soft    nofile          65536
asterisk        hard    nofile          131072
```

For systemd-managed services, the limits.conf values may be ignored. Override at the service level:

```bash
# For Asterisk
mkdir -p /etc/systemd/system/asterisk.service.d/
cat > /etc/systemd/system/asterisk.service.d/limits.conf << EOF
[Service]
LimitNOFILE=131072
LimitNPROC=65536
LimitMEMLOCK=infinity
EOF

# For MariaDB/MySQL
mkdir -p /etc/systemd/system/mariadb.service.d/
cat > /etc/systemd/system/mariadb.service.d/limits.conf << EOF
[Service]
LimitNOFILE=131072
LimitNPROC=65536
EOF

systemctl daemon-reload
```

**How to verify:** After restarting the services, check the actual limits:

```bash
# Find Asterisk's PID and check its limits
cat /proc/$(pidof asterisk)/limits | grep "Max open files"
# Should show: Max open files  131072  131072  files

# Same for MySQL
cat /proc/$(pidof mysqld)/limits | grep "Max open files"
```

If you see `1024` or `4096` here, the limits aren't taking effect. Check that PAM is loading `pam_limits.so` and that systemd overrides are in place.

### CPU Governor and Power Management

Modern servers ship with power-saving CPU governors enabled by default. On a telephony server, that means the CPU scales down during "quiet" periods and takes milliseconds to scale back up when a burst of calls arrives. Those milliseconds cause jitter on active calls.

```bash
# Check current governor
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor

# Set to performance (maximum clock speed always)
for cpu in /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor; do
    echo performance > $cpu
done

# Make persistent via tuned (RHEL/CentOS)
tuned-adm profile throughput-performance
```

On bare-metal servers, also disable C-states and P-states in the BIOS. Intel SpeedStep and AMD Cool'n'Quiet are great for power bills, terrible for real-time telephony.

---

## Asterisk Channel Optimization

Asterisk is the telephony engine. Every call — inbound and outbound — flows through Asterisk. At 500 agents with a 3:1 [dial ratio](/settings/auto-dial-level/), you're pushing 1,500 concurrent channels across your [cluster](/glossary/vicidial-cluster/). Each channel consumes CPU for [codec](/glossary/codec/) processing, memory for channel state, and file descriptors for SIP sockets and recording files. Tuning Asterisk for this load means eliminating waste and allocating resources where they matter.

### asterisk.conf and modules.conf Tuning

Module loading is controlled in `/etc/asterisk/modules.conf`:

```ini
; /etc/asterisk/modules.conf
[modules]
autoload=yes

; Disable unused channel drivers to save memory and startup time.
; VICIdial only uses SIP (chan_sip or chan_pjsip) and Local channels.
noload => chan_iax2.so
noload => chan_skinny.so
noload => chan_unistim.so
noload => chan_mgcp.so
noload => chan_oss.so
noload => chan_alsa.so
noload => chan_phone.so

; Disable unused applications
noload => app_voicemail.so
noload => app_queue.so
noload => app_followme.so
noload => app_directory.so

; Disable unused codecs you're not using
; (keep the ones your carriers require)
noload => codec_speex.so
noload => codec_ilbc.so
noload => codec_g726.so
noload => codec_gsm.so
noload => codec_lpc10.so
```

And in `/etc/asterisk/asterisk.conf`:

```ini
[options]
; Reduce console output overhead
verbose = 1
debug = 0

; Cache settings
cache_record_files = yes
record_cache_dir = /tmp

; Max calls per Asterisk instance
maxcalls = 2048

; Max simultaneous file descriptors Asterisk will use
maxfiles = 131072
```

**The `noload` approach:** Each loaded Asterisk module consumes memory and contributes to startup time and runtime overhead. The default Asterisk installation loads dozens of modules that VICIdial never uses — voicemail, queues, Skinny protocol (for Cisco phones), IAX2, MGCP, and others. Unloading them isn't just housekeeping. On a 500-agent telephony server, the memory freed by not loading unused modules is significant enough to allow 50-100 more concurrent channels.

### SIP Channel Configuration (chan_sip)

For high-volume outbound dialing, the SIP stack needs specific tuning. In `sip.conf`:

```ini
[general]
; Reduce SIP transaction timers for faster call teardown.
; Default Timer B is 32 seconds — way too long for an outbound
; dialer that needs to detect failed calls quickly.
timer1 = 500
timerb = 16000

; Use TCP for SIP signaling if your carrier supports it.
; UDP fragments at high packet rates and drops packets
; more readily than TCP under load.
; tcpenable = yes
; transport = tcp

; Reduce qualify frequency — VICIdial doesn't need per-second
; peer monitoring. Every 60 seconds is sufficient.
qualifyfreq = 60

; Max SIP registrations / concurrent dialogs
maxexpiry = 3600
defaultexpiry = 360

; RTP settings — moved to rtp.conf in newer Asterisk
; but some versions still read from sip.conf
rtptimeout = 60
rtpholdtimeout = 300

; Disable DNS SRV lookups for trunks with known IPs
srvlookup = no

; Compact SIP headers — reduces packet size,
; matters at 1000+ concurrent calls
compactheaders = yes
```

### RTP Port Range Optimization

Each active call uses one RTP port for audio. At 1,500 concurrent channels, you need at least 1,500 available RTP ports. The default range (10000-20000) provides 10,000 ports, which is ample — but the range should be tuned to match your firewall rules and avoid conflicts.

In `rtp.conf`:

```ini
[general]
rtpstart = 10000
rtpend = 30000
; 20,000 ports — handles up to 20,000 concurrent calls
; (you'll run out of CPU long before you run out of ports)

; Enable strict RTP — drops RTP from unexpected sources.
; Prevents RTP bleed and saves CPU on invalid packets.
strictrtp = yes

; RTP checksums — disable for performance on trusted networks
; rtpchecksums = no
```

**Firewall alignment:** Whatever range you set here must be open in iptables/firewalld for UDP traffic. Mismatched port ranges between rtp.conf and your firewall are a common cause of one-way audio that only appears under load (when the RTP ports fall outside the firewall-allowed range).

### Asterisk Memory and Threading

Asterisk's memory footprint per channel depends primarily on the codec used:

| Codec | Bandwidth per Call | Memory per Channel | CPU per Channel |
|-------|-------------------|-------------------|-----------------|
| G.711 (ulaw/alaw) | 87.2 kbps | ~120 KB | Low |
| G.729 | 31.2 kbps | ~120 KB + transcoding | **High** |
| GSM | 13.2 kbps | ~120 KB + transcoding | Medium |
| Opus | Variable | ~150 KB | Medium-High |

**G.711 is the correct choice for VICIdial outbound dialing.** It uses more bandwidth but requires zero transcoding — Asterisk passes the audio straight through. G.729 halves the bandwidth but requires software transcoding on every channel, which consumes significant CPU. At 500 concurrent calls, G.729 transcoding can consume 4-8 CPU cores that would otherwise handle more channels. Unless bandwidth is genuinely constrained (which it isn't on any modern data center connection), use G.711.

Calculate per-server capacity:

```
G.711 channels per core ≈ 150-200 (passthrough, no transcoding)
G.729 channels per core ≈ 15-25 (with transcoding)
```

A 16-core telephony server running G.711 can handle approximately 1,500-2,000 concurrent channels — well beyond what you'd allocate to a single server in a properly designed [VICIdial cluster](/blog/vicidial-cluster-guide/). With G.729, that same server tops out at 200-400 channels before CPU saturation.

---

## MySQL/MariaDB Performance Tuning

We covered MySQL optimization in depth in the [VICIdial MySQL Optimization guide](/blog/vicidial-mysql-optimization/), including buffer pool sizing, slow query analysis, MEMORY table conversions, and archival strategies. Here, we'll focus specifically on the performance tuning aspects that interact with OS-level and cluster-level optimization.

### Buffer Pool Sizing for High-Agent Counts

The relationship between agent count and database memory requirements is non-linear. Each agent generates database activity through multiple pathways simultaneously — screen refreshes (SELECT against `vicidial_live_agents`), call events (INSERT/UPDATE on `vicidial_log`), [hopper](/glossary/hopper/) queries, recording triggers, and status updates. At 500 agents, the database isn't doing 10x the work of 50 agents — it's doing 25-30x, because the interaction between concurrent queries creates lock contention that multiplies with agent count.

**Recommended MySQL memory allocation by deployment size:**

| Agents | Dedicated DB RAM | key_buffer_size | max_heap_table_size | sort_buffer_size | max_connections |
|--------|-----------------|-----------------|---------------------|------------------|-----------------|
| 50 | 16 GB | 512M | 64M | 4M | 2000 |
| 100 | 32 GB | 1024M | 128M | 8M | 3000 |
| 200 | 32 GB | 2048M | 128M | 8M | 4000 |
| 300 | 64 GB | 3072M | 256M | 16M | 5000 |
| 500+ | 64-128 GB | 4096M | 256M | 16M | 6000 |

**Why `max_connections` scales with agents:** In a VICIdial cluster, every telephony server, web server, and cron process maintains multiple persistent connections to the database. A single telephony server running the standard set of VICIdial daemons holds 40-60 active database connections during peak operation. A single web server handling 100 agent sessions holds another 100-200 connections (each agent's browser polls the server, which polls the database). At 500 agents across 4 telephony servers and 2 web servers, you're looking at 600-1,000 active connections routinely, with spikes to 2,000+ during shift changes.

### Query Cache: Disable It

This deserves its own subsection because it's counterintuitive. MySQL's query cache sounds like it should help — it caches the results of SELECT queries and returns them instantly if the same query runs again. For read-heavy applications with infrequent writes, it's a significant performance boost.

VICIdial is not that application. VICIdial writes to its core tables multiple times per second. Every write to a table invalidates every cached query result for that table. The query cache becomes a high-overhead bookkeeping system that caches results only to immediately invalidate them. The cache lock itself becomes a contention point.

```ini
[mysqld]
query_cache_size = 0
query_cache_type = 0
```

This is not optional. Every VICIdial MySQL optimization guide from the community arrives at the same conclusion. The query cache hurts VICIdial performance. Turn it off.

### MEMORY Tables for Real-Time Performance

The tables that VICIdial queries most frequently — `vicidial_live_agents`, `vicidial_auto_calls`, `vicidial_hopper`, `vicidial_live_inbound_agents`, `vicidial_live_sip_channels` — should run on the MEMORY storage engine. MEMORY tables live entirely in RAM, eliminate disk I/O for the hottest queries, and reduce lock duration from milliseconds to microseconds.

The full conversion process is detailed in our [MySQL optimization guide](/blog/vicidial-mysql-optimization/). The critical sizing parameter:

```ini
[mysqld]
# MEMORY table size limit.
# Must accommodate ALL MEMORY tables simultaneously.
# At 500 agents: vicidial_live_agents ≈ 2-4 MB,
# vicidial_auto_calls ≈ 1-2 MB, vicidial_hopper ≈ 1-5 MB,
# others ≈ 1-2 MB. Total: ~10-15 MB actual use.
# Set much higher for safety margin and tmp table operations.
max_heap_table_size = 256M
tmp_table_size = 256M
```

When a MEMORY table hits the `max_heap_table_size` ceiling, INSERTs fail. In VICIdial, this manifests as agents unable to log in (can't INSERT into `vicidial_live_agents`) or calls not dialing (can't INSERT into `vicidial_auto_calls`). The error log will show "table is full" messages. Set the limit high enough that you never hit it.

### Concurrent Insert Optimization

MyISAM supports a feature called concurrent inserts — new rows can be inserted at the end of a table while SELECT queries read from it, without table-level locking. This is a significant performance feature for VICIdial's mixed read/write workload, but it requires a specific setting:

```ini
[mysqld]
concurrent_insert = 2
```

The values: `0` = disabled, `1` = enabled only for tables with no deleted rows (gaps), `2` = enabled always, inserting at end of table regardless of gaps. VICIdial tables frequently have deleted rows (archival, hopper drains), so `concurrent_insert = 2` is necessary to get the benefit consistently.

> **Running 200+ Agents and the Database Is Your Bottleneck?**
> ViciStack deployments include dedicated database servers with pre-tuned MySQL, MEMORY table conversions, and weekly performance reviews. We've solved this problem hundreds of times. [Get a Free Performance Audit →](/free-audit/)

---

## Apache vs Nginx for VICIdial Web

VICIdial's agent interface runs through Apache/Nginx serving PHP scripts. Every logged-in agent has a browser session that polls the server every second for real-time data (agent status, call information, script content, callback alerts). At 500 agents, that's 500 HTTP requests per second just from agent polling — before managers load reports, before supervisors check real-time dashboards, before any API calls.

### Apache: The Default (and Its Problems)

VICIdial's install scripts configure Apache with `mod_php` (prefork MPM). This is the simplest setup and works fine under 100 agents. The architecture: each incoming HTTP request gets its own Apache child process, and each child process has a PHP interpreter embedded in it. The problem at scale is memory.

A single Apache+mod_php process with VICIdial loaded uses 30-60 MB of RAM. At 500 concurrent connections (one per agent, plus overhead), you need 500 Apache children. That's 15-30 GB of RAM just for the web server — on top of whatever else the server is running.

**Default Apache configuration (what VICIdial installs):**

```apache
# /etc/httpd/conf/httpd.conf — prefork MPM defaults
<IfModule prefork.c>
    StartServers         8
    MinSpareServers      5
    MaxSpareServers     20
    ServerLimit        256
    MaxClients         256
    MaxRequestsPerChild  4000
</IfModule>
```

`MaxClients 256` means Apache can handle 256 concurrent connections. Above that, requests queue. At 300 agents all polling simultaneously, you exceed the limit and agents see timeouts.

**Tuned Apache configuration for 500 agents:**

```apache
<IfModule prefork.c>
    StartServers         50
    MinSpareServers      25
    MaxSpareServers      75
    ServerLimit         600
    MaxClients          600
    MaxRequestsPerChild  10000
</IfModule>

# Disable unused modules
# Every loaded module adds memory to each child process
LoadModule status_module modules/mod_status.so
# Comment out or remove:
# LoadModule dav_module modules/mod_dav.so
# LoadModule dav_fs_module modules/mod_dav_fs.so
# LoadModule autoindex_module modules/mod_autoindex.so
# LoadModule negotiation_module modules/mod_negotiation.so
# LoadModule userdir_module modules/mod_userdir.so

# Connection tuning
KeepAlive On
MaxKeepAliveRequests 100
KeepAliveTimeout 5

# Timeout for stale connections
Timeout 60
```

**Memory budget:** 600 children × 50 MB average = 30 GB. You need a web server (or web servers) with enough RAM to support this. In a [VICIdial cluster](/blog/vicidial-cluster-guide/), splitting agents across two web servers (300 agents each) cuts the per-server requirement in half.

### Nginx + PHP-FPM: The Better Architecture

Nginx with PHP-FPM separates the web server from PHP processing. Nginx handles HTTP connections using an event-driven model (thousands of connections with minimal memory), and forwards PHP requests to a pool of PHP-FPM worker processes. This architecture uses dramatically less memory because:

1. Nginx connection handling: ~2-10 KB per connection (vs. 30-60 MB per Apache child)
2. PHP-FPM workers: only as many as you need for concurrent PHP execution (not one per connection)
3. Static files (CSS, JS, images) served directly by Nginx without touching PHP at all

**Nginx configuration for VICIdial:**

```nginx
# /etc/nginx/conf.d/vicidial.conf

server {
    listen 80;
    server_name dialer.example.com;
    root /var/www/html;
    index index.php index.html;

    # Static file caching — agent interface assets don't change
    location ~* \.(js|css|png|jpg|gif|ico|woff|woff2)$ {
        expires 7d;
        add_header Cache-Control "public, immutable";
        access_log off;
    }

    # PHP processing via FPM
    location ~ \.php$ {
        fastcgi_pass unix:/run/php-fpm/www.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;

        # VICIdial agent polling generates thousands of
        # requests/sec. Keep connections to FPM alive.
        fastcgi_keep_conn on;

        # Timeout tuning — some VICIdial report scripts
        # take 30+ seconds on large datasets
        fastcgi_read_timeout 120;
        fastcgi_send_timeout 60;
    }

    # Connection tuning
    keepalive_timeout 65;
    keepalive_requests 1000;

    # Buffer tuning — VICIdial responses are small
    # (HTML fragments, JSON data for agent screens)
    client_body_buffer_size 16k;
    client_max_body_size 50m;  # For list uploads

    # Gzip compression — reduces bandwidth for agent polling
    gzip on;
    gzip_types text/html text/css application/javascript text/xml;
    gzip_min_length 256;
}
```

**Nginx worker configuration (`/etc/nginx/nginx.conf`):**

```nginx
worker_processes auto;  # One worker per CPU core
worker_rlimit_nofile 65536;

events {
    worker_connections 4096;
    use epoll;
    multi_accept on;
}

http {
    # Connection pooling to PHP-FPM
    upstream php-fpm {
        server unix:/run/php-fpm/www.sock;
        keepalive 64;
    }

    # Reduce logging overhead for agent polling requests
    # (every agent generates 1 request/second — that's a LOT of log lines)
    map $request_uri $loggable {
        ~*vicidial/vdc_db_query\.php  0;
        default                        1;
    }
    access_log /var/log/nginx/access.log combined if=$loggable;
}
```

### PHP-FPM Tuning

Whether you use Apache+mod_php or Nginx+PHP-FPM, the PHP configuration matters. With PHP-FPM, you have direct control over the process pool:

```ini
; /etc/php-fpm.d/www.conf

[www]
user = apache
group = apache

; Use a Unix socket — faster than TCP for same-server communication
listen = /run/php-fpm/www.sock

; Static process management — predictable resource usage.
; Dynamic is fine for web apps. VICIdial's constant polling
; means you always need all your workers, so pre-spawn them.
pm = static
pm.max_children = 128

; At 500 agents polling every second, you need enough FPM
; workers to handle the concurrent requests. Each agent poll
; takes ~5-20ms of PHP execution time.
; Throughput: 128 workers × (1000ms / 15ms avg) = ~8,500 req/sec
; Required: 500 agents × 1 req/sec = 500 req/sec (minimum)
; The 128 workers provide massive headroom for report queries
; and burst traffic.

; Recycle workers after 10,000 requests to prevent memory leaks
pm.max_requests = 10000

; PHP settings for VICIdial
php_admin_value[memory_limit] = 256M
php_admin_value[max_execution_time] = 120
php_admin_value[max_input_time] = 120
php_admin_value[upload_max_filesize] = 50M
php_admin_value[post_max_size] = 50M

; OPcache — cache compiled PHP bytecode.
; VICIdial serves the same PHP files thousands of times per second.
; OPcache eliminates the parsing/compilation overhead.
php_admin_value[opcache.enable] = 1
php_admin_value[opcache.memory_consumption] = 128
php_admin_value[opcache.max_accelerated_files] = 10000
php_admin_value[opcache.validate_timestamps] = 0
; Set validate_timestamps=0 in production. VICIdial PHP files
; don't change between SVN updates, and disabling timestamp
; checks saves a stat() call per request.
```

**The Apache vs Nginx verdict:** For new deployments at 200+ agents, Nginx + PHP-FPM is the right choice. It uses 60-70% less memory, handles connection surges more gracefully, and the static file serving is dramatically faster. For existing Apache deployments that work, switching is a weekend project — and the performance gain scales with agent count. At 100 agents, the difference is marginal. At 500, it's the difference between needing one web server and needing two.

**One caveat:** VICIdial's `.htaccess` files and some `mod_rewrite` rules won't work in Nginx. You need to convert them to Nginx `location` and `rewrite` directives. Test thoroughly in staging before cutting over.

---

## Process Management

VICIdial runs its core processes inside GNU `screen` sessions. The main keepalive script (`AST_VDauto_dial.pl`, `AST_manager_listen.pl`, `AST_update.pl`, and dozens more) are launched inside screen and monitored by cron-driven keepalive scripts. When a process dies, the keepalive script is supposed to restart it. In practice, this system has gaps that become critical at scale.

### Understanding VICIdial's Screen Architecture

When VICIdial starts, it launches a screen session (typically named `ASTscreen`) containing multiple windows, each running a critical Perl daemon:

| Process | Purpose | Impact if Dead |
|---------|---------|----------------|
| `AST_manager_listen.pl` | Listens to Asterisk AMI events | **Complete dialer failure** — no calls placed or tracked |
| `AST_VDauto_dial.pl` | Auto-dial engine | Outbound dialing stops |
| `AST_VDremote_agents.pl` | Remote agent management | Remote/PJSIP agents stop receiving calls |
| `AST_update.pl` | Updates real-time agent/call data | Real-time reports freeze, agent screens stale |
| `AST_VDadapt.pl` | Adaptive dial-level calculations | Dial ratios stop adjusting, over/under-dialing |
| `AST_VDhopper.pl` | Fills the [hopper](/glossary/hopper/) | Hopper drains, dialing stops when empty |
| `AST_VDauto_dial_FILL.pl` | Supplemental dialing | Reduced throughput |
| `AST_conf_update.pl` | Conference bridge management | Agent-to-call bridging fails |

**The failure mode nobody talks about:** Screen itself can die. If the `ASTscreen` session is killed (OOM killer, manual `kill`, system resource exhaustion), every process inside it dies simultaneously. The keepalive scripts run via cron, typically every minute. That means up to 60 seconds of complete dialer downtime before the keepalives detect the failure and restart the screen session. At 500 agents and a 3:1 [dial ratio](/settings/auto-dial-level/), 60 seconds of downtime means ~1,500 calls that don't get placed and ~500 agents sitting idle.

### Hardening the Keepalive System

VICIdial's standard keepalive cron entries look like this:

```cron
### recording hepatic
* * * * * /usr/share/astguiclient/AST_CRON_audio_1_move_mix.pl
* * * * * /usr/share/astguiclient/AST_CRON_audio_1_move_VDonly.pl
### keepalive processes
* * * * * /usr/share/astguiclient/ADMIN_keepalive_ALL.pl --cu3way
```

`ADMIN_keepalive_ALL.pl` is the master keepalive. It checks whether each expected process is running and restarts anything that's dead. The `--cu3way` flag includes the three-way call process. This runs every minute via cron.

**Improving keepalive response time:**

For 500+ agent deployments, one-minute keepalive intervals are too slow. Add a secondary check at 30-second intervals:

```cron
# Primary keepalive — every minute
* * * * * /usr/share/astguiclient/ADMIN_keepalive_ALL.pl --cu3way
# Secondary keepalive — 30 seconds offset
* * * * * sleep 30 && /usr/share/astguiclient/ADMIN_keepalive_ALL.pl --cu3way
```

This ensures no process stays dead for more than 30 seconds. The `sleep 30 &&` trick runs the keepalive at the :30 second mark of each minute, interleaving with the standard :00 execution.

### OOM Killer Protection

The Linux OOM (Out of Memory) killer terminates processes when the system runs out of memory. It picks the process using the most memory — which on a VICIdial server is often MySQL or Asterisk. Both are catastrophic to lose.

Protect critical processes:

```bash
# Find PIDs and set OOM score adjustment
# -1000 = never kill, -999 to -1 = less likely to kill
echo -1000 > /proc/$(pidof asterisk)/oom_score_adj
echo -1000 > /proc/$(pidof mysqld)/oom_score_adj
```

Make this persistent by adding it to your startup scripts or systemd unit overrides:

```bash
# /etc/systemd/system/asterisk.service.d/oom.conf
[Service]
OOMScoreAdjust=-1000

# /etc/systemd/system/mariadb.service.d/oom.conf
[Service]
OOMScoreAdjust=-1000
```

**The real solution:** Don't run out of memory in the first place. If the OOM killer is firing, you're either under-provisioned or leaking memory somewhere. But protecting Asterisk and MySQL buys you time to diagnose while the system stays up.

### Process Monitoring with Custom Checks

Beyond VICIdial's built-in keepalives, add process count verification:

```bash
#!/bin/bash
# /usr/local/bin/vicidial-process-check.sh
# Run via cron every 5 minutes

EXPECTED_PROCESSES=(
    "AST_manager_listen.pl"
    "AST_VDauto_dial.pl"
    "AST_update.pl"
    "AST_VDadapt.pl"
    "AST_VDhopper.pl"
    "AST_conf_update.pl"
)

for proc in "${EXPECTED_PROCESSES[@]}"; do
    COUNT=$(pgrep -fc "$proc")
    if [ "$COUNT" -eq 0 ]; then
        echo "CRITICAL: $proc is not running" | \
            mail -s "VICIdial Process Alert: $proc DOWN" ops@example.com
    fi
done

# Check screen session exists
if ! screen -list | grep -q "ASTscreen"; then
    echo "CRITICAL: ASTscreen session is missing" | \
        mail -s "VICIdial Alert: Screen Session DOWN" ops@example.com
fi
```

---

## Capacity Planning: Agents to Server Resources

This is the section people skip during planning and regret during production. Capacity planning for VICIdial is not guesswork — there are concrete formulas based on call volume, agent count, recording requirements, and the architectural role of each server.

### The Core Formula

Every VICIdial cluster has three resource-consumption dimensions:

1. **Telephony (Asterisk):** CPU-bound. Scales with concurrent calls, not agents. A paused agent uses zero telephony resources.
2. **Web (Apache/Nginx + PHP):** Memory-bound. Scales linearly with logged-in agents regardless of call activity.
3. **Database (MySQL):** I/O-bound at scale. Scales non-linearly with agents due to lock contention.

**Telephony server sizing:**

```
Concurrent calls = Agents × Dial ratio × (Talk time / (Talk time + Wrap time + Pause time))
```

Example for 500 agents, outbound:
- [Auto-dial level](/settings/auto-dial-level/): 3.0
- Average talk time: 90 seconds
- Average wrap time: 15 seconds
- Average pause time: 5 seconds
- Agent utilization: 90 / (90 + 15 + 5) = 82%

```
Concurrent calls = 500 × 3.0 × 0.82 = 1,230 concurrent calls
```

At G.711 with no transcoding: **1,230 calls ÷ 200 calls/core ≈ 7 cores** of raw telephony processing. Add 30% for Asterisk overhead (AMI, recording initiation, channel management): **~9 cores.**

Spread across servers: 3 telephony servers with 8 cores each provides 24 cores of capacity — plenty of headroom for spikes, recording processing, and the OS overhead.

**Web server sizing:**

```
Required memory = Agents × Memory per agent session
```

With Apache+mod_php: `500 × 50 MB = 25 GB`
With Nginx+PHP-FPM: `500 × 5 MB (Nginx) + 128 workers × 60 MB (PHP-FPM) = 10 GB`

Web server CPU is rarely the bottleneck — the PHP scripts execute in milliseconds. Memory is the constraint.

**Database server sizing:**

```
Minimum RAM = key_buffer_size + max_heap_table_size + (max_connections × per_thread_buffers) + OS reserve
```

For 500 agents:
```
4096M + 256M + (6000 × 0.5M average) + 8192M OS = ~15.5 GB minimum
Recommended: 64 GB (to maximize OS filesystem cache for MyISAM .MYD files)
```

Storage: NVMe SSDs, not SATA SSDs, not spinning disks. At 500 agents, the database performs thousands of random I/O operations per second. NVMe latency (50-100 μs) vs SATA SSD (200-500 μs) vs spinning disk (5,000-10,000 μs) directly translates to query latency. The investment in NVMe storage pays for itself in reduced server count because each server can handle more load before I/O-bound contention degrades performance.

### Server Role Allocation at Scale

| Deployment Size | Telephony Servers | Web Servers | Database Server | Total Servers |
|----------------|-------------------|-------------|-----------------|---------------|
| 50 agents | 1 (combined) | 1 (combined) | 1 (combined) | 1 |
| 100 agents | 1-2 | 1 | 1 (dedicated) | 3-4 |
| 200 agents | 2-3 | 1-2 | 1 (dedicated) | 4-6 |
| 300 agents | 3-4 | 2 | 1 (dedicated, 64 GB) | 6-7 |
| 500 agents | 4-6 | 2-3 | 1 (dedicated, 64-128 GB) | 7-10 |
| 800+ agents | 6-8 | 3-4 | 1-2 (primary + replica) | 10-14 |

**The database stays single.** VICIdial cannot cluster its database. You can add a replica for reporting (see [MySQL optimization guide](/blog/vicidial-mysql-optimization/)), but the primary database is a single instance. This is VICIdial's fundamental scaling constraint, and it's why database server hardware should be the most powerful machine in the cluster — faster CPUs, more RAM, NVMe storage, and battery-backed RAID.

### Bandwidth Planning

VICIdial's bandwidth consumption is dominated by RTP audio. SIP signaling, database queries, and web traffic are negligible by comparison.

```
Bandwidth per call (G.711) = 87.2 kbps × 2 (in + out) = 174.4 kbps
Bandwidth at 1,230 concurrent calls = 1,230 × 174.4 kbps = 214 Mbps
```

Add recording storage bandwidth (calls being recorded to disk): approximately 87 kbps per recorded call. If 100% of calls are recorded:

```
Recording bandwidth = 1,230 × 87 kbps = 107 Mbps additional disk I/O
Total: ~320 Mbps sustained
```

A single gigabit ethernet port handles this. But if your telephony servers have both management traffic and RTP on the same NIC, consider dedicated NICs for RTP traffic at this scale. Network congestion on a shared NIC manifests as jitter and packet loss — audible quality problems on active calls.

### Recording Storage Planning

Call recordings are the other storage dimension that catches people off guard:

```
Storage per recording minute (G.711 WAV) = ~960 KB/min
Storage per recording minute (compressed MP3/OGG) = ~120-240 KB/min
```

At 500 agents, 8 hours/day, 82% utilization, average 90-second calls:

```
Calls per day = 500 × (8 × 3600) / (90 + 15 + 5) × 0.82 ≈ 120,000 calls/day
Recording minutes per day = 120,000 × 1.5 min = 180,000 minutes
Storage per day (WAV) = 180,000 × 0.96 MB = ~173 GB/day
Storage per day (MP3) = 180,000 × 0.2 MB = ~36 GB/day
Storage per month (MP3) = ~1 TB/month
```

Plan your recording storage accordingly. Many operations keep 90 days of recordings online, which means 3 TB of fast storage at 500 agents. Archive older recordings to cheaper storage (NFS, S3, tape).

> **Let Us Size Your Cluster.**
> ViciStack provides capacity planning as part of every deployment — we calculate server specs, bandwidth requirements, and storage needs based on your actual campaign parameters. No undersizing, no overspending. [Get Your Free Audit →](/free-audit/)

---

## Monitoring and Alerting

You cannot tune what you cannot measure. A 500-agent VICIdial deployment without monitoring is a ticking clock. Problems at this scale don't announce themselves gradually — they hit a tipping point and cascade within minutes.

### What to Monitor

**System-level metrics (every server):**

| Metric | Warning Threshold | Critical Threshold | Tool |
|--------|-------------------|-------------------|------|
| CPU usage | > 70% sustained | > 90% sustained | Prometheus node_exporter, Nagios NRPE |
| Memory usage | > 80% | > 90% | Same |
| Swap usage | > 0 MB | > 100 MB | Same |
| Disk usage | > 80% | > 90% | Same |
| Disk I/O wait | > 15% | > 30% | iostat, sar |
| Network errors | > 0 | > 10/min | ethtool, /proc/net/dev |
| Load average | > CPU cores × 1.5 | > CPU cores × 3 | Same |

**VICIdial-specific metrics:**

| Metric | How to Check | Warning Sign |
|--------|-------------|--------------|
| Hopper level | `SELECT COUNT(*) FROM vicidial_hopper;` | Dropping below [hopper level setting](/settings/hopper-level/) |
| Active calls vs agents | `SELECT COUNT(*) FROM vicidial_auto_calls;` | Ratio deviating from expected dial level |
| Agent login count | `SELECT COUNT(*) FROM vicidial_live_agents WHERE status NOT IN ('PAUSED');` | Sudden drops |
| MySQL connections | `SHOW STATUS LIKE 'Threads_connected';` | > 80% of max_connections |
| MySQL slow queries | `SHOW STATUS LIKE 'Slow_queries';` | Increasing faster than 10/hour |
| Screen session health | `screen -list \| grep ASTscreen` | Missing session |
| Asterisk channels | `asterisk -rx 'core show channels count'` | Exceeding expected concurrent calls |
| SIP trunk registration | `asterisk -rx 'sip show registry'` | Unregistered trunks |

### Prometheus + Grafana Stack

For comprehensive monitoring, deploy Prometheus with Grafana dashboards. See our [Grafana dashboards guide](/blog/vicidial-grafana-dashboards/) for the complete setup. The short version:

1. **node_exporter** on every server — system metrics
2. **mysqld_exporter** on the database server — MySQL metrics
3. **Custom VICIdial exporter** — a script that queries VICIdial tables and exposes metrics in Prometheus format

A minimal custom exporter:

```bash
#!/bin/bash
# /usr/local/bin/vicidial-metrics.sh
# Output Prometheus-format metrics for VICIdial
# Serve via node_exporter textfile collector

MYSQL_CMD="mysql -u cron -pYOUR_PASSWORD asterisk -N -B"
OUTPUT="/var/lib/prometheus/node-exporter/vicidial.prom"

AGENTS=$($MYSQL_CMD -e "SELECT COUNT(*) FROM vicidial_live_agents;")
ACTIVE_CALLS=$($MYSQL_CMD -e "SELECT COUNT(*) FROM vicidial_auto_calls;")
HOPPER=$($MYSQL_CMD -e "SELECT COUNT(*) FROM vicidial_hopper;")
PAUSED=$($MYSQL_CMD -e "SELECT COUNT(*) FROM vicidial_live_agents WHERE status='PAUSED';")

cat > "$OUTPUT" <<EOF
# HELP vicidial_agents_total Total logged-in agents
# TYPE vicidial_agents_total gauge
vicidial_agents_total $AGENTS
# HELP vicidial_active_calls Current auto-dial calls
# TYPE vicidial_active_calls gauge
vicidial_active_calls $ACTIVE_CALLS
# HELP vicidial_hopper_count Leads in hopper
# TYPE vicidial_hopper_count gauge
vicidial_hopper_count $HOPPER
# HELP vicidial_paused_agents Agents in PAUSED status
# TYPE vicidial_paused_agents gauge
vicidial_paused_agents $PAUSED
EOF
```

Run this via cron every 15 seconds:

```cron
* * * * * /usr/local/bin/vicidial-metrics.sh
* * * * * sleep 15 && /usr/local/bin/vicidial-metrics.sh
* * * * * sleep 30 && /usr/local/bin/vicidial-metrics.sh
* * * * * sleep 45 && /usr/local/bin/vicidial-metrics.sh
```

### Alerting Rules That Matter

Don't alert on everything. Alert on conditions that require human intervention:

```yaml
# Prometheus alerting rules — /etc/prometheus/rules/vicidial.yml
groups:
  - name: vicidial
    rules:
      - alert: HopperDraining
        expr: vicidial_hopper_count < 20
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Hopper below 20 leads for 5+ minutes"

      - alert: NoActiveCalls
        expr: vicidial_active_calls == 0 and vicidial_agents_total > 10
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "No active calls despite {{ $value }} agents logged in"

      - alert: HighMySQLConnections
        expr: mysql_global_status_threads_connected / mysql_global_variables_max_connections > 0.8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "MySQL connections at {{ $value | humanizePercentage }} of max"

      - alert: SwapActive
        expr: node_memory_SwapFree_bytes < node_memory_SwapTotal_bytes
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Server is swapping — VICIdial performance degraded"

      - alert: HighIOWait
        expr: rate(node_cpu_seconds_total{mode="iowait"}[5m]) > 0.2
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "I/O wait above 20% — possible disk bottleneck"
```

### Quick Command-Line Health Check

For operators who need a fast snapshot without dashboards:

```bash
#!/bin/bash
# /usr/local/bin/vicidial-healthcheck.sh
# Quick health check — run anytime

echo "=== VICIdial Health Check ==="
echo ""

# System
echo "--- System ---"
uptime
free -h | head -2
df -h / /var/spool/asterisk/monitor 2>/dev/null | tail -n +2
echo ""

# Asterisk
echo "--- Asterisk ---"
asterisk -rx 'core show channels count' 2>/dev/null || echo "Asterisk not responding"
asterisk -rx 'sip show registry' 2>/dev/null | tail -n +2 | head -5
echo ""

# MySQL
echo "--- MySQL ---"
mysql -u cron -pYOUR_PASSWORD asterisk -N -B -e "
    SELECT 'Agents:', COUNT(*) FROM vicidial_live_agents
    UNION ALL
    SELECT 'Active calls:', COUNT(*) FROM vicidial_auto_calls
    UNION ALL
    SELECT 'Hopper:', COUNT(*) FROM vicidial_hopper;
" 2>/dev/null
mysql -N -B -e "SHOW STATUS LIKE 'Threads_connected';" 2>/dev/null
echo ""

# Processes
echo "--- Screen Sessions ---"
screen -list 2>/dev/null | grep -c "ASTscreen"
echo "VICIdial screen sessions running"
echo ""

echo "--- Critical Processes ---"
for proc in AST_manager_listen AST_VDauto_dial AST_update AST_VDhopper; do
    COUNT=$(pgrep -fc "$proc")
    echo "$proc: $COUNT instance(s)"
done
```

---

## Advanced Tuning: Network and Storage I/O

Once you've addressed the kernel, Asterisk, MySQL, and web server layers, the remaining performance gains come from network and storage optimization.

### Network Tuning for SIP/RTP

SIP and RTP are UDP-based protocols. UDP has no built-in flow control — when a buffer fills, packets are silently dropped. On a busy telephony server, this manifests as audio quality problems (choppy voice, one-way audio) that only appear under load.

**Increase NIC ring buffer sizes:**

```bash
# Check current ring buffer settings
ethtool -g eth0

# Increase to maximum (values vary by NIC)
ethtool -G eth0 rx 4096 tx 4096
```

**Verify no packet drops:**

```bash
# Check for RX/TX errors and drops
ethtool -S eth0 | grep -i "drop\|error\|discard"

# Also check kernel-level drops
cat /proc/net/udp | awk '{print $5}' | sort -rn | head
```

If you see RX drops under load, the solutions in order of impact:
1. Increase ring buffers (above)
2. Enable RX interrupt coalescing: `ethtool -C eth0 rx-usecs 50`
3. Pin NIC interrupts to specific CPU cores (IRQ affinity)
4. Enable Receive Side Scaling (RSS) if the NIC supports it
5. Dedicated NIC for RTP traffic

### Storage I/O for Recordings

Call recording writes are a significant I/O pattern that's easy to overlook in capacity planning. Each active recorded call writes a continuous audio stream to disk. At 1,000 concurrent recorded calls with G.711:

```
Write throughput = 1,000 × 64 kbps = 64 Mbps = 8 MB/sec sustained
IOPS = 1,000 (one write stream per recording)
```

This isn't much for modern storage, but it's sustained and concurrent with everything else the disk is doing (Asterisk binary, configuration reads, log writes, MySQL if on the same server). The problem comes when recordings are stored on the same disk as MySQL data or Asterisk spool files.

**Best practice:** Separate mount point for recordings, preferably on a different physical disk:

```bash
# Dedicated recording volume
/dev/sdb1  /var/spool/asterisk/monitor  xfs  noatime,nodiratime  0 0
```

The `noatime` and `nodiratime` mount options eliminate the filesystem metadata writes that normally occur on every file access. Since recording files are written once and read rarely (only during playback), there's no benefit to tracking access times, and the I/O savings are measurable.

### tmpfs for Asterisk Temporary Files

Asterisk creates temporary files during call processing — particularly for recording mixing and format conversion. These temp files can be moved to RAM-backed tmpfs to eliminate disk I/O entirely:

```bash
# Add to /etc/fstab
tmpfs  /var/spool/asterisk/tmp  tmpfs  size=2G,noatime  0 0

# Mount it
mount -a
```

In `asterisk.conf`:

```ini
[directories]
astspooldir => /var/spool/asterisk
tmpdir => /var/spool/asterisk/tmp
```

This is especially beneficial on servers that do real-time recording mixing (dual-channel to mono conversion) during the call. The temporary intermediate files never touch disk.

---

## Where ViciStack Fits In

Every optimization in this guide — from the kernel sysctl parameters to the Nginx configuration to the Prometheus alerting rules — is baked into every ViciStack deployment from the first boot. We don't hand you a server with default settings and a tuning guide. We hand you a server that's already tuned for your specific agent count, codec configuration, recording requirements, and campaign type.

More importantly, we maintain the tuning over time. A deployment that runs 200 agents today and 400 agents next quarter needs different kernel parameters, different MySQL buffer sizes, different web server process limits, and different monitoring thresholds. ViciStack adjusts these as your operation scales — proactively, before performance degrades, not reactively after agents start complaining.

Performance tuning isn't a project. It's a practice. ViciStack exists so you can practice running your business instead of practicing Linux system administration.

[Get a free performance audit →](/free-audit/)

---

*This guide is maintained by the ViciStack team and updated as VICIdial, Linux kernel, and infrastructure best practices evolve. Last updated: March 2026.*

---

## Frequently Asked Questions

### How do I know if my VICIdial system needs performance tuning?

Three reliable indicators: (1) Real-time reports take more than 2 seconds to refresh — this means database queries are slow, which cascades to everything else. (2) Agents report delays between clicking a disposition and getting the next call — this indicates hopper drain issues, dial-level calculation lag, or MySQL lock contention. (3) Call quality degrades during peak hours (choppy audio, one-way audio, dropped calls) — this points to CPU saturation on telephony servers or network buffer exhaustion. Check `top` on every server during peak hours. If any server shows CPU above 80% sustained, I/O wait above 15%, or any swap usage at all, tuning is overdue. The healthcheck script in the monitoring section gives you all of these in one command.

### Can I run 500 agents on a single server?

Technically possible but practically a bad idea. A single server running Asterisk, MySQL, Apache, and all VICIdial processes creates a single point of failure and forces every component to compete for the same CPU, memory, and I/O. We've seen single-server deployments work up to 80-100 agents with proper tuning. Beyond that, component contention creates problems that no amount of tuning can solve — Asterisk needs CPU for audio processing at the same moment MySQL needs CPU for a complex report query, and one of them loses. The correct architecture at 500 agents is a [VICIdial cluster](/blog/vicidial-cluster-guide/) with dedicated telephony servers, web servers, and a database server. Clusters also provide redundancy — losing one telephony server in a four-server cluster reduces capacity by 25% instead of causing a total outage.

### What's the most impactful single optimization for VICIdial performance?

It depends on where the bottleneck is, but across hundreds of deployments, the answer is almost always one of three things: (1) MySQL `key_buffer_size` — the default is far too small for VICIdial's MyISAM workload, and increasing it to match your index size eliminates the most common I/O bottleneck. (2) The `skip-name-resolve` directive in `my.cnf` — this single line prevents DNS lookups on every database connection and has fixed more "unexplained slowness" than any other change. (3) Converting the hot real-time tables (`vicidial_live_agents`, `vicidial_auto_calls`, `vicidial_hopper`) to the MEMORY storage engine — this eliminates disk I/O for the most frequently accessed tables and reduces lock contention dramatically. All three are covered in our [MySQL optimization guide](/blog/vicidial-mysql-optimization/). If you haven't done any tuning yet, start there.

### How much RAM do I need per server role?

**Telephony servers:** 8-16 GB is sufficient. Asterisk's memory footprint is modest — approximately 120 KB per concurrent channel plus overhead. At 500 channels per server, Asterisk uses ~1 GB. The rest goes to VICIdial's Perl daemons and the OS. **Web servers:** 16-32 GB with Apache+mod_php (memory-hungry), 8-16 GB with Nginx+PHP-FPM (memory-efficient). The dominant factor is the number of concurrent agent sessions. **Database server:** As much as you can afford. 64 GB is the sweet spot for 500 agents — MySQL buffers use 8-10 GB, and the remaining 50+ GB serves as OS filesystem cache for MyISAM data files, which is where the real database performance comes from. At 800+ agents, move to 128 GB. RAM is the single most cost-effective upgrade for a VICIdial database server.

### Should I use VICIdial's Asterisk configuration or tune it myself?

Start with VICIdial's stock [Asterisk configuration](/blog/vicidial-asterisk-configuration/) and tune on top of it. The stock configuration establishes the SIP settings, dialplan contexts, and AMI permissions that VICIdial requires. Changing these without understanding VICIdial's call flow breaks things in subtle ways — calls connect but don't bridge to agents, recordings start but don't stop, transfers drop. The tuning in this guide (noload directives, timer adjustments, RTP settings, [codec](/glossary/codec/) selection) sits on top of the stock configuration and doesn't alter VICIdial's functional requirements. Never modify `extensions.conf` entries that VICIdial manages, never change AMI user permissions, and never alter the dialplan contexts without testing in a non-production environment first.

### What monitoring tool should I use for VICIdial?

For deployments under 100 agents where simplicity matters: Nagios or Zabbix with the checks described in this guide. Both are mature, well-documented, and can monitor everything VICIdial needs with standard plugins plus the custom checks shown above. For deployments above 200 agents where you need historical trending and dashboards: Prometheus + Grafana. The time-series database handles VICIdial's high-frequency metrics efficiently, and Grafana dashboards give managers and operations staff visual insight into system health. See our [Grafana dashboards guide](/blog/vicidial-grafana-dashboards/) for VICIdial-specific dashboard templates. Regardless of the tool, the non-negotiable alerts are: swap usage, hopper drain, screen session death, MySQL connection saturation, and disk space. If you monitor nothing else, monitor those five.

### How do I handle load balancing across multiple web servers?

VICIdial supports [load balancing](/glossary/load-balancing/) across web servers through its `server_ip` assignment in the `phones` table — each phone/agent is assigned to a specific web server. This is static assignment, not dynamic HTTP load balancing. For true HTTP load balancing (HAProxy, Nginx upstream, F5), you need to configure session persistence (sticky sessions) because VICIdial's agent interface maintains server-side state. Without sticky sessions, an agent's requests bounce between web servers and the session data doesn't follow, causing login loops and lost call state. The recommended approach: use DNS round-robin or a simple load balancer with cookie-based persistence, assigning each agent session to one web server for its entire duration. Failover to the other web server happens only on server failure, not per-request.

### How often should I revisit performance tuning?

After any significant change: adding 50+ agents, launching new campaigns with different call patterns (short calls vs. long calls, high [dial ratios](/settings/auto-dial-level/) vs. preview dialing), adding servers to the cluster, upgrading VICIdial SVN, or upgrading the OS/kernel. Also review quarterly even if nothing changes — tables grow, indexes fragment, and workload patterns shift gradually. The monitoring stack should catch acute problems (CPU spikes, swap usage, hopper drain), but gradual degradation requires periodic review. Run `pt-query-digest` on the slow query log monthly, check `SHOW TABLE STATUS` for table sizes that are growing faster than expected, and verify that your capacity planning assumptions still hold against actual agent counts and call volumes. If you're on ViciStack, we handle this automatically through weekly performance reviews and proactive tuning adjustments.

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-performance-tuning).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
