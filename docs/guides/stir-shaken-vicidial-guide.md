# STIR/SHAKEN for VICIdial: The Complete 2026 Implementation Guide

*Published by ViciStack — the managed VICIdial platform built by operators, for operators.*

---

If you're running a VICIdial call center in 2026 and you think STIR/SHAKEN is just another compliance checkbox you can safely ignore — congratulations, you're about to learn what a 50% drop in answer rates feels like.

The uncomfortable reality that nobody in the VICIdial community is spelling out clearly: **STIR/SHAKEN compliance is necessary but nowhere near sufficient.** Getting your calls signed with A-level attestation is Layer 1 of a 13-layer compliance and reputation stack. Most VICIdial operators stop at Layer 1 and then wonder why their numbers are getting flagged as "Scam Likely" six days into a campaign.

We manage over 100 VICIdial-based contact centers processing roughly 600,000 dials per month across solar, roofing, and home services verticals. We've watched this landscape evolve from "it'll never affect us" to "our calls aren't connecting and we don't know why" to "we need to rebuild everything from the ground up."

This guide is the rebuild manual. Every word is backed by data. No hedging, no filler, no "consult your legal team" cop-outs at the point where it actually gets useful.

Let's get into it.

---

## What STIR/SHAKEN Actually Does (and What It Doesn't)

STIR/SHAKEN is a cryptographic call authentication framework. That's it. STIR (Secure Telephone Identity Revisited) defines the IETF standards (RFC 8224, 8225, 8226) for digitally signing phone calls. SHAKEN (Signature-based Handling of Asserted information using toKENs) is the North American deployment framework built on top of those standards.

When your VICIdial server fires off a SIP INVITE through Asterisk, that call hits your SIP trunk provider. Their Authentication Service (STI-AS) looks at three things: Do they know who you are (KYC)? Did they assign you this phone number? Did the call originate on their network?

If the answer to all three is yes, they generate a PASSporT token — a three-part JSON Web Signature containing your originating number, the destination, a timestamp, and a cryptographic signature — and inject it as a SIP Identity header. That call now carries a digital passport proving its identity.

The terminating carrier's Verification Service (STI-VS) checks the signature, validates the certificate, and slaps a `verstat` parameter on the call: `TN-Validation-Passed`, `TN-Validation-Failed`, or `No-TN-Validation`. That result gets fed to analytics engines, and those engines decide whether your call rings through normally, gets a "Spam Likely" label, or goes straight to voicemail jail.

**The critical misconception costing VICIdial operators money every single day**: STIR/SHAKEN does NOT block calls. It does NOT label calls as spam. It authenticates identity. Period.

The actual blocking and labeling decisions? Those are made by carrier analytics engines — T-Mobile's Scam Shield (powered by First Orion), AT&T's ActiveArmor (powered by Hiya), and Verizon's Call Filter (powered by TNS). These three systems control call reputation for over 200 million US wireless subscribers, and they update their models **every six minutes**.

A-level attestation tells these engines "this caller is who they say they are." It does NOT tell them "this caller is someone you should trust." The difference between those two statements is where most outbound call centers are losing the game.

> **STIR/SHAKEN Is Layer 1 of 13. Are You Stuck There?**
> ViciStack manages the full compliance and reputation stack so your calls actually connect. [Get Your Free Audit →](/free-audit/)

---

## The Three Attestation Levels and Why Only One Matters

**Full Attestation (A):** Your carrier verified your identity, you own the number, and the call started on their network. This is the only level that moves the needle on deliverability. Some devices show a "Verified" checkmark. Analytics engines treat it as a positive (but not conclusive) signal.

**Partial Attestation (B):** Your carrier knows who you are, but can't confirm you own the specific number. This is the default for most call centers using DIDs from one carrier routed through another. Bandwidth says B-attested calls "should not automatically be blocked," but industry data shows B and C attested calls are roughly **three times more likely** to be flagged as robocalls, triggering significantly heavier scrutiny from analytics engines.

