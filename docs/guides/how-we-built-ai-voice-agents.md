# How We Built an AI Voice Agent from 500 Real Cold Calls

Here is the math that keeps call center operators up at night.

You are running 50 agents on an outbound roofing campaign. Fully loaded cost per agent — wages, seats, dialer licenses, management overhead — lands somewhere between $15 and $25 an hour depending on whether you are running domestic or offshore. Call it $20 on average. That is $1,000 an hour in labor just to keep the floor running. Eight-hour shift, five days a week, and you are burning north of $160,000 a month before a single appointment hits the board.

Now look at what those 50 agents actually produce. VICIdial is pushing 200+ dials per hour per agent through predictive mode. Your contact rate — the percentage of dials that reach a human — sits between 3% and 7% on a typical residential list. That means out of 10,000 dials an hour, you are getting maybe 500 conversations. Out of those 500 conversations, your top closer books appointments at three or four times the rate of your worst agent. But your top closer is one person. They call in sick. They burn out. They quit and take the job across the street for fifty cents more an hour.

This is the fundamental problem with human-powered outbound at scale: **your best performance is not repeatable, and your average performance is expensive.**

We have watched this play out across hundreds of VICIdial deployments. The top 20% of agents generate 60-70% of the results. The bottom 30% barely cover their own cost. And the operational overhead of recruiting, training, and retaining enough warm bodies to maintain volume is a second full-time job that nobody signed up for.

So we decided to build something better. But we did not start where most people start.

Most AI voice products are built on theoretical scripts — someone sits in a conference room, writes what they think a good cold call sounds like, feeds it into a language model, and hopes for the best. That approach produces agents that sound like they are reading a script because they are. Real cold calls are messy. Prospects interrupt. They object in ways no script anticipates. The timing of a pause matters as much as the words.

We started with data. Specifically, we pulled 500+ real cold call recordings from a live roofing appointment-setting campaign running through VICIdial. Five human agents, all working the same lists with the same dialer settings. We transcribed every call. We tagged every disposition. We mapped which agents converted and which ones fumbled, then we went line by line through the conversations that actually booked appointments to figure out *why* they worked.

The insight was simple: **do not invent a new playbook — clone the one that is already winning.**

Record everything. Transcribe everything. Analyze everything. Identify the exact phrases, objection handlers, pacing patterns, and conversation structures that your best human agents use on the calls that convert. Then build an AI agent that replicates those patterns at scale, across every dial, every hour, without calling in sick or asking for a raise.

That is what we built. And it started with those 500 recordings.

---

## Building a Transcription Engine That Actually Works on Phone Audio

Here is the thing nobody tells you about Whisper: it was trained on podcasts, audiobooks, and YouTube videos. Clean audio. Controlled environments. Maybe a little background music.

Phone audio is none of that.

You are dealing with narrow-band codec compression (8kHz G.711, if you are lucky), background noise from job sites, crosstalk where both parties talk over each other, and the particular joy of a prospect on a cell phone standing next to a busy road. Feed that raw into Whisper and you get hallucinated bible verses, repeated phrases, and confident-sounding garbage.

We needed transcription that actually worked on VICIdial call recordings. No cloud APIs, no per-minute costs that scale into absurdity at volume. So we built the whole pipeline locally with faster-whisper.

### Choosing the Right Model

We tested every model worth testing. `base` scored maybe a 5 out of 10 on accuracy — fine as a supplement but completely unusable standalone. `medium` was better but still stumbled on domain vocabulary. `distil-large-v3` offered a decent speed-accuracy tradeoff. But for production transcription where the output actually matters, `large-v3` was the only model that consistently got it right.

### The Configuration That Made It Work

Getting the model right was only half the battle. The real gains came from tuning the transcription parameters:

```python
from faster_whisper import WhisperModel

model = WhisperModel("large-v3", device="cpu", compute_type="int8")

segments, info = model.transcribe(
    processed_audio_path,
    beam_size=10,
    initial_prompt=(
        "This is a phone call about roofing services. "
        "Terms: siding, fascia, eaves, gutters, downspouts, soffit, "
        "flashing, ridge cap, drip edge, ice dam, shingle."
    ),
    word_timestamps=True,
    vad_filter=True,           # Silero VAD
    vad_parameters=dict(
        min_silence_duration_ms=500,
    ),
    temperature=[0.0, 0.2, 0.4, 0.6, 0.8, 1.0],
    condition_on_previous_text=False,
)
```

