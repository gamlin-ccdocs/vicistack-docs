# The Complete VICIdial Setup Guide (2026): From Bare Metal to First Dial in Under 2 Hours

**Last updated: March 2026 | Reading time: ~18 minutes**

You've done the math. [Convoso wants $150/seat/month](/blog/vicidial-vs-convoso/). [Five9 wants even more](/blog/vicidial-vs-five9/). Meanwhile, VICIdial — the same open-source predictive dialer powering 14,000+ installations worldwide — costs exactly zero in licensing fees.

There's just one problem: actually setting it up.

Every guide you'll find on Google right now is either a ViciBox 7 PDF from 2018, a forum thread with 47 conflicting answers, or a blog post that reads like it was translated three times. CentOS 7 — the OS that 90% of those guides reference — hit end-of-life in June 2024. If you follow those instructions in 2026, you're building on a dead foundation.

This guide fixes that. We're covering ViciBox 12.0.2 (the current stable release), AlmaLinux 9 scratch installs, single-server deployments, multi-server clusters, SIP trunk configuration with real carrier examples, WebRTC setup for remote agents, and every common gotcha that'll eat your weekend if you don't know about it in advance.

**Or — and hear us out — you could skip all of this and let ViciStack handle it overnight.** We migrate VICIdial operators to fully optimized, bare-metal infrastructure with [AMD accuracy](/blog/vicidial-amd-guide/) hitting 92–96% out of the box. No compiling Asterisk from source. No debugging one-way audio at 2 AM. But if you want to do it yourself, keep reading. We respect the DIY spirit. We just want you to know there's a better option when you're ready.

---

## What Actually Changed in 2024–2026 (And Why Your Old Guide Is Lying to You)

Let's get this out of the way first, because if you skip this section and follow an outdated tutorial, you're going to waste a solid 6–8 hours before realizing something is fundamentally broken.

**CentOS 7 is dead.** Like, actually dead. EOL June 2024. No more security patches, no more updates. Every "VICIdial installation guide" from striker24x7, every Udemy course, every forum walkthrough that says `yum install centos-release-scl` — that's all historical fiction now.

**ViciBox jumped from v9 to v12.** The version numbers aren't a typo. ViciBox 12.0.2 dropped in January 2025, running on OpenSuSE Leap 15.6 with Asterisk 18, MariaDB 10.11.9, and PHP 8.2. ViciBox 13.0 is already in beta with OpenSuSE 16.0 and SELinux support coming. If you're following a guide that mentions ViciBox 8 or 9, you're reading ancient history.

**Asterisk 18 is now the default.** The jump from Asterisk 13/16 to 18 brought PJSIP support, improved WebRTC handling, and better codec negotiation. Matt Florell officially confirmed full Asterisk 18 support in September 2025. The VICIdial-specific patches now target Asterisk 18 exclusively for new installs.

**PHP 8.2 is standard.** VICIdial code older than 4 years will throw deprecation warnings or outright break on PHP 8.x. The `mysql_*` functions your old installation scripts reference? Gone since PHP 7.0.

**The SVN trunk is at revision 3939+**, version 2.14b0.5, database schema 1729. Still hosted at `svn://svn.eflo.net:3690/agc_2-X/trunk` because the VICIdial project has zero plans to move to Git. Some things never change.

Here's what this means in practice: **the only two paths worth taking for a new VICIdial install in 2026 are ViciBox 12.0.2 (recommended) or a scratch install on AlmaLinux 9 / Rocky Linux 9.** Everything else is a waste of your time.

> **Your Old Guide Is Lying to You. We Won't.**
> ViciStack deploys fully optimized VICIdial on Asterisk 18, AlmaLinux 9, with every gotcha pre-solved. [Skip the Setup →](/free-audit/)

---

## Hardware: What You Actually Need (Not What the Forum Told You in 2015)

Let's talk real numbers. The ViciBox 12 documentation finally has proper dimensioning specs, and they're different from what you'll find in old forum posts.