**Gateway Attestation (C):** Your carrier doesn't know where the call came from. C-level calls are functionally dead on arrival. Some carriers block all C traffic outright. VICIdial community veterans say it plainly: calls with C attestation end up in voicemail.

**The golden rule:** Buy your DIDs directly from your SIP trunk provider. If the carrier assigned the number and you're sending the call from their network, that's automatic A-level. The moment you introduce numbers from an external source without a signed Letter of Authorization (LOA), you're looking at B-level at best.

> **B-Level Attestation Is Costing You Calls.**
> ViciStack ensures A-level attestation on every DID, every carrier, every call. [Fix Your Attestation →](/free-audit/)

---

## Your Carrier Choice Is the Whole Ballgame

VICIdial doesn't implement STIR/SHAKEN. Your carrier does.

When Asterisk generates a SIP INVITE, it sends a standard, unsigned call to your SIP trunk. The carrier's signing infrastructure handles everything — constructing the PASSporT, signing with their private key, injecting the Identity header. No VICIdial configuration changes. No Asterisk module tweaks. The call leaves your server unsigned and arrives at the carrier's Session Border Controller where the magic happens.

This means your carrier selection is literally the most important decision you'll make for call deliverability. What to look for:

**They must have their own SPC token and certificate.** As of September 18, 2025, the FCC banned third-party STIR/SHAKEN certificates. Every provider must now obtain their own Service Provider Code from iconectiv and sign calls with their own credentials. Before this rule, shady carriers were borrowing certificates from downstream providers and slapping A-level attestation on calls they couldn't properly vet. The Lingo Telecom case — a **$1 million fine** for giving A-level attestation to spoofed deepfake Biden robocalls — was the wake-up call. Ask your carrier for their SPC token registration. If they can't produce it, run.

**They must be on the Robocall Mitigation Database (RMD).** This isn't optional. Under 47 CFR § 64.6305, every intermediate and voice service provider is legally required to **cease accepting traffic** from any provider not listed in the RMD. In August 2025, the FCC removed 1,388 providers from the RMD in a single month — 185 on August 6th and 1,203 on August 25th. Call centers using those carriers saw their operations cease within 48 hours. No workaround. No grace period. Verify your carrier's RMD status and check quarterly.

**They should be a CLEC with their own numbering resources.** A Competitive Local Exchange Carrier that owns its own number blocks can assign DIDs that automatically qualify for A-level attestation. You want the carrier that knows you, assigned the number, and can prove both under audit.

**They need infrastructure built for predictive dialers.** VICIdial's [predictive dialer](/blog/vicidial-predictive-dialer-settings/) needs 2-3x more concurrent channels than active agents. A 50-seat center at peak can easily require 150+ simultaneous channels. Your carrier needs serious capacity, multiple Points of Presence for redundancy, and support for G.711 ulaw codec, RFC 2833 DTMF, and IP-based authentication.

---

## The "Scam Likely" Problem Is Worse Than You Think

Let's talk about what's actually killing your answer rates.

Hiya's 2024 data shows **28% of all unknown calls** are flagged as spam or fraud. YouMail counted **52.5 billion robocalls in 2025**, with unwanted calls surging 15.4% year-over-year. Consumer behavior has shifted accordingly: **48% of consumers now never answer unidentified calls**.

For outbound call centers, the numbers are grim. Realistic daily connection rates sit at **15-25%** on fresh, consented lists with healthy DIDs. The elite operations — running automated DID rotation, real-time reputation monitoring, and branded calling — hit 60-70%. Everyone else? Single digits.

PhoneBurner surveyed 351 sales professionals and found **81% believe their company has lost revenue** from incorrect spam flags. **15% reported losses exceeding $100,000.** And the number that should make every call center owner sit up: **53% reported more than 10 positions eliminated** directly due to incorrect flagging.

