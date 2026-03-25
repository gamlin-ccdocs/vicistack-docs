# VICIdial ROI Case Study: How One Center Doubled Connects in 14 Days

Numbers don't lie. This case study walks through exactly how a 45-agent insurance call center went from a 3.2% connect rate to 7.1% in 14 days using ViciStack's VICIdial optimization -- and what that meant for their bottom line.

We're sharing the day-by-day metrics, the specific changes we made, and the reasoning behind each optimization. If you're running a VICIdial-based call center and your connect rates are stuck below 5%, everything in this article is directly applicable to your operation.

## Background: The Starting Point

**The center:** A Medicare supplement insurance operation based in the Southeast, running 45 agents on VICIdial across two campaigns (inbound and outbound). They'd been on VICIdial for three years with a self-hosted setup managed by an in-house IT generalist.

**The problem:** Connect rates had been declining for months. When they reached out to ViciStack, they were at 3.2% -- meaning for every 100 calls the dialer placed, only 3.2 resulted in a live agent-to-human conversation. The rest were answering machines, dead air, busy signals, disconnected numbers, or drops.

**What 3.2% means in real terms:**

| Metric | Value |
|--------|-------|
| Daily outbound attempts | ~42,000 |
| Daily live connects | ~1,344 |
| Connects per agent per day | ~30 |
| Average talk time per connect | 4.2 minutes |
| Sales conversion rate (of connects) | 8.5% |
| Daily sales | ~114 |
| Revenue per sale | $320 (first-year commission) |
| Daily revenue | ~$36,480 |

The numbers looked acceptable in isolation. The problem was that they were leaving massive amounts of money on the table. Their agents were spending more time waiting than talking, their AMD was killing live calls, and their DID reputation was in the gutter.

## The Problem Analysis

We started with a comprehensive audit of their VICIdial configuration, carrier setup, and call data. Here's what we found.

### Problem 1: AMD False Positives (Estimated Impact: 35% of Lost Connects)

Their AMD settings were using VICIdial's default CPD parameters:

```
CPD Settings (as found):
  initial_silence: 2500
  greeting: 1500
  after_greeting_silence: 800
  total_analysis_time: 5000
  minimum_word_length: 100
  between_words_silence: 50
  maximum_number_of_words: 3
  silence_threshold: 256
```

The default `initial_silence` of 2500ms was too aggressive for their demographic (Medicare-eligible adults, age 65+). Older populations often take longer to answer and have longer greeting pauses. The system was classifying 8-12% of live humans as answering machines.

We confirmed this by analyzing call duration patterns:

```sql
-- Calls dispositioned as Answering Machine with suspiciously short durations
-- These are likely live humans killed by AMD
SELECT
    DATE(call_date) AS day,
    COUNT(*) AS amd_dispos,
    SUM(CASE WHEN length_in_sec BETWEEN 1 AND 4 THEN 1 ELSE 0 END) AS under_4sec,
    SUM(CASE WHEN length_in_sec BETWEEN 4 AND 8 THEN 1 ELSE 0 END) AS sec_4_to_8,
    ROUND(SUM(CASE WHEN length_in_sec BETWEEN 1 AND 4 THEN 1 ELSE 0 END) / NULLIF(COUNT(*), 0) * 100, 1) AS pct_under_4sec
FROM vicidial_log
WHERE status IN ('AA','AM')
  AND call_date >= DATE_SUB(CURDATE(), INTERVAL 7 DAY)
GROUP BY DATE(call_date)
ORDER BY day;
```

Results showed 9.3% of all calls were being AMD-killed within 4 seconds -- a clear sign of false positives. For their volume of 42,000 daily attempts, that's approximately 3,900 calls per day where a live human was potentially being dropped.

### Problem 2: Flagged and Burned DIDs (Estimated Impact: 40% of Lost Connects)

Their outbound caller ID setup was using a rotation of 20 local DIDs that had been in service for over a year without rotation. We checked the reputation:

```sql
-- Answer rate by caller ID number
SELECT
    vdl.outbound_cid AS caller_id,
    COUNT(*) AS attempts,
    SUM(CASE WHEN vl.status NOT IN ('A','AA','AM','AL','N','NI','NP','B','DC','NA','ADC') THEN 1 ELSE 0 END) AS answers,
    ROUND(SUM(CASE WHEN vl.status NOT IN ('A','AA','AM','AL','N','NI','NP','B','DC','NA','ADC') THEN 1 ELSE 0 END) / NULLIF(COUNT(*), 0) * 100, 2) AS answer_rate
FROM vicidial_dial_log vdl
JOIN vicidial_log vl ON vdl.uniqueid = vl.uniqueid
WHERE vl.call_date >= DATE_SUB(CURDATE(), INTERVAL 7 DAY)
GROUP BY vdl.outbound_cid
ORDER BY answer_rate ASC;
```

The results were damning. Their worst DIDs had answer rates below 2%. Their best were at 5.8%. For reference, a clean DID in their market should achieve 8-12% answer rates. Every single one of their 20 DIDs had been flagged as spam by one or more carrier databases (Hiya, Nomorobo, First Orion, TNS Call Guardian).

### Problem 3: Suboptimal Predictive Pacing (Estimated Impact: 25% of Lost Connects)

Their campaign was configured with:

```
Dial Method: RATIO
Auto Dial Level: 3.0 (fixed)
```

They were using RATIO mode instead of ADAPT, meaning the system dialed exactly 3 lines per available agent regardless of conditions. During the first hour of the shift when all 45 agents logged in simultaneously, this created a massive spike in outbound attempts -- far more than agents could handle. The result was a 7.8% drop rate during the first hour (well above the 3% compliance target) and wasted calls that burned through their best leads.

Later in the day, when agents were in various states (talking, paused, wrapping up), the 3:1 ratio was too conservative, leaving agents waiting 15-25 seconds between calls.

## The Solution: ViciStack Optimization

We implemented changes in three phases over 14 days, carefully measuring the impact of each change before moving to the next.

### Days 1-3: AMD Tuning

We adjusted CPD parameters specifically for their Medicare-age demographic:

```
CPD Settings (optimized):
  initial_silence: 3200    (was 2500 -- more time for elderly pickup)
  greeting: 2000           (was 1500 -- longer greeting tolerance)
  after_greeting_silence: 1100  (was 800 -- pause tolerance for older speakers)
  total_analysis_time: 5500     (was 5000 -- slightly more analysis time)
  minimum_word_length: 120      (was 100 -- filter very short sounds)
  between_words_silence: 75     (was 50 -- natural speech pauses)
  maximum_number_of_words: 4    (was 3 -- some greetings are longer)
  silence_threshold: 220        (was 256 -- slightly more sensitive)
```

We also implemented a post-AMD verification step using VICIdial's message detection feature. Instead of immediately dropping AMD-detected calls, the system waited an additional 1.5 seconds listening for live speech patterns before disconnecting. This caught approximately 60% of false positives with minimal impact on agent efficiency.

**Day 1 Results:**

| Metric | Before | Day 1 |
|--------|--------|-------|
| Connect rate | 3.2% | 3.8% |
| AMD false positive rate (estimated) | 9.3% | 5.1% |
| Agent connects/day | 30 | 35 |

**Day 3 Results (after fine-tuning):**

| Metric | Before | Day 3 |
|--------|--------|-------|
| Connect rate | 3.2% | 4.1% |
| AMD false positive rate (estimated) | 9.3% | 3.2% |
| AMD accuracy | ~85% | 93.1% |
| Agent connects/day | 30 | 38 |

### Days 4-7: DID Reputation Recovery

This was the most impactful change. We implemented a three-pronged DID strategy:

**1. Replaced all 20 burned DIDs** with fresh numbers from a clean DID provider. We selected numbers that:
- Had no prior outbound call history
- Were from local area codes matching their target geography
- Were spread across multiple underlying carriers to avoid single-carrier flagging

**2. Implemented intelligent DID rotation** using VICIdial's CID Group Rotation feature:

```
Campaign > Caller ID:
  Use Custom CID: Y
  CID Group: MEDICARE_ROTATION

CID Group Configuration:
  CID Count: 60 (3x their previous pool)
  Rotation Method: ROUND_ROBIN
  Max Calls Per CID Per Hour: 30
  Max Calls Per CID Per Day: 150
```

The key setting: capping each DID at 30 calls/hour and 150 calls/day. Carrier spam algorithms flag numbers that make more than ~200 calls/day from a single number. By spreading volume across 60 DIDs, no single number exceeded the threshold.