### Single Server (The "I Have 10–25 Agents" Setup)

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| CPU | 4 cores @ 2.0+ GHz | 6+ cores @ 2.0+ GHz |
| RAM | 8 GB | 16 GB ECC |
| Storage | 160 GB SSD | 500 GB RAID1 SSD |
| Network | 1 Gbps | 1 Gbps dedicated |

**SSDs are mandatory in 2026.** Not recommended — mandatory. The ViciBox documentation explicitly states SATA SSD as the minimum. If someone tries to sell you a VICIdial server on spinning rust, they're either stuck in 2016 or they don't care about your agents sitting idle waiting for database queries.

A single Express server realistically handles **15–20 outbound agents** with predictive dialing active, or roughly 50 inbound-only agents under ideal conditions. Once you push past 25 outbound agents, you're playing with fire.

### Multi-Server Cluster (The "I'm Actually Running a Business" Setup)

When you outgrow a single box, VICIdial splits into four roles. For the full deep-dive on [cluster architecture, capacity planning, and every configuration detail](/blog/vicidial-cluster-guide/), see our dedicated cluster guide.

**Database server** — The brain. One per cluster, always. For 150 agents: 8+ cores, 32 GB ECC RAM, NVMe RAID1.

**Telephony/dialer servers** — The lungs. Each one handles about 25 outbound agents with heavy recording and 4:1 dial ratios.

**Web servers** — The face. 2–4 cores, 4–8 GB RAM. SSL cuts capacity roughly in half due to TLS overhead.

**Archive server** — The memory. This is the one place where spinning drives are actually fine.

> **Stop Guessing at Server Specs.**
> ViciStack provisions purpose-built bare metal for your exact agent count and dial ratio. [Get Your Custom Quote →](/pricing/)

---

## Installation Method 1: ViciBox ISO (The Sane Path)

ViciBox is the official pre-built ISO maintained by Kumba (the ViciBox developer). It bundles OpenSuSE Leap 15.6, Asterisk 18, MariaDB, Apache, PHP, and VICIdial into a single bootable image. It's the path of least resistance and the one we recommend for anyone who values their time.

### Download and Boot

Grab ViciBox 12.0.2 from `download.vicidial.com/iso/vicibox/server/`. Two flavors: **Standard** (single disk, hardware RAID, VMs) and **MD** (software RAID1 across two drives). Burn to USB with Rufus or dd, boot it up, and select "Install ViciBox" from the menu.

The installer copies the OS to disk, you log in as root, and it walks you through locale, keyboard, timezone, and root password. Reboot when prompted. Total time: about 10 minutes.

### Pre-VICIdial Configuration (Don't Skip This)

Before you touch VICIdial, nail down your networking. VICIdial's configuration is permanently tied to your server's IP address — changing it later is a painful multi-file surgery.

**Set a static IP:**
```bash
yast lan
```
Select your interface, choose "Statically assigned IP Address," enter the IP/subnet/gateway, and set DNS. Hit ALT-O to apply. Verify with `ping -4 google.com`.

**Set timezone (use the ViciBox command, not yast):**
```bash
vicibox-timezone
```
The regular yast timezone command doesn't update PHP's timezone. Ask me how I know.

**Update the system:**
```bash
zypper ref
zypper up
reboot
```

**Critical warning:** Always `zypper up`, never `zypper dup`. The `dup` command (distribution upgrade) can downgrade MariaDB or break DAHDI compatibility. Multiple forum posts document this destroying production systems.

### Install VICIdial (The One-Command Miracle)

For a single server with ≤20 agents:
```bash
vicibox-express
```

Type Y. Wait. Reboot. That's it. VICIdial is running.

Verify with `screen -ls` — you should see 10–12 screen sessions. Hit `http://<your-server-IP>/vicidial/welcome.php` with default credentials **6666 / 1234** and you're looking at a working dialer.

For a cluster, run `vicibox-install` on each server (database first, then web, then telephony), select which roles to enable, and point non-DB servers at the database server's IP. Same process, just repeated.