But it gets sneakier. Carrier analytics engines don't just flag or not-flag. They operate on a **reputation spectrum** — a continuous scoring system where your numbers can appear "clean" in monitoring tools while the carrier algorithms quietly deprioritize your calls. Your calls might ring fewer times, route directly to voicemail, or display with lower caller ID confidence. You'd never know unless you're tracking answer rates by carrier at the per-DID level, which almost nobody does.

> **48% of Consumers Never Answer Unknown Calls.**
> That number only goes down from here. ViciStack's reputation management keeps your DIDs clean. [See How →](/pricing/)

---

## Your VICIdial Settings Are Feeding the Spam Algorithms

The connection nobody makes explicitly enough: **your VICIdial campaign configuration generates specific call patterns that carrier analytics engines interpret as spam signatures.**

**[AMD](/blog/vicidial-amd-guide/) is the stealth reputation killer.** When Asterisk's AMD module detects a voicemail greeting and disconnects after 2-3 seconds of audio analysis, it generates massive volumes of very short-duration calls. Calls under 30 seconds are the single strongest spam signal in carrier algorithms. With AMD accuracy hovering at 65-80% out of the box, real humans frequently pick up, hear silence, and get hung up on. That generates both consumer complaints AND short-duration call data. Double whammy.

**Your dial method matters more than you think.** RATIO at 3:1 with 10 agents fires 30 simultaneous calls when all agents are ready. If your contact rate is 40%, roughly 12 calls answer, 2 agents are available, and 10 calls get abandoned. That creates a steady stream of short-duration, high-volume calls — textbook robocall signature. ADAPT_HARD_LIMIT creates a sawtooth pattern (aggressive → spike → drop to 1:1 → recover → aggressive again) that ML models are trained to flag. **ADAPT_AVERAGE** produces the smoothest traffic pattern because it maintains drop rate as a running average instead of a hard limit.

**The [DID rotation](/blog/vicidial-did-management/) trap.** Matt Florell himself warns that aggressive rotation "in scenarios where people are dialing to wireless numbers more often can cause the big four to tag the caller ID as 'Scam Likely' prematurely." The industry consensus on calls-per-DID-per-day converges around **50-100 calls**, with 75 as the sweet spot.

**Optimal VICIdial settings for reputation management:**

- Dial method: **ADAPT_AVERAGE** for 35+ agents, RATIO at 1.5-2.0 for smaller teams
- Drop percentage limit: **1-3%** (the FCC cap is 3%, but analytics engines start penalizing well before that)
- Drop action: **IN_GROUP or MESSAGE** — never HANGUP. Playing a 15-20 second safe harbor recording extends call duration past the short-duration penalty threshold
- AMD: Either disabled, or configured to leave a voicemail instead of hanging up immediately
- Dial timeout: **26-30 seconds** — shorter timeouts generate masses of very short call attempts
- Available Only Tally: **Y** (critical for accurate adapt calculations)

Example campaign settings in the VICIdial admin (Campaigns > Campaign Detail):

```
Dial Method:           ADAPT_AVERAGE
Auto Dial Level:       1.0  (let adapt raise it)
Adaptive Drop %:       2.0
Drop Action:           MESSAGE
Drop Exten:            8304  (safe harbor recording)
Dial Timeout:          28
Available Only Tally:  Y
Calls per DID per day: 75   (under DID Rotation settings)
```

> **Your Campaign Settings Are a Spam Signature.**
> RATIO at 3:1 + AMD + aggressive rotation = carrier blacklist. Let us fix your config. [Get Your Free Audit →](/free-audit/)

---

## The Canada-US Cross-Border Problem Nobody Mentions

If you're running VICIdial operations from Canada and calling US numbers — and a lot of operations in Ottawa, Toronto, and Montreal are — you've got an extra layer of complexity.