Every parameter here exists for a reason. `beam_size=10` gives the decoder more room to explore candidate transcriptions instead of greedily committing to the first path. The `initial_prompt` is seeded with domain-specific vocabulary pulled directly from the client's calling script — without it, Whisper hears "soffit" and writes "saw fit" every single time. `vad_filter` engages Silero Voice Activity Detection to skip silence, which prevents the hallucination problem where Whisper invents text to fill quiet gaps. The temperature fallback starts at 0 for deterministic output but steps up automatically when confidence is low. And `condition_on_previous_text=False` stops one bad segment from poisoning everything that follows.

### Preprocessing: Fix the Audio Before Whisper Ever Sees It

Before any of that runs, every recording passes through an ffmpeg preprocessing pipeline. We normalize volume levels, apply noise reduction, run a high-pass filter to cut low-frequency phone hum, and — critically — upsample the audio. Most VICIdial recordings are narrow-band 8kHz. Whisper expects 16kHz. Upsampling with proper interpolation gives the model something closer to what it was trained on. This single step measurably improved word error rates.

Recordings process in parallel across CPU cores. A 5-minute call transcribes in roughly 30-40 seconds on modern hardware, and batch jobs distribute cleanly across available threads.

### Post-Processing: Where the Transcript Becomes Useful

Raw transcripts, even good ones, still need work. Every transcript gets post-processed by Claude, which fixes domain-specific mishearings that Whisper consistently gets wrong ("eve" back to "eave," "gutter" pluralization, brand names). It also corrects proper nouns — addresses, street names, homeowner names — by cross-referencing the matched lead data from the dialer. If the lead record says the prospect lives at 4217 Birchwood Lane, and Whisper wrote "4217 Birch Wood Lane," the post-processor fixes it.

Speaker labeling happens through a combination of script pattern matching and word-level timestamps. The agent's lines follow predictable script patterns — greetings, qualifying questions, rebuttals. By aligning those patterns against the word timestamps, we reliably tag each segment as Agent or Prospect without needing a separate diarization model.

The key insight is simple but easy to miss: phone audio transcription is a pipeline problem, not a model problem. No single model, no matter how large, will give you clean transcripts from dirty audio. You preprocess aggressively, configure carefully, and post-process intelligently. Then it actually works.

---

## Context Injection: Matching Every Recording to Its Lead

A raw transcript is just words on a page. Without context, you cannot tell whether the agent butchered the prospect's address or Whisper did. You cannot tell if this is a first-contact cold call or the eighth redial on a lead that has been dodging callbacks for three weeks. You cannot tell if the agent went off-script or if the script itself is the problem.

We started by reading everything the agents were supposed to be working from. The cold calling script — the exact words they were told to say on every dial. The objection handling playbook — prescribed responses to "I'm not interested," "I already have coverage," "How did you get my number," and every other common pushback. The leads CSV with phone numbers, names, addresses, and cities. The campaign context briefing with company details and service offerings. All of it went into the pipeline before a single recording was processed.

Every recording filename in VICIdial contains the phone number. That phone number is a direct key into the `vicidial_list` table. So step one was automated matching: for each recording, pull the full lead record. Name, street address, city, state, insurance status, every field VICIdial stores. Then pull the call metadata — talk time, wait time, hold time, who hung up (agent or prospect), disposition, timestamps. Then pull the agent ID and cross-reference it against the VICIdial user table to get the actual agent name. Then pull campaign and list metadata for additional context about what this agent was supposed to be selling and to whom.

This context fed directly into transcription. Whisper accepts an `initial_prompt` parameter that biases its vocabulary toward expected words and phrases. We loaded script terminology, objection playbook language, company names, and product terms into that prompt. The model stopped hallucinating "solar panel" when the agent said "supplemental plan."

The matched lead data fed into transcript cleanup. When Whisper output "123 Oak," and the lead record showed "123 Oakwood Drive, Tallahassee," the correction was automatic. Prospect names, street addresses, city names — all verified against the actual lead data and fixed in the final transcript.

But the most valuable context was call history. Some leads in this dataset were called eight or more times before they booked an appointment. A first-contact cold call and a fifth redial to the same prospect are completely different conversations with completely different dynamics. The agent's tone is different. The prospect's patience is different. What counts as "good" objection handling is different. We identified every call as either first-contact or follow-up and tagged it accordingly, because analyzing a redial with first-contact expectations produces garbage insights.

The key insight is simple: a transcript without context is just words. When you know the lead's name, address, insurance status, and complete call history, you can fix transcription errors AND understand what is actually happening in the conversation. VICIdial stores all of this data. Most shops never connect it back to their recordings. That connection is where the analysis starts.

---

## Win/Loss Analysis: What Makes a Call Convert