### The One Bug You Need to Fix Immediately

ViciBox 12 ships with a MariaDB version that deprecated implicit TIMESTAMP behavior. This can silently break tables. Fix it before you do anything else:

```bash
echo "explicit_defaults_for_timestamp = Off" >> /etc/my.cnf.d/general.cnf
systemctl restart mariadb.service
```

> **ViciBox Gets You Running. ViciStack Gets You Results.**
> We go beyond installation — AMD tuning, DID management, carrier optimization, all included. [See the Difference →](/free-audit/)

---

## Installation Method 2: Scratch Install on AlmaLinux 9 (The Control Freak Path)

Some people need RHEL-family Linux. Some people want to understand every component. Some people just enjoy compiling software from source on a Friday night. We don't judge.

The best option in 2026 is the **carpenox auto-installer** maintained by Chris at CyburDial/Dialer.one. It's the most actively maintained community script and handles AlmaLinux 9 + Rocky Linux 9 with Asterisk 18:

```bash
timedatectl set-timezone America/New_York
yum check-update && yum update -y
yum -y install epel-release && yum update -y
yum install git kernel* --exclude=kernel-debug* -y
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
cd /usr/src
git clone https://github.com/carpenox/vicidial-install-scripts.git
reboot
cd /usr/src/vicidial-install-scripts
chmod +x alma-rocky9-ast18.sh
./alma-rocky9-ast18.sh
```

**SELinux must be disabled.** This is non-negotiable. VICIdial's Perl scripts, Asterisk's file operations, and Apache's config all assume SELinux is off. Every single scratch install guide starts with disabling it.

The script handles dependency installation, Asterisk 18 compilation with VICIdial patches, DAHDI, LAME, Jansson, VICIdial SVN checkout, database setup, crontab configuration, and boot scripts. The five VICIdial-specific Asterisk patches (AMD stats, IAX peer status, SIP peer logging, and two timeout reset patches) are applied automatically.

---

## Post-Install: From "It's Running" to "We're Making Calls"

This is where every other guide on the internet stops. "Congratulations, you installed VICIdial! Here's a screenshot of the login page. Good luck!" Not helpful. Let's actually configure this thing.

### Lock Down the Defaults (Do This First)

VICIdial ships with training wheels that are also security holes:

1. **Change admin password** — Admin → Users → Modify user 6666. The default 6666/1234 credentials are known to literally everyone who has ever Googled "vicidial."
2. **Set MySQL root password** — `mysqladmin -u root password 'SOMETHING_STRONG'`
3. **Change phone registration passwords** — Default is `test`. Yes, really.
4. **Move SSH off port 22** — Every bot on the internet is hammering port 22 right now.

### Setting Up Your SIP Trunk

This is where most DIY installs stall. You need a VoIP carrier to actually make phone calls, and VICIdial's carrier configuration has some non-obvious quirks.

Navigate to Admin → Carriers → Add A New Carrier. For an IP-authenticated trunk (most business carriers):

**Account Entry:**
```
[your-carrier]
disallow=all
allow=ulaw
allow=g729
type=peer
insecure=port,invite
host=sip.yourcarrier.com
dtmfmode=rfc2833
context=trunkinbound
canreinvite=no
```

**Dialplan Entry:**
```
exten => _91NXXNXXXXXX,1,AGI(agi://127.0.0.1:4577/call_log)
exten => _91NXXNXXXXXX,2,Dial(${CARRIER}/${EXTEN:1},60,tTor)
exten => _91NXXNXXXXXX,3,Hangup
```

**Global String:** `CARRIER=SIP/your-carrier`

The `9` prefix is a dial string convention — when your campaign uses dial prefix `9`, VICIdial prepends it to each number, and the dialplan strips it before sending to the carrier. Verify your trunk with:

```bash
asterisk -rx "sip show registry"
asterisk -rx "sip show peers"
```