**3. Set up DID health monitoring** using a daily automated check:

```bash
#!/bin/bash
# Daily DID health check - runs at 6 AM before dialing starts
# Checks answer rate per DID from previous day

mysql -u reports -p'password' asterisk -e "
SELECT
    vdl.outbound_cid,
    COUNT(*) AS attempts,
    SUM(CASE WHEN vl.status NOT IN ('A','AA','AM','AL','N','NI','NP','B','DC','NA','ADC') THEN 1 ELSE 0 END) AS answers,
    ROUND(SUM(CASE WHEN vl.status NOT IN ('A','AA','AM','AL','N','NI','NP','B','DC','NA','ADC') THEN 1 ELSE 0 END) / NULLIF(COUNT(*), 0) * 100, 2) AS answer_rate
FROM vicidial_dial_log vdl
JOIN vicidial_log vl ON vdl.uniqueid = vl.uniqueid
WHERE vl.call_date >= DATE_SUB(CURDATE(), INTERVAL 1 DAY)
  AND vl.call_date < CURDATE()
GROUP BY vdl.outbound_cid
HAVING answer_rate < 4.0
ORDER BY answer_rate ASC;
" | while read line; do
    echo "$(date): LOW ANSWER RATE DID: $line" >> /var/log/did-health.log
done
```

Any DID dropping below 4% answer rate gets flagged for investigation and potential replacement.

**Day 7 Results:**

| Metric | Before | Day 3 | Day 7 |
|--------|--------|-------|-------|
| Connect rate | 3.2% | 4.1% | 5.9% |
| DID average answer rate | 4.1% | 4.1% | 9.2% |
| Agent connects/day | 30 | 38 | 54 |
| Daily sales | 114 | 137 | 195 |

The DID change alone pushed connect rates from 4.1% to 5.9% -- a 44% improvement. This is the single most common optimization we see across call centers. Most operations are dialing with burned numbers and don't realize it.

### Days 8-14: Predictive Algorithm Optimization

With AMD and DIDs optimized, we turned to the dialing algorithm itself.

**Changed from RATIO to ADAPT_TAPERED:**

```
Dial Method: ADAPT_TAPERED
Auto Dial Level: 1.0 (starting point -- system auto-adjusts)
Adaptive Maximum Level: 8.0
Adaptive Dropped Percentage: 3.00
Adaptive Target Drop Rate: 2.00
Adaptive Intensity: 30
Adaptive DL Diff Target: 1
```

Key changes and reasoning:

- **ADAPT_TAPERED** gradually ramps up dial ratio over the first 30 minutes of a shift, preventing the drop rate spike they were experiencing with fixed RATIO mode at shift start.
- **Adaptive Maximum Level of 8.0** (vs their fixed 3.0) allows the system to dial more aggressively when agents are available and data supports it. With improved AMD and clean DIDs, higher ratios produce more connects without proportionally more drops.
- **Target Drop Rate of 2.00%** keeps them comfortably under the 3% FCC threshold while maximizing throughput.
- **Intensity of 30** controls how quickly the system adjusts. We started conservative and would increase if needed.

We also optimized the hopper settings:

```
Campaign > Hopper:
  Hopper Level: 500 (was 200)
  Lead Order: UP PHONE with 2nd NEW
  Lead Recycling: Enabled
    - NA: recycle after 2 hours
    - B: recycle after 1 hour
    - A: recycle after 4 hours (AMD)
    - DROP: recycle after 30 minutes
```

Increasing the hopper level from 200 to 500 ensured the dialer always had enough leads queued to support the higher dial ratios without stalling. The lead recycling rules ensured no-answer and busy leads got retried at appropriate intervals rather than being wasted.

**Day 10 Results:**

| Metric | Before | Day 7 | Day 10 |
|--------|--------|-------|--------|
| Connect rate | 3.2% | 5.9% | 6.5% |
| Drop rate | 4.1% | 2.8% | 1.9% |
| Agent wait time (avg) | 18 sec | 14 sec | 9 sec |
| Agent connects/day | 30 | 54 | 59 |

**Day 14 Final Results:**

