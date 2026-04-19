# VICIdial Pricing Guide

**Last updated: March 2026 | Reading time: ~25 minutes**

"VICIdial is free." You've read this on every comparison site, every G2 review, every blog post written by someone trying to sell you hosting. And it's technically true — the software is open source under AGPLv2, and you can download the entire codebase from vicidial.org without paying a cent.

But if you've ever tried to run VICIdial in production, you know that the software license is maybe 2% of what you'll spend. The other 98% is hardware, SIP trunking, DIDs, softphone licenses, compliance tools, and — the big one everybody conveniently ignores — the labor to keep the thing running.

I'm going to give you the real numbers. Not the marketing numbers. Not the "starting at" numbers. The actual [total cost of ownership](/blog/vicidial-cost-2026/) at three different scales, with line items for everything you'll pay for.

---

## The Three Deployment Models

### Model 1: Self-Hosted (Your Hardware, Your Headache)

You buy or rent dedicated servers, install VICIbox (the official VICIdial ISO), configure everything yourself, and manage it yourself. This is the cheapest option on paper and the most expensive option when something breaks at 2 AM.

**Best for**: Organizations with 50+ agents AND in-house Linux/VoIP expertise.

### Model 2: VICIhost (Official Hosted Platform)

Vicidial Group (the company behind VICIdial) offers managed hosting through VICIhost. They handle the servers, Asterisk configuration, and basic system administration. You handle campaign configuration and day-to-day operations.

**Best for**: Organizations that want VICIdial specifically but don't have sys admin staff.

### Model 3: Third-Party Managed Hosting

Companies like RockyDialer, OXtrys, DialerKing, and others offer VICIdial hosting with varying levels of managed services. Prices range wildly.

**Best for**: Organizations that need custom modifications or integrations not available through VICIhost.

---

## Line-Item Cost Breakdown

### VICIdial Software License

**Cost: $0.00/month**

It's AGPLv2 open source. Download it, use it, modify it. The only requirement is that if you modify the source code and distribute it, you must share your modifications under the same license.

### Server Hardware / Hosting

This is where the real spending starts.

**Self-Hosted [Bare Metal](/blog/vicidial-setup-guide/):**

For a 50-agent operation, you need at minimum:
- 1x Web/Database server: 8 cores, 32 GB RAM, 1 TB SSD — $150-250/month (dedicated hosting) or $3,000-5,000 one-time (purchased hardware + colocation)
- 1x Asterisk/Telephony server: 8 cores, 16 GB RAM, 500 GB SSD — $120-200/month (dedicated)
- For 100+ agents, add a second telephony server and potentially a separate database server

Realistic monthly hosting costs for bare metal:
| Scale | Servers Needed | Monthly Cost |
|-------|---------------|-------------|
| 10-20 agents | 1 (all-in-one) | $150-200 |
| 20-50 agents | 2 (web + asterisk) | $300-450 |
| 50-100 agents | 3 (web + 2 asterisk) | $450-700 |
| 100-200 agents | 4-5 (web + DB + 3 asterisk) | $700-1,200 |

**Cloud (AWS/GCP/DigitalOcean):**

Cloud hosting costs 2-3x more than bare metal for the same specs because you're paying for on-demand flexibility you probably don't need for a call center that runs predictable hours. A c5.2xlarge on AWS (8 vCPU, 16 GB) runs $247/month on-demand or $152/month reserved 1-year. Multiply by your server count.

Realistic cloud costs:
| Scale | Monthly Cloud Cost |
|-------|-------------------|
| 10-20 agents | $200-350 |
| 50 agents | $500-900 |
| 100 agents | $900-1,500 |
| 200 agents | $1,500-2,500 |

**VICIhost:**

Official VICIhost pricing as of 2026:
- $400/month per server
- $1,000 one-time setup and training fee (includes first month)
- Call minute charges vary by carrier/route

For a 50-agent operation, you'd typically need 2 servers: $800/month.

**Third-Party Managed Hosting:**

Per-seat pricing varies from $20/seat/month to $120/seat/month depending on the provider and included services. At $40/seat average:
| Scale | Monthly Managed Cost |
|-------|---------------------|
| 10 agents | $400 |
| 50 agents | $2,000 |
| 100 agents | $4,000 |
| 200 agents | $8,000 |

