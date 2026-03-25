# VICIdial Cluster Configuration: The Complete Multi-Server Setup Guide [2026]

*Your single-server VICIdial box just hit a wall. Agents are getting "time synchronization" errors, the hopper keeps emptying, and your real-time report looks like it's running through molasses. You've Googled "VICIdial cluster setup" and found... nothing useful. A 15-year-old PDF, some AI-generated marketing fluff from offshore companies, and a dozen fragmented forum threads where the answer is always "search the forum."*

*This is the guide that should have existed years ago.*

We've built, managed, and troubleshot over 100 VICIdial-based contact centers. We've read every relevant thread on vicidial.org, studied the PoundTeam manual (which still references Core2Duo processors), and consolidated thousands of hours of tribal knowledge from Matt Florell and the community into one place.

This isn't a conceptual overview. This is the real thing — the architecture decisions, the exact configuration parameters, the mistakes that will cost you a weekend, and the optimizations that separate a cluster that handles 500 agents from one that melts down at 130.

---

## Why Your Single Server Has a Ceiling (And It's Lower Than You Think)

Here's the uncomfortable truth about VICIdial: **Asterisk doesn't scale vertically.** You can throw a 64-core monster at it and Asterisk will still choke at the same agent count as a quad-core. The VICIdial Group's Director of Engineering, Michael Cargile, said it plainly: "The biggest limiting factor is Asterisk itself. It has locking issues. Throwing monstrously powerful servers at it does not help."

A single all-in-one VICIdial server maxes out around **20-25 agents** for predictive outbound dialing. That's not a hardware limitation — it's an architectural one. Each agent running predictive at a 5:1 dial ratio generates 4-6 simultaneous Asterisk channels. Twenty-five agents means 125+ live channels fighting for the same CPU, the same MySQL instance, the same Apache processes, and the same disk I/O.

The solution isn't a bigger server. It's **more servers, each doing one job well.**

That's what a VICIdial cluster is: a group of purpose-built servers sharing a single database, each handling the specific workload it's optimized for. And it's the architecture behind every large-scale VICIdial deployment on the planet — including confirmed production environments running 1,000+ agents across 40 telephony servers processing 1.5 million calls per day.

> **Still Running Everything on One Box?**
> That ceiling doesn't move — no matter how much RAM you throw at it. ViciStack deploys optimized clusters overnight. [Get Your Free Cluster Assessment →](/free-audit/)

---

## The Four Server Roles Every Cluster Needs

Every VICIdial cluster is built from four core roles. You can combine some of them at smaller scales, but understanding what each one does is non-negotiable.

### The Database Server — Your Single Most Important Investment

This is the brain. Every agent login, every call disposition, every hopper query, every real-time report — all of it flows through one MySQL/MariaDB instance. The database server is both VICIdial's greatest strength (single source of truth, no sync issues) and its Achilles' heel (single point of failure, can't be clustered the way you'd cluster Postgres).

VICIdial uses **MyISAM exclusively** — not InnoDB. Matt Florell has been unambiguous: "We do not recommend using InnoDB under any circumstances." MyISAM's table-level locking is a limitation, but VICIdial's architecture is designed around it, and switching breaks things in spectacular ways.

Hardware guidelines based on real production deployments:

| Agent Count | CPU | RAM | Storage | Network |
|-------------|-----|-----|---------|---------|
| Up to 50 | 4 cores, 2.0+ GHz | 8 GB | 250 GB RAID-1 SSD | 1 Gbps |
| Up to 150 | 8 cores, 2.0+ GHz | 16 GB | RAID-10 SSD (LSI MegaRAID) | 1 Gbps |
| Up to 300 | 16 cores | 32 GB | NVMe RAID-10, dedicated controller | 1 Gbps |
| Up to 600 | 2× octal-core | 64 GB | NVMe RAID-10, LSI MegaRAID | 10 Gbps |

The RAID controller matters more than you think. Use **LSI Logic MegaRAID only.** Not 3ware. Not Adaptec. Not Dell PERC. One forum user hired Percona for a $3,000 MySQL audit and the answer was the same: swap the RAID card. At 200+ agents, switch from SSD to NVMe — the latency difference becomes measurable.

### Telephony (Dialer) Servers — Where Calls Actually Happen

Each dialer runs Asterisk and handles outbound/inbound call processing for a slice of your agents. The golden rule: **20 agents per telephony server** for predictive outbound. You can push to 25 on a high-clock quad-core, maybe 35 on dual hex-core Xeons, but you're borrowing from your stability margin.