| Metric | Before | Day 14 | Change |
|--------|--------|--------|--------|
| Connect rate | 3.2% | 7.1% | +122% |
| AMD accuracy | ~85% | 94.3% | +9.3 points |
| Drop rate | 4.1% | 1.7% | -59% |
| Agent wait time (avg) | 18 sec | 7 sec | -61% |
| Agent connects/day | 30 | 65 | +117% |
| Daily sales | 114 | 235 | +106% |
| Agent utilization | 61% | 78% | +17 points |

## ROI Calculation

Let's do the math at $150/agent/month.

### Costs

| Item | Monthly |
|------|---------|
| ViciStack: 45 agents x $150 | $6,750 |
| Additional DIDs (40 new numbers) | $80 |
| **Total new monthly cost** | **$6,830** |

They were previously spending approximately $3,500/month on server hosting and $2,000/month on their IT person's time allocated to VICIdial management, plus $500/month on DIDs. So their previous VICIdial-related spend was about $6,000/month.

**Net incremental cost: $830/month.**

### Revenue Impact

| Metric | Before | After | Delta |
|--------|--------|-------|-------|
| Daily sales | 114 | 235 | +121 |
| Monthly sales (22 work days) | 2,508 | 5,170 | +2,662 |
| Revenue per sale | $320 | $320 | -- |
| Monthly revenue | $802,560 | $1,654,400 | +$851,840 |

**ROI: $851,840 additional monthly revenue for $830 additional monthly cost.**

Even if you're conservative and assume that only half the improvement came from ViciStack (the other half from market conditions, list quality, etc.), the ROI is still over 500:1.

### Revenue Per Agent Per Day

This is the metric most call center operators track closest:

| Metric | Before | After |
|--------|--------|-------|
| Connects per agent/day | 30 | 65 |
| Sales per agent/day | 2.5 | 5.2 |
| Revenue per agent/day | $800 | $1,664 |
| Revenue per agent/month | $17,600 | $36,608 |

At $150/agent/month, ViciStack's cost represents 0.4% of the revenue each agent generates. There's no investment in a call center that comes close to this return.

## Day-by-Day Improvement Timeline

For operators who want to see the progression:

| Day | Connect Rate | AMD Accuracy | Drop Rate | Connects/Agent | What Changed |
|-----|-------------|-------------|-----------|----------------|-------------|
| 0 (baseline) | 3.2% | ~85% | 4.1% | 30 | -- |
| 1 | 3.8% | 89.2% | 3.9% | 35 | AMD initial tuning |
| 2 | 3.9% | 91.4% | 3.8% | 36 | AMD refinement |
| 3 | 4.1% | 93.1% | 3.7% | 38 | AMD fine-tuning complete |
| 4 | 4.4% | 93.0% | 3.5% | 40 | First 20 new DIDs deployed |
| 5 | 5.1% | 93.2% | 3.2% | 46 | Full 60-DID rotation active |
| 6 | 5.6% | 93.4% | 2.9% | 51 | DID settling + rotation calibration |
| 7 | 5.9% | 93.5% | 2.8% | 54 | DID optimization complete |
| 8 | 6.0% | 93.6% | 2.4% | 55 | Switched to ADAPT_TAPERED |
| 9 | 6.2% | 93.8% | 2.1% | 56 | Hopper + recycling tuned |
| 10 | 6.5% | 94.0% | 1.9% | 59 | Adaptive intensity adjusted |
| 11 | 6.7% | 94.1% | 1.8% | 61 | Lead order optimization |
| 12 | 6.8% | 94.1% | 1.8% | 62 | Filter rules refined |
| 13 | 7.0% | 94.2% | 1.7% | 64 | Final pacing adjustments |
| 14 | 7.1% | 94.3% | 1.7% | 65 | Stabilized |

Note the pattern: AMD changes showed immediate but moderate improvement (Days 1-3). DID changes showed the largest single jump (Days 4-7). Predictive algorithm optimization provided the final push and improved compliance metrics (Days 8-14).

## Lessons Learned

### Lesson 1: DID Reputation is the #1 Silent Killer

Most centers don't monitor DID health. They buy numbers, use them until clients complain about low volume, and never connect the dots. In our experience across 100+ centers, DID reputation issues account for 30-50% of lost connects. It's almost always the biggest single optimization.

### Lesson 2: Default AMD Settings Are Wrong for Everyone