### SIP Trunking (Carrier Costs)

This is your biggest recurring cost regardless of deployment model. SIP trunking pays for the actual phone calls.

**[Outbound calling](/blog/vicidial-vs-gohighlevel/) costs:**
- Per-minute pricing: $0.005-0.015/minute for US domestic
- At $0.008/min average (competitive bulk rate)
- A 50-agent center making 300 calls/day at 2.5 minutes average: 37,500 minutes/day
- Monthly: ~825,000 minutes = $6,600/month in trunking costs

**Inbound calling costs:**
- Typically $0.003-0.008/minute
- Lower volume for most outbound-focused operations

**Channel fees:**
- Some carriers charge per concurrent channel ($1-5/channel/month)
- 50 agents need ~75-100 concurrent channels (1.5-2x agent count for predictive dialing)
- $75-500/month for channel access

Realistic SIP trunking costs:
| Scale | Monthly Trunking |
|-------|-----------------|
| 10 agents | $1,200-2,000 |
| 50 agents | $5,000-8,000 |
| 100 agents | $10,000-16,000 |
| 200 agents | $20,000-35,000 |

### DID Numbers (Caller ID)

You need DIDs for outbound caller ID. STIR/SHAKEN attestation requires that you own the numbers you display. With [caller ID reputation monitoring](/blog/vicidial-caller-id-reputation/), you'll want to rotate numbers to avoid "Spam Likely" flags.

- DID cost: $0.50-2.00/number/month
- Minimum for a serious operation: 20-50 DIDs
- Large operations: 100-500 DIDs

Monthly DID costs:
| Scale | DIDs Needed | Monthly Cost |
|-------|-------------|-------------|
| 10 agents | 10-20 | $10-40 |
| 50 agents | 50-100 | $50-200 |
| 100 agents | 100-250 | $100-500 |
| 200 agents | 200-500 | $200-1,000 |

### Softphone Licenses

VICIdial agents connect via SIP softphone. Zoiper, the most commonly used option, requires per-seat licensing for commercial use.

- Zoiper Pro: ~$42/seat one-time (not monthly)
- MicroSIP: Free (Windows only, basic)
- WebRTC (built into VICIdial): Free, but requires HTTPS and proper certificate setup

Many operations use WebRTC now to avoid softphone costs entirely. The trade-off is setup complexity and occasionally higher CPU usage on the agent workstation.

If you go the WebRTC route, here's the minimum you need in your Asterisk PJSIP config:

```ini
; /etc/asterisk/pjsip.conf — WebRTC endpoint template
[webrtc-agent](!)
type=endpoint
transport=transport-wss
context=from-internal
disallow=all
allow=opus
allow=ulaw
dtls_auto_generate_cert=yes
webrtc=yes
use_avpf=yes
media_use_received_transport=yes
rtcp_mux=yes
ice_support=yes
media_encryption=dtls
```

And your Apache config needs WSS proxy support:

```apache
# /etc/httpd/conf.d/vicidial-wss.conf
ProxyPass "/ws" "ws://127.0.0.1:8088/ws"
ProxyPassReverse "/ws" "ws://127.0.0.1:8088/ws"
SSLProxyEngine on
```

This replaces the $42/seat Zoiper cost with zero ongoing cost. We've deployed WebRTC for centers up to 120 agents without issues.

### DNC Scrubbing

If you're doing outbound calling in the US, you need to scrub your lists against the National [Do Not Call](/blog/vicidial-dnc-management/) Registry and state-level registries.

- DNC.com or similar service: $500-2,000/month depending on volume
- Internal DNC management: Included in VICIdial, but you still need to purchase and import the federal/state lists
- Federal DNC list subscription: $75/area code/year or $19,534/year for all area codes

Realistic monthly DNC costs:
| Scale | Monthly DNC |
|-------|------------|
| 10 agents | $200-500 |
| 50 agents | $500-1,500 |
| 100+ agents | $1,500-3,000 |

### STIR/SHAKEN Compliance