The critical spec is **single-thread CPU performance**, not core count. Asterisk's internal locking means it can't efficiently parallelize across many cores. A 4-core chip running at 4.5 GHz will outperform an 8-core at 2.5 GHz every time.

If you're running [AMD (Answering Machine Detection)](/blog/vicidial-amd-guide/), cut your per-server agent count nearly in half. AMD uses a loopback trunk architecture that effectively doubles the channel count on each call.

### Web Servers — Stateless and Scalable

Web servers run Apache and serve the agent interface, admin panels, and real-time reports. They're stateless — no server-side sessions — which means you can scale them horizontally behind a load balancer without any sticky-session complexity.

One web server handles approximately **150 agents** on standard hardware, dropping to about 75 if you're running SSL/TLS (which you should be, especially with WebRTC). The web server is the one role where virtualization is genuinely acceptable. William Conley's take: "Web is the only server you can get away with running virtual."

### Archive/Recording Server — Don't Skip This

Every telephony server records calls locally. Those recordings need to get consolidated somewhere searchable. The archive server is just an FTP/HTTP target — it doesn't even need VICIdial installed. But without it, your recordings are scattered across every dialer, and the playback links in your reports will randomly return "Object not found" depending on which server handled the call.

Budget **1-2 TB per 100 agents per month** for full call recording in MP3 format.

---

## The Port Matrix Nobody Published Until Now

Before you install anything, your firewall rules need to be correct. Getting one port wrong means you'll spend two days chasing phantom audio issues. Here's the complete inter-server port reference:

| Port | Protocol | Direction | Purpose |
|------|----------|-----------|---------|
| 3306 | TCP | All servers → DB | MySQL/MariaDB |
| 4569 | UDP | Dialer ↔ Dialer | IAX2 inter-server calls |
| 5060 | UDP | Carriers/phones → Dialers | SIP signaling |
| 5038 | TCP | Scripts → Local Asterisk | AMI (Asterisk Manager Interface) |
| 4577 | TCP | Dialers (local) | FastAGI |
| 10000-20000 | UDP | Bidirectional (all) | RTP audio streams |
| 80/443 | TCP | Agents → Web servers | HTTP/HTTPS |
| 8089 | TCP | Agents → Dialers | WebSocket (ViciPhone/WebRTC) |
| 21 | TCP | Dialers → Archive | FTP recordings |

**Pro tip:** Set up dual NICs on every telephony server. One NIC on the public network for carriers and agents, one on a **private gigabit switch** for inter-server communication. No firewall between cluster nodes on the private network. This eliminates 90% of the audio issues people spend weeks debugging.

---

## Keepalive Flags: The Configuration That Breaks Every New Cluster

Here's where most cluster deployments go sideways. VICIdial uses a set of numbered "keepalive flags" in `/etc/astguiclient.conf` to control which background processes run on each server. Set them wrong and you'll get duplicate calls, erratic dial levels, or a system that silently stops dialing.

The `VARactive_keepalives` setting tells `ADMIN_keepalive_ALL.pl` which processes to babysit. Here's the cheat sheet:

| Flag | Process | Run Where |
|------|---------|-----------|
| 1 | AST_update.pl | ALL telephony servers |
| 2 | AST_send_listen.pl | ALL telephony servers |
| 3 | AST_VDauto_dial.pl | ALL telephony servers |
| 4 | AST_VDremote_agents.pl | ALL telephony servers |
| **5** | **AST_VDadapt.pl** | **ONE telephony server ONLY** |
| 6 | FastAGI_log.pl | ALL telephony servers |
| **7** | **AST_VDauto_dial_FILL.pl** | **ONE telephony server ONLY** |
| 8 | ip_relay | ALL telephony servers |
| **9** | **Timeclock auto-logout** | **ONE server ONLY** |
| E | Email inbound parser | ONE server ONLY |
| X | No keepalives | DB-only or web-only servers |

**The single most dangerous misconfiguration:** running flag 5 or 7 on more than one server. Flag 5 is the adaptive predictive algorithm — it calculates dial levels for the entire cluster. Flag 7 is the fill/balance dialer that redistributes calls across servers with spare capacity. Run either on two servers simultaneously and you'll get conflicting dial-level calculations, double-dialed leads, and erratic agent utilization that'll make you question your sanity.

