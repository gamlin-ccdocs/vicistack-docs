# VICIdial Voicemail Drop Configuration and Compliance Guide

Voicemail drop is one of the most powerful and most misunderstood features in VICIdial. Done correctly, it turns answering machine contacts from dead weight into a marketing touchpoint — a pre-recorded message plays on the prospect's voicemail without consuming agent time. Done incorrectly, it violates FCC regulations, wastes list records, and can result in fines that dwarf your entire campaign budget.

This guide covers the technical configuration, compliance framework, and per-campaign strategy for voicemail drops in VICIdial. If you're running 25+ agents and not using voicemail drops effectively, you're leaving a significant number of callbacks on the table.

## How Voicemail Drop Works in VICIdial

The voicemail drop process relies on Answering Machine Detection (AMD) to identify when a call reaches a voicemail system rather than a live person. Here's the flow:

1. **Dialer places the call** — VICIdial's adaptive dialer sends the call through Asterisk to the carrier.
2. **Call is answered** — The remote side picks up (could be human or machine).
3. **AMD analyzes the audio** — Asterisk's AMD application listens to the first few seconds of audio and classifies the answer as HUMAN or MACHINE.
4. **If MACHINE → voicemail drop** — The system plays a pre-recorded audio file and hangs up. No agent is involved.
5. **If HUMAN → route to agent** — The call bridges to an available agent as normal.

The critical component is AMD accuracy. A false positive (human classified as machine) means a live person hears your voicemail message and the agent never connects — that's a dropped live contact. A false negative (machine classified as human) means an agent gets connected to a voicemail greeting and wastes 15-30 seconds before manually dispositioning.

### VICIdial AMD Settings

AMD is configured at the campaign level under **Campaign Settings > Answering Machine Detection**.

Key settings:

| Setting | Description | Default |
|---------|-------------|---------|
| `amd_send_to_vmx` | Send AMD-detected machines to voicemail drop | N |
| `amd_type` | AMD method: `AMD` (Asterisk native) or `CPD` (Call Progress Detection) | AMD |
| `amd_agent_route_options` | What to do with AMD results | NONE |
| `campaign_vdad_exten` | Extension to route AMD machines to | 8369 |

The `campaign_vdad_exten` is where the voicemail drop magic happens. When AMD detects a machine, it routes the call to this extension. In the Asterisk dialplan, this extension plays the voicemail drop audio file.

## Audio File Requirements and Configuration

### Recording the Voicemail Drop Message

Your voicemail drop message needs to accomplish several things in 20-30 seconds:

1. **Identify the caller** — FCC requires this for prerecorded messages
2. **State the purpose** — Brief reason for the call
3. **Provide a callback mechanism** — Phone number or website
4. **Sound natural** — Not robotic, not rushed

Example script for an insurance campaign:

> "Hi, this is [agent name] calling from [company name] about the insurance information you requested. I wanted to make sure you got everything you needed. Give me a call back at [number] at your convenience, or visit [website]. Thanks, and have a great day."