Canada implemented STIR/SHAKEN under CRTC Decision 2021-123, effective November 30, 2021, with **no exemptions for small carriers** (unlike the US). The governance structure mirrors the US but uses separate authorities: the Canadian Secure Token Governance Authority (CST-GA) and TransUnion/Neustar as the Canadian STI-PA.

The problem: **there is no formal bilateral agreement between Canadian and US authorities for STIR/SHAKEN interoperability.** Each country maintains a separate list of approved Certificate Authorities. When a call signed by a Canadian CA arrives at a US terminating carrier, that carrier checks the certificate against its US-approved CA list. If the Canadian CA isn't recognized, verification fails automatically — your beautiful A-level attestation arrives looking like an unsigned call.

The workaround: since October 2021, both PAs have made their CA lists publicly available, allowing terminating providers to optionally merge foreign lists with their own. TransUnion/Neustar announced that carriers using their Certified Caller platform can verify cross-border calls at no additional cost. But this is a **provider-by-provider decision**, not a guarantee.

For Canadian VICIdial operators calling US numbers, either use a carrier with explicit cross-border STIR/SHAKEN support, or route US-bound calls through a US-based SIP trunk provider with US certificates. Don't gamble that your Canadian attestation will survive the border crossing.

---

## The Complete Compliance Stack: 13 Layers, Not Just 1

No competitor in this space has ever published the complete picture. They all treat STIR/SHAKEN in isolation. The full stack a VICIdial operator needs:

**The Deliverability Chain:**
1. STIR/SHAKEN A-level attestation (carrier-side)
2. CNAM registration ($0.15-$2/number, 15 characters, covers landlines)
3. Free Caller Registry enrollment (freecallerregistry.com — free, submits to all three analytics engines)
4. Individual analytics engine registration (Hiya, First Orion/CallTransparency, TNS/VoiceSpamFeedback)
5. Continuous number reputation monitoring (Numeracle, CallerID Reputation, or similar)
6. Branded calling (optional but increasingly important — CTIA BCID at ~$0.12/call)

**The Legal Compliance Chain:**
7. Consent documentation (TrustedForm or Jornaya, $0.15-$0.50/lead)
8. Federal DNC scrubbing ($1,886/month for all area codes)
9. State DNC scrubbing (11-13 states maintain separate lists)
10. Cell phone identification and TCPA compliance
11. Reassigned Numbers Database queries
12. Litigator scrubbing ($500-$2,000/month)
13. Internal DNC management and call recording (VICIdial handles this natively)

**For a 50-seat center**, minimum viable compliance runs **$3,000-$5,000/month**. The recommended full stack hits **$10,000-$18,000/month** ($200-$360/seat). The largest variable is consent management — TrustedForm alone can exceed $10,000/month at volume.

That sounds expensive until you calculate the alternative. Non-compliance costs a 50-seat center an estimated **$143,000-$768,000 per month** in lost connections, wasted agent wages, accelerated DID burnout, and remediation costs. TCPA class action filings hit **507 in Q1 2025** — a 112% year-over-year increase — with average settlements running **$6.6 million**.

The math isn't even close.

> **$143K-$768K/Month in Lost Revenue. That's the Cost of Ignoring This.**
> ViciStack builds the full 13-layer stack so you don't have to. [See Our Compliance Package →](/pricing/)

---

## What's Coming in 2026-2027 That You Need to Prepare For Now

Three FCC proceedings will reshape this landscape within 18 months:

**Mandatory branded calling (FCC 25-76, October 2025).** The FCC unanimously proposed requiring terminating carriers to display verified caller names alongside A-level attestation. This effectively makes branded calling — showing your business name and logo on the recipient's phone — a regulatory requirement, not just a nice-to-have. Comments closed February 2026. Final rules expected H2 2026.