Your database server gets `VARactive_keepalives = X`. Your web servers get `X`. Your telephony servers get `1238` (or `12368` with FastAGI). Exactly ONE telephony server gets `123456789` — that's your "primary" dialer running the one-server-only processes.

---

## MySQL Tuning That Actually Matters

Out of the box, MySQL/MariaDB is configured for a general-purpose workload. VICIdial's access patterns — high-frequency writes to narrow tables, constant polling of MEMORY tables, bulk reads for reports — need specific tuning. Here are the parameters that make a measurable difference:

```ini
[mysqld]
# THE most important single setting for VICIdial
skip-name-resolve

# Connection capacity (default is often 100-151, way too low)
max_connections = 4000

# MyISAM key buffer — VICIdial's primary cache
key_buffer_size = 640M        # Standard; 4096M for enterprise

# Intentionally disabled — write invalidation overhead exceeds benefit
query_cache_size = 0

# Table cache (VICIdial opens hundreds of tables)
table_open_cache = 8192

# MEMORY table ceiling
max_heap_table_size = 64M
tmp_table_size = 64M

# Allow concurrent inserts during reads (MyISAM-specific)
concurrent_insert = 2
```

**The `skip-name-resolve` story deserves its own paragraph.** Without this setting, MySQL performs a reverse DNS lookup on every single connection. In a cluster where telephony servers, web servers, and cron jobs are all hammering the database, those DNS lookups pile up and create a connection backlog that looks exactly like "too many connections" — except raising `max_connections` doesn't fix it. Add `skip-name-resolve` and the problem vanishes. This single line has probably saved more VICIdial clusters than any other configuration change.

> **Skip the 3 AM MySQL Tuning Sessions.**
> Every ViciStack cluster ships with skip-name-resolve, MEMORY tables, and archive jobs pre-configured. [See What Optimized Looks Like →](/pricing/)

### The MEMORY Table Optimization

For clusters above 200 agents, converting high-traffic tables to the MEMORY engine is transformational. `vicidial_live_agents` — the table that tracks every agent's real-time state — gets hammered by every dialer, every web server, and every real-time report simultaneously. On MyISAM, table-level locks create contention that manifests as agents freezing and calls not being placed. Converting to MEMORY moves the entire table into RAM, and forum users consistently describe the difference as "night and day."

The catch: MEMORY tables use fixed-width CHAR fields, so you need to shorten VARCHAR columns first. And the table loses all data on MySQL restart — but VICIdial repopulates it automatically when agents log in.

### Archive Your Logs or Die

This is the maintenance task nobody does until their cluster melts down. `vicidial_log`, `call_log`, `vicidial_log_extended`, `vicidial_carrier_log` — these tables grow indefinitely. On a high-volume cluster, they can reach hundreds of millions of rows within months.

Run `ADMIN_archive_log_tables.pl --daily` in cron at 1 AM. It moves old records to `_archive` suffix tables, keeping only the last 24 hours in the hot tables. For high-volume systems (500+ agents), run it more aggressively and keep `vicidial_list` under 10 million leads. Un-archived tables are the #1 cause of the cascading failure pattern: slow queries → table locks → too many connections → cluster-wide outage.

---

## The Recording Pipeline Nobody Explains

Recordings in a cluster follow a three-stage FTP pipeline that runs every 3 minutes on each telephony server, staggered by 1-minute offsets:

1. **Stage 1 (audio_1_move_mix):** Converts raw recordings in `/var/spool/asterisk/monitor/` to the configured format and moves them to `monitorDONE/`
2. **Stage 2 (audio_2_compress):** Compresses to MP3 (or configured format) and stages for transfer
3. **Stage 3 (audio_3_ftp):** FTPs completed recordings to the archive server

When recordings go missing — and they will — check the `monitorDONE/FTP/` directory on each telephony server. If files are accumulating there, your FTP transfer to the archive server is failing. Common culprits: DNS resolution failure on a single dialer (the other dialers work fine, making it maddening to diagnose), special characters in the FTP password that need Perl escaping, or the archive server's disk filling up silently.

The pipeline moves files regardless of transfer success. If the archive server is unreachable, recordings pile up in the FTP staging directory. They're not lost — but they're not where your web interface expects them, so playback links break.

---

## The "Time Synchronization Problem" Error Is Almost Never About Time

