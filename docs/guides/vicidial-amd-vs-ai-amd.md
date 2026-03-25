# VICIdial Answering Machine Detection vs AI-Based AMD: Which Is Better?

Answering machine detection is the single most impactful technology in an outbound call center's stack. A good AMD system connects your agents to live humans and filters out voicemail boxes. A bad one either drops live calls (false positives) or wastes agent time on answering machines (false negatives). Either way, you lose money.

VICIdial ships with traditional CPD-based (Call Progress Detection) AMD built into Asterisk. It has been the standard for over a decade. In the last few years, AI-based AMD systems have emerged that use machine learning models to analyze audio in real time. These systems promise significantly higher accuracy -- but they come with tradeoffs in latency, cost, and integration complexity.

This article provides a detailed technical comparison. We will examine how each approach works at the signal-processing level, compare real-world accuracy numbers, analyze latency impacts on agent utilization, break down the cost math, and help you decide which approach fits your operation.

## How Traditional CPD/AMD Works in Asterisk

VICIdial's AMD relies on Asterisk's `AMD()` application, which implements Call Progress Detection using audio signal analysis. Here is what happens at the technical level when an outbound call is answered.

### The Detection Process

1. **Call answer detection.** Asterisk detects that the far end has answered (SIP 200 OK or energy detection on analog lines). The CPD engine begins listening.

2. **Energy detection.** The CPD engine monitors audio energy levels. When energy exceeds the silence threshold, it marks the start of a "sound event" -- what it considers a word or speech segment.

3. **Silence detection.** When energy drops below the threshold and stays there for `cpd_amd_between_words_silence` milliseconds, the current sound event ends. The silence gap is counted as a word boundary.

4. **Word counting.** Each sound event that lasts longer than `cpd_amd_minimum_word_length` milliseconds is counted as a word. Sound events shorter than this are discarded as noise.

5. **Decision logic.** If the number of words exceeds `cpd_amd_maximum_number_of_words`, or if any single word exceeds `cpd_amd_maximum_word_length`, the call is classified as a machine. Otherwise, it is classified as human.

6. **Timeout.** If the CPD engine cannot reach a decision within the total analysis window (typically 2-5 seconds), it applies a default classification.

### The Detection Parameters

```
; VICIdial AMD parameter defaults
cpd_amd_maximum_word_length = 5000      ; Max ms for a single word
cpd_amd_minimum_word_length = 100       ; Min ms to count as a word
cpd_amd_between_words_silence = 50      ; Ms of silence between words
cpd_amd_maximum_number_of_words = 3     ; Max words before machine classification
cpd_amd_minimum_number_of_words = 1     ; Min words before human classification
cpd_amd_silence_threshold = 256         ; Energy threshold for silence vs speech
```

### Strengths of Traditional AMD

- **Zero external dependencies.** Runs entirely within Asterisk. No API calls, no network hops, no third-party services.
- **Zero per-call cost.** The CPU overhead of running AMD on a modern server is negligible.
- **Deterministic behavior.** Given the same audio input and parameters, you always get the same result. This makes debugging straightforward.
- **Sub-second detection.** For clearly human answers ("Hello"), the system makes a decision in 500-800ms. For clearly machine answers (long greetings), it decides as soon as the word count exceeds the threshold.
- **Tunable.** The parameters are well-understood and can be adjusted per campaign, per carrier, and per time of day. See our [detailed AMD tuning guide](/blog/vicidial-amd-false-positive-reduction).

### Weaknesses of Traditional AMD

- **No semantic understanding.** The system does not know what is being said -- only how many words there are and how long they are. A human saying "Hello, this is Michael from accounting" (6 words) looks identical to a short voicemail greeting to the CPD engine.
- **Sensitive to audio quality.** Network jitter, codec compression artifacts, background noise, and silence suppression all affect word boundary detection. Different carriers require different parameter tuning.
- **Cannot distinguish voicemail types.** A cell phone voicemail ("Hey, it's Sarah, leave a message") and a human answering informally ("Hey, it's Sarah, what's up") have very similar acoustic profiles.
- **Fixed analysis window.** The system must make a decision within a few seconds. Some modern voicemail greetings are very short (1-2 words), making them indistinguishable from human answers based on word count alone.

### Real-World Accuracy: Traditional AMD

Across more than 100 VICIdial deployments we have managed, here are the accuracy ranges we observe:

