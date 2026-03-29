# Podcast Ep. 4: The Complete VICIdial Setup Guide: From Bare Metal to First Dial in Under 2 Hours

**Episode 4 of [ViciStack Call Center Tech](/blog/contact-rate-optimization/)** -- the step-by-step VICIdial installation guide that every existing tutorial gets wrong.

Most VICIdial setup guides on the internet are 3-5 years out of date. They reference CentOS 7 (dead), ViciBox 10 (old), and skip the SIP trunk configuration that actually matters. This episode walks through the 2026 installation process from ISO to first outbound dial.

---

## Listen Now

<audio controls preload="metadata" style="width:100%">
  <source src="https://mcdn.podbean.com/mf/web/vicidial-setup-guide.mp3" type="audio/mpeg">
  Your browser does not support the audio element.
</audio>

**Duration:** 7:43

[Subscribe via RSS](https://feed.podbean.com/jasong7h/feed.xml) to get notified when this episode goes live on Podbean.

---

## Timestamps

- **0:00** -- Intro: Why every existing VICIdial guide is outdated
- **0:50** -- Choosing your install method: ViciBox vs AlmaLinux 9
- **1:45** -- ViciBox 12.0.2 ISO installation walkthrough
- **2:50** -- AlmaLinux 9 manual installation with SVN scripts
- **3:45** -- Asterisk and SIP trunk configuration
- **4:40** -- Creating your first campaign: dial method, auto-dial, caller ID
- **5:35** -- Agent phone setup: SIP softphones and WebRTC
- **6:30** -- Multi-server clustering rules
- **7:10** -- Common gotchas and troubleshooting
- **7:35** -- Outro

---

## Key Takeaways

1. **ViciBox 12.0.2 is the fastest path.** Download the ISO, burn it, answer 4 questions, and you have a working VICIdial server. AlmaLinux 9 scratch installs are for people who need custom kernel configs or non-standard hardware.
2. **SIP trunk configuration is where most installs stall.** Every carrier has slightly different registration requirements. Get the trunk working in Asterisk CLI before you touch VICIdial's campaign settings.
3. **WebRTC changes the remote agent game.** No more softphone installs, no more NAT headaches for home-based agents. VICIdial's built-in WebRTC support (SVN 3600+) is production-ready in 2026.
4. **Don't skip the firewall rules.** SIP needs UDP 5060, RTP needs UDP 10000-20000, and WebRTC needs TCP 8089. Getting these wrong causes one-way audio [and registration failures](/blog/vicidial-carrier-selection/) that waste hours to debug.
5. **Multi-server clustering has strict rules.** Database server is always separate first. Web and telephony separate second. Don't try to run everything on one box past 30-40 agents.

---

## Read the Full Article

The full written guide includes exact commands, configuration file examples, and carrier-specific SIP trunk templates:

[The Complete VICIdial Setup Guide](/blog/vicidial-setup-guide/)

---

## Subscribe to the Podcast

Never miss an episode of [ViciStack Call Center Tech](/blog/contact-rate-optimization/):

- [Subscribe via RSS](https://feed.podbean.com/jasong7h/feed.xml)
- [Listen on Podbean](https://jasong7h.podbean.com/)
- [All Episodes](/blog/vicistack-podcast/)

---

## Get a Free Call Center Audit

Need help with your VICIdial setup? We've deployed 100+ systems. We'll review your architecture and configuration for free.

[Request Your Free Audit](https://vicistack.com/free-audit/)

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/podcast-ep-4-vicidial-setup-guide).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