As of 2026, STIR/SHAKEN attestation is mandatory for all VoIP calls in the US. Your carrier handles the signing, but you need to ensure proper attestation levels.

- Most carriers include basic STIR/SHAKEN in their service
- Full attestation (A-level) may require additional verification: $50-200 one-time
- Some carriers charge for attestation upgrades: $0-100/month

### Maintenance Labor (The Big One)

This is the cost that every "VICIdial is free" article conveniently omits. Someone has to keep the system running, apply security patches, troubleshoot issues, manage carriers, and configure campaigns.

**Self-hosted with in-house staff:**
- Full-time VICIdial/Linux admin: $55,000-85,000/year ($4,600-7,100/month)
- Part-time contractor: $1,500-4,000/month (10-20 hours at $25-50/hour)
- Overseas contractor (Philippines, India): $800-2,000/month

**Managed hosting:**
- Usually included in the per-seat price
- Custom development or complex issues: $100-200/hour

**No maintenance at all:**
- Works great until it doesn't. Then you pay emergency rates ($200-400/hour) to whoever can help you while your agents sit idle.

---

## Total Cost of Ownership: Three Scenarios

### Scenario 1: 10-Agent Operation (Self-Hosted)

| Line Item | Monthly Cost |
|-----------|-------------|
| VICIdial license | $0 |
| Server hosting (1 server, bare metal) | $180 |
| SIP trunking | $1,500 |
| DIDs (15 numbers) | $22 |
| Softphones (WebRTC) | $0 |
| [DNC scrubbing](/blog/contact-rate-optimization-guide/) | $300 |
| Maintenance (part-time contractor, 5 hrs/month) | $500 |
| **Total** | **$2,502/month** |
| **Per agent** | **$250/agent/month** |

### Scenario 2: 50-Agent Operation (VICIhost)

| Line Item | Monthly Cost |
|-----------|-------------|
| VICIdial license | $0 |
| VICIhost (2 servers) | $800 |
| SIP trunking | $6,600 |
| DIDs (75 numbers) | $112 |
| Softphones (WebRTC) | $0 |
| DNC scrubbing | $1,000 |
| Maintenance (included in VICIhost + part-time for campaigns) | $1,000 |
| **Total** | **$9,512/month** |
| **Per agent** | **$190/agent/month** |

### Scenario 3: 200-Agent Operation (Self-Hosted, In-House Admin)

| Line Item | Monthly Cost |
|-----------|-------------|
| VICIdial license | $0 |
| Server hosting (5 servers, bare metal) | $1,000 |
| SIP trunking | $28,000 |
| DIDs (300 numbers) | $450 |
| Softphones (WebRTC) | $0 |
| DNC scrubbing | $2,500 |
| STIR/SHAKEN premium | $100 |
| Full-time VICIdial admin | $6,000 |
| **Total** | **$38,050/month** |
| **Per agent** | **$190/agent/month** |

---

## VICIdial vs. Proprietary Dialers: The Real Comparison

Here's where VICIdial's value proposition becomes clear — or doesn't.

### Five9 (Cloud)
- 50 agents at $175/agent: $8,750/month (plus minutes and DIDs)
- Total comparable cost: $16,000-20,000/month

### Convoso (Cloud)
- 50 agents at $90-150/agent: $4,500-7,500/month (plus minutes)
- Total comparable cost: $12,000-16,000/month

### VICIdial (Self-Hosted/VICIhost)
- 50 agents: $9,512/month (from Scenario 2 above)

At 50 agents, VICIdial saves you roughly $3,000-10,000/month compared to proprietary alternatives. Over a year, that's $36,000-120,000.

**But here's the catch**: The savings only materialize if you have someone competent managing VICIdial. If you're spending $4,000/month on emergency troubleshooting because your contractor doesn't know what they're doing, the math flips fast.

### Running the Comparison Yourself

If you want to test VICIdial before committing, here's the fastest path to a proof-of-concept:

1. Download VICIbox ISO from vicibox.com
2. Install on a bare-metal server or VM with at minimum 4 cores, 8 GB RAM, 100 GB SSD
3. Get a SIP trunk from Telnyx (pay-as-you-go, no commitment): sign up, add $25 credit, configure a trunk with your server's IP

