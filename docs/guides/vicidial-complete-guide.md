# VICIdial: The Complete Guide to Open-Source Call Center Software

**Last updated: March 2026 | Reading time: ~26 minutes**

VICIdial is the most widely deployed open-source contact center platform in the world. Over 14,000 installations. Zero licensing fees. It handles predictive dialing, inbound routing, IVR, blended campaigns, call recording, real-time reporting, and agent management — all through a web browser.

If you're reading this, you're probably in one of three situations. You're evaluating VICIdial as an alternative to expensive hosted dialers. You've already installed it and you're trying to figure out what half the settings do. Or you've been running it for years and want to know if you're getting everything out of it.

This guide covers all three scenarios. No vendor pitch. Just an honest assessment of what VICIdial is, what it's not, where it shines, and where it'll hurt you if you're not prepared.

---

## What VICIdial Actually Is

VICIdial is a PHP web application that sits on top of Asterisk (the open-source PBX). Asterisk handles the actual phone calls — SIP signaling, RTP media, codec negotiation. VICIdial handles everything else — agent screens, lead management, campaign configuration, recording, reporting, and the predictive dialing algorithm.

The stack:

```
┌─────────────────────────────────────┐
│  VICIdial Admin + Agent Web UI      │  PHP, Apache/Nginx
│  (campaign config, reporting, etc.)  │
├─────────────────────────────────────┤
│  VICIdial Backend Scripts            │  Perl daemons
│  (auto-dial, hopper, timeclock,etc.) │  (AST_VDauto_dial.pl, etc.)
├─────────────────────────────────────┤
│  MySQL / MariaDB                     │  Lead data, CDRs, config
│  (vicidial_list, vicidial_log, etc.) │
├─────────────────────────────────────┤
│  Asterisk PBX                        │  SIP, RTP, dialplan,
│  (handles actual calls)              │  AMI, recording
├─────────────────────────────────────┤
│  Linux OS                            │  ViciBox (OpenSuSE) or
│  (AlmaLinux, Rocky, Debian)          │  manual install
└─────────────────────────────────────┘
```

The Perl backend scripts are the real engine. `AST_VDauto_dial.pl` is the predictive dialer — it monitors agent availability, calculates dial ratios, and sends originate commands to Asterisk via the [Manager Interface (AMI)](/blog/asterisk-manager-interface-guide/). `AST_VDhopper.pl` loads leads from the database into the [dial hopper](/blog/vicidial-dial-hopper-guide/) — a pre-filtered buffer that the dialer pulls from.

Everything runs in a web browser. Agents need nothing installed on their machines. Admins configure everything through the admin panel. Reports generate in the browser. Even the agent phone can run in the browser using WebRTC (since VICIdial 2.14 with Asterisk 16+).

---

## What VICIdial Can Do

The feature list is genuinely massive. Here's what matters for most operations:

### Outbound Dialing

- **Predictive dialing** — The algorithm monitors agent answer rates, call durations, and availability to calculate how many leads to dial simultaneously. When tuned correctly, it keeps agents on calls 85-95% of their logged-in time with minimal dropped calls.
- **Progressive dialing** — Dials one call per available agent. Lower abandonment risk than predictive but lower agent utilization.
- **Preview dialing** — Shows lead info to the agent before dialing. Agent reviews and clicks to dial.
- **Manual dialing** — Agent types or selects a number and clicks dial. For high-value outreach.
- **Ratio dialing** — Fixed ratio of calls per agent (e.g., 2:1 or 3:1).

### Inbound Call Handling

- **ACD (Automatic Call Distribution)** — Routes inbound calls to available agents based on skills, priority, and wait time.
- **IVR (Interactive Voice Response)** — Multi-level menus with DTMF input. "Press 1 for sales, press 2 for support."
- **Inbound DID routing** — Map specific DIDs to specific in-groups, IVRs, or agents.
- **Skills-based routing** — Route calls based on agent skills, language, product knowledge.
- **Queue management** — Hold music, position announcements, callback offers.

### Blended Campaigns

This is where VICIdial really differentiates from basic dialers. An agent can handle both inbound and outbound calls in the same session. When inbound calls are slow, the agent gets outbound leads. When an inbound call comes in, the system pauses outbound dialing for that agent and routes the inbound call.