This is the most misunderstood error in VICIdial, and it drives new cluster operators crazy. The community's most experienced consultant, William Conley, put it definitively: "5% of the time this is the actual problem. The other 95% of the time, the agent's session timestamp simply isn't being updated."

When you see "time synchronization problem" on an agent's screen, here's your actual diagnostic checklist:

**Step 1: Check screen sessions.** SSH into each telephony server and run `screen -ls`. You should see 7-9 active sessions. If any are missing, your keepalive scripts have crashed. This is the cause ~60% of the time.

**Step 2: Check the AMI connection.** Verify `/etc/asterisk/manager.conf` has the correct `bindaddr`, username, and secret. Port 5038 must be accessible on localhost. Wrong AMI credentials = scripts can't talk to Asterisk = stale timestamps.

**Step 3: Check server load.** If the system load exceeds the number of CPU cores, keepalive scripts can't update the `server_updater` table fast enough, and VICIdial interprets the stale timestamp as a time sync issue.

**Step 4: Check DAHDI timing.** Run `dahdi_test` — you should see 99.99%+ accuracy. Without a proper timing source, everything downstream breaks.

**Step 5: THEN check NTP.** And only then. Designate one server as the NTP master syncing to `pool.ntp.org`, point every other cluster node at that single internal master. Offsets should be under 1 millisecond.

> **"Time Synchronization Problem" Giving You PTSD?**
> Our ops team monitors your cluster 24/7 so the only time you think about screen sessions is in your nightmares. [Let Us Handle It →](/free-audit/)

---

## No Audio Between Servers: The Cluster Problem Hall of Fame

This is the #1 reported cluster issue on the VICIdial forum, with 10+ threads and counting. The symptom is maddening: calls ring, the agent picks up, and... silence. Or one-way audio. But only on calls that cross servers. Same-server calls work perfectly.

Here's the complete fix checklist, in order of likelihood:

**1. Add externip and localnet to sip.conf on every dialer.** Without these, Asterisk advertises its internal IP in SDP packets, and the remote party sends RTP audio to an unreachable address. This is the cause in roughly half of all cluster audio issues.

```ini
[general]
externip=YOUR.PUBLIC.IP.HERE
localnet=192.168.1.0/255.255.255.0
```

**2. Open UDP 10000-20000 bidirectionally between ALL nodes.** SIP on port 5060 handles signaling (the phone rings), but the actual audio travels on a random high UDP port. Block those ports and you get a ringing phone with no voice.

**3. Set canreinvite=no and directmedia=no.** These settings tell Asterisk to keep media flowing through itself rather than negotiating direct paths between endpoints — essential when NAT is involved.

**4. Use a private switch between servers with zero firewall.** This is the nuclear option that eliminates the entire class of inter-server audio problems. Public NICs face carriers and agents. Private NICs face each other on an unfiltered gigabit switch. All inter-server SIP, IAX2, and RTP traffic uses the private network.

> **One-Way Audio Is a Rite of Passage. You've Passed.**
> ViciStack clusters come with dual-NIC, proper externip, and every port rule pre-configured. [Skip the Pain →](/free-audit/)

---

## Capacity Planning: From 50 to 500+ Agents

Here are reference architectures based on real production deployments, not theoretical calculations:

| Scale | DB Servers | Web Servers | Telephony Servers | Archive | Total Servers |
|-------|-----------|-------------|-------------------|---------|---------------|
| 50 agents | 1 | 1 (can share with DB) | 2-3 | 1 | 4-5 |
| 100 agents | 1 (dedicated) | 1-2 | 4-5 | 1 | 7-9 |
| 200 agents | 1 (beefy) | 2-3 | 8-10 | 1 | 12-15 |
| 300 agents | 1 (enterprise) + 1 slave | 3-4 | 12-15 | 1-2 | 18-22 |
| 500 agents | 1 (maxed) + 1 slave | 4-5 | 20-25 | 2 | 28-33 |

**The 125-agent threshold:** William Conley warns that around 125 agents, "you will enter a new realm of 'why did THAT break?'" This is where MyISAM table-lock contention on `vicidial_live_agents` starts causing cascading performance issues. MEMORY table conversion becomes essential, database hardware needs to be serious, and your monitoring game needs to be tight.

**The 300-agent decision point:** Above 300 agents with aggressive dial ratios, consider splitting into **multiple independent clusters** rather than scaling one cluster further. The database is a single point of failure, and the math gets scary: one DB outage takes out 300+ agents for hours. Two clusters of 150 each means one failure only impacts half your operation.