| Metric | Default Parameters | Tuned Parameters |
|--------|-------------------|------------------|
| Overall accuracy | 75-82% | 85-92% |
| False positive rate | 15-25% | 4-8% |
| False negative rate | 10-15% | 8-12% |
| Detection latency | 800-2000ms | 600-1500ms |

"Tuned parameters" means per-carrier calibration with the methodology described in our [AMD tuning guide](/blog/vicidial-amd-false-positive-reduction).

## How AI-Based AMD Works

AI-based AMD systems take a fundamentally different approach. Instead of counting words and measuring silence gaps, they analyze the audio signal using machine learning models trained on millions of call recordings.

### The Detection Process

1. **Audio capture.** When the call is answered, the system begins capturing the audio stream. This is typically done by forking the media to an external analysis service.

2. **Feature extraction.** The raw audio is converted into features that a machine learning model can process. Common features include Mel-frequency cepstral coefficients (MFCCs), spectral characteristics, pitch contours, and speech rate patterns.

3. **Model inference.** The features are fed into a trained neural network (typically a convolutional neural network or transformer model) that has been trained on millions of labeled recordings. The model outputs a probability score: the likelihood that the audio is from a human versus a machine.

4. **Confidence thresholding.** If the human probability exceeds a threshold (typically 0.7-0.85), the call is classified as human. If the machine probability exceeds its threshold, it is classified as machine. If neither threshold is met, the system continues listening or returns an uncertain classification.

5. **Continuous analysis.** Unlike traditional AMD which makes a single binary decision, AI-based systems continuously update their classification as more audio arrives. The confidence score typically stabilizes within 1-2 seconds.

### What the Model Learns

AI-based AMD models learn features that traditional CPD cannot detect:

- **Speech prosody.** Humans answering a ringing phone have a rising intonation ("Hello?"). Voicemail greetings tend to have a flat or falling intonation ("Hi, you've reached...").
- **Background characteristics.** Voicemail greetings are typically recorded in quiet environments. Live answers often have ambient noise, TV, traffic, or other people.
- **Microphone characteristics.** Voicemail greetings are often recorded on the phone's internal microphone at close range. Live answers come through the earpiece microphone at variable distances.
- **Beep detection.** AI models learn to detect the voicemail beep with high accuracy, even when it has not played yet, based on the greeting patterns that precede it.
- **Carrier-specific voicemail patterns.** The model learns that Verizon's voicemail system sounds different from AT&T's, which sounds different from T-Mobile's.

### Integration with VICIdial

AI-based AMD systems integrate with VICIdial through one of two approaches:

**Approach 1: Media forking.** The Asterisk dial plan forks the media stream to an external analysis service. The service returns the classification via an AMI event or AGI callback. This approach keeps the call flow within Asterisk and VICIdial.

```ini
; Example: Fork media to AI AMD service
exten => _1NXXNXXXXXX,1,Answer()
exten => _1NXXNXXXXXX,n,Set(AUDIOHOOK_INHERIT(MixMonitor)=yes)
exten => _1NXXNXXXXXX,n,MixMonitor(/tmp/amd_${UNIQUEID}.wav)
; Send audio to AI service via WebSocket or HTTP stream
exten => _1NXXNXXXXXX,n,AGI(ai_amd.agi)
; ai_amd.agi reads the stream, calls the API, returns result
exten => _1NXXNXXXXXX,n,GotoIf($["${AMD_RESULT}" = "MACHINE"]?machine:human)
```

**Approach 2: SIP REFER.** The call is initially routed to the AI AMD service, which analyzes the audio, makes a classification, and then transfers the call back to VICIdial with the classification result in a SIP header. This approach is cleaner but requires more complex SIP routing.

### Real-World Accuracy: AI-Based AMD

Based on published benchmarks and our own testing across multiple AI AMD providers:

| Metric | AI-Based AMD |
|--------|-------------|
| Overall accuracy | 92-96% |
| False positive rate | 2-4% |
| False negative rate | 4-8% |
| Detection latency | 1000-2500ms |

## Head-to-Head Comparison

### Accuracy

This is where AI-based AMD clearly wins. The jump from 85-92% (tuned traditional) to 92-96% (AI) may seem modest in percentage terms, but at scale the impact is significant.

For a center running 50 agents at 200 dials/hour:

- **Traditional AMD (tuned, 8% FP rate):** 10,000 dials -> 4,000 answered -> 320 false positives = 320 lost live connections/day
- **AI AMD (3% FP rate):** 10,000 dials -> 4,000 answered -> 120 false positives = 120 lost live connections/day