Every call in the dataset got a tag. **WIN** meant an appointment was booked (disposition code APPTBK). **LOSS** got a sub-reason: *not interested, already has a contractor, bad timing, wrong number, voicemail, gatekeeper block, do-not-call request, hostile, or disconnected*. No ambiguity. No "maybe callback." Every call lands in a bucket so we can count what actually happens on the phones.

Then we went deeper.

We extracted the **exact moment** a prospect commits. Not the general vibe of the call — the precise words and the timestamp. In winning calls, commitment sounds like this: *"Yeah, that'd be fine"* at 1:47. *"Sure, go ahead and put me down"* at 2:12. *"What time works?"* at 1:33. These aren't enthusiastic yeses. They're quiet surrenders. The prospect stops resisting and agrees to a slot. We captured every one of them.

In losing calls, we extracted the **exact moment of disengagement** — the sentence where the prospect checks out and the call is effectively over, even if it keeps going for another thirty seconds.

Then we mapped **what the agent said immediately before each turning point**. The sentence right before the prospect books. The sentence right before the prospect shuts down. This is where the data gets sharp.

Here is a real pattern from the dataset. Prospect says: *"I'm not really interested."* Agent A responds: *"I totally understand — most folks say the same thing until they see what the estimate looks like. It's free either way. Would morning or afternoon work better for you?"* Prospect books. Agent B, same objection, responds: *"Oh, okay. Well, if you change your mind..."* Call over.

**The difference between a booked appointment and a burned lead is often a single sentence.** The agent who recovers from "I'm not interested" with the right words books it. The agent who fumbles it loses that lead permanently.

We calculated **time-to-book** for every winning call — seconds from pickup to confirmed appointment. We tracked how many objections the agent handled before the yes came. We asked: do more objections mean longer calls? Yes. Do they still convert? Surprisingly often. Some of the strongest bookings came after three or four objections. The agent just kept going.

We counted the **number of "no's" a winning agent pushes through** before the prospect converts. Top performers absorbed two to three rejections per booked call. They didn't hear "no" as a stop sign. They heard it as a request for more information.

We categorized objections into **soft no** and **hard no** patterns. A soft no sounds like *"I'm not sure"* or *"Maybe not right now."* A hard no sounds like *"Put me on your do-not-call list"* or a dial tone. The distinction matters because soft no's are recoverable — but only if the agent recognizes them in real time and responds accordingly.

We identified **"save" moments** — calls where the prospect said no, the agent recovered with a specific technique, and the call ended with an appointment. These are the highest-value training examples in the entire dataset. We also identified **"kill" moments** — calls where the agent said something that actively lost a prospect who was otherwise still in play. Talking over the prospect. Jumping to price before building any value. Asking *"Is this a bad time?"* in the first five seconds. These patterns showed up repeatedly.

We mapped the **exact word or phrase that triggers the shift from skeptical to interested**. In this campaign, phrases referencing "free estimate" and "no obligation" consistently preceded the turning point in winning calls. We tracked **competitor mentions** to see which names came up and how often. We logged **pricing expectations** that prospects volunteered. We captured **insurance status distribution** across the call population to understand the lead pool.

All of this was extracted programmatically from transcripts and dispositions already sitting in VICIdial's database. No manual call reviews. No listening to recordings one at a time. Just structured analysis of data the dialer already collected.

The win/loss tags told us *what* happened. The turning-point extraction told us *why*.

---

## Conversation Flow Mapping: Finding the Golden Path

Raw sentiment scores and talk ratios tell you *what* happened on a call. Conversation flow mapping tells you *why* it happened — and more importantly, *where* it went right or wrong.

We took every call in the dataset and mapped it as a decision tree. Opener leads to response, response leads to a branch, each branch leads to another decision point or a terminal outcome: booking, callback, or dead air followed by a click. Every winning call got mapped. Every losing call got mapped the same way. Then we overlaid the two sets and looked for divergence points — the exact moments where winning calls and losing calls stop looking alike.

What emerged was what we call the **golden path**: the most common conversational flow that ends in a booked appointment. Not a script. A decision tree with four or five critical branch points where the agent's word choice is the difference between a booking and a hangup.

But the golden path is not the *only* path. We identified multiple alternative winning flows — agents who skip qualification and go straight to a value pitch, agents who lead with neighborhood framing instead of weather framing, agents who close on urgency instead of convenience. All of them book appointments. The golden path is just the highest-probability route through the tree.

**Opener framing turned out to be the first major branch point.** We mapped opener variations across the entire agent pool and tracked success rates for each. Weather-based openers ("due to the recent storms in your area") performed differently than neighborhood-based openers ("we've got a crew on your street this week") and both performed differently than pure value-based openers ("we're offering a free inspection"). The performance gap between the best and worst opener frames was significant — and it varied by region, by time of day, and by list source. One frame does not fit all.