Your VICIbox install gives you:

```bash
# Check your install is working
asterisk -rx "pjsip show endpoints"
# Should show your carrier endpoint registered

# Verify VICIdial cron processes are running
ps aux | grep AST_ | wc -l
# Should show 8-12 background processes

# Check the web interface
curl -s -o /dev/null -w "%{http_code}" http://localhost/vicidial/welcome.php
# Should return 200
```

From install to first dial: about 4 hours if you know Linux, 8-12 hours if you're learning. The software cost for this proof-of-concept: $0. The server cost: $150/month or less.

### The Breakeven Point

In our experience, VICIdial becomes clearly cost-effective at **30+ agents** with **competent technical management**. Below 30 agents, the per-agent overhead of maintaining the platform (whether through in-house staff or contractors) erodes most of the savings versus a per-seat [hosted dialer](/blog/hosted-vs-self-hosted-dialer-cost/).

At 100+ agents, VICIdial is almost always the cheapest option by a wide margin — you're saving $5,000-15,000/month versus cloud alternatives.

---

## The Costs Everyone Forgets

### Training
New VICIdial admins need 40-80 hours of learning time. New agents need 2-4 hours on the agent interface. Campaign managers need 10-20 hours to learn the admin panel. Budget for lost productivity during this ramp-up.

### Downtime
Self-hosted VICIdial has no SLA unless you build one. If your server goes down during business hours, agents are idle at $15-25/hour each. A 2-hour outage with 50 agents costs $1,500-2,500 in idle labor alone, plus lost sales.

### Compliance Fines
TCPA violations are $500-1,500 per call. A misconfigured DNC filter or [timezone dialing](/blog/vicidial-timezone-dialing-tcpa/) rule can generate six-figure liability in a single day. The risk isn't unique to VICIdial, but proprietary platforms often include compliance guardrails that you have to build yourself on VICIdial.

### Upgrades
VICIdial releases updates via SVN. Applying updates to a production system requires testing, staging, and a maintenance window. Budget 4-8 hours per major update, times your admin's hourly rate.

---

## How to Save Money on VICIdial

### 1. Use WebRTC Instead of Softphones
VICIdial has built-in WebRTC support. Set it up properly (HTTPS, valid SSL cert, SRTP, WSS) and you eliminate softphone licensing entirely. The setup takes a few hours but saves $42/seat permanently.

### 2. Negotiate SIP Trunking Volume Discounts
SIP trunking is your biggest cost. Get quotes from at least three carriers (Telnyx, Bandwidth, VoIP Innovations) and negotiate. Going from $0.012/min to $0.008/min at 500,000 monthly minutes saves $2,000/month.

### 3. Compress Call Recordings
Switch from WAV to MP3 or Opus recording format. A 3-minute call recording drops from 1.5 MB (WAV) to 200 KB (MP3). That's 7x less storage, which means cheaper disks and less frequent rotation.

### 4. Right-Size Your Servers
Don't run a 32-core, 128 GB server for 20 agents. A single 8-core, 16 GB server handles 20-30 agents comfortably. Oversized servers waste money every month.

### 5. Automate Everything
Automated monitoring, automated backups, automated maintenance (see our [database maintenance guide](/blog/vicidial-database-maintenance/)) — every hour you don't spend on routine tasks is an hour your admin can spend on optimization or campaign support.

A basic monitoring cron that catches 90% of problems:

```bash
#!/bin/bash
# /usr/local/bin/vicidial-health-check.sh
# Run every 5 minutes via cron

# Check if Asterisk is running
if ! pgrep -x asterisk > /dev/null; then
 echo "CRITICAL: Asterisk is down on $(hostname)" | mail -s "VICIdial Alert" admin@company.com
 systemctl restart asterisk
fi

# Check disk space (alert at 85%)
DISK=$(df / | tail -1 | awk '{print $5}' | sed 's/%//')
if [ "$DISK" -gt 85 ]; then
 echo "WARNING: Disk at ${DISK}% on $(hostname)" | mail -s "VICIdial Disk Alert" admin@company.com
fi

# Check MySQL
if ! mysqladmin ping -u root -p'PASS' > /dev/null 2>&1; then
 echo "CRITICAL: MySQL is down on $(hostname)" | mail -s "VICIdial Alert" admin@company.com
fi

# Check Apache
if ! systemctl is-active httpd > /dev/null 2>&1; then
 echo "CRITICAL: Apache is down on $(hostname)" | mail -s "VICIdial Alert" admin@company.com
 systemctl restart httpd
fi
```