VICIdial's default CPD parameters are a compromise designed to work "okay" across all demographics. They're not optimized for anyone. Every population, every region, every time of day has different answer patterns. AMD tuning should be campaign-specific and reviewed monthly.

For a comprehensive guide, see our [AMD Optimization article](/blog/vicidial-amd-optimization).

### Lesson 3: Fixed Ratio Dialing is Leaving Money on the Table

RATIO mode with a fixed dial level is the most common VICIdial misconfiguration we see. ADAPT_TAPERED exists specifically because fixed ratios create problems: too aggressive at shift start (drops), too conservative mid-shift (agent wait time). If you're running RATIO mode, switch to ADAPT. It's free, it's built in, and it works better in virtually every scenario.

### Lesson 4: Optimization is Iterative, Not One-Shot

We didn't change everything on Day 1. We made one category of change, measured the impact, then moved to the next. This approach has two benefits: you can attribute improvement to specific changes (so you know what's working), and you avoid the risk of multiple changes interacting in unexpected ways.

### Lesson 5: The Math Always Works at Scale

A 1% connect rate improvement on 42,000 daily attempts is 420 additional connects. At their 8.5% sales conversion rate, that's 35.7 extra sales per day, or $11,424 in daily revenue. At $150/agent/month, ViciStack costs $225/day for 45 agents. The math works at almost any scale above 20 agents.

## Applicability to Your Center

This case study was an insurance operation, but the optimization principles apply across industries:

| Vertical | Typical Pre-Optimization Connect Rate | Post-ViciStack Connect Rate |
|---------|--------------------------------------|---------------------------|
| Insurance (Medicare, ACA) | 2.5-4.0% | 6.0-8.5% |
| Solar/Home Improvement | 3.0-4.5% | 6.5-9.0% |
| Debt Settlement/Financial | 2.0-3.5% | 5.5-7.5% |
| Real Estate | 3.5-5.0% | 7.0-10.0% |
| B2B Appointment Setting | 4.0-6.0% | 8.0-12.0% |
| Political/Polling | 2.5-4.0% | 5.5-8.0% |

The specifics change -- different CPD parameters, different DID rotation strategies, different pacing algorithms -- but the three pillars (AMD, DIDs, predictive algorithm) drive improvement in every case.

## How ViciStack Helps

Every optimization described in this case study is part of ViciStack's standard managed service. There's no premium tier, no add-on packages, no "advanced optimization" upsell. At $150/agent/month, you get:

- **AMD tuning** optimized for your specific campaigns, demographics, and carrier mix -- reviewed and adjusted monthly
- **DID management** including health monitoring, automatic rotation, reputation tracking, and proactive replacement of flagged numbers
- **Predictive algorithm optimization** using ADAPT_TAPERED with settings calibrated to your agent count, talk time patterns, and compliance requirements
- **24/7 server management** including monitoring, updates, security patches, and capacity planning
- **Custom reporting** with the metrics that matter for your operation, delivered daily
- **Expert support** from people who've optimized VICIdial for 100+ call centers -- not tier-1 script readers

The center in this case study is still a ViciStack customer. Six months after the initial optimization, their connect rate has stabilized at 7.3% (slightly above the Day 14 result) as we continue to fine-tune AMD parameters, rotate DIDs proactively, and adjust pacing for seasonal volume changes.

**Want to see what your numbers could look like?** We offer a free, no-obligation analysis of your current VICIdial performance. We'll pull the same metrics we described in this case study -- AMD accuracy, DID answer rates, dial efficiency, drop rates -- and show you exactly where your connects are being lost.

No sales pitch. Just data.

[Get Your Free Analysis at vicistack.com/proof/](https://vicistack.com/proof/) -- we respond within 5 minutes during business hours.

## Related Articles

- [VICIdial AMD Optimization: Eliminating False Positives](/blog/vicidial-amd-optimization)
- [VICIdial Predictive Dialer Optimization](/blog/vicidial-predictive-dialer-optimization)
- [VICIdial Custom MySQL Reports Guide](/blog/vicidial-custom-mysql-reports)
- [VICIdial vs CallTools: Detailed Comparison](/blog/vicidial-vs-calltools)
- [VICIdial to Hosted Migration Checklist](/blog/vicidial-hosted-migration-checklist)

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-roi-case-study).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