**The non-IP gap closure (FCC 25-25, April 2025).** The FCC proposed requiring all providers with TDM network segments to implement caller ID authentication or complete IP transition within two years. This addresses the "TDM-in-the-middle" problem where STIR/SHAKEN attestation gets stripped when calls cross legacy SS7 infrastructure — currently affecting roughly **57-62% of signed calls** that arrive at the terminating end without verification data intact.

**Higher RMD fines and state-level action.** New rules establish **$10,000 fines** for false RMD information. Virginia has introduced legislation with a **$5,000-per-call private right of action**. Multiple states are pushing their own caller ID and STIR/SHAKEN mandates, creating a potential patchwork of compliance requirements.

The FCC is also exploring whether verified caller identity should become **a condition of A-level attestation itself** — creating a distinction between basic A (number authenticated) and enhanced A (number + identity verified). The trajectory is clear: unsigned, unbranded calls from unverified callers will increasingly never reach consumers at all.

---

## The VICIdial Implementation Path: What to Do Right Now

Your 90-day playbook, no ambiguity:

**Week 1-2: Carrier Audit**
- Verify your carrier's RMD status
- Confirm they have their own SPC token and certificate (not borrowed — this is now illegal)
- Request attestation level confirmation for your specific DIDs
- If your carrier can't confirm A-level for carrier-assigned DIDs, start evaluating alternatives

**Week 3-4: Number Hygiene**
- Register every outbound DID on FreeCallerRegistry.com
- Set up CNAM for all numbers (15 characters, your business name)
- Run a baseline reputation check across all three analytics engines
- Flag and retire any currently spam-labeled numbers. Don't try to remediate them — replace and move on

**Week 5-8: VICIdial Configuration**
- Switch to ADAPT_AVERAGE if you're running RATIO above 2.0
- Set drop percentage to 2% maximum
- Change Drop Action from HANGUP to MESSAGE or IN_GROUP
- Set dial timeout to 28 seconds minimum
- Either disable AMD or configure voicemail-leave instead of immediate hangup
- Limit calls per DID per day to 75 maximum. Set up rotation accordingly

**Week 9-12: Monitoring and Optimization**
- Implement daily tracking of answer rates by carrier (parse `vicidial_carrier_log` channel field)
- Track short-duration call ratio (answered calls ≤ 6 seconds as % of total). Keep under 15%
- Monitor SIP 603 response codes in carrier logs — a sudden spike means active blocking
- Set up weekly per-DID reputation checks through your monitoring service
- Establish a DID retirement/replacement pipeline for flagged numbers

> **90 Days Is a Long Time. We Can Do It in 48 Hours.**
> ViciStack migrates your entire operation to compliant, optimized infrastructure overnight. [Start Your Migration →](/free-audit/)

---

## The Bottom Line

STIR/SHAKEN is the operating system of modern call deliverability. It's not the whole story — it's Layer 1 of 13 — but without it, nothing else matters. The VICIdial community has been navigating this largely without institutional support, relying on forum threads and Matt Florell's guidance. That's not sustainable when the FCC is removing 1,388 carriers from the network in a month and fining companies millions for improper attestation.

The operators who build the full compliance and reputation management stack now will see answer rates, conversion rates, and revenue per seat that make the investment self-evident. The operators who keep treating STIR/SHAKEN as somebody else's problem will find themselves spending more money to reach fewer people until the economics collapse entirely.

We built ViciStack because this is exactly the problem we solve. We manage the carrier relationships, the reputation monitoring, the compliance stack, and the VICIdial configuration optimization so you can focus on what you're actually good at: running campaigns and closing deals.

The future belongs to operators who treat call deliverability as a competitive advantage, not a regulatory burden.

**Stop guessing. Start building.** [Contact ViciStack →](/free-audit/)

---

*ViciStack is the managed VICIdial platform that handles STIR/SHAKEN compliance, carrier optimization, number reputation management, and dialer configuration — so your calls actually connect.*

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/stir-shaken-vicidial-guide).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