That is 200 additional live connections per day reaching agents. At $5 per live connection, that is $1,000/day or $22,000/month in recovered value.

But the accuracy advantage diminishes with excellent traditional AMD tuning. If you have already optimized your CPD parameters per carrier and implemented adaptive thresholds (getting your false positive rate to 4-5%), the incremental gain from AI AMD drops to 50-80 additional live connections per day.

### Latency

This is where traditional AMD has the advantage. Here is why it matters.

When AMD is running, the called party has picked up and is hearing silence (or ringback, depending on your configuration). Every millisecond of AMD processing is a millisecond where the prospect is saying "Hello? Hello?" into silence. If the AMD latency is too high, the prospect hangs up before the agent connects -- creating a different kind of lost connection.

| AMD Type | Detection Latency | Agent Connection Delay |
|----------|-------------------|------------------------|
| Traditional (clear human) | 500-800ms | 1-1.5 seconds |
| Traditional (machine) | 800-2000ms | N/A (dropped) |
| AI-based (clear human) | 1000-1500ms | 1.5-2.5 seconds |
| AI-based (ambiguous) | 2000-3000ms | 2.5-3.5 seconds |

The 500-1000ms additional delay with AI AMD is noticeable. Studies show that each additional second of silence after the called party answers reduces the conversation rate by 5-8%. At 2.5 seconds of total delay, some prospects hang up, defeating the purpose of the AMD system entirely.

Some AI AMD providers mitigate this with "early classification" -- making a preliminary decision at 800ms based on initial audio features and refining it as more audio arrives. This helps but introduces the risk of premature misclassification.

### Cost

Traditional AMD costs nothing beyond the Asterisk server CPU it runs on. AI-based AMD introduces per-call costs.

| Provider Type | Cost Per Analyzed Call | Monthly Cost (10,000 calls/day) |
|--------------|----------------------|-------------------------------|
| Traditional (Asterisk built-in) | $0.00 | $0 |
| AI AMD SaaS (budget tier) | $0.005 - $0.01 | $1,500 - $3,000 |
| AI AMD SaaS (premium tier) | $0.01 - $0.03 | $3,000 - $9,000 |
| Self-hosted AI AMD | $0.001 - $0.003 | $300 - $900 (+ server costs) |

Self-hosted AI AMD (running your own ML model on a GPU server) is the most cost-effective option if you have the engineering capability to deploy and maintain it. But the GPU server itself costs $500-2,000/month depending on specs.

### Reliability and Dependencies

Traditional AMD has no external dependencies. If Asterisk is running, AMD is running.

AI-based AMD introduces dependencies:

- **Network connectivity** to the AMD service (for SaaS solutions)
- **Service availability** of the AMD provider
- **API latency** which varies with provider load
- **Model updates** which can change classification behavior without warning

If the AI AMD service goes down or becomes unreachable, you need a fallback -- which is typically traditional AMD. So you end up maintaining both systems.

### Maintenance

Traditional AMD requires ongoing parameter tuning as carriers change and call patterns shift. This is manual work that requires VICIdial expertise.

AI-based AMD models are typically maintained by the provider. They retrain on new data and push model updates. This is hands-off from your perspective, but it also means you have limited control over changes to classification behavior.

## Decision Framework: When to Use Each Approach

### Use Traditional AMD When:

1. **Your center has fewer than 50 agents.** The cost of AI AMD does not justify the incremental accuracy gain at lower volumes.
2. **You have per-carrier AMD tuning in place.** Tuned traditional AMD at 4-5% false positive rate provides diminishing returns from AI.
3. **Latency is critical.** If your campaigns target demographics that hang up quickly (B2C cold calls), the additional 500-1000ms of AI AMD latency may cost more connections than it saves.
4. **You want zero external dependencies.** For mission-critical operations, keeping AMD entirely within your infrastructure eliminates a class of failure modes.
5. **Budget is constrained.** Traditional AMD is free. AI AMD at $3,000-9,000/month is a meaningful expense.

### Use AI-Based AMD When:

1. **Your center has 100+ agents.** The accuracy improvement translates to hundreds of additional live connections per day, easily justifying the cost.
2. **Your campaigns have high per-connection value.** If each live connection is worth $10+ (insurance, solar, real estate), the 2-4% false positive reduction from AI AMD pays for itself many times over.
3. **You dial cell phones predominantly.** Cell phone voicemail greetings are shorter and more conversational than landline voicemails, making them harder for traditional AMD to distinguish. AI AMD handles this better.
4. **You have already exhausted traditional tuning.** If you have per-carrier tuning, adaptive thresholds, and recording-validated parameters and still see 6-8% false positives, AI AMD is the next step.
5. **Your agents handle inbound transfers.** If the called party hears a brief IVR message or hold music during the AMD analysis window (masking the silence), the latency penalty is eliminated.