Record as a WAV file (8kHz, 16-bit, mono — Asterisk's native format):

```bash
# Convert from any format to Asterisk-compatible WAV
sox input_message.mp3 -r 8000 -c 1 -b 16 vm_drop_insurance.wav

# Place in Asterisk sounds directory
cp vm_drop_insurance.wav /var/lib/asterisk/sounds/

# Verify the file is readable
asterisk -rx "core show file formats" | grep wav
```

### Multiple Audio Files for Different Campaigns

Each campaign can have its own voicemail drop message. This is essential when running multiple verticals — a solar voicemail should not reference insurance.

Store files with a clear naming convention:

```
/var/lib/asterisk/sounds/vm_drop_insurance_medicare.wav
/var/lib/asterisk/sounds/vm_drop_solar_residential.wav
/var/lib/asterisk/sounds/vm_drop_political_gotv.wav
/var/lib/asterisk/sounds/vm_drop_retention_callback.wav
```

### Dialplan Configuration for Voicemail Drop

The voicemail drop plays through a custom Asterisk dialplan context. The standard VICIdial installation includes a framework for this, but you'll need to customize it for your audio files.

In your Asterisk extensions configuration (typically `extensions_custom.conf` or within VICIdial's generated dialplan):

```
; Voicemail drop context for SALESCAMP
[vm-drop-salescamp]
exten => s,1,Answer()
exten => s,n,Wait(1)
exten => s,n,Playback(vm_drop_insurance_medicare)
exten => s,n,Wait(1)
exten => s,n,Hangup()

; Voicemail drop context for SOLARCAMP
[vm-drop-solarcamp]
exten => s,1,Answer()
exten => s,n,Wait(1)
exten => s,n,Playback(vm_drop_solar_residential)
exten => s,n,Wait(1)
exten => s,n,Hangup()
```

The `Wait(1)` before playback is important — it gives the voicemail system time to start recording after its beep. Without it, the first second of your message gets cut off.

### VICIdial Campaign Configuration for AMD + Voicemail Drop

Configure the campaign to use AMD and route machines to your voicemail drop context:

```sql
UPDATE vicidial_campaigns
SET
    amd_send_to_vmx = 'Y',
    amd_type = 'AMD',
    campaign_vdad_exten = '8369',
    amd_agent_route_options = 'AMD'
WHERE campaign_id = 'SALESCAMP';
```

In VICIdial's admin interface, this corresponds to:

1. Go to **Campaigns > [Your Campaign] > Detail View**
2. Set **AMD Send To VMX:** `Y`
3. Set **AMD Type:** `AMD`
4. Set **Answering Machine Message:** Select your audio file from the system recordings

You can also use VICIdial's built-in audio store by uploading the recording through **Admin > Audio Store** and referencing it by the recording ID.

## AMD Integration for Accurate Drops

AMD accuracy determines whether voicemail drop is an asset or a liability. The default Asterisk AMD settings are tuned for American English telephone greetings, but they're far from perfect out of the box.

### Understanding AMD Parameters

Asterisk's AMD module analyzes audio patterns based on these parameters (set in `amd.conf`):

```ini
; /etc/asterisk/amd.conf
[general]
initial_silence = 2500         ; Max silence before classifying as MACHINE (ms)
greeting = 1500                ; Max greeting length before classifying as MACHINE (ms)
after_greeting_silence = 800   ; Silence after greeting to confirm MACHINE (ms)
total_analysis_time = 5000     ; Max total time for AMD analysis (ms)
min_word_length = 100          ; Minimum duration of a "word" (ms)
between_words_silence = 50     ; Max silence between words before end-of-greeting (ms)
maximum_number_of_words = 3    ; Max words before classifying as MACHINE
silence_threshold = 256        ; Audio level threshold for silence detection
maximum_word_length = 5000     ; Maximum single word duration (ms)
```

### How AMD Decides

The logic is roughly:

1. **Initial silence** — If there's silence for `initial_silence` ms after the call is answered, it's likely a machine (voicemail systems have a brief pause before the greeting).
2. **Word count** — If more than `maximum_number_of_words` words are spoken without a long pause, it's a voicemail greeting (humans typically say "hello" and wait).
3. **Greeting length** — If continuous speech exceeds `greeting` ms, it's a machine playing a recorded message.
4. **After-greeting silence** — After speech stops, if silence exceeds `after_greeting_silence` ms, it confirms a machine (the beep/recording period).

### Tuning AMD for Your Environment

The defaults work reasonably well for standard American voicemail, but specific adjustments can improve accuracy significantly.

**Reducing False Positives (Humans Classified as Machines):**

False positives are the more dangerous error — you lose a live connection. They happen when a person answers with a long greeting ("Hello, this is John Smith, who's calling?") that exceeds the word count or greeting length threshold.

```ini
; More tolerant settings — reduces false positives
greeting = 2000                ; Allow longer greetings before machine classification
maximum_number_of_words = 5    ; More words before machine classification
total_analysis_time = 5500     ; More time for analysis
```

The tradeoff: more false negatives (machines routed to agents). But an agent can disposition a voicemail in 5 seconds; a lost live connection is gone forever.

**Reducing False Negatives (Machines Classified as Humans):**

If agents are constantly getting connected to voicemail greetings, tighten the thresholds:

```ini
; Stricter settings — reduces false negatives
greeting = 1200
maximum_number_of_words = 2
initial_silence = 2000
```

The tradeoff: more false positives. This is only appropriate when you're confident your list has very few live answers (e.g., heavily recycled aged lists where 80%+ go to voicemail).

### Testing AMD Accuracy

Measure your AMD accuracy over a sample period:

```sql
-- Compare AMD classification with agent disposition
-- Agents who get connected should disposition AM (answering machine)
-- if they hear a voicemail greeting

SELECT
    DATE(call_date) AS day,
    COUNT(*) AS total_calls,
    SUM(CASE WHEN status = 'AA' THEN 1 ELSE 0 END) AS amd_machine,
    SUM(CASE WHEN status = 'AM' THEN 1 ELSE 0 END) AS agent_marked_machine,
    SUM(CASE WHEN status = 'AL' THEN 1 ELSE 0 END) AS amd_live,
    ROUND(
        SUM(CASE WHEN status = 'AM' THEN 1 ELSE 0 END) * 100.0 /
        NULLIF(SUM(CASE WHEN status IN ('AA','AL','AM') THEN 1 ELSE 0 END), 0),
        1
    ) AS agent_machine_pct
FROM vicidial_log
WHERE call_date >= DATE_SUB(CURDATE(), INTERVAL 7 DAY)
  AND campaign_id = 'SALESCAMP'
GROUP BY DATE(call_date);
```

If `agent_machine_pct` is high (agents are frequently marking calls as answering machines that AMD sent to them as live), your AMD is producing too many false negatives. If your `amd_machine` count is disproportionately high relative to industry norms (typically 20-40% of answered calls), you may have too many false positives.

A good target: AMD accuracy of 85%+ for machine detection, with false positive rate under 5%.

## FCC/TCPA Compliance Considerations

Voicemail drops exist in a legal gray area that has become less gray and more regulated over time. The compliance landscape as of 2026:

### The Core Question: Is a Voicemail Drop a "Call"?

The FCC has ruled that leaving a prerecorded voicemail constitutes a "call" under the TCPA. This means:

1. **Prior express consent** is required for non-emergency prerecorded messages to residential lines
2. **Prior express written consent** is required for prerecorded telemarketing messages to cell phones
3. The **National Do Not Call Registry** applies
4. **Identification requirements** apply — the message must identify the caller and provide a callback number

### Consent Requirements by Scenario

| Scenario | Consent Needed | Notes |
|----------|---------------|-------|
| B2C marketing to cell phone | Prior express written consent | Web form opt-in qualifies |
| B2C marketing to landline | Prior express consent (verbal OK) | Must have documentation |
| B2B to business line | Established business relationship exemption may apply | Verify per state |
| Existing customer relationship | Prior consent likely exists | Check original agreement |
| Political campaign | Exempt from many TCPA rules | Still needs caller ID |
| Debt collection | FDCPA overlays apply | Consult compliance counsel |

### Safe Harbor Considerations for Dropped Calls

When AMD fails and a live call is dropped (no agent available), the dropped call safe harbor requires:

1. A prerecorded message plays within 2 seconds of the person's greeting
2. The message identifies the caller and provides a callback number
3. The call is disconnected within 2 seconds of the message completing

This is distinct from voicemail drop — safe harbor applies to live answer drops, while voicemail drop applies to machine-detected calls. Configure both:

```sql
-- Safe harbor for dropped live calls
UPDATE vicidial_campaigns
SET drop_action = 'AUDIO',
    safe_harbor_audio = 'safe_harbor_msg',
    safe_harbor_exten = '8300'
WHERE campaign_id = 'SALESCAMP';
```

### State-Specific Requirements

Several states have additional restrictions beyond federal TCPA requirements:

- **California (CCPA):** Additional consent and disclosure requirements
- **Florida:** Stricter prerecorded message rules; requires prior express written consent for most prerecorded calls
- **New York:** Separate telemarketing registration and bond requirements
- **Indiana, Kentucky, Louisiana, North Dakota, Wyoming:** Various additional prerecorded message restrictions

If you're calling nationally, your voicemail drop message and consent documentation must comply with the strictest state in your calling area.

### Practical Compliance Checklist

Before enabling voicemail drops on any campaign:

- [ ] Verify prior express written consent exists for all leads (web form, signed agreement)
- [ ] Scrub against the National DNC Registry (updated within 31 days)
- [ ] Scrub against your internal DNC list
- [ ] Confirm voicemail drop message includes caller identity and callback number
- [ ] Verify message does not exceed 60 seconds (shorter is better)
- [ ] Document AMD accuracy testing results
- [ ] Review state-specific requirements for all states in your calling area
- [ ] Configure safe harbor message separately for live answer drops
- [ ] Set up call recording for compliance documentation

## Message Blasting vs. AMD-Triggered Drops

VICIdial supports two distinct approaches to automated voicemail delivery, and they serve very different purposes.

### AMD-Triggered Voicemail Drop

This is the standard approach described above:

1. Dialer places a normal outbound call
2. AMD analyzes the answer
3. If machine → play message and hang up
4. If human → route to agent

**Pros:**
- Live answers still reach agents — you don't sacrifice live connections
- Drop is triggered only on verified machine answers
- Agents handle the valuable human connections

**Cons:**
- AMD is imperfect — some live answers get voicemail dropped (false positives)
- Still consumes dialer capacity and trunks for machine-answered calls
- Requires careful AMD tuning

**Best for:** Active campaigns where agents are logged in and handling live calls. The voicemail drop is a secondary benefit of calls that would otherwise be wasted.

### Message Blasting

Message Blasting (also called "broadcast" or "blast dial") sends a prerecorded message to every answered call, regardless of whether a human or machine answers. There are no agents involved.

VICIdial supports this through the `campaign_vdad_exten` setting combined with a dedicated dialplan that plays audio on all answered calls:

```sql
-- Message blast campaign configuration
UPDATE vicidial_campaigns
SET
    dial_method = 'RATIO',
    auto_dial_level = 1.0,
    campaign_vdad_exten = '8368',  -- Non-AMD extension
    amd_send_to_vmx = 'N',        -- Not using AMD routing
    closer_campaigns = ''           -- No agent routing
WHERE campaign_id = 'MSGBLAST';
```

The dialplan for a message blast sends all answered calls to the audio playback:

```
[msg-blast]
exten => s,1,Answer()
exten => s,n,Wait(1)
exten => s,n,Playback(blast_message_audio)
exten => s,n,Wait(1)
exten => s,n,Hangup()
```

**Pros:**
- No agents needed — pure automated outreach
- 100% of answered calls hear the message
- Very high throughput — limited only by trunk capacity

**Cons:**
- Every live answer hears a prerecorded message — no live agent interaction
- Much stricter TCPA requirements (prior express written consent for all numbers)
- Higher complaint rates from consumers
- Many states have additional restrictions on prerecorded message broadcasts

**Best for:** Appointment reminders, event notifications, political GOTV messages, and other informational messages where agent interaction isn't the goal. Not recommended for sales campaigns — the conversion rate from a broadcast message is negligible compared to a live agent conversation.

### Hybrid Approach: AMD-Gated Blast with Press-1 Transfer

A middle ground that some operations use:

1. Call goes out through the dialer
2. AMD detects the answer
3. If MACHINE → play voicemail drop message
4. If HUMAN → play a brief recorded message with a "press 1 to speak with an agent" option
5. If the person presses 1 → transfer to an available agent

This requires more complex dialplan configuration:

```
[hybrid-vmdrop]
exten => s,1,Answer()
exten => s,n,AMD()
exten => s,n,GotoIf($["${AMDSTATUS}" = "MACHINE"]?machine:human)

exten => s,n(machine),Playback(vm_drop_message)
exten => s,n,Hangup()

exten => s,n(human),Playback(press1_intro_message)
exten => s,n,Read(DTMF_INPUT,,1,,,5)
exten => s,n,GotoIf($["${DTMF_INPUT}" = "1"]?transfer:hangup)

exten => s,n(transfer),Dial(Local/8300@from-internal,,tTo)
exten => s,n,Hangup()

exten => s,n(hangup),Playback(goodbye_message)
exten => s,n,Hangup()
```

This approach has specific TCPA implications — the initial prerecorded message to a live person still requires proper consent. Consult with your compliance team before implementing.

## Per-Campaign Voicemail Drop Strategies

### High-Value Lead Campaigns (Insurance, Financial)

For campaigns where each lead costs $5-50+:

- **AMD sensitivity:** Conservative (more false negatives tolerated, fewer false positives)
- **Message tone:** Professional, mention the specific information they requested
- **Message length:** 20-25 seconds
- **Strategy:** Drop voicemail only on first 2 attempts, then switch to agent-only for subsequent attempts

```sql
-- Check if lead has been voicemail-dropped before
-- Use this in custom logic to skip VM drop after 2 attempts
SELECT
    phone_number,
    COUNT(*) AS vm_drop_count
FROM vicidial_log
WHERE phone_number = '5551234567'
  AND status = 'AA'  -- AMD-detected answering machine
  AND campaign_id = 'INS_FRESH'
GROUP BY phone_number;
```

### High-Volume Cold Campaigns (Solar, Home Services)

For campaigns with cheap, replaceable lists:

- **AMD sensitivity:** Standard or slightly aggressive
- **Message tone:** Conversational, create curiosity
- **Message length:** 15-20 seconds
- **Strategy:** Voicemail drop on all attempts — maximize touchpoints

### Retention / Win-Back Campaigns

For calling existing or lapsed customers:

- **AMD sensitivity:** Very conservative — these people know you, and a dropped live call damages the relationship
- **Message tone:** Familiar, reference their history with you
- **Message length:** 20 seconds max
- **Strategy:** Personalize where possible — use the customer's name if your system supports TTS (Text-to-Speech) integration

### Political Campaigns

- **AMD sensitivity:** Standard
- **Message tone:** Varies by campaign type (voter ID vs. GOTV vs. fundraising)
- **Message length:** 15-25 seconds
- **Strategy:** Voicemail drop on every attempt. Volume matters. Coordinate message content with the overall campaign's media buy.

### Message Rotation

Leaving the same voicemail three times makes you sound robotic and decreases callback likelihood. VICIdial supports multiple audio files through dialplan logic:

```
[vm-drop-rotation]
exten => s,1,Answer()
exten => s,n,Wait(1)
; Randomly select from 3 messages
exten => s,n,Set(MSG=${RAND(1,3)})
exten => s,n,GotoIf($["${MSG}" = "1"]?msg1)
exten => s,n,GotoIf($["${MSG}" = "2"]?msg2)
exten => s,n,Goto(msg3)

exten => s,n(msg1),Playback(vm_drop_v1)
exten => s,n,Goto(done)

exten => s,n(msg2),Playback(vm_drop_v2)
exten => s,n,Goto(done)

exten => s,n(msg3),Playback(vm_drop_v3)
exten => s,n,Goto(done)

exten => s,n(done),Wait(1)
exten => s,n,Hangup()
```

Alternatively, use sequential rotation based on the number of previous voicemails left for that number. This requires more complex logic that queries the VICIdial database within the dialplan using `func_odbc` or an AGI script.

## Measuring Voicemail Drop Effectiveness

Track callback rates from voicemail drops to measure ROI:

```sql
-- Calls that came inbound within 48 hours of a voicemail drop
SELECT
    vl.phone_number,
    vl.call_date AS vm_drop_time,
    vcl.call_date AS callback_time,
    TIMESTAMPDIFF(HOUR, vl.call_date, vcl.call_date) AS hours_to_callback
FROM vicidial_log vl
JOIN vicidial_closer_log vcl
    ON vl.phone_number = vcl.phone_number
    AND vcl.call_date > vl.call_date
    AND vcl.call_date <= DATE_ADD(vl.call_date, INTERVAL 48 HOUR)
WHERE vl.status = 'AA'  -- AMD machine detection
  AND vl.call_date >= DATE_SUB(CURDATE(), INTERVAL 7 DAY)
  AND vl.campaign_id = 'SALESCAMP'
ORDER BY vl.call_date;
```

Healthy voicemail drop campaigns see 2-5% callback rates. If you're below 1%, your message needs work or your list quality is the bottleneck.

## How ViciStack Helps

Voicemail drop sits at the intersection of technical configuration (AMD tuning, dialplan programming, audio engineering), compliance (TCPA, state laws, consent documentation), and strategy (message content, rotation, per-campaign optimization). Getting any one of these wrong costs money or creates legal exposure.

ViciStack handles all three:

- **AMD optimization** — We tune AMD parameters based on your specific carrier and list characteristics, targeting 90%+ accuracy
- **Compliance review** — We verify your consent documentation, DNC scrubbing, and message content against current federal and state requirements
- **Message strategy** — We design per-campaign voicemail drop strategies including rotation, attempt limits, and callback tracking
- **Audio production guidance** — We spec the technical requirements for your recordings and verify proper Asterisk integration
- **Ongoing monitoring** — We track AMD accuracy, callback rates, and compliance metrics weekly

Flat rate $150/agent/month. No per-minute. No per-message.

**Find out if your voicemail drops are helping or hurting your campaigns** — get a free analysis: [Request Your Free ViciStack Analysis](https://vicistack.com/proof/)

## Related Resources

- [VICIdial Auto-Dial Level Tuning by Campaign Type](/blog/vicidial-auto-dial-level-tuning) — AMD and voicemail drop settings interact directly with dial level optimization
- [VICIdial Asterisk CDR Analysis for Connect Rate Optimization](/blog/vicidial-asterisk-cdr-analysis) — CDR data reveals AMD accuracy and voicemail drop impact on connect rates
- [VICIdial Pause Codes and Agent Accountability Systems](/blog/vicidial-pause-codes-accountability) — Agent behavior when receiving AMD false negatives (voicemail connects) affects accountability metrics

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-voicemail-drop).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