**Question architecture was the second major divergence point.** We tracked every question type — open, closed, leading, and assumptive — and logged the order they appeared in across winning and losing calls. Winning agents front-loaded open questions to build rapport, then shifted to assumptive questions to drive toward the close. The classic assumptive frame — "so when we come out there to take a look..." — appeared in a disproportionate number of booked calls. Losing agents asked closed questions too early and got one-word answers that killed momentum.

**Transition mapping revealed the third critical branch.** How an agent moves from qualification to pitch to close is not a smooth gradient — it is a series of micro-commitments. Winning agents bridged each phase with a summary statement that locked in agreement before moving forward. Losing agents jumped from qualifying questions straight into a close with no pitch phase at all, and the prospect balked.

**Close technique analysis exposed the fourth branch point.** We compared open-ended availability framing ("when works for you?") against choice architecture ("I have Monday afternoon and Wednesday morning — which works better for you?"). Choice architecture won by a wide margin. Offering two specific options outperformed open-ended scheduling in nearly every segment. The best closers narrowed the window even further: "I've got a 2 o'clock on Thursday — can you be there for that?"

Finally, we mapped **post-booking behavior** — what happens after the prospect says yes. Top-performing agents repeated the appointment details back, set explicit expectations for what the inspector would do on-site, and in many cases introduced a soft cross-sell or upsell before ending the call. Agents who booked and immediately hung up had higher cancellation rates downstream. The confirmation protocol is not administrative overhead. It is part of the close.

The takeaway is this: the golden path is not a script you hand to an agent and tell them to read. It is a map of decision points. At each branch, the agent has two or three options — and the data shows which option leads to a booking most often.

---

## Timing, Pacing, and Agent Performance: The Numbers Behind the Best Closer

Most call centers measure agents by conversion rate and nothing else. That single number tells you *who* is winning but absolutely nothing about *why*. We pulled timing and pacing data from every completed call across five agents, mapped agent IDs back to real names through VICIdial's user table, and built per-agent performance profiles that go far deeper than a leaderboard.

### The Timing Metrics That Actually Matter

Every call was broken into measurable timing segments. **Talk-to-listen ratio** captures how much of the call an agent spends speaking versus letting the prospect speak. Across all five agents, the average was 62:38 — agents talked 62% of the time. But the range was enormous: our worst performer ran at 71:29. Our best ran at 57:43.

**Average pause length** — the gap between a prospect finishing a sentence and the agent responding — sat at 1.4 seconds across the board. The top agent averaged 1.8 seconds. The worst averaged 0.7 seconds. That 0.7-second gap wasn't efficient. It was interrupting. Call review confirmed it: the worst performer was stepping on prospect sentences 34% of the time compared to the top performer's 11%.

There is an optimal response delay window. Below 0.9 seconds, prospects perceive the agent as robotic or not listening. Above 2.5 seconds, you get dead air that signals confusion or disinterest. The sweet spot in our data was **1.4 to 2.0 seconds** — long enough to feel considered, short enough to feel natural.

**Words per minute** told a similar story. The worst performer spoke at 178 WPM. The best spoke at 149 WPM. The top performer was not the fastest talker. They were the most deliberate one.

### The First 15 Seconds Decide Everything

Calls where the prospect disengaged within the first 60 seconds had a shared pattern: the agent failed the first 15 seconds. Specifically, agents who reached their **value proposition within 8 to 12 seconds** kept prospects on the line 74% of the time. Agents who took longer than 18 seconds to state why they were calling saw a 41% early dropout rate.

### Call Duration and Segment Breakdown

Booked appointments came from calls averaging **4 minutes 12 seconds**. Calls shorter than 2:30 almost never converted — not enough rapport. Calls longer than 6:30 showed diminishing returns and often signaled an agent trapped in circular objection loops.

We segmented each call into three phases: rapport building, business/value discussion, and booking logistics. The optimal split for converted calls was roughly **20% rapport, 55% business, 25% booking**.

One critical metric: **silence after the ask**. When an agent asks for the appointment and then waits, the prospect decides. The top performer averaged 3.1 seconds of silence after asking for the booking before speaking again. The worst performer averaged 1.1 seconds — they would ask for the appointment and then immediately start soft-selling again, undercutting their own close.

### The Agent Scorecards