Blended mode is how you maximize agent utilization without overstaffing. It's also one of the harder things to configure correctly. The balance between inbound priority and outbound dial levels takes experimentation.

### Call Recording

Every call can be recorded automatically. Recordings are stored on the server and accessible through the admin interface. You can configure recording at the campaign, in-group, or individual call level.

Recording storage adds up fast. At 1 MB per minute (standard quality), a 100-agent operation doing 8 hours of calls daily generates about 48 GB per day. Plan your storage accordingly.

### Agent Management

- **Real-time monitoring** — See every agent's status (ready, on call, paused, in wrap-up), current call duration, caller ID, and campaign.
- **Whisper/barge** — Listen to agent calls silently, whisper coaching the agent can hear (lead can't), or barge in for three-way conversation.
- **Pause codes** — Track why agents pause (break, lunch, bathroom, coaching). Essential for accountability.
- **Agent timeclock** — Built-in time tracking with login/logout and pause time reporting.
- **Screen scripting** — Display custom scripts to agents based on lead data, disposition, or campaign.

### Lead Management

- **List management** — Upload, filter, shuffle, and segment lead lists. CSV import, API import, manual entry.
- **Lead recycling** — Automatically re-attempt leads after configurable delays based on disposition.
- **DNC management** — Internal DNC lists, campaign DNC, and system-wide DNC. Scrub leads before dialing.
- **Callback scheduling** — Agents can schedule callbacks (agent-specific or anyone-available). The system automatically redials at the scheduled time.
- **Dial count limits** — Cap how many times any lead gets called. For TCPA compliance and to avoid burning your reputation.

### Reporting

VICIdial includes dozens of built-in reports:

- **Real-Time Report** — Live view of all agents, campaigns, and call activity
- **Agent Performance** — Calls per hour, talk time, wait time, pause time, disposition breakdown
- **Campaign Statistics** — Contact rate, dial-to-connect ratio, drops, abandon rate
- **Call Detail Records** — Every call with timestamps, duration, disposition, recording link
- **Outbound Calling Report** — Detailed per-call data for compliance auditing
- **DID Report** — Inbound call tracking per number
- **Export** — Most reports export to CSV

The reporting is functional but not pretty. If you need polished dashboards, you'll either use the [Grafana integration](/blog/vicidial-grafana-dashboards/) or build custom reports against the MySQL database.

---

## What VICIdial Is Not

Let's be honest about the limitations, because the forums and Reddit are full of people who went in with the wrong expectations.

**It's not a cloud SaaS product.** There's no sign-up page. You run it on your own servers (or pay someone to host it). You manage the infrastructure, the updates, the backups. If Asterisk crashes at midnight, that's your problem.

**The UI is from 2006.** The admin interface is functional — it does everything — but it looks like it was designed by someone whose primary tool was a text editor. Because it was. This is PHP from the era of tables-for-layout. No responsive design. No modern JavaScript framework. It works, but don't show it to a designer.

**No built-in CRM.** VICIdial manages leads and call activity, but it's not a CRM. There's no deal pipeline, no email tracking, no marketing automation. You'll need to integrate with a separate CRM (Salesforce, HubSpot, custom) using the [VICIdial API](/blog/vicidial-api-integration/).

**Documentation is thin.** The official documentation is the Manager manual (PDF) and the vicidial.org forums. There is no getting-started tutorial. There is no API reference website. The forums have the answers, but they're buried in 15 years of threads. You'll spend time searching.

**No mobile app.** The agent screen is a web page designed for desktop browsers. There's no native mobile experience. Agents on phones would use the WebRTC interface, which works but isn't optimized for small screens.

---

## Who Uses VICIdial

VICIdial's user base is larger than most people realize:

- **Outbound sales operations** — The largest user segment. Solar, insurance, home services, political campaigns, debt consolidation, real estate.
- **Collection agencies** — Predictive dialing for debt collection with TCPA-compliant settings.
- **Political campaigns** — Volunteer phone banks and paid canvassing operations. VICIdial's [political campaign features](/blog/vicidial-political-campaigns/) include survey scripting and voter file integration.
- **Customer service centers** — Inbound ACD with IVR and skills-based routing.
- **BPOs (Business Process Outsourcing)** — Multi-tenant VICIdial clusters serving multiple clients. Common in the Philippines, India, and Latin America.
- **Government agencies** — Some municipalities and state agencies run VICIdial for citizen outreach and survey programs.

The common thread: organizations that make or receive enough calls to justify dedicated infrastructure but don't want to pay $100-200/seat/month for a hosted solution.

---

## What VICIdial Costs

The software is free. The infrastructure isn't. Here's a realistic breakdown.

### Server Hardware

For a 20-agent operation, a single server handles everything:

| Component | Spec | Cost |
|-----------|------|------|
| CPU | Intel Xeon E-2388G (8 cores) | $350 |
| RAM | 64 GB ECC DDR4 | $200 |
| Storage | 2× 1TB NVMe (RAID 1) | $200 |
| OS Drive | 500GB NVMe | $50 |
| NIC | Intel X710 dual-port 10GbE | $150 |
| Server chassis | Supermicro 1U or similar | $400 |
| **Total** | | **~$1,350** |

For 50+ agents, you need a [multi-server cluster](/blog/vicidial-cluster-guide/) — separate database server, web server, and one or more telephony servers. Double or triple the hardware cost.

Colocation for a single server runs $50-150/month at most data centers. If you go cloud (AWS, GCP), expect $200-500/month for an equivalent instance.

We have a [full cost analysis here](/blog/vicidial-cost-2026/) if you want the detailed numbers.

### SIP Trunking

You need a SIP carrier to route calls to the PSTN. Costs vary by volume:

- **Low volume** (< 50K minutes/month): $0.01-0.02/minute
- **Medium volume** (50K-500K minutes/month): $0.005-0.01/minute
- **High volume** (500K+ minutes/month): $0.003-0.007/minute

For a 20-agent operation doing 8 hours/day, 20 business days/month, roughly 50% talk time:

```
20 agents × 8 hours × 60 min × 20 days × 50% = 96,000 minutes
96,000 × $0.008/min = $768/month in carrier costs
```

Add $20-50/month for DIDs (caller ID numbers). [Carrier selection guide here](/blog/vicidial-carrier-selection/).

### Support and Maintenance

This is where people underestimate costs. VICIdial doesn't maintain itself.

- **Updates:** VICIdial releases SVN updates regularly. Someone needs to apply them, test, and handle any issues.
- **Asterisk maintenance:** Security patches, version upgrades, configuration tuning.
- **Database maintenance:** Table optimization, log rotation, backup verification.
- **Troubleshooting:** SIP issues, one-way audio, registration failures, [hopper problems](/blog/vicidial-dial-hopper-guide/), agent connection issues.

You're either paying a sysadmin ($4-8K/month for a good one) or you're doing it yourself. Or you hire a managed VICIdial provider.

### Total Cost Comparison

| | VICIdial (Self-Hosted) | VICIdial (Managed) | Hosted Dialer |
|---|---|---|---|
| 20 agents | ~$1,200/month | ~$2,000/month | $2,000-4,000/month |
| 50 agents | ~$2,500/month | ~$4,000/month | $5,000-10,000/month |
| 100 agents | ~$4,000/month | ~$6,500/month | $10,000-20,000/month |

The savings compound with scale. At 100 agents, VICIdial costs a fraction of hosted alternatives. But you're taking on the operational responsibility. That's the trade-off.

---

## Getting Started with VICIdial

### Installation Options

**ViciBox (Recommended for New Installs)**

ViciBox is a pre-built ISO image that installs a complete VICIdial system. Boot from the ISO, answer a few questions, and you have a working system in 30-45 minutes. ViciBox 12.0.2 (current) runs on OpenSuSE Leap 15.6 with Asterisk 18 and MariaDB 10.11.

```bash
# After ViciBox install, the system is at:
# Admin: http://your-server-ip/vicidial/admin.php
# Agent: http://your-server-ip/agc/vicidial.php
# Default admin login: 6666 / 1234 (change immediately!)
```

**Manual Install on AlmaLinux 9**

If you want control over the base OS, install AlmaLinux 9 minimal and then install VICIdial from SVN. This is harder but gives you full control. Our [setup guide](/blog/vicidial-setup-guide/) covers this path in detail.

**Cloud Deployment**

VICIdial runs fine on cloud instances (AWS EC2, GCP Compute Engine, DigitalOcean Droplets), but there are caveats. Cloud networking adds latency and jitter that can degrade [call quality](/blog/voip-mos-score-guide/). Bare metal is better for voice. If you go cloud, use dedicated instances (not shared/burstable) and pick a region close to your SIP carrier.

### First Campaign Setup

Once VICIdial is installed, here's the path to your first outbound call:

**1. Configure a SIP Trunk (Carrier)**

In the admin GUI, go to **Carriers** → **Add New Carrier**. Enter your SIP provider's details:
- Carrier name and description
- Registration string: `username:password@sip.carrier.com`
- Codec preferences (G.711 ulaw first)
- Dial prefix (1 for US domestic)

**2. Create a Campaign**

**Campaigns** → **Add New Campaign**. Key settings:
- **Dial Method:** Start with RATIO until you understand the system. Predictive is more efficient but drops calls if misconfigured.
- **Auto-Dial Level:** Start at 1.0 (one call per available agent). Increase after you've verified everything works.
- **Local Call Time:** Set your allowed calling window. This affects the [dial hopper](/blog/vicidial-dial-hopper-guide/).
- **Answering Machine Detection:** Start with AMD OFF. Turn it on later after you've read the [AMD guide](/blog/vicidial-amd-guide/).

**3. Create a List and Load Leads**

**Lists** → **Add New List**. Set the list to Active and assign it to your campaign. Then load leads via CSV:

The CSV format VICIdial expects:
```
vendor_lead_code,source_id,list_id,phone_code,phone_number,title,first_name,middle_initial,last_name,address1,address2,address3,city,state,province,postal_code,country_code,gender,date_of_birth,alt_phone,email,security_phrase,comments
```

Minimum required fields: list_id, phone_code (1 for US), phone_number. Everything else can be blank.

**4. Create a User Group and Agent Accounts**

**User Groups** → Create a group. **Users** → Create agent accounts assigned to that group. Give agents a phone login extension.

**5. Set Up Agent Phones**

Agents need a SIP phone registered to your Asterisk. Options:
- Hardware SIP phone (Polycom, Yealink)
- Softphone (Zoiper, MicroSIP)
- WebRTC (built into VICIdial 2.14+ with Asterisk 16+)

Register the phone to Asterisk using the SIP credentials you configured. The agent's phone extension must match what's in their VICIdial user settings.

**6. Start Dialing**

Agent logs in at the agent web interface, selects their campaign, enters their phone login. Campaign starts dialing.

---

## Predictive Dialer: The Core Feature

The predictive dialer is why most people choose VICIdial. It's also the feature that causes the most problems when misconfigured.

### How Predictive Dialing Works

The algorithm watches three things:
1. **How many agents are available** (in READY status, waiting for a call)
2. **Average handle time** (how long calls last once connected)
3. **Connect rate** (what percentage of dials actually reach a human)

From these, it calculates how many simultaneous dials to make per available agent (the auto-dial level). If your connect rate is 30% (only 30% of calls get answered), it needs to dial roughly 3.3 numbers per available agent to keep everyone busy.

The auto-dial level adjusts dynamically. When agents are idle, it dials more aggressively. When all agents are on calls, it backs off. The target is zero idle agents with minimal dropped calls (calls that connect but no agent is available).

### The Drop Rate Problem

Federal regulations (FTC Telemarketing Sales Rule) limit abandoned calls to 3% of answered calls. A dropped call is one where a human answers but no agent is available within 2 seconds. If your predictive dialer is too aggressive, you'll exceed this threshold and face fines.

VICIdial tracks drop rate per campaign and can auto-adjust the dial level to stay under the threshold. Configure this in **Campaign Settings**:

- **Dial Level Difference Target:** How far above the current auto-dial level the system can go. Start conservative (0.5).
- **Available Only Ratio Tally:** Set to Y. This means the dialer only counts agents in READY status, not agents on calls. Much safer for compliance.
- **Drop Lockout Time:** How many seconds after the line is answered before the system considers it "dropped." Usually 2 seconds.
- **Adaptive Dropped Percentage:** Target drop rate. Set to 3.0 (3%) to match FTC safe harbor.

### Tuning for Performance

The gap between a well-tuned and poorly-tuned VICIdial campaign is enormous. A well-tuned operation gets agents on calls 45-55 minutes per hour. A poorly-tuned one gets 20-30 minutes.

Key tuning areas:
- **[Auto-dial level](/blog/vicidial-auto-dial-level-tuning/)** — The #1 performance lever
- **[AMD configuration](/blog/vicidial-amd-guide/)** — Answering machine detection saves agent time but has false positives
- **[Lead recycling](/blog/vicidial-lead-recycling/)** — Don't let failed leads die after one attempt
- **[Dial hopper](/blog/vicidial-dial-hopper-guide/)** — Keep it full to prevent dialer stalls
- **Carrier capacity** — Make sure your SIP trunk can handle the concurrent calls your dialer needs

---

## TCPA and Compliance

Running a predictive dialer in the US means dealing with TCPA (Telephone Consumer Protection Act) compliance. VICIdial has built-in features to help, but they only work if you configure them.

### Built-In Compliance Features

- **Local Call Time enforcement** — Only dial leads during allowed hours in their local timezone
- **DNC list management** — Internal and external DNC scrubbing
- **Drop rate limiting** — Auto-adjust dial levels to stay under 3% abandon rate
- **Dial count limits** — Cap attempts per lead
- **Call recording consent** — Play a recording notification at the start of calls (where required by state law)
- **Manual dial options** — For leads that require TCPA opt-in before automated dialing

### What VICIdial Doesn't Do

VICIdial does not:
- Automatically check if a number is a cell phone (you need a scrub service for this)
- Manage TCPA consent records (your CRM should handle this)
- Handle state-specific calling restrictions beyond timezone rules
- Provide legal compliance advice (hire a telecom attorney)

For wireless numbers, you need prior express written consent before using a predictive dialer. VICIdial has no way to verify this — it's your responsibility to only load consented leads into campaigns that use automated dialing.

---

## Scaling VICIdial

### Single Server (1-30 Agents)

One server handles everything. ViciBox on modern hardware (8+ cores, 64GB RAM) can handle 30 agents with recording, AMD, and real-time reporting without breaking a sweat.

### Multi-Server Cluster (30-200 Agents)

Split the workload:
- **Database server:** Dedicated MySQL/MariaDB on fast storage
- **Web server:** Admin interface and agent screens
- **Telephony server(s):** Asterisk, call processing, recording

VICIdial supports automatic agent phone routing across multiple telephony servers. The web server handles the UI while telephony servers handle the actual calls. Our [cluster guide](/blog/vicidial-cluster-guide/) covers the setup.

### Large Scale (200+ Agents)

At this scale, you need:
- MySQL replication for read scaling
- Multiple telephony servers with load distribution
- Dedicated recording storage (NFS or SAN)
- Redundant web servers with load balancing
- Monitoring and alerting (Grafana, Nagios, or similar)

VICIdial clusters running 500+ agents exist in production. They work, but they require dedicated infrastructure teams.

---

## VICIdial vs. The Competition

Quick comparison against the most common alternatives:

### VICIdial vs. Five9

Five9 is the enterprise hosted option. $150-250/seat/month. Beautiful interface. Zero infrastructure management. Great if you have the budget and don't need customization. Our [full comparison is here](/blog/vicidial-vs-five9/).

**Choose VICIdial if:** Cost is a factor, you need custom integrations, you have technical staff.
**Choose Five9 if:** You want zero admin overhead, you need enterprise support SLAs, budget isn't a constraint.

### VICIdial vs. Convoso

Convoso is built for outbound sales. $100-175/seat/month. Great AMD and compliance features. Strong analytics. Our [comparison is here](/blog/vicidial-vs-convoso/).

**Choose VICIdial if:** Your operation is price-sensitive, you want to own your data, you run blended inbound/outbound.
**Choose Convoso if:** You're purely outbound, you need turnkey analytics, you don't want to manage servers.

### VICIdial vs. GoAutoDial

GoAutoDial is a VICIdial fork with a modern UI layer on top. Free and open-source. If VICIdial's admin interface makes you cringe, GoAutoDial puts a cleaner coat of paint on it. Under the hood, it's still VICIdial. Our [migration guide is here](/blog/goautodial-to-vicidial-migration/).

**Choose VICIdial if:** You want direct access to the latest features and community support.
**Choose GoAutoDial if:** The UI matters and you want a friendlier admin experience.

---

## Common Problems and Solutions

The problems people hit most often, in order of frequency:

### "The hopper is empty and agents are sitting idle"

This is the [dial hopper](/blog/vicidial-dial-hopper-guide/) running dry. Causes: leads exhausted, lists inactive, timezone restrictions, lead filter too aggressive, VDHopper cron not running. Check Dialable Leads Count first.

### "Calls connect but there's no audio / one-way audio"

NAT problem 95% of the time. Verify `externip`, `localnet`, and `nat=force_rport,comedia` in sip.conf. Open RTP ports (10000-20000 UDP). Check for SIP ALG on your firewall. Full guide in our [SIP troubleshooting post](/blog/vicidial-sip-troubleshooting/).

### "SIP trunk won't register"

Wrong credentials, firewall blocking port 5060, DNS not resolving, or carrier-side issue. Our [SIP registration guide](/blog/sip-registration-failed-fix/) covers every error code.

### "AMD is killing too many live calls"

VICIdial's AMD classifies human answers as machines at a rate of 5-15% depending on your settings. That's 5-15% of your live leads getting dropped. Either tune the [AMD settings](/blog/vicidial-amd-guide/) or disable AMD and route everything to agents.

### "Reports are slow / admin interface is laggy"

Database needs optimization. On large systems, `vicidial_log` and `vicidial_closer_log` grow to hundreds of millions of rows. Archive old data, optimize tables, and ensure proper indexes. Our [MySQL optimization guide](/blog/vicidial-mysql-optimization/) covers this.

### "Recordings are missing"

Usually a disk space issue or a permissions problem. Check that the recording directory has space and that Asterisk can write to it:

```bash
# Check disk space
df -h /var/spool/asterisk/monitor/

# Check permissions
ls -la /var/spool/asterisk/monitor/
```

---

## The VICIdial Community

VICIdial development is led by Matt Florell at the Vicidial Group. Updates come through SVN (yes, Subversion, not Git). The community lives on the vicidial.org forums, which are active but have a learning curve — long-time users can be terse with questions that have been answered before.

There's also a small but active community on Reddit (r/vicidial) and various Discord/Slack groups.

For paid support, the Vicidial Group offers hosted solutions and support contracts. There are also third-party providers (like us) who specialize in VICIdial infrastructure and optimization.

---

## Is VICIdial Right for You?

**VICIdial is a good fit if:**
- You make or receive 10,000+ calls per month
- You have technical resources (sysadmin or managed service provider)
- Cost savings matter (VICIdial saves 50-80% vs. hosted alternatives at scale)
- You need deep customization (custom reports, CRM integration, workflow automation)
- You want to own your data and infrastructure

**VICIdial is a bad fit if:**
- You have fewer than 5 agents and minimal technical knowledge
- You need a turnkey solution with zero setup
- Nobody on your team can troubleshoot Linux/Asterisk/MySQL issues
- You need a modern, polished user interface
- You're in a heavily regulated industry that requires vendor compliance certifications

If you're somewhere in between — you want the cost savings and control of VICIdial but don't want to deal with the infrastructure — that's exactly where managed VICIdial providers come in.

---

**Want VICIdial's power without the operational headaches?** ViciStack deploys, optimizes, and manages VICIdial on bare-metal infrastructure. We handle the servers, the Asterisk config, the SIP trunks, the [MOS monitoring](/blog/voip-mos-score-guide/), the updates — you focus on making calls. Our standard engagement is $5K ($1K deposit, $4K on completion) and we'll have you dialing on optimized infrastructure within 2 weeks. Most clients see [50% improvement in contact rates](/blog/vicidial-roi-case-study/) from optimization alone. [Get started](/contact/).

---

*Related: [VICIdial Setup Guide](/blog/vicidial-setup-guide/) | [VICIdial Predictive Dialer Settings](/blog/vicidial-predictive-dialer-settings/) | [VICIdial Cost Breakdown 2026](/blog/vicidial-cost-2026/) | [VICIdial AMD Guide](/blog/vicidial-amd-guide/)*

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-complete-guide).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
