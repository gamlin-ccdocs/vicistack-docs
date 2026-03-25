# How to Reduce VICIdial AMD False Positives from 20% to Under 5%

If you run a VICIdial outbound call center, you already know the pain of answering machine detection gone wrong. A live prospect picks up, says "Hello?" and your AMD algorithm decides it heard a voicemail greeting. The call gets dropped or sent to a message queue. That prospect never gets a callback. At scale -- say 50 agents making 200+ dials per hour -- a 20% false positive rate on AMD means you are burning through 40 live connections every single hour. Over an 8-hour shift, that is 320 lost conversations per day. At even a modest $5 revenue per live connection, you are leaving $1,600 on the table daily.

This guide walks through the exact AMD parameters inside VICIdial and Asterisk, how to tune them per carrier, how to build adaptive thresholds, and how to validate your changes with real recordings. We have applied this methodology across more than 100 VICIdial call centers, and consistently bring false positive rates from the 15-25% range down to 3-5%.

## Understanding How VICIdial AMD Works Under the Hood

VICIdial's answering machine detection relies on Asterisk's built-in CPD (Call Progress Detection) engine. When a call is answered, Asterisk listens to the initial audio and makes a binary decision: human or machine. It does this by analyzing speech patterns -- specifically, the length of words, the silence between words, and the total number of words spoken in the initial greeting.

The logic is straightforward. Humans typically answer with one or two short words: "Hello," "Yeah," or "This is Mike." Answering machines play a longer greeting: "Hi, you've reached the Johnson residence. We can't come to the phone right now..." The CPD engine counts words, measures their length, and checks silence gaps to distinguish between these two patterns.

The problem is that the default parameters were tuned for a generic North American English speaker on a landline in the early 2000s. Today's calls traverse VoIP networks with variable latency, hit cell phones with background noise, and reach voicemail systems that have gotten shorter and more conversational. The defaults produce false positive rates of 15-25% on most modern campaigns.

## The Four Core AMD Parameters

VICIdial exposes four critical CPD parameters through the system settings and campaign-level configuration. Each one directly controls how the detection algorithm interprets audio.

### cpd_amd_maximum_word_length

This parameter defines the maximum duration (in milliseconds) that a single "word" can last before the system considers it part of a machine greeting. The default is typically 5000ms (5 seconds).

**Why it matters:** If a human says "Helloooo?" and draws it out to 600ms, a tight maximum_word_length setting might flag it as a machine word. Conversely, if the threshold is too loose, a machine greeting where words blend together will pass as human.

**Recommended starting point:** 2000ms

```
; In your Asterisk configuration or VICIdial system settings
cpd_amd_maximum_word_length = 2000
```

Reduce this value to make AMD more aggressive at catching machines (but risk more false positives on humans who speak slowly). Increase it to be more permissive (fewer false positives, but more machines slip through as human).

### cpd_amd_minimum_word_length

The minimum duration a sound must last to be counted as a "word." Sounds shorter than this threshold are treated as noise or clicks and ignored. The default is often 100ms.

**Why it matters:** Network artifacts, line clicks, and brief mouth sounds can be miscounted as words. If minimum_word_length is too low, background noise on a cell phone call gets counted as speech, inflating the word count and triggering a machine classification.

**Recommended starting point:** 120ms

```
cpd_amd_minimum_word_length = 120
```

On carriers with noisy connections or a lot of cell phone traffic, push this up to 140-160ms. On clean SIP trunks to mostly landlines, you can drop it to 100ms.

### cpd_amd_between_words_silence

The duration of silence (in milliseconds) between sounds that the system uses to determine where one "word" ends and the next begins. The default is usually 50ms.

**Why it matters:** This is arguably the most impactful parameter. If between_words_silence is too short, the system will merge multiple words into one long word, making a human greeting look like a machine greeting. If it is too long, the system will split a single word into multiple words, making a machine greeting look like a human saying multiple short words.

**Recommended starting point:** 70ms

```
cpd_amd_between_words_silence = 70
```

