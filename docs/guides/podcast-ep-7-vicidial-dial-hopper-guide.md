# Podcast Ep. 7: VICIdial Dial Hopper: How It Works and Why Yours Is Empty

**Episode 7 of ViciStack Call Center Tech** -- the episode about the error message that makes call center managers lose sleep: "No leads in hopper."

The dial hopper is VICIdial's lead staging mechanism. It pre-loads leads from vicidial_list into a queue so the predictive dialer can fire calls instantly. When it goes empty, your agents sit idle and your operation bleeds money. This episode explains exactly how the hopper works and the 8 most common reasons it goes dry.

---

## Listen Now

<audio controls preload="metadata" style="width:100%">
  <source src="https://mcdn.podbean.com/mf/web/vicidial-dial-hopper-guide.mp3" type="audio/mpeg">
  Your browser does not support the audio element.
</audio>

**Duration:** 7:12

[Subscribe via RSS](https://feed.podbean.com/jasong7h/feed.xml) to get notified when this episode goes live on Podbean.

---

## Timestamps

- **0:00** -- Intro: The "No leads in hopper" error and why it panics managers
- **0:50** -- Hopper architecture: vicidial_list to vicidial_hopper pipeline
- **1:45** -- How VDHopper cron job loads leads every minute
- **2:40** -- Diagnosing an empty hopper: checklist of 8 common causes
- **3:35** -- Hopper level settings and how they affect throughput
- **4:30** -- List status, lead filters, and timezone restrictions
- **5:20** -- Campaign settings that silently block leads
- **6:10** -- Optimizing hopper for high-volume operations (500+ seats)
- **6:55** -- Outro

---

## Key Takeaways

1. **The hopper runs on a cron job (VDHopper), not in real time.** It loads leads once per minute by default. If you burn through leads faster than the cron cycle, you'll see empty hopper errors even when you have millions of leads in the list.
2. **Eight things can empty your hopper.** List not set to active, campaign not assigned to the list, timezone restrictions filtering out all leads, lead filter too aggressive, all leads in non-dialable statuses, hopper level set too low, DNC scrub removing everything, or VDHopper cron not running.
3. **Hopper level is the most misunderstood setting.** It controls how many leads to pre-load per agent. Too low (the default) and high-volume campaigns starve. Too high and you waste memory. For 500+ seat operations, set it to 100-200 per agent.
4. **Timezone restrictions are the silent killer.** If all your leads are in a timezone that's outside your configured dial window, the hopper correctly loads zero leads. This catches people every time daylight saving changes hit.
5. **For high-volume operations, optimize the VDHopper query itself.** Add indexes to vicidial_list for the columns used in your lead filters. The default table structure works fine for 100K leads but slows down past 5M.

---

## Read the Full Article

The full written guide includes SQL queries, cron configuration, and troubleshooting flowcharts:

[VICIdial Dial Hopper: The Complete Guide](/blog/vicidial-dial-hopper-guide/)

---

## Subscribe to the Podcast

Never miss an episode of ViciStack Call Center Tech:

- [Subscribe via RSS](https://feed.podbean.com/jasong7h/feed.xml)
- [Listen on Podbean](https://jasong7h.podbean.com/)
- [All Episodes](/blog/vicistack-podcast/)

---

## Get a Free Call Center Audit

Hopper problems killing your throughput? We'll review your campaign settings, list configuration, and cron jobs for free.

[Request Your Free Audit](https://vicistack.com/free-audit/)

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/podcast-ep-7-vicidial-dial-hopper-guide).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
