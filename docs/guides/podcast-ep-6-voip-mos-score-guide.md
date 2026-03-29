# Podcast Ep. 6: VoIP MOS Score: What It Means and How to Fix Bad Call Quality

**Episode 6 of [ViciStack Call Center Tech](/blog/contact-rate-optimization/)** -- the VoIP call quality episode for people who are tired of agents saying "the phones sound bad."

MOS scores, jitter, packet loss, latency -- these terms get thrown around a lot, but most call center operators don't know what they actually mean or how to fix them. This episode translates the ITU standards into plain-English troubleshooting for Asterisk and VICIdial environments.

---

## Listen Now

<audio controls preload="metadata" style="width:100%">
  <source src="https://mcdn.podbean.com/mf/web/voip-mos-score-guide.mp3" type="audio/mpeg">
  Your browser does not support the audio element.
</audio>

**Duration:** 6:03

[Subscribe via RSS](https://feed.podbean.com/jasong7h/feed.xml) to get notified when this episode goes live on Podbean.

---

## Timestamps

- **0:00** -- Intro: Why agents complain about call quality and what MOS means
- **0:45** -- The MOS scale: 1.0 to 5.0 and what each range sounds like
- **1:35** -- ITU-T P.800 methodology and how measurement actually works
- **2:25** -- Measuring VoIP quality: RTCP stats, Asterisk CLI, network tools
- **3:15** -- Jitter, packet loss, and latency thresholds for call centers
- **4:10** -- Codec selection: G.711 vs G.729 and impact on MOS
- **4:55** -- QoS configuration for VICIdial networks
- **5:30** -- Quick wins for improving call quality today
- **5:55** -- Outro

---

## Key Takeaways

1. **MOS 4.0+ is the target for call centers.** Below 3.5, calls sound noticeably degraded. Below 3.0, agents and prospects both have trouble understanding each other. Aim for 4.0-4.3 on every trunk.
2. **Jitter kills call quality faster than latency.** 150ms of consistent latency is workable. 30ms of jitter is not. Jitter buffers help, but only if you configure them -- Asterisk defaults are often too small for high-volume environments.
3. **G.711 gives better quality but uses more bandwidth.** For local trunks and LAN agents, always use G.711 (ulaw/alaw). G.729 saves bandwidth for remote/WAN connections but drops your MOS ceiling by 0.3-0.5 points.
4. **QoS tagging is pointless without end-to-end enforcement.** DSCP marking on your VICIdial server does nothing if your ISP or the carrier's network ignores it. Verify QoS is honored at every hop.
5. **The fastest fix is usually the network, not the software.** Dedicated VLAN for voice traffic, wired (not Wi-Fi) connections for agents, and a separate ISP circuit for SIP trunks solve 80% of call quality complaints.

---

## Read the Full Article

The full written guide includes Asterisk CLI commands, network diagnostic tools, and QoS configuration examples:

[VoIP MOS Score: The Complete Guide](/blog/voip-mos-score-guide/)

---

## Subscribe to the Podcast

Never miss an episode of [ViciStack Call Center Tech](/blog/contact-rate-optimization/):

- [Subscribe via RSS](https://feed.podbean.com/jasong7h/feed.xml)
- [Listen on Podbean](https://jasong7h.podbean.com/)
- [All Episodes](/blog/vicistack-podcast/)

---

## Get a Free Call Center Audit

Agents complaining about call quality? We'll diagnose your VoIP stack and give you a fix list -- free.

[Request Your Free Audit](https://vicistack.com/free-audit/)

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/podcast-ep-6-voip-mos-score-guide).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