This parameter is highly carrier-dependent. VoIP carriers that use aggressive jitter buffers or silence suppression will create artificial gaps in audio, artificially inflating word counts. We have seen cases where a single carrier change required a 30ms adjustment to this value.

### cpd_amd_maximum_number_of_words

The maximum number of words allowed before the system classifies the call as a machine. If the initial greeting exceeds this word count, it is flagged as an answering machine. The default is typically 3.

**Why it matters:** Most humans answer with 1-2 words. Most answering machine greetings contain 8-20 words. A threshold of 3 catches the majority of machines while allowing for humans who say "Hello, this is Mike" (3 words). Setting it to 2 would catch more machines but would also flag any human who identifies themselves.

**Recommended starting point:** 3

```
cpd_amd_maximum_number_of_words = 3
```

For B2B campaigns where people often answer with their name and company ("Mike Johnson, Acme Corp"), consider raising this to 4. For residential campaigns where people tend to just say "Hello," you could tighten this to 2 -- but test carefully first.

## The Per-Carrier Tuning Approach

Here is where most VICIdial admins leave massive performance on the table. They set AMD parameters globally and call it done. But different SIP carriers have radically different audio characteristics.

### Why Carriers Differ

Each SIP carrier has its own:

- **Codec negotiation preferences** -- G.711 ulaw vs G.729 vs Opus all produce different audio characteristics
- **Jitter buffer implementation** -- affects silence gaps and word boundaries
- **Silence suppression (VAD)** -- some carriers strip silence from the audio stream, which destroys between_words_silence detection
- **Post-dial delay** -- affects when the CPD engine starts listening
- **Comfort noise generation** -- injects artificial background noise that can be counted as speech

### Building a Per-Carrier Tuning Matrix

The approach is to run controlled tests against each carrier independently and build a tuning matrix.

**Step 1: Identify your carriers.** List every SIP trunk you route outbound calls through. In Asterisk, check your `sip.conf` or `pjsip.conf`:

```bash
grep -E "^\[trunk" /etc/asterisk/sip.conf
# or for pjsip
grep -E "^\[carrier" /etc/asterisk/pjsip.conf
```

In VICIdial, check Admin > Carriers for your active carrier configurations.

**Step 2: Tag calls by carrier.** Modify your VICIdial dial plan or use the carrier_id field in the hopper to track which carrier handled each call. You need this to correlate AMD outcomes with specific trunks.

**Step 3: Pull AMD outcomes per carrier.** Query the vicidial_log table, joining against vicidial_carrier_log to identify the trunk:

```sql
SELECT
    SUBSTRING_INDEX(cl.channel, '/', 2) AS carrier_trunk,
    COUNT(*) as total_calls,
    SUM(CASE WHEN vl.status = 'AA' THEN 1 ELSE 0 END) as amd_machine,
    SUM(CASE WHEN vl.status = 'AL' THEN 1 ELSE 0 END) as amd_live,
    ROUND(SUM(CASE WHEN vl.status = 'AA' THEN 1 ELSE 0 END) / COUNT(*) * 100, 1) as machine_pct
FROM vicidial_log vl
JOIN vicidial_carrier_log cl ON vl.uniqueid = cl.uniqueid
WHERE vl.call_date > DATE_SUB(NOW(), INTERVAL 24 HOUR)
    AND vl.status IN ('AA', 'AL', 'A', 'B', 'N', 'NA')
GROUP BY carrier_trunk
ORDER BY total_calls DESC;
```

**Step 4: Compare against known baselines.** For most B2C campaigns in the US, you should expect 40-60% of answered calls to be actual answering machines. If a carrier shows 70%+ machine rate, AMD is likely over-classifying. If it shows under 30%, AMD is likely under-classifying.

**Step 5: Tune per carrier.** VICIdial allows you to set AMD parameters at the campaign level. If you route specific campaigns through specific carriers, you can effectively tune per carrier. For more granular control, you can modify the Asterisk dial plan to set AMD parameters dynamically based on the outbound trunk:

```
; In extensions.conf, before the AMD application
exten => _NXXNXXXXXX,1,Set(CARRIER=${CHANNEL(peername)})
exten => _NXXNXXXXXX,n,GotoIf($["${CARRIER}" = "carrier_a"]?carrier_a_amd)
exten => _NXXNXXXXXX,n,GotoIf($["${CARRIER}" = "carrier_b"]?carrier_b_amd)
exten => _NXXNXXXXXX,n,Goto(default_amd)

exten => _NXXNXXXXXX,n(carrier_a_amd),Set(cpd_amd_between_words_silence=85)
exten => _NXXNXXXXXX,n,Set(cpd_amd_maximum_word_length=2200)
exten => _NXXNXXXXXX,n,Goto(run_amd)

exten => _NXXNXXXXXX,n(carrier_b_amd),Set(cpd_amd_between_words_silence=60)
exten => _NXXNXXXXXX,n,Set(cpd_amd_maximum_word_length=1800)
exten => _NXXNXXXXXX,n,Goto(run_amd)

exten => _NXXNXXXXXX,n(default_amd),Set(cpd_amd_between_words_silence=70)
exten => _NXXNXXXXXX,n,Set(cpd_amd_maximum_word_length=2000)

exten => _NXXNXXXXXX,n(run_amd),AMD()
```

## Real-Time Adaptive Thresholds

Static tuning gets you from 20% down to maybe 8-10%. To push below 5%, you need adaptive thresholds that respond to changing conditions throughout the day.

### The Feedback Loop Concept

The idea is simple: monitor AMD outcomes in real time, detect when the false positive rate drifts upward, and adjust parameters automatically. This requires:

1. **A scoring mechanism** -- agents disposition calls as "actually was a machine" or "actually was a human" when they connect
2. **A monitoring script** -- polls the database every few minutes and calculates the current false positive rate
3. **An adjustment engine** -- modifies AMD parameters when the rate exceeds a threshold

### Implementing a Basic Adaptive System

Here is a simplified monitoring script that checks AMD accuracy every 5 minutes:

```bash
#!/bin/bash
# amd_monitor.sh - Run via cron every 5 minutes
# Checks AMD false positive rate and adjusts parameters

MYSQL_CMD="mysql -u cron -pcronpass -D asterisk -N -e"

# Get false positive rate for last 30 minutes
# AMD statuses: AA = AMD machine, AL = AMD live agent connect
# If agents frequently disposition AA-routed calls as live answers, AMD is over-classifying
FP_RATE=$($MYSQL_CMD "
    SELECT ROUND(
        SUM(CASE WHEN status = 'AA' THEN 1 ELSE 0 END) /
        NULLIF(SUM(CASE WHEN status IN ('AA','AL') THEN 1 ELSE 0 END), 0) * 100, 1
    ) FROM vicidial_log
    WHERE call_date > DATE_SUB(NOW(), INTERVAL 30 MINUTE)
    AND status IN ('AA','AL');
")

# Get current between_words_silence setting from Asterisk AstDB
CURRENT_BWS=$(asterisk -rx "database get cpd amd_between_words_silence" 2>/dev/null | grep -oP '\d+')
[ -z "$CURRENT_BWS" ] && CURRENT_BWS=50  # Default if not set

# If false positive rate exceeds 8%, loosen the parameters
if (( $(echo "$FP_RATE > 8.0" | bc -l) )); then
    NEW_BWS=$((CURRENT_BWS + 5))
    # Cap at 120ms to prevent machines from slipping through
    if [ $NEW_BWS -le 120 ]; then
        asterisk -rx "database put cpd amd_between_words_silence $NEW_BWS"
        logger "AMD Monitor: FP rate ${FP_RATE}%, increased between_words_silence to ${NEW_BWS}ms"
    fi
fi

# If false positive rate is under 3%, tighten slightly to catch more machines
if (( $(echo "$FP_RATE < 3.0" | bc -l) )); then
    NEW_BWS=$((CURRENT_BWS - 3))
    # Floor at 40ms
    if [ $NEW_BWS -ge 40 ]; then
        asterisk -rx "database put cpd amd_between_words_silence $NEW_BWS"
        logger "AMD Monitor: FP rate ${FP_RATE}%, decreased between_words_silence to ${NEW_BWS}ms"
    fi
fi
```