### Use Both (Hybrid Approach):

The optimal setup for large centers is a hybrid approach:

1. Run traditional AMD as the primary detector with aggressive tuning
2. For calls where traditional AMD confidence is low (borderline decisions), route the audio to AI AMD for a second opinion
3. Use AI AMD exclusively for cell phone-only campaigns where traditional AMD struggles

This hybrid approach gives you the speed of traditional AMD for clear-cut cases (which are 70-80% of calls) and the accuracy of AI AMD for ambiguous cases.

```ini
; Hybrid AMD dial plan example
exten => _1NXXNXXXXXX,1,AMD()
exten => _1NXXNXXXXXX,n,Set(TRADITIONAL_RESULT=${AMDSTATUS})
exten => _1NXXNXXXXXX,n,Set(TRADITIONAL_CAUSE=${AMDCAUSE})

; If traditional AMD is uncertain (borderline word count, borderline timing)
exten => _1NXXNXXXXXX,n,GotoIf($["${TRADITIONAL_CAUSE}" = "TOOLONG-3-3000"]?ai_check)
exten => _1NXXNXXXXXX,n,GotoIf($["${TRADITIONAL_CAUSE}" = "MAXWORDS-3-2800"]?ai_check)

; Clear traditional result - use it
exten => _1NXXNXXXXXX,n,GotoIf($["${TRADITIONAL_RESULT}" = "MACHINE"]?machine:human)

; Borderline - get AI second opinion
exten => _1NXXNXXXXXX,n(ai_check),AGI(ai_amd_check.agi)
exten => _1NXXNXXXXXX,n,GotoIf($["${AI_RESULT}" = "MACHINE"]?machine:human)

exten => _1NXXNXXXXXX,n(human),NoOp(HUMAN - connecting to agent)
exten => _1NXXNXXXXXX,n,Goto(agent_connect,s,1)

exten => _1NXXNXXXXXX,n(machine),NoOp(MACHINE - sending to voicemail drop)
exten => _1NXXNXXXXXX,n,Goto(vm_drop,s,1)
```

## The Future of AMD in VICIdial

The trajectory is clear: AI-based AMD will eventually replace traditional CPD for most call centers. Model inference costs are dropping rapidly (50% year-over-year for the past three years). Latency is improving as edge-deployed models eliminate network round trips. And accuracy continues to improve as training datasets grow.

However, traditional AMD will remain relevant for:
- Small centers where the cost-benefit does not justify AI
- Environments with strict data sovereignty requirements (no audio leaving the premise)
- As a fallback when AI services are unreachable

The winning strategy today is to master traditional AMD tuning first -- it is free and immediately impactful -- and layer AI AMD on top when your scale and per-connection value justify the investment.

## How ViciStack Helps

ViciStack manages AMD optimization as part of our flat-rate VICIdial optimization service. For every client, we:

- **Audit current AMD performance** with recording-level false positive analysis
- **Deploy per-carrier traditional AMD tuning** to get false positives below 5%
- **Evaluate AI AMD ROI** based on your specific call volume and per-connection value
- **Integrate AI AMD** when the math supports it, handling all technical integration
- **Monitor ongoing performance** and adjust parameters as conditions change

We have deployed both traditional and AI AMD across more than 100 VICIdial call centers. We know when each approach delivers ROI and when it does not -- and we will tell you honestly which one is right for your operation.

**All included at $150/agent/month. No per-minute charges. No AI AMD surcharges.**

**[Get your free AMD analysis -- we will measure your current false positive rate and calculate exactly how many live connections you are losing.](https://vicistack.com/proof/)**

5-minute response time. No commitment, no sales pitch. Just your data.

## Further Reading

- [How to Reduce VICIdial AMD False Positives from 20% to Under 5%](/blog/vicidial-amd-false-positive-reduction) -- the complete traditional AMD tuning guide
- [VICIdial SIP Trunk Failover and Redundancy: Complete Setup Guide](/blog/vicidial-sip-trunk-failover) -- carrier changes that affect AMD performance
- [VICIdial Caller ID Reputation Monitoring and Recovery Guide](/blog/vicidial-caller-id-reputation) -- caller ID impacts answer rates, which impacts AMD performance

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-amd-vs-ai-amd).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