**The 500-agent ceiling:** VICIdial's MEMORY tables use single-threaded operations capped at roughly the CPU's effective clock speed. Around 500 agents with predictive dialing, you're pushing the physical limits of a single MySQL instance. The proven path beyond this is **Cross-Cluster Communication (CCC)** — connecting multiple independent VICIdial clusters via IAX2 trunks and API lookups during transfers. The largest confirmed CCC deployment: 1,600+ agents across multiple clusters processing 6 million calls per day.

> **Planning a 200-Agent Cluster? We've Done It 100+ Times.**
> Stop guessing at server counts. Get a custom architecture recommendation based on your actual workload. [Get Your Free Assessment →](/free-audit/)

---

## The Virtualization Question — Answered Once and For All

This comes up on every single cluster planning thread, so let's settle it.

**Matt Florell (VICIdial creator):** "You will actually need more hardware and spend more money when you virtualize. Best case scenario is 50% of normal bare metal capacity."

**William Conley (PoundTeam):** "If you go Virtual, you're entering a war you cannot win."

**But the nuance matters.** Cloud hosting on **dedicated instances** works. Over 100 agents have been confirmed running outbound on AWS EC2 dedicated hosts. The key distinction: dedicated cloud instances give you actual hardware with no noisy neighbors and no CPU throttling. Shared/burstable instances (t3, t4g) will fail because DAHDI needs precise 1ms kernel timer ticks, and CPU stealing destroys timing accuracy.

The practical rule:
- **Telephony servers:** Bare metal or dedicated cloud instances. Period.
- **Database server:** Bare metal strongly preferred. NVMe I/O matters too much.
- **Web servers:** Virtualization is fine. They're stateless HTTP handlers.
- **Archive server:** Anything with enough storage. Cloud object storage works.

If you're choosing between hosting providers for bare metal, Hetzner's AX-series and OVH Rise servers run $50-85/month each — a fraction of what you'd pay for equivalent AWS instances. A full 500-agent cluster on Hetzner bare metal costs roughly **$860/month for 14 servers**. That's $1.72 per agent per month for compute. Compare that to [Convoso at $120-225 per agent per month](/blog/vicidial-vs-convoso/), or [Five9 at $149-199](/blog/vicidial-vs-five9/). The math isn't close.

> **Bare Metal. Pre-Configured. Ready to Dial.**
> Hetzner AX-series + ViciStack optimization = cluster-grade performance at $1.72/agent/month in compute. [Get Your Custom Quote →](/free-audit/)

---

## WebRTC and ViciPhone in a Cluster