This is a simplified example. A production system would also adjust maximum_word_length and maximum_number_of_words, implement rate limiting on changes (no more than one adjustment per 15 minutes), and include safeguards against oscillation.

### Time-of-Day Patterns

AMD accuracy varies throughout the day. During morning hours (8-10 AM), you reach more voicemail because people are commuting or in meetings. During evening hours (5-7 PM), more people answer, but they often answer with background noise (TV, kids, cooking). Your adaptive system should account for these patterns.

Consider maintaining separate parameter profiles for different time blocks:

```sql
-- Example: time-based AMD profiles
-- Morning (8-11 AM): More aggressive machine detection
-- Afternoon (11 AM - 4 PM): Balanced
-- Evening (4-8 PM): More permissive to account for background noise

CREATE TABLE amd_time_profiles (
    time_start TIME,
    time_end TIME,
    cpd_amd_between_words_silence INT,
    cpd_amd_maximum_word_length INT,
    cpd_amd_maximum_number_of_words INT,
    cpd_amd_minimum_word_length INT
);

INSERT INTO amd_time_profiles VALUES
('08:00:00', '11:00:00', 65, 1800, 3, 130),
('11:00:00', '16:00:00', 70, 2000, 3, 120),
('16:00:00', '20:00:00', 80, 2200, 4, 110);
```

## Testing Methodology with Recordings

You cannot tune AMD blindly. You need a systematic testing approach using real call recordings.

### Building a Test Corpus

**Step 1: Pull recordings from your VICIdial server.** VICIdial stores recordings in `/var/spool/asterisk/monitorDONE/` by default. Pull a sample of 200-300 recordings:

- 100 calls that AMD classified as machine (status AA)
- 100 calls that AMD classified as human (status AL)
- 50-100 calls where the agent dispositioned differently than AMD predicted

```bash
# Pull recent recordings classified as answering machine
find /var/spool/asterisk/monitorDONE/ -name "*.wav" -mtime -1 | head -100 > /tmp/test_corpus_machine.txt

# Pull recent recordings classified as live
find /var/spool/asterisk/monitorDONE/ -name "*.wav" -mtime -1 | tail -100 > /tmp/test_corpus_live.txt
```

**Step 2: Manually review and label.** Yes, this is tedious. Listen to each recording for the first 3 seconds and label it as HUMAN or MACHINE. This gives you ground truth to test against.

**Step 3: Run AMD against the corpus with different parameters.** Asterisk provides a command-line way to test AMD against a recording file:

```bash
# Test a recording against current AMD settings
asterisk -rx "channel originate Local/s@amd-test application Playback /path/to/recording"
```

For batch testing, write a script that iterates through your corpus with different parameter combinations:

```bash
#!/bin/bash
# amd_batch_test.sh - Test AMD parameters against labeled recordings

RESULTS_FILE="/tmp/amd_test_results.csv"
echo "file,actual,predicted,bws,mwl,mnw" > $RESULTS_FILE

for BWS in 50 60 70 80 90; do
    for MWL in 1500 1800 2000 2200 2500; do
        for MNW in 2 3 4; do
            # Update AMD parameters
            asterisk -rx "database put cpd amd_between_words_silence $BWS"
            asterisk -rx "database put cpd amd_maximum_word_length $MWL"
            asterisk -rx "database put cpd amd_maximum_number_of_words $MNW"

            # Test against corpus
            while IFS=, read -r file actual_label; do
                predicted=$(test_amd_on_file "$file")
                echo "$file,$actual_label,$predicted,$BWS,$MWL,$MNW" >> $RESULTS_FILE
            done < /tmp/labeled_corpus.csv
        done
    done
done
```

**Step 4: Analyze results.** Calculate false positive rate (humans classified as machines) and false negative rate (machines classified as humans) for each parameter combination. Find the sweet spot where false positives are under 5% without letting too many machines through.

### The Confusion Matrix

For each parameter set, build a confusion matrix:

```
                    Predicted Human    Predicted Machine
Actual Human        True Positive      FALSE POSITIVE (bad!)
Actual Machine      False Negative     True Negative
```