| Metric | Top Agent | Agent 2 | Agent 3 | Agent 4 | Worst Agent |
|---|---|---|---|---|---|
| Conversion Rate | 23.7% | 19.4% | 17.1% | 14.8% | 11.2% |
| Avg Call Duration | 4:08 | 4:22 | 3:47 | 4:41 | 5:03 |
| Talk:Listen Ratio | 57:43 | 60:40 | 63:37 | 64:36 | 71:29 |
| Script Adherence | 78% | 91% | 85% | 88% | 93% |
| Objection Handling Success | 47% | 38% | 31% | 29% | 22% |
| Recovery Rate (No to Yes) | 19.3% | 12.7% | 9.4% | 8.1% | 5.6% |
| WPM | 149 | 155 | 162 | 168 | 178 |
| Interruption Rate | 11% | 16% | 21% | 24% | 34% |

The top agent ranked first in conversion, objection handling, and recovery rate — while having the **lowest** script adherence of all five agents. The worst agent followed the script most closely and converted the least. That is not a coincidence. The top performer deviated from the script *because* they were reading the prospect and adjusting. The worst performer clung to the script because they were not.

The booking language comparison reinforced this. The top agent used assumptive closes ("I have Tuesday at 2 or Thursday at 10 — which works better?") on 83% of booking attempts. The worst performer used permission-based closes ("Would you maybe be interested in scheduling something?") on 67% of attempts. Assumptive framing converted at 2.1x the rate of permission-based framing in our dataset.

Recovery rate is the most undervalued metric here. Nearly one in five prospects who initially told the top agent "no" ended up booking. The worst performer saved barely one in twenty. The difference was timing: the top agent responded to initial rejection with a 2.3-second pause followed by an empathy statement and a redirect. The worst performer responded in under a second with another pitch, which prospects read as pressure.

---

## Prospect Intelligence and the Real Objection Map

Most AI voice deployments treat every prospect the same. One script, one tone, one cadence. That works fine until it doesn't — which is about 60% of the time.

We categorized every prospect in the dataset into personas based on their behavior in the first 15 seconds of the call. Not demographics. Behavior. The insurance claimant who opens with "I already filed a claim." The skeptical homeowner who responds to every statement with "uh huh." The gatekeeper — a spouse, a parent, an adult child — who isn't the decision-maker but controls access. The hostile pickup: "Take me off your list." The chatty prospect who wants to talk about the weather, their dog, their neighbor's roof, anything except scheduling an appointment. The renter who can't authorize work. The homeowner who already has a contractor lined up.

Eight distinct personas emerged. Each one responds to fundamentally different conversational strategies.

The skeptical homeowner shuts down when the agent leads with urgency. What works: slowing down, offering the free inspection with zero commitment language, letting silence do the work. The insurance claimant wants process clarity — they've already decided they need help, they just need to know you can navigate the claim. The "already have a contractor" prospect isn't a dead end; 23% of those calls converted when the agent positioned the inspection as a second opinion rather than a replacement. The gatekeeper converts when you give them something specific to relay — not "have them call us back" but "we found damage on homes in their area and the free inspection takes about 30 minutes, available Tuesday or Thursday."

Agent adaptation speed matters. The best-performing call sequences showed persona identification within the first two exchanges. Not the first two minutes — the first two exchanges. The agent that waited until objection three to shift strategy had already lost.

Edge cases surfaced that no playbook anticipated. Multi-property owners who want to know if you'll inspect all three houses. Insurance complications where the prospect is mid-dispute with their carrier. Partial jobs — "I just need the flashing fixed, not the whole roof." These represented roughly 8% of calls and had near-zero conversion rates, largely because the agent had no pathway for them.

Geographic patterns showed real variance. Certain area codes converted at nearly double the rate of others — driven by recent storm activity, insurance carrier density, and average home age. Time-of-day analysis confirmed what you'd expect but quantified it: late morning (10-11:30 AM) outperformed every other window by a wide margin. Early afternoon calls (1-2 PM) showed the highest hostile-pickup rate.

We also flipped the lens. Instead of only analyzing what the agent said, we cataloged what prospects asked and what they volunteered unprompted. Prospects frequently asked about licensing, insurance, and whether the company was "local" — none of which appeared in the original script's proactive talking points. Unprompted, prospects volunteered their roof's age, previous storm damage, and neighbor activity ("the house two doors down just got a new roof") at remarkably high rates. That volunteered information is a buying signal the agent never acted on.

Decision-making styles split into four categories: instant decision-makers (roughly 12%), check-with-spouse (31%), need-to-think (38%), and need-multiple-quotes (19%). Each requires a different closing approach and a different follow-up cadence. The original script treated all four the same.

### The Real Objection Map