**Pro tip from running 100+ VICIdial centers:** [STIR/SHAKEN attestation](/blog/stir-shaken-vicidial-guide/) matters enormously in 2026. You need A-level attestation, which requires your DIDs and termination on the same carrier. The dual-stack approach gives you redundancy while maintaining A-level on both.

> **Carrier Config Is Where DIY Installs Go to Die.**
> Wrong STIR/SHAKEN setup = "Scam Likely" within a week. ViciStack configures your carriers correctly from day one. [Get It Right →](/free-audit/)

### Creating Your First Campaign

Admin → Campaigns → Add a New Campaign. The critical setting is **Dial Method:**

- **RATIO** — Fixed calls per agent (e.g., 2.0 = two simultaneous calls per idle agent). Simple, predictable, good for small teams.
- **ADAPT_HARD_LIMIT** — Predictive dialing with a hard ceiling on drop rate. Set this to 3% for TCPA compliance. This is what most outbound operations should use.
- **ADAPT_TAPERED** — More aggressive early, more conservative as the day goes on. Good for experienced operations that understand the tradeoffs.
- **MANUAL** — Agent clicks to dial. For compliance-heavy environments or testing.

Key settings to get right from the start: **Hopper Level** (100–200 leads pre-loaded), **Dial Timeout** (26–30 seconds — carrier-dependent), **Available Only Tally = Y** (only dial when agents are actually idle), and **Auto Dial Level** (start at 1.5 for adaptive modes and adjust based on performance). For the complete breakdown of [every dialer setting that matters](/blog/vicidial-predictive-dialer-settings/), see our dedicated guide.

### Loading Leads

Lists → Add A New List → assign to your campaign → Lists → Load New Leads. Upload a CSV with at minimum: `phone_number`, `first_name`, `last_name`, `state`. Always test with a small batch first. VICIdial's lead loader is powerful but unforgiving with formatting issues.

### Agent Phone Setup

Two options in 2026:

**SIP Softphone (MicroSIP, Zoiper, X-Lite):** Create a phone in Admin → Phones with an extension (e.g., 1001), server IP, and registration password. Agent configures their softphone with these credentials. Works, but requires software on every agent machine.

**WebRTC/ViciPhone (the modern way):** Requires SSL/TLS on your web server and port 8089 open. Set up with `vicibox-ssl` on ViciBox, or certbot on scratch installs. Enable WebRTC phone templates in Admin, set phones to "As Webphone = Y," and agents get a browser-based phone built right into the agent interface. No software installs, works from anywhere. This is how most remote operations run in 2026.

---

## Multi-Server Clustering: The Rules Nobody Writes Down

Once you grow past 20–25 outbound agents, you need a [cluster](/blog/vicidial-cluster-guide/). Here are the rules that will save you from the forum's graveyard of broken cluster posts:

**Rule 1: One adaptive process, one server.** The `AST_VDadapt` process (keepalive 5) manages the predictive algorithm. It runs on **exactly one server** in the entire cluster. Running it on two servers causes dial-level conflicts that look like random drops. Same goes for `AST_VDauto_dial_FILL` (keepalive 7).

**Rule 2: Same LAN, no routers.** All cluster servers must be on the same local network with sub-1ms latency. A router between your database and dialer servers adds enough latency to break agent sessions. Use IAX2 (not SIP) for inter-server trunks.

**Rule 3: NTP from one source.** All servers sync their clocks to the database server or one designated NTP source. Independent NTP sync to external servers causes clock drift that breaks agent sessions, disconnects calls, and corrupts reporting.

**Rule 4: Know your ceiling.** VICIdial's MEMORY tables are single-threaded. One cluster maxes out around 450–500 agents. Plan your growth accordingly.

### When to Add What

| You Hit... | You Add... |
|------------|-----------|
| 20 outbound agents | Split DB from telephony |
| 25 more agents | Second dialer server |
| 50+ agents | Dedicated DB server |
| 70+ agents | Dedicated web server |
| 150+ agents | Slave database for reporting |
| 450+ agents | Second cluster |

