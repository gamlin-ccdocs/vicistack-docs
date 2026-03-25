# VICIdial AMD Configuration: The Only Guide That Doesn't Waste Your Time

**The complete, no-BS guide to Answering Machine Detection in VICIdial — from the algorithm internals to the dialplan code to the iOS 26 extinction event nobody's talking about.**

---

If you've ever watched your agents sit through dead air, listened to a voicemail greeting play into an empty headset, or wondered why your AMD seems to work great on Tuesday and like garbage on Thursday — congratulations, you've experienced the single most frustrating feature in outbound dialing.

VICIdial's Answering Machine Detection has been argued about on forums since 2008. Matt Florell himself — the guy who *built* VICIdial — told an Astricon audience in 2012 that his team "almost never suggests using" stock AMD. William Conley at PoundTeam routinely tells clients to skip it entirely. And yet, when you're pushing 600,000 dials a month across 100 contact centers, letting 50% of answered calls waste agent time on voicemail greetings isn't exactly a business strategy.

So here's the deal. After analyzing every forum thread, every Asterisk source file, every carrier interaction, and every real-world deployment we could find — this is the guide. Not the "here's what the settings do" guide. The guide that tells you *what actually happens* when AMD runs, why it breaks, and what to do about it in 2026.

---

## How AMD Actually Works (It's Dumber Than You Think)

Let's kill the mystery. The AMD application in Asterisk (`app_amd.c`) is a two-state finite state machine that processes 20-millisecond audio frames at 8kHz sample rate. That's it. There's no machine learning, no neural network, no AI. It's a silence detector with a word counter bolted on top.

Here's the state machine: it starts in a **SILENCE** state. When it detects audio energy above the `silenceThreshold` (default 256), it transitions to **WORD** state. When audio drops below threshold, it transitions back to **SILENCE**. The algorithm counts these transitions and measures their durations. Then it makes a terminal decision based on whichever threshold gets hit first.

Five possible terminal states, in priority order:

**INITIALSILENCE** — No speech detected within `initialSilence` milliseconds. The line connected but nobody's home. This triggers a MACHINE classification because real humans typically say "hello" within 2 seconds.

**MAXWORDS** — The word count exceeded `maximumNumberOfWords`. If the algorithm counts 4+ distinct voice segments, it assumes it's a voicemail greeting listing off options. MACHINE.

**LONGGREETING** — Total cumulative voice duration exceeded `greeting` milliseconds. Long, continuous speaking patterns match voicemail greetings. MACHINE.