We extracted every objection from every call — the actual words, not summaries. "I need to talk to my husband" is not the same objection as "my wife handles this stuff." "How much does it cost?" is not the same as "we can't afford a new roof right now." The written playbook contained 15 objection-response pairs. The calls surfaced 43 distinct objections.

That gap — between what you think prospects say and what they actually say — is where most AI voice agents fall apart. They're trained on the playbook. The playbook covers 35% of reality.

We ranked every objection by frequency, then mapped each one in two directions: to the agent response that led to a booking, and to the response that lost the prospect. "I already have a roofer" converted at 23% when met with the second-opinion frame. It converted at 0% when the agent said "we might be able to offer a better price." Same objection, radically different outcomes based on a single sentence.

"Not interested" — the most common objection at 22% of all calls — is recoverable roughly 15% of the time, but only with a specific pattern: acknowledge, pivot to the inspection being free and fast, and offer a specific day. Any variation that included the word "just" ("I just want to let you know about...") killed the call dead.

Multi-objection sequences revealed the real structure. A prospect who objects once is testing. A prospect who objects twice with the same objection is done — conversion rate drops to under 3%. But a prospect who raises two different objections is actually engaged; they're processing. Those calls converted at 18% when the agent treated the second objection as a new conversation rather than a continuation of the first.

Objection timing told its own story. Early objections — within the first 30 seconds — are screening mechanisms. "Is this a sales call?" isn't a real objection; it's a filter. The answer matters less than the tone. Late objections, after the agent has delivered the value proposition, are genuine concerns that require real answers.

The objections that are truly unrecoverable? "I'm a renter" (0% conversion, no pathway to decision-maker). "I just got a new roof" with a timeframe under two years. Everything else — including "take me off your list" — has a nonzero conversion rate with the right response at the right time. The difference between a dead objection and a recoverable one is almost never the objection itself. It's what the agent says next.

---

## Designing the AI Agent: From Data to a Digital Closer

Most AI voice agents are built from a blank page. Someone writes a script they think sounds good, plugs it into a text-to-speech engine, and hopes for the best. That is not engineering. That is guessing.

We had weeks of call data, thousands of analyzed conversations, a complete objection map, golden path models, and per-agent performance breakdowns. Every design decision for the AI agent came from that dataset. Nothing was theoretical.

### Script Optimization: Written vs. Spoken Reality

The first step was reconciling two versions of the truth: the script the client wrote and what agents actually said on calls.

They diverged significantly. We compared the written script against transcriptions from live calls and categorized every deviation. Some deviations improved conversion. The top performer's ad-libbed opening line — a direct, slightly informal greeting that skipped the robotic compliance preamble — converted at 34% higher than the scripted opener. Her reframing of the value proposition as a question rather than a statement increased engagement. Her transition to the booking ask was faster and more assumptive than the script called for.

Other deviations killed deals. Mid-tier agents who freelanced on objection handling introduced hesitation language ("I think," "maybe," "I'm not sure but") that tanked confidence. Agents who deviated from the confirmation protocol had a 23% higher cancellation rate on booked appointments.

We extracted the best-performing language from every stage — opening, value prop, transition, objection response, booking ask, confirmation — and merged it with the structural discipline of the written script. The result was a revised script built from proven performance, not copywriting instinct.

We then branched it into three distinct flows: first-contact outbound, callback (prospect requested a call back), and follow-up (prior contact, no booking). Each flow has different opener energy, different context assumptions, and different conversion mechanics.

### AI Agent Architecture

With the optimized script as the foundation, we built the AI agent's core architecture.

**System prompt and persona.** The agent's communication style was modeled directly on the top human performer. Her pacing, her sentence length, her ratio of statements to questions, her tone calibration between professional and conversational — all of it was encoded into the system prompt. Voice selection matched her energy and cadence profile. This was not a generic "friendly assistant" persona. It was a reconstruction of a specific, proven communication style.

**Conversation flow logic.** The decision tree follows the golden path model extracted from high-converting calls. Every branch point corresponds to a real fork we observed in the data — not a hypothetical "what if the prospect says X" scenario, but an actual pattern that occurred with measurable frequency.

**Objection handling module.** Built entirely from the real objection map. The 14 objection categories we identified each have response sequences pulled from calls where that specific objection was successfully overcome. The AI's rebuttal to "I need to talk to my spouse" is the exact language that converted that objection 41% of the time in live calls.

**Persona detection.** Within the first 15 seconds of conversation, the AI classifies the prospect's communication type — direct/analytical, social/rapport-driven, skeptical/guarded, or disengaged — and adjusts its approach. Pacing slows for analytical prospects. Rapport-building extends for social types. Proof points front-load for skeptics.