Cron it:
```
*/5 * * * * /usr/local/bin/vicidial-health-check.sh 2>&1
```

This 30-minute setup catches server crashes, disk-full situations, and service failures [before they](/blog/tcpa-compliance-2026/) become agent-reported outages. The alternative is finding out when your operations manager calls you at 9 AM asking why nobody can dial.

---

## What ViciStack Charges (And Why)

Since we're being transparent about costs: ViciStack offers a $5,000 engagement ($1,000 down, $4,000 on completion) to increase your contact center's conversions by 50% within two weeks. The engagement includes:

- Complete VICIdial configuration audit and optimization
- AMD tuning for your specific carrier and traffic patterns
- Dial level optimization based on your actual data
- Campaign configuration review and adjustment
- Performance baseline and measurement

After the initial engagement, ongoing support is $1,500/month. That replaces the $1,500-7,000/month you'd spend on a dedicated admin, with the difference being we've done this across 100+ deployments and know exactly which settings move the needle.

Is it worth it? If you're a 50-agent center leaving $3,000-10,000/month in efficiency gains on the table because your VICIdial isn't properly tuned — which is almost every center we audit — then the math works out in your favor by the end of month one.

The software is free. The expertise to make it perform is what costs money. Budget accordingly, and don't let "free" fool you into thinking cheap.

---

---

**Related reading:**
- [VICIdial Pricing in 2026: The Real Numbers Nobody Wants to Publish](https://vicistack.com/blog/vicidial-pricing-guide/)
- [STIR/SHAKEN for VICIdial: The Complete 2026 Implementation Guide](https://vicistack.com/blog/stir-shaken-vicidial-guide/)
- [VICIdial AMD Configuration: The Only Guide That Doesn't Waste Your Time](https://vicistack.com/blog/vicidial-amd-guide/)
- [VICIdial Caller ID Reputation Monitoring and Recovery Guide](https://vicistack.com/blog/vicidial-caller-id-reputation/)
## The Hidden Savings: What Most Cost Analyses Miss

Most VICIdial cost comparisons focus on line items. They miss the operational efficiency gains that change the math entirely.

### Dial Level Optimization

A well-tuned VICIdial predictive dialer at dial level 2.5 puts agents on the phone 55 minutes per hour. A poorly tuned one at dial level 1.5 puts them on the phone 35 minutes per hour. That's 57% more productivity for the same seat cost.

At 50 agents earning $15/hour (loaded), 20 minutes of wasted time per agent per hour = $250/hour in idle labor = $2,000/day = $40,000/month. Proper dial [level tuning](/blog/vicidial-auto-dial-level-tuning/) recovers most of that.

### AMD Accuracy

A VICIdial AMD configuration that misclassifies 15% of human answers as machines (sending them to voicemail) is losing 15% of your live conversations. For a center making 15,000 dials per day with a 25% [contact rate](/blog/contact-rate-optimization/), that's 562 lost human contacts per day.

If your conversion rate on human contacts is 10% at $300 per deal, those lost contacts cost $16,860/day in missed revenue. Tuning AMD accuracy from 85% to 95% recovers $5,600/day in revenue.

These operational improvements don't show up in a pricing spreadsheet, but they're the difference between a profitable call center and one that's bleeding money despite "cheap" technology.

If you want to shortcut the learning curve and skip the expensive mistakes, check out what [ViciStack](https://vicistack.com/) offers. Our $5K engagement ($1K down, $4K on results) typically pays for itself within the first month through improved dial efficiency and reduced carrier waste. We've seen centers cut their per-seat costs by 15-25% while simultaneously increasing agent talk time by 30% — and that math works at any scale.

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-pricing-guide).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