**MAXWORDLENGTH** — A single continuous voice segment exceeded `maximumWordLength` (not exposed in VICIdial's default parameters but exists in the code). This catches those marathon voicemail greetings. MACHINE.

**HUMAN** — None of the above triggered, and after-greeting silence was detected. The algorithm heard a short utterance ("Hello?") followed by silence (the person waiting for a response). HUMAN.

If *none* of these conditions are met within `totalAnalysisTime` (default 5000ms), the result is **NOTSURE** with cause **TOOLONG**. And this is where the fun begins.

> **Your AMD Is Running a 2008 Algorithm in 2026.**
> ViciStack's AI-powered AMD Crusher hits 92-96% accuracy. Same VICIdial, dramatically better detection. [See the Difference →](/free-audit/)

### The Default Parameters vs. What VICIdial Actually Uses

| Parameter | Asterisk Default | VICIdial Typical | What It Does |
|---|---|---|---|
| initialSilence | 2500ms | 2000ms | Max silence before speech |
| greeting | 1500ms | 2000ms | Max cumulative voice duration |
| afterGreetingSilence | 800ms | 1000ms | Silence after greeting = HUMAN |
| totalAnalysisTime | 5000ms | 5000ms | Hard timeout for analysis |
| minimumWordLength | 100ms | 120ms | Min voice segment to count as "word" |
| betweenWordsSilence | 50ms | 50ms | Min silence between words |
| maximumNumberOfWords | 2 | 4 | Max words before MACHINE |
| silenceThreshold | 256 | 256 | Audio energy threshold |

VICIdial bumps `greeting` to 2000ms (catching longer voicemail greetings) and `maximumNumberOfWords` to 4 (reducing false positives on humans who say "Hello, this is Jason" — that's 4 words, which would trip the default of 2).

These defaults came from extensive community calibration. As one forum user noted, they ran 10,000+ calls to arrive at the current default values. But here's the catch everyone misses: these parameters were calibrated on Asterisk 1.4/1.8 era calling patterns. Voicemail greetings in 2026 are different. Carrier behavior is different. The entire audio landscape has shifted.

---

## The Dialplan: What Actually Happens When a Call Connects

When VICIdial dials a number with AMD enabled, the call routes to extension 8369 (or 8373 for survey/press-1 campaigns). Here's what the dialplan looks like on Asterisk 13.20+ — annotated with what's actually happening at each step:

```
exten => 8369,1,AGI(agi://127.0.0.1:4577/call_log)
exten => 8369,n,Playback(sip-silence)
exten => 8369,n,AMD(2000,2000,1000,5000,120,50,4,256)
exten => 8369,n,AGI(VD_amd.agi,${EXTEN})
exten => 8369,n,AGI(agi-VDAD_ALL_outbound.agi,NORMAL-----LB-----${CONNECTEDLINE(name)})
exten => 8369,n,Hangup()
```

**Priority 1 — `call_log` AGI:** FastAGI call on port 4577. Logs the call initiation to `vicidial_log`, sets channel variables like CAMPCUST and campaign metadata. On Asterisk 13+, this *must* run before any audio operations. If you have this after `sip-silence` (which was the Asterisk 11 order), your call logging will silently break. This is the number one mistake operators make when upgrading from Asterisk 11 to 13.

**Priority 2 — `Playback(sip-silence)`:** This is the single most important line in the AMD dialplan and the one nobody understands. It plays a silent audio file (typically 250ms of silence in GSM format) to "prime" the audio channel. Without this line, 40%+ of your calls will return NOAUDIODATA — meaning AMD received literally zero audio frames to analyze. Some operators with problematic carriers run this line *twice*. If you're seeing high NOAUDIODATA rates, doubling this Playback is your first move.

**Priority 3 — `AMD()`:** The actual AMD analysis. This line *blocks dialplan execution* for up to 5 seconds (the `totalAnalysisTime` parameter). During this time, the person on the other end hears dead air. AMD sets two channel variables: `AMDSTATUS` (HUMAN, MACHINE, NOTSURE, or HANGUP) and `AMDCAUSE` (the specific reason).

**Priority 4 — `VD_amd.agi`:** The routing brain. This Perl script reads `AMDSTATUS` and `AMDCAUSE`, queries the campaign's `cpd_amd_action` setting, checks the `amd_agent_route_options` Settings Container if enabled, and routes accordingly. MACHINE calls get dispositioned (AA), sent to voicemail drop (AM→AL), or transferred to an ingroup/call menu. HUMAN calls continue to the next line.

**Priority 5 — `agi-VDAD_ALL_outbound.agi`:** Routes the human-classified call to an available agent via load-balanced server assignment. This is where the person who's been sitting in dead air for 2-5 seconds finally hears a human voice. If they haven't already hung up.

For comparison, extension 8368 (no AMD) skips the AMD() and VD_amd.agi lines entirely — calls go straight from sip-silence to agent routing.

---

## The NOTSURE Problem (And Why AMD Agent Route Options Changed Everything)

Here's the dirty secret of VICIdial AMD: the default behavior routes both HUMAN *and* NOTSURE calls to agents. That means when AMD can't make a decision within 5 seconds (TOOLONG), the call goes to an agent anyway — after the customer has already endured 5+ seconds of dead air. When AMD receives no audio data from the carrier (NOAUDIODATA), it also routes to agents — who then get a silent channel with nobody on the other end.

In SVN revision 3200+, VICIdial added **AMD Agent Route Options** — a Settings Container that lets you define exactly which `AMDSTATUS,AMDCAUSE` combinations route to agents. This is genuinely the most important AMD feature added in years, and most operators don't know it exists.

The Settings Container lives at the campaign level. Each line is a comma-separated pair:

**Conservative setup (only definitive humans):**
```
HUMAN,HUMAN
```

**Moderate setup (humans + ambiguous calls, block dead channels):**
```
HUMAN,HUMAN
NOTSURE,TOOLONG
```

**Community consensus minimum — block NOAUDIODATA at bare minimum:**
```
HUMAN,HUMAN
NOTSURE,TOOLONG
NOTSURE,INITIALSILENCE
NOTSURE,MAXWORDS
```

Striker24x7's tutorial specifically recommends blocking NOAUDIODATA, and further suggests blocking TOOLONG as well. The logic is sound: TOOLONG calls have already consumed 5+ seconds of dead air. Routing them to agents creates a terrible customer experience. Yes, you'll lose some legitimate humans who happen to have unusual greeting patterns. But those humans were already irritated by 5 seconds of silence anyway.

> **How Many Live Humans Are You Hanging Up On?**
> If you haven't configured AMD Agent Route Options, the answer is "way too many." [Get Your Free AMD Audit →](/free-audit/)

---

## The Asterisk Version War: Pick Your Poison

This is the part nobody wants to hear. The Asterisk version you're running fundamentally determines how well AMD works, and there is no version that does everything well.

**Asterisk 11 (the golden era, now EOL):** AMD works at 70-80% accuracy with the community-tuned defaults. Multiple operators confirmed: "100 calls, 1 or 2 answering machines reach agents." The problem? Asterisk 11 is end-of-life, has known security vulnerabilities, no WebRTC support, and — according to Kumba — "will randomly crash/hang/go-zombie 3-4 times a day on a high load system."

**Asterisk 13.0–13.17 (the broken years):** AMD is *fundamentally non-functional*. The VICIdial ASTERISK_13.txt documentation explicitly states: "NOTE: Asterisk 13.20 and higher has a functioning version of the AMD app, unlike earlier Asterisk 13 versions." GOautodial 4 shipped with Asterisk 13.17.2 and AMD was broken out of the box. If you're running anything below 13.20, your AMD is returning TOOLONG on the majority of calls. Full stop.

**Asterisk 13.20+ (the compromise):** AMD works again but noticeably worse than Asterisk 11. Forum user bbakirtas: "100 calls 1 or 2 AA with Asterisk 11, same settings on Asterisk 13 = 10 AA." Another operator: "When I switch back to Asterisk 11 it's almost cut in half." The SIP stack refactoring in Asterisk 13 introduced timing changes that affect AMD's audio stream handling. You can mitigate with AMD Agent Route Options, but you can't fix the underlying accuracy regression.

**Asterisk 18 (the current):** The version VICIdial is actively developing for. Ships with AudioSocket support for external AMD integration, PJSIP replacing chan_sip, and ConfBridge replacing MeetMe. The AMD algorithm itself hasn't changed — same `app_amd.c` code — but the improved SIP handling and modern audio stack may provide marginal improvements. More importantly, AudioSocket enables streaming call audio to external AI AMD services, which is where the real future lies.

**The hybrid approach (what smart operators do):** Run AMD on Asterisk 11 telephony servers that only handle dialing and AMD analysis. Run agents on Asterisk 13/18 WebRTC servers for the modern agent interface. VICIdial's [cluster architecture](/blog/vicidial-cluster-guide/) supports this natively. One operator shared their setup: 5 telephony servers running Asterisk 11/13 for AMD, separate agent-only servers for WebRTC. All connected via IAX to the same database cluster.

> **Still Running AMD on Asterisk 13.17? Yikes.**
> ViciStack deploys on Asterisk 18 with every AMD optimization in this guide baked in. [Talk to Us →](/free-audit/)

---

## The AMDMINLEN Feature: Saving Your DIDs (SVN 3873)

In September 2024, Matt Florell added a small but critical feature to `VD_amd.agi` that addresses one of the most expensive AMD side effects: carrier Short Duration Percent (SDP) penalties.

When AMD correctly identifies a voicemail and hangs up, the call duration is often 2-4 seconds. Carriers track SDP as a quality metric. Too many short-duration calls and your carrier starts penalizing you — degraded routing, higher per-minute rates, or outright termination. Your [DIDs get flagged](/blog/vicidial-did-management/), their reputation tanks, and your connect rates crater.

The fix is a single dialplan variable set *before* the Dial line in your carrier entry (not in `extensions.conf`):

```
exten => _8567.,1,AGI(agi://127.0.0.1:4577/call_log)
exten => _8567.,n,Set(__AMDMINLEN=7)
exten => _8567.,n,Dial(SIP/carrier/${EXTEN:4},,tToR)
exten => _8567.,n,Hangup
```

`AMDMINLEN=7` ensures `VD_amd.agi` keeps the call alive for a minimum of 7 seconds before hanging up, even if AMD already determined it's a machine. The call doesn't go to an agent — it just stays connected silently until the minimum is reached. This eliminates the short-duration penalty without affecting agent productivity.

William Conley at PoundTeam took a different approach: "We skipped straight to 'all calls answered minimum 7 seconds' via dialplan modification." Same principle, different implementation. Carpenox confirmed it's been working great for extending DID lifespan and eliminating SDP penalties.

If you're running AMD and *not* using AMDMINLEN, you are actively burning your DIDs. This is non-optional in 2026.

> **Your DIDs Are Dying and AMD Is Pulling the Trigger.**
> AMDMINLEN is non-negotiable in 2026. ViciStack configures it automatically across every carrier. [Protect Your Numbers →](/free-audit/)

---

## iOS 26: The AMD Extinction Event

Here's the part that should terrify every outbound operator reading this.

In September 2025, Apple released iOS 26 with system-level call screening. When enabled, calls from unknown numbers are automatically answered by iOS itself, which plays a prompt: "Please state your name and reason for calling." The caller's response is transcribed live on the lock screen. The iPhone owner — who never heard it ring — reads the transcription and decides whether to pick up.

Think about what this means for AMD.

The call connects. The carrier sends a 200 OK. Asterisk sees an answered call. AMD starts analyzing. But instead of hearing a human say "Hello?" or a voicemail greeting, AMD hears Apple's screening prompt — which is a pre-recorded female voice saying a full sentence. To AMD's two-state finite state machine, this looks exactly like a voicemail greeting: a long voice segment, potentially multiple words, followed by silence while the screening system waits for the caller to speak.

AMD classifies it as MACHINE. Hangs up. The iPhone owner never knew you called.

Or AMD classifies it as NOTSURE/TOOLONG because the screening interaction exceeds `totalAnalysisTime`. Call goes to an agent. Agent hears "Please state your name and reason for calling." Agent is confused. Customer sees the transcription of an agent saying "Uh... hello?" and ignores the call.

Pure CallerID ran extensive lab testing across AT&T, Verizon, and T-Mobile. Their findings: **UNKNOWN is the default state.** Any caller not in Contacts, without a recent inbound interaction (roughly 120-minute bypass window), or surfaced by Siri Suggestions gets flagged as UNKNOWN and screened. CNAM, [STIR/SHAKEN attestation](/blog/stir-shaken-vicidial-guide/), and call branding do not change UNKNOWN status. Read that again.

With iPhone holding 59% US market share and iOS adoption historically reaching 70%+ within 6 months, this isn't a future problem. This is a *now* problem. Traditional AMD — the dumb silence-detector we've been nursing along since 2008 — cannot distinguish between an iOS screening prompt, a voicemail greeting, and a human saying hello.

The implications for VICIdial operators are existential. Contact rates will decline. AMD accuracy will degrade against iPhones specifically. The percentage of calls that "succeed" through AMD will increasingly be a selection bias: only calls to Android users, older phones, and landlines will behave the way AMD expects.

> **iOS 26 Changed Everything. Your AMD Doesn't Know That Yet.**
> Our AI AMD distinguishes humans, voicemails, AND iOS screening prompts. Stock AMD can't. [See Our AMD Accuracy →](/pricing/)

---

## When to Use AMD (And When to Kill It)

After everything above, here's the honest assessment.

**AMD makes sense when:**
- Your voicemail hit rate exceeds 40% (common in residential B2C)
- You have 20+ agents and the productivity math works (see below)
- You can tolerate 20-30% false positive rate (legitimate humans hung up on)
- You're running AMDMINLEN to protect DIDs
- You have AMD Agent Route Options configured to block NOAUDIODATA and TOOLONG
- Your campaign's FCC abandonment rate has headroom below 3%

**AMD doesn't make sense when:**
- You're calling cell-heavy lists where iOS 26 screening is prevalent
- Your agents can identify voicemails in under 1 second (train them to disposition quickly)
- You're running fewer than 10 agents (the dead-air hangups hurt more than voicemail time)
- Your carrier is generating high FAS (carrier answers before human does)
- You're in a regulated vertical where the 3% abandonment rule is being actively enforced

**The dial timeout alternative:** Set your campaign dial timeout to 22-24 seconds. Most voicemail systems answer at 24-25 seconds (4-5 rings). By hanging up before the voicemail picks up, you avoid the entire AMD problem. You'll miss humans who take a long time to answer, but you eliminate false positives, dead air, and FCC abandonment risk. Adjust the timeout based on your carrier's post-dial delay.

**The press-1 alternative:** Survey campaigns (extension 8366/8373) play an audio message first. Humans hear the message and press 1 to connect to an agent. Voicemail systems don't press buttons. This elegantly sidesteps AMD entirely — the human self-selects. The downside: regulatory complexity (FCC requires prior consent for pre-recorded messages to cell phones) and lower connect rates (not everyone presses 1 even if interested).

---

## The Economic Math: When AMD Pays for Itself

Let's do the math that nobody else publishes.

**Assumptions:** 50 agents, $14/hour fully loaded, 8-hour shift. Without AMD, agents handle ~15 calls/hour, 50% are voicemails requiring ~20 seconds each to identify and disposition. That's 7.5 voicemails × 20 seconds = 150 seconds per hour per agent wasted on voicemails. Across 50 agents over 8 hours: **16,667 minutes of wasted agent time per day.** At $14/hour, that's roughly **$3,889/day in wasted payroll.**

**With AMD at 80% accuracy:** AMD catches 80% of those voicemails (6 of 7.5 per hour). Agent voicemail time drops to 30 seconds/hour. Savings: ~**$3,111/day.** But AMD adds 2-3 seconds of dead air to *every* human-answered call. With a ~3% customer hangup rate due to dead air (conservative), you lose approximately 18 live conversations per day. If your conversion rate is 5% and average revenue per conversion is $500, that's **$450/day in lost revenue.**

**Net daily benefit: ~$2,661.** That's roughly $66,500/month — significant at scale. But that 20-30% false positive rate means AMD is also hanging up on 90-135 *actual humans* per day who answered the phone and got disconnected. Those humans are not calling you back. They're filing complaints.

The calculus changes dramatically with AI AMD solutions claiming 95-99% accuracy. At 95% accuracy, you catch nearly all voicemails while dropping false positives to ~5%. The dead-air penalty shrinks because AI solutions can classify faster (sub-1-second vs. 2-5 seconds for stock AMD). The commercial options (AMDY.IO at $79-85/month, VoiceDetect, Twilio's AMD) start making economic sense at roughly 20+ seats.

> **The Math Says You're Losing $2,600/Day.**
> ViciStack customers see 7-16% more live conversations per hour. Same agents, same leads, more revenue. [Prove It Free →](/free-audit/)

---

## The Configuration Checklist: If You're Going to Use AMD, Do It Right

Here's the practical implementation, synthesized from everything above:

**Asterisk version:** 13.38.3-vici minimum. Ideally 18.26.0 for future AI AMD integration. Never run 13.0-13.17.

**Dialplan order (Asterisk 13+):** `call_log` → `sip-silence` → `AMD()` → `VD_amd.agi` → `agi-VDAD_ALL_outbound.agi`. If you upgraded from Asterisk 11 and didn't swap the call_log/sip-silence order, fix it now.

**AMDMINLEN:** Add `Set(__AMDMINLEN=7)` before the Dial line in your carrier dialplan entry. Non-negotiable.

**AMD Agent Route Options:** Enable and create a Settings Container blocking at minimum NOTSURE-NOAUDIODATA. Strongly recommended to also block NOTSURE-TOOLONG.

**WaitForSilence for voicemail drops:** Default `2000,2,30` works for most carriers. The first parameter (2000ms) is the silence duration to detect. The second (2) means detect it twice (once after the greeting, once after the beep). The third (30) is the timeout in seconds. If voicemail drops are cutting off early or not playing after the beep, increase the silence detection to 3000ms or add a third iteration.

**Codec:** Use ulaw (G.711) on carrier trunks whenever possible. G.729 requires CPU-intensive transcoding to slin for AMD analysis, and the transcoding process can introduce artifacts that degrade AMD accuracy. As William Conley bluntly put it: "the default codec is ulaw. the more preferred codec is ulaw. the best codec is ulaw."

**Monitoring:** Enable `$AMD_LOG` in `VD_amd.agi`. Review the AMD Log Report in Admin Utilities weekly. Track your AMDSTATUS distribution: target less than 5% NOTSURE-TOOLONG and less than 2% NOTSURE-NOAUDIODATA. If NOAUDIODATA exceeds 5%, you have a carrier problem — not an AMD problem.

**CPU sizing:** AMD performs DSP processing on every active channel simultaneously. On a dedicated telephony server, plan for roughly 100-150 concurrent AMD channels per modern quad-core CPU. Monitor for non-linear degradation — AMD accuracy tanks when CPU utilization exceeds 70-80% because frame processing starts missing its 20ms timing windows. This is likely what's behind the community reports of "Asterisk 13 works fine until channels start to shoot up."

---

## The Bottom Line

VICIdial's stock AMD is a 2008-era algorithm running in a 2026 world. It was never great — Matt Florell said so himself — and the landscape has shifted dramatically against it. iOS call screening, [STIR/SHAKEN carrier filtering](/blog/stir-shaken-vicidial-guide/), and modern voicemail systems have all changed the audio patterns that AMD was calibrated to detect.

But for high-volume outbound operations where 40-60% of answered calls are voicemails, even a 70-80% accurate AMD provides meaningful economic value. The key is implementing it correctly (AMDMINLEN, Agent Route Options, proper dialplan order, codec hygiene) and knowing when to turn it off.

The future is AI-powered AMD — sub-second classification using real-time audio streaming, capable of distinguishing between humans, voicemail greetings, iOS screening prompts, and carrier IVRs. Asterisk 18's AudioSocket support makes this architecturally feasible today. The operators who move to AI AMD first will have a measurable competitive advantage in connect rates and agent productivity.

Until then, you've got a silence detector with a word counter and a whole lot of configuration to get right.

*At ViciStack, we've tuned AMD across 100+ contact centers processing 600,000+ monthly dials. Our platform optimizes every parameter covered in this guide automatically — from AMDMINLEN configuration to carrier-specific AMD tuning to real-time AMDSTATUS monitoring. If your AMD accuracy is costing you money, [we should talk](/free-audit/).*

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-amd-guide).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