**Operational modules.** Dynamic scheduling queries real-time availability and confirms appointment times conversationally. Cross-sell detection flags when a prospect mentions adjacent services and routes appropriately. The appointment confirmation protocol mirrors the exact sequence that correlated with lowest cancellation rates.

**Transfer-to-human triggers.** The AI is not trying to replace every interaction. Specific conditions initiate a warm transfer: high-value prospect signals, emotional escalation beyond threshold, compliance-sensitive topics, explicit request for a human, or conversation loops exceeding two cycles on the same objection. The handoff includes a structured summary so the human agent has full context.

**Edge case handling.** Voicemail detection triggers a separate script optimized for callback generation. Silence handling follows calibrated wait times — not too eager, not too passive — derived from the timing analysis of real calls. Interruption handling uses natural back-off patterns. Response latency was tuned to the cadence data from the top performer: fast enough to feel responsive, slow enough to feel human.

**Guardrails and compliance.** DNC handling is immediate and absolute. Topic boundaries are hard-coded — the AI will not discuss pricing outside approved ranges, will not make claims beyond approved language, and will not engage with off-topic conversation beyond a single redirect. Maximum call duration enforces a cutoff to prevent resource waste on non-converting calls.

**Structured data capture.** At call termination, the AI writes a structured record: disposition, objections encountered, prospect type classification, key information gathered, next-action recommendation, and full conversation summary. Every call produces actionable data, not just a recording to review later.

The fundamental principle: this AI agent is not a chatbot with a phone number. It is a digital reconstruction of the highest-performing human closer on the team, built from empirical data about what actually works on these calls, constrained by operational discipline, and designed to produce structured output that feeds the next optimization cycle.

---

## Testing Against Real Calls and the ROI Math

You don't ship a dialer feature without testing it against live traffic. You don't ship an AI voice agent without doing the same thing, except harder.

### How We Tested

We started with replay testing. We took the prospect's side of hundreds of real recorded calls — the objections, the pauses, the "I'm not interested" delivered seventeen different ways — and played them against the AI. Then we scored every AI response against what the top-performing human agent actually said on that same call.

This wasn't a vibes check. We built a QA rubric derived directly from the highest-converting human calls in the dataset. Every AI call got scored automatically against that rubric: opener quality, objection handling, appointment close technique, compliance adherence. No subjectivity. No "that sounded pretty good." Numbers or nothing.

From there, we A/B tested everything. Different openers. Different objection responses. Different voices. Different scheduling approaches — "I have a 2pm tomorrow" versus "What time works best for you this week?" We measured conversion rate on every variant against the human baseline, broken out per agent.

Then we stress-tested the edge cases that break most automation: multi-property scenarios where the prospect owns three rentals and a primary residence. Insurance complications. Prospects who are openly hostile. And compliance scenarios that carry real legal weight — DNC requests, "stop calling me," "take me off your list." The AI has to handle those correctly every single time, not most of the time.

Finally, regression testing. Every change to prompts, voice configuration, or conversation logic got tested against the full suite of winning patterns. If a tweak to objection handling improved insurance callbacks but broke the solar opener, we caught it before it hit production.

### The ROI Math

Here are the numbers that matter.

A human agent fully loaded — salary, benefits, management overhead, training, workspace — runs $15 to $25 per hour. After you account for AMD filtering, no-answers, voicemails, and dispositions, that agent is having maybe 15 to 25 actual live conversations per hour. That puts your cost per live conversation somewhere between $0.60 and $1.67, before you factor in turnover, sick days, training ramp, and the Monday-after-a-long-weekend performance dip.

An AI voice agent costs a fraction of a cent per second of conversation. It runs 24 hours a day, 7 days a week. It handles unlimited concurrent calls. It doesn't call in sick. It doesn't quit after three weeks. It doesn't have bad days. And critically, it performs at the level of your *best* agent — not your average, not your newest hire who's still reading from a script they don't understand.

Run the math on cost per appointment. Take your current human conversion rate, divide by your fully loaded cost per hour, and compare that to the AI's conversion rate at its per-minute cost. For most operations we've tested, the AI delivers appointments at 40 to 60 percent lower cost — and that's before you factor in the 24/7 availability capturing leads that would otherwise hit voicemail at 8pm.

But cost savings aren't even the real story. The real value is consistency and scalability. Your best human agent has a 12% conversion rate on Tuesdays and a 7% rate on Fridays. The AI runs the same playbook at the same level every single call. A 50-seat center can deploy the AI alongside their human agents, measure head-to-head for two weeks, and scale based on actual data — not projections.