**False positives** are your primary concern -- these are lost live connections. A 5% false positive rate on 1000 daily live answers means 50 lost conversations.

**False negatives** matter too, but less. If a machine slips through as human, an agent hears a voicemail greeting, dispositions it, and moves on. It wastes 3-5 seconds of agent time, not the entire lead.

Optimize for minimizing false positives first, false negatives second.

## Before and After: Real-World Results

Here is what we typically see when applying this full methodology to a VICIdial call center running 50+ agents:

### Before Tuning (Default Parameters)

| Metric | Value |
|--------|-------|
| Total daily dials | 12,000 |
| Answered calls | 4,800 (40%) |
| AMD: classified as machine | 2,880 (60% of answered) |
| AMD: classified as human | 1,920 (40% of answered) |
| False positive rate | 18.5% |
| Live connections lost to false positives | 533/day |
| Agent talk time utilization | 62% |

### After Tuning (Per-Carrier + Adaptive)

| Metric | Value |
|--------|-------|
| Total daily dials | 12,000 |
| Answered calls | 4,800 (40%) |
| AMD: classified as machine | 2,640 (55% of answered) |
| AMD: classified as human | 2,160 (45% of answered) |
| False positive rate | 4.2% |
| Live connections lost to false positives | 111/day |
| Agent talk time utilization | 78% |

The difference: **422 additional live connections per day** reaching agents. At even a conservative $3 per live connection value, that is $1,266 per day or **$27,852 per month in recovered revenue** -- from parameter tuning alone.

## Common Pitfalls to Avoid

**1. Tuning in production during peak hours.** Always test parameter changes during low-volume periods first. A bad parameter change during peak dialing can spike false positives across thousands of calls before you catch it.

**2. Ignoring codec differences.** If you switch carriers or change codec preferences, re-tune AMD. G.729 compression changes audio characteristics enough to shift word boundary detection by 20-30ms.

**3. Not accounting for IVR pre-greetings.** Some cell phone carriers play a brief tone or message ("The subscriber you are calling...") before the actual person answers. This pre-greeting can be counted as machine speech. Work with your carriers to understand their pre-connect audio behavior.

**4. Over-tuning on a small sample.** Never tune based on fewer than 500 calls per parameter set. Statistical noise in small samples will lead you to parameters that perform well on the test set but poorly in production.

**5. Forgetting about DNC and compliance implications.** If AMD drops a call classified as machine, but it was actually a human, you may have just made a "dead air" call that counts against your abandoned call rate. FTC regulations limit abandoned calls to 3% of answered calls. False positives can push you over this limit.

## How ViciStack Helps

Everything described in this article -- per-carrier tuning, adaptive thresholds, recording analysis, and ongoing monitoring -- is part of what we deploy for every ViciStack customer. Our managed optimization service includes:

- **Initial AMD audit** with recording-level analysis of your current false positive rate
- **Per-carrier parameter tuning** based on your specific trunk configuration
- **Adaptive monitoring** that adjusts parameters in real time throughout the day
- **Monthly re-tuning** as carrier networks and call patterns change
- **Flat pricing at $150/agent/month** -- no per-minute charges, no surprises

We have tuned AMD for over 100 VICIdial call centers. The median improvement is a reduction from 18% false positives to 4.1%.

**[Get your free VICIdial analysis -- we will measure your current AMD false positive rate and show you exactly what is recoverable.](https://vicistack.com/proof/)**

We respond within 5 minutes during business hours. No commitment, no credit card. Just data.

## Further Reading

- [VICIdial Caller ID Reputation Monitoring and Recovery Guide](/blog/vicidial-caller-id-reputation) -- false positives are not the only thing killing your contact rates
- [VICIdial Answering Machine Detection vs AI-Based AMD: Which Is Better?](/blog/vicidial-amd-vs-ai-amd) -- when traditional AMD tuning is not enough, AI-based alternatives can push accuracy above 95%
- [VICIdial Kamailio Load Balancing for 100+ Agent Call Centers](/blog/vicidial-kamailio-load-balancing) -- scaling your infrastructure affects AMD performance across servers

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-amd-false-positive-reduction).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