ViciPhone (VICIdial's built-in WebRTC softphone) adds a layer of complexity in clustered environments that catches people off guard.

Each telephony server needs its own **FQDN with a valid SSL certificate.** WebRTC requires HTTPS — no exceptions, no self-signed certs in production. Every dialer needs its own WebSocket URL configured (`wss://dial1.yourdomain.com:8089`), its own webRTC phone template, and phones assigned to that specific server's template.

The latest ViciPhone v3.0 (a complete rewrite using SIP.js) requires `rtcp_mux=yes` in your pjsip or SIP configuration. STUN configuration in `/etc/asterisk/rtp.conf` is required on every dialer. If you're running multiple web servers, they can sit behind HAProxy or a load balancer, but each agent's phone connection must route to their assigned telephony server — you can't load-balance WebSocket connections across dialers.

---

## The Asterisk 21 Situation: What You Need to Know Right Now

Asterisk 21+ removes three things VICIdial has depended on for two decades: `app_meetme` (the conferencing engine that makes every call work), `res_monitor` (call recording), and `chan_sip` (SIP channel driver). This isn't a "someday" problem — Asterisk 20 is the last LTS with these modules, with full support through 2026 and security patches through 2027.

The good news: **VICIdial already has ConfBridge support production-ready.** As of SVN revision 3894 (January 2025), ConfBridge code is fully integrated and has been running in production at multiple companies for over a year. PJSIP support is also in the codebase. New cluster deployments should target Asterisk 18 with ConfBridge enabled.

The performance caveat matters for cluster planning: MeetMe does audio mixing in DAHDI kernel space. ConfBridge does it in userspace. Michael Cargile's assessment: ConfBridge uses roughly **20% more CPU** for the same workload, and the performance gap widens significantly on virtual machines. This means your per-dialer agent capacity drops slightly with ConfBridge — plan for 18 agents per server instead of 20-25 if you're being conservative.

Set `mixing_interval=20` in `confbridge.conf` to prevent out-of-order RTP packets, and use `res_timing_dahdi.so` instead of the default `res_timing_timerfd.so`.

---

## The 10 Things That Will Break Your Cluster (And How to Fix Them)

After mining every cluster-related thread on vicidial.org, these are the problems you'll actually hit:

| # | Problem | Real Cause | Fix |
|---|---------|-----------|-----|
| 1 | No audio between servers | Missing externip/localnet in sip.conf | Add externip + localnet; open UDP 10000-20000 |
| 2 | "Time synchronization" errors | Screen sessions crashed (~60%) | `screen -ls`; verify keepalives running |
| 3 | Recordings missing | FTP pipeline broken on one dialer | Check monitorDONE/FTP directory size |
| 4 | "Too many connections" | Missing skip-name-resolve | Add to my.cnf; set max_connections=4000 |
| 5 | Agents waiting, no calls placed | Hopper empty or trunk exhaustion | Check screen sessions; verify TRUNK SHORT/FILL |
| 6 | Duplicate calls | Flag 5 or 7 on multiple servers | Only ONE server runs 5 and 7 |
| 7 | Agent screens freezing | DB table locks from un-archived logs | Run ADMIN_archive_log_tables.pl daily |
| 8 | New server won't join cluster | IP not in servers table | Run ADMIN_update_server_ip.pl |
| 9 | IAX2 between dialers not working | Registration only works one direction | Known bug; verify iax.conf on both ends |
| 10 | DB load at 100% | RAID-5, missing key_buffer_size tuning | Switch to RAID-10; tune MySQL per above |

> **That's 10 Problems You Don't Have to Solve.**
> ViciStack clusters ship with every fix in this table pre-applied. Zero debugging required. [See Our Architecture →](/pricing/)

---

## Why Convoso's "VICIdial Breaks at Scale" Narrative Is Nonsense

Let's address the elephant in the room. If you've researched VICIdial scaling, you've probably seen [Convoso's content claiming](/blog/vicidial-vs-convoso/) the platform "breaks at scale" and "suffers dropped calls beyond 20-30 seats."

Here's what they're actually saying: **a single all-in-one server** maxes out around 20-30 agents. That's true. It's also like saying "a single EC2 instance can't run Netflix" — technically correct about one server, deliberately misleading about the platform.

Convoso's flagship blog post on VICIdial's "hidden costs" cites $400/server/month from VICIhost — the premium managed hosting tier — as if it's the only way to run VICIdial. They never mention that self-hosted bare metal runs [$50-85/month per server](/blog/vicidial-cost-2026/). At 500 agents, the annual cost difference between a VICIdial cluster and Convoso is over **$400,000.**

What Convoso conspicuously avoids mentioning in any of their anti-VICIdial content: the word "cluster." Their entire scaling argument depends on their audience not knowing that VICIdial's documented, proven multi-server architecture exists and is running 1,000+ agent operations right now.

Fun fact: Convoso itself was built on VICIdial code for 11 years before pivoting to a proprietary platform. They know exactly how capable VICIdial clustering is. They just don't want you to know.

---

## Where ViciStack Fits In

Look, we just handed you the keys to the kingdom. Everything above is exactly how we build production clusters for our own operation — 100 centers, ~600,000 monthly dials. We're not gatekeeping this knowledge behind a sales call.

But here's the reality: building a cluster right takes time, expertise, and ongoing maintenance. Archiving logs, monitoring MySQL processlist, rotating SSL certs, patching Asterisk, replacing failed drives — it never stops.

**ViciStack exists for operators who want cluster-grade VICIdial performance without becoming full-time sysadmins.** We ship pre-configured bare metal clusters on dedicated hardware with every optimization in this guide already baked in. Your dials go up, your [AMD accuracy](/blog/vicidial-amd-guide/) hits 92-96%, and you stop paying $150/agent/month to companies that built their platform on VICIdial's code.

Same architecture. Fraction of the effort. Dramatically lower cost than the cloud dialers.

[Get a custom cluster assessment →](/free-audit/)

---

*This guide is maintained by the ViciStack team and updated as VICIdial, Asterisk, and best practices evolve. Last updated: March 2026.*

*Have a cluster question this guide didn't answer? Drop it in the comments or reach out directly — if it's good, we'll add it to the guide.*

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-cluster-guide).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