> **Rule #1 of Clustering: Don't Learn It the Hard Way.**
> We've built 100+ clusters. Let our scars save your weekend. [Talk to a Cluster Expert →](/free-audit/)

---

## The Troubleshooting Hall of Fame

These are the issues that fill the VICIdial forum's 13,400+ support threads. Learn from others' pain:

**No audio / one-way audio** — 80% chance it's the firewall blocking UDP 10000–20000 (RTP ports). 15% chance it's a missing `externip` in sip.conf. 5% chance it's SIP ALG on a NAT router. Temporarily disable the firewall and test. If audio works, it's the firewall.

**"No available sessions"** — Conference extensions aren't populated for your server IP. Admin → Conferences → Show VICIDIAL Conferences. Each server needs its own conference range.

**Recordings missing** — Check the full pipeline: Is SOX installed? Is campaign recording set to ALLCALLS? Are the cron jobs running? Check `/var/spool/asterisk/monitor/` for raw files. The user-level recording setting can silently override the campaign setting — check both.

**Database schema mismatch warning** — You updated the SVN code but forgot the database. Run:
```bash
mysql -p -f --database=asterisk < /usr/src/astguiclient/trunk/extras/upgrade_2.14.sql
```

---

## The Honest Truth About DIY vs. Managed VICIdial

Look, we wrote this entire guide because we believe in transparency. VICIdial is incredible software. It's free, it's powerful, and in the right hands, it outperforms dialers that [cost 10x more](/blog/vicidial-cost-2026/).

But "the right hands" is doing a lot of heavy lifting in that sentence.

Running VICIdial yourself means you're the sysadmin, the DBA, the telephony engineer, the security auditor, and the carrier relationship manager. When Asterisk crashes at 9 AM on a Monday and 50 agents are sitting idle, you're the one in the terminal. When your [DIDs get flagged](/blog/vicidial-did-management/) as spam and your connect rates drop 40%, you're the one calling carriers.

**ViciStack exists because we've done this 200+ times.** We've built and sold over 200 call centers. We've hired 10,000+ agents. We've spent 15+ years learning every VICIdial quirk, every Asterisk gotcha, every carrier optimization that moves the needle.

Here's what we deliver that this guide can't:

- **AMD accuracy of 92–96%** (vs. the 80–85% most self-managed installs achieve). That delta means 7–16% more live conversations per hour.
- **Overnight migration** — Your entire VICIdial environment, moved to our optimized bare-metal infrastructure while your agents sleep.
- **[STIR/SHAKEN A-level attestation](/blog/stir-shaken-vicidial-guide/)** configured correctly from day one.
- **DID reputation management** — We rotate and monitor your numbers so "Spam Likely" doesn't eat your connect rates.

> **You Read the Whole Guide. Respect.**
> Now imagine skipping all of it and making calls tomorrow instead. That's ViciStack. [Get Your Free Proof of Concept →](/free-audit/)

---

## Essential Resources (Bookmark These)

- **ViciBox 12 Documentation:** docs.vicibox.com — Hardware specs, installation phases, networking, firewall
- **VICIdial Forum:** forum.vicidial.org — 13,400+ topics. Search before posting. Matt Florell (mflorell) and William Conley (williamconley) are the most authoritative voices
- **VICIdial SVN:** svn://svn.eflo.net:3690/agc_2-X/trunk — The source code
- **Carpenox Install Scripts:** github.com/carpenox/vicidial-install-scripts — Best maintained auto-installer for Alma/Rocky
- **VICIdial Manager Manual:** Amazon, $45–65 — Matt Florell's comprehensive reference
- **ViciStack:** vicistack.com — When you're done doing it yourself

---

*This guide is maintained by ViciStack and updated as the VICIdial ecosystem evolves. Last verified against ViciBox 12.0.2 and VICIdial SVN trunk 2.14b0.5, March 2026. Found something outdated? [Let us know.](/free-audit/)*

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-setup-guide).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/)
- [ViciStack Pricing](https://vicistack.com/pricing/)
- [All ViciStack Guides](https://vicistack.com/blog/)