The AI doesn't need to be better than your best agent. It needs to be better than your *average* agent, available around the clock, at a fraction of the cost. That math changes everything.

---

## What We Deliver

When we hand over the final package, you're not getting a slide deck and a handshake. You're getting a production-ready system backed by every piece of documentation you need to understand it, maintain it, and improve it over time.

**Transcripts and Core Analysis.** Every call gets a clean, speaker-labeled transcript. These feed into the master analysis CSV — one row per call with date, agent name, prospect name, phone number, call duration, outcome, appointment date (if booked), objections raised, lead score, and agent score. Sort it, filter it, pivot it — it's your campaign in a spreadsheet.

**Pattern Intelligence.** The winning patterns document pulls exact language from calls that booked appointments — not paraphrased, not summarized, the actual words that worked. The losing patterns document does the same for calls that died. When you read them side by side, the differences are obvious and immediate. The objection map takes every real objection your team encountered and pairs it with proven winning responses and proven losing responses, straight from the transcripts. No theoretical rebuttals. These are battle-tested.

**Prospect Profiles.** Persona profiles break your prospect base into behavioral types and document what works and what fails for each one. Edge cases get their own documentation — the weird calls, the unusual scenarios, the situations your script never anticipated.

**Agent Performance.** Individual agent scorecards show per-agent metrics with specific coaching recommendations. The agent comparison report benchmarks everyone against your top performer, highlighting exactly where the gaps are and what to do about them.

**Script and Flow Optimization.** A full script effectiveness analysis showing which sections convert and which sections lose people, plus the optimized script. Conversation flow decision trees map out every branch point. Timing and pacing analysis shows how fast your best closers talk, when they pause, and how long their calls need to run. Geographic and time-of-day analysis tells you when and where your calls hit hardest.

**The AI Agent.** The complete AI agent system prompt — ready to deploy. A structured knowledge base the agent draws from during live calls. A test plan with scenarios covering every persona, objection, and edge case we identified. And an ROI model projecting what this agent does to your cost-per-appointment at scale.

You own all of it. Every document, every dataset, every prompt. Nothing lives behind our login.

---

## Frequently Asked Questions

### How many calls do you need to build an AI agent?

We recommend 500 or more recorded calls for full statistical significance — that gives us enough data to identify patterns with confidence, build reliable persona profiles, and validate objection handling across a meaningful sample. That said, even 100 calls will surface the major patterns. We've started builds with smaller datasets and expanded as more recordings came in. The analysis just gets sharper with more data.

### Does this work for industries other than roofing?

Absolutely. The methodology works for any outbound calling campaign where you're trying to set appointments, qualify leads, or move prospects through a pipeline. We've applied this to insurance, solar, home services, political outreach, and general B2B appointment setting. If you're running outbound calls through VICIdial, the process is the same — the industry-specific details just change the content of the scripts and objection maps.

### Does the AI agent integrate with VICIdial?

Yes — that's the whole point. The AI agent pulls leads from the same lead lists your human agents use, writes dispositions back to VICIdial in real time, and uses the same DIDs and caller ID settings. From VICIdial's perspective, the AI agent is just another agent logged into the system. Your existing reporting, list management, and campaign logic all stay intact.

### What about compliance and TCPA?

Compliance is built into the system from the start. The AI agent handles DNC list checking, consent tracking, and state-specific calling rules automatically. It knows when to identify itself, how to handle opt-out requests immediately, and respects time-of-day restrictions by state. We build these guardrails into the agent's core logic — they're not bolted on after the fact.

### Can you still use human agents alongside the AI?

Yes, and most operations do exactly that. Blended campaigns are a natural fit — the AI agent handles first-contact dials and initial qualification, then routes warm transfers or scheduled callbacks to your human closers. Your best agents spend their time on the calls that actually need a human touch, and the AI handles the volume work. You can scale the ratio up or down depending on staffing and call volume.

### How long does the whole process take?

From the time we receive your call recordings to a production-ready AI agent, expect two to four weeks. The first week is transcription and deep analysis. The second week is pattern extraction, script optimization, and building the agent's knowledge base. Weeks three and four cover system prompt engineering, testing against real scenarios, and tuning until the agent performs at or above your benchmarks. Smaller datasets or simpler campaigns can move faster.

### What's the conversion rate compared to human agents?

It varies by campaign, but the AI agent consistently matches or exceeds the performance of your average human agent. It won't outperform your single best closer on their best day — but it performs at that above-average level on every single call, every hour, without fatigue, without bad days, and without turnover. When you factor in the volume it handles and the consistency it maintains, the overall conversion numbers typically come out ahead.

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/how-we-built-ai-voice-agents).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
