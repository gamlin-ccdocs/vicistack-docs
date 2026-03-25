# VICIdial Auto-Dial Level Tuning by Campaign Type

The `auto_dial_level` setting is the single most impactful parameter in VICIdial for outbound campaigns. Set it too low and your agents sit idle, burning payroll while waiting for calls. Set it too high and you drop calls, violating FTC regulations and burning through your list. The right level depends on your campaign type, list quality, agent count, and connect rate — and it changes throughout the day.

This guide covers how auto_dial_level interacts with VICIdial's pacing system, provides specific tuning recommendations for different campaign verticals, and lays out a testing methodology to find your sweet spot.

## How auto_dial_level Interacts with Pacing Modes

VICIdial supports several dialing modes, and `auto_dial_level` behaves differently in each.

### Ratio Dialing (Manual Dial Level)

When `dial_method` is set to `RATIO`, the `auto_dial_level` is a fixed multiplier. A level of 3.0 means the system dials 3 lines for every agent in READY or INCALL status. This is the simplest mode but also the crudest — it doesn't adapt to changing connect rates.

```
Calls placed = auto_dial_level × available_agents
```

If you have 30 agents and a dial level of 3.0, the system maintains approximately 90 active call attempts. If your connect rate is 33%, you'll land roughly 30 live calls — one per agent. If the connect rate drops to 20%, you'll only get 18 connections, and 12 agents sit idle.

### Adaptive Dialing (Predictive)

When `dial_method` is set to `ADAPT_TAPERED`, `ADAPT_AVERAGE`, or `ADAPT_HARD_LIMIT`, VICIdial automatically adjusts the dial level based on real-time metrics. The `auto_dial_level` setting becomes the **maximum** level the system can reach.

In adaptive mode, VICIdial calculates a target dial level using:

- Current drop rate percentage
- Average agent wait time
- Average talk time
- Number of available agents
- The `adaptive_intensity` modifier

The system then adjusts the actual dial level up or down within the bounds of 1.0 (minimum) and `auto_dial_level` (maximum).

### Key Adaptive Settings

```
dial_method:          ADAPT_HARD_LIMIT  (or ADAPT_TAPERED / ADAPT_AVERAGE)
auto_dial_level:      5.0               (maximum the system will reach)
adaptive_intensity:   0                 (adjustment factor, see below)
adaptive_dl_diff_target: 0             (target difference between dropped % and threshold)
available_only_ratio_tally: Y          (count only READY agents, not INCALL)
drop_rate_group:      CAMPAIGN_ONLY    (calculate drop rate per campaign)
```

The difference between adaptive modes:

- **ADAPT_HARD_LIMIT:** Never exceeds the drop rate threshold, even momentarily. Most conservative.
- **ADAPT_AVERAGE:** Targets the drop rate threshold as an average. May briefly exceed it.
- **ADAPT_TAPERED:** Gradually reduces aggressiveness as drop rate approaches the threshold. Best balance for most operations.

## B2B vs. B2C Optimal Settings

Business-to-business and business-to-consumer campaigns have fundamentally different calling dynamics.

### B2B Campaign Characteristics

- **Lower connect rates** (10-20%) — gatekeepers, voicemail systems, IVRs
- **Longer talk times** when connected (3-10 minutes for qualifying conversations)
- **Smaller lists** — fewer total records, each one more valuable
- **Calling windows** restricted to business hours (9 AM - 5 PM)
- **Higher cost per lead** — burning a record through a dropped call is expensive

### B2B Recommended Settings

```
dial_method:              ADAPT_TAPERED
auto_dial_level:          3.5
adaptive_intensity:       -1
available_only_ratio_tally: Y
drop_rate_group:          CAMPAIGN_ONLY
dial_timeout:             26
```

Why these values:
- **Max level 3.5:** B2B connect rates are low enough that you need moderate overdial, but not aggressive — each contact is valuable.
- **Intensity -1:** Slightly conservative bias. B2B lists are expensive to replace, so protecting records matters more than minimizing agent wait time.
- **Dial timeout 26:** Business phones often route through PBX systems that add ring delay. Give them time to answer.

### B2C Campaign Characteristics

- **Higher connect rates** (25-45%) — direct consumer phones, mobile numbers
- **Shorter initial talk times** (30 seconds to 3 minutes for the pitch)
- **Larger lists** — higher volume, more replaceable records
- **Extended calling hours** (8 AM - 9 PM in many jurisdictions)
- **Lower cost per record** — volume matters more than individual preservation

### B2C Recommended Settings

```
dial_method:              ADAPT_TAPERED
auto_dial_level:          6.0
adaptive_intensity:       0
available_only_ratio_tally: Y
drop_rate_group:          CAMPAIGN_ONLY
dial_timeout:             22
```

Why these values:
- **Max level 6.0:** Higher connect rates need more headroom for the adaptive algorithm. During peak hours you may only need 3.0, but during off-peak the system needs room to compensate.
- **Intensity 0:** Neutral — let the algorithm find the balance.
- **Dial timeout 22:** Consumer phones answer quickly or go to voicemail. A 22-second timeout captures most live answers while avoiding voicemail pickup.

## Campaign Type Tuning: Insurance, Solar, Political

### Insurance Lead Campaigns

Insurance leads (Medicare, ACA, final expense, auto) have specific characteristics:

- Leads are often pre-qualified through web forms — higher connect expectation
- Compliance requirements vary by state — some require specific disclosures before pitch
- Talk times range widely: 2 minutes for a quick disqualification to 20+ minutes for a full quote
- Lead freshness matters enormously — a 5-minute-old lead has a 400% higher connect rate than a 24-hour-old one

**Insurance Settings:**

```
dial_method:              ADAPT_TAPERED
auto_dial_level:          5.0
adaptive_intensity:       0
dial_timeout:             24
hopper_level:             200
available_only_ratio_tally: Y
```

For fresh lead campaigns (leads under 1 hour old), consider a lower max dial level (3.0-4.0) with higher priority. These leads are too valuable to risk dropping. For aged leads (7+ days), you can be more aggressive (6.0-7.0) since connect rates drop dramatically and agent wait becomes the bigger cost.

**Pro tip:** Create separate campaigns for fresh vs. aged leads within the same vertical. In the VICIdial admin under Campaigns > Campaign Detail, set each campaign differently:

```
Fresh leads campaign (< 24 hours):
  Auto Dial Level:      4.0
  Adaptive Intensity:   -1

Aged leads campaign (7+ days):
  Auto Dial Level:      7.0
  Adaptive Intensity:   1
```

### Solar / Home Improvement Campaigns

Solar and home improvement campaigns typically involve:

- Cold or semi-warm lists (homeowner data, not inbound leads)
- Moderate connect rates (15-30%)
- Qualification-heavy conversations (homeowner? roof age? electric bill? credit score?)
- High appointment set value ($100-300+ per qualified appointment)
- Aggressive competition — the same homeowner is being called by multiple companies

**Solar Settings:**

```
dial_method:              ADAPT_TAPERED
auto_dial_level:          5.5
adaptive_intensity:       0
dial_timeout:             24
available_only_ratio_tally: Y
```

Solar campaigns benefit from slightly aggressive dialing because the qualification process itself weeds out most calls quickly. An agent who determines the prospect is a renter will disposition in 20-30 seconds and be back in the queue. This rapid turnover of short calls means agents become available faster than the algorithm predicts based on average talk time.

Consider setting `available_only_ratio_tally` to `N` for solar campaigns with very short average talk times (under 90 seconds). This tells the dialer to count INCALL agents as partially available, which prevents over-dialing during periods of rapid agent turnover.

### Political Campaigns

Political calling (voter ID, GOTV, fundraising, survey) has unique requirements:

- Massive lists (entire voter files for a district or state)
- Very high volume needs (millions of attempts in days/weeks before an election)
- Moderate connect rates (20-35% for voter files)
- Short conversations for voter ID/survey (30-90 seconds)
- Longer conversations for fundraising (2-5 minutes)
- TCPA considerations for cell phones (which is most of the file now)

**Political Voter ID Settings:**

```
dial_method:              ADAPT_HARD_LIMIT
auto_dial_level:          8.0
adaptive_intensity:       1
dial_timeout:             20
hopper_level:             500
available_only_ratio_tally: N
```

Note the ADAPT_HARD_LIMIT — political campaigns face intense scrutiny from regulators and opponents. A dropped call to a voter is not just a lost contact; it's a potential news story. The hard limit ensures the system never exceeds the FTC's 3% drop rate threshold, even momentarily.

The high max dial level (8.0) is appropriate because voter ID calls are extremely short. With 60-second average talk times and 20-25% connect rates, agents cycle fast and the dialer needs room to keep up.

**Political Fundraising Settings:**

```
dial_method:              ADAPT_TAPERED
auto_dial_level:          5.0
adaptive_intensity:       0
dial_timeout:             24
available_only_ratio_tally: Y
```

Fundraising calls last longer (agents are making an ask and potentially processing a donation), so the dialing profile looks more like B2C sales than voter ID.

## Drop Rate Management and TCPA Compliance

### The 3% Rule

The FTC's Telemarketing Sales Rule (TSR) establishes a 3% maximum abandoned (dropped) call rate, measured per campaign per 30-day rolling period. VICIdial tracks drop rate in real time and through the `vicidial_drop_rate_groups` system.

Check your current drop rate:

```sql
SELECT
    campaign_id,
    COUNT(*) AS total_calls,
    SUM(CASE WHEN status = 'DROP' THEN 1 ELSE 0 END) AS dropped,
    ROUND(
        SUM(CASE WHEN status = 'DROP' THEN 1 ELSE 0 END) * 100.0 /
        NULLIF(SUM(CASE WHEN status IN ('DROP','SALE','XFER','CALLBK','A','B','N','NI','NP','DNCL') THEN 1 ELSE 0 END), 0),
        2
    ) AS drop_rate_pct
FROM vicidial_log
WHERE call_date >= DATE_SUB(CURDATE(), INTERVAL 30 DAY)
GROUP BY campaign_id;
```

Important: The TSR calculates drop rate as abandoned calls divided by calls answered by a person (not total attempts). This means your denominator should exclude no-answers, busies, and machine-answered calls.

### VICIdial Drop Rate Settings

Key campaign-level settings for drop rate management:

```
dial_method:              ADAPT_HARD_LIMIT    (strictest enforcement)
adaptive_dropped_percentage: 3.0              (target threshold)
adaptive_maximum_level:   5.0                 (absolute ceiling)
drop_rate_group:          CAMPAIGN_ONLY       (per-campaign calculation)
```

If your operation runs multiple campaigns, `drop_rate_group` determines whether VICIdial calculates drop rate per campaign or across all campaigns. Per-campaign is safer and is what the TSR expects.

### The Safe Message Requirement

When a call is dropped (no agent available when the person answers), the FTC requires that a pre-recorded message play within 2 seconds, including the caller's identity and a callback number. In the VICIdial admin, go to Campaigns > Campaign Detail and set:

```
Drop Action:           AUDIO
Safe Harbor Audio:     safe_harbor_message_audio
Safe Harbor Exten:     8300
```

Record your safe harbor message and place the audio file in `/var/lib/asterisk/sounds/`. The message should be brief: "Hello, this is [company name] calling. We apologize for the inconvenience. Please call us back at [number]. Thank you."

## adapt_intensity and Related Parameters

The `adaptive_intensity` parameter modifies how aggressively the adaptive algorithm adjusts the dial level. Understanding it requires knowing what the base algorithm does.

### How Adaptive Intensity Works

VICIdial's adaptive algorithm calculates an "ideal" dial level based on:

```
ideal_level = agents_waiting / (connect_rate × agent_availability_factor)
```

The `adaptive_intensity` then shifts this calculation:

| Value | Effect | Use When |
|-------|--------|----------|
| -3 to -1 | Conservative — dials fewer lines | Expensive lists, compliance-sensitive, B2B |
| 0 | Neutral — trusts the algorithm | Default, good starting point |
| +1 to +3 | Aggressive — dials more lines | Cheap lists, high volume needed, short talks |

Each unit of intensity roughly corresponds to a 10-15% shift in dialing aggressiveness. An intensity of +2 might push a calculated dial level of 3.0 up to approximately 3.5-3.6.

### Related Parameters

**adaptive_dl_diff_target:** The target difference between your actual drop rate and the threshold. Set to `0` to target the exact threshold. Set to `-1` to maintain a 1% buffer (if threshold is 3%, target 2%). For a conservative approach, go to Campaigns > Campaign Detail and set `Adaptive DL Diff Target` to `-1`.

**available_only_ratio_tally:** Controls which agents are counted when calculating dial ratios.

- `Y` — Only count agents in READY status (waiting for a call). More conservative.
- `N` — Count READY and INCALL agents. More aggressive, assumes INCALL agents will become available soon.

For campaigns with average talk times under 90 seconds, `N` can improve efficiency. For campaigns with 3+ minute average talks, use `Y`.

**hopper_level:** How many leads to keep pre-loaded in the hopper. Low hopper means the dialer can stall while waiting for leads to load. Rule of thumb: `hopper_level >= auto_dial_level x agent_count x 2`. For aggressive dialing with 50 agents at a max dial level of 4.0, set Hopper Level to at least 400 in Campaigns > Campaign Detail.

## Testing Methodology for Finding Optimal Levels

Don't guess at dial levels. Use a structured testing approach.

### Step 1: Baseline Measurement

Run your current settings for a full day and record:

```sql
SELECT
    HOUR(vl.call_date) AS hour,
    COUNT(*) AS attempts,
    SUM(CASE WHEN vl.status = 'DROP' THEN 1 ELSE 0 END) AS drops,
    ROUND(SUM(CASE WHEN vl.status = 'DROP' THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 2) AS drop_pct,
    ROUND(AVG(CASE WHEN al.talk_sec > 0 THEN al.talk_sec ELSE NULL END), 0) AS avg_talk,
    ROUND(AVG(CASE WHEN al.wait_sec > 0 THEN al.wait_sec ELSE NULL END), 0) AS avg_wait
FROM vicidial_log vl
LEFT JOIN vicidial_agent_log al ON vl.uniqueid = al.uniqueid
WHERE vl.call_date >= CURDATE()
  AND vl.campaign_id = 'SALESCAMP'
GROUP BY HOUR(vl.call_date)
ORDER BY hour;
```

Record the average agent wait time, drop rate, and calls per hour per agent. These are your baselines.

### Step 2: Incremental Adjustment

Change one parameter at a time. Start with `auto_dial_level`:

- If agent wait time > 15 seconds average: Increase max dial level by 0.5
- If drop rate > 2%: Decrease max dial level by 0.5
- If both are fine: Increase intensity by 1

Run each new setting for at least 2 hours (ideally a full day) before evaluating.

### Step 3: Time-of-Day Profiling

Connect rates change throughout the day. The optimal dial level at 9 AM is not the same as at 2 PM. In adaptive mode, the system adjusts automatically — but you should verify it's doing so effectively:

```sql
SELECT
    HOUR(vl.call_date) AS hour,
    COUNT(DISTINCT vl.user) AS active_agents,
    ROUND(
        SUM(CASE WHEN vl.status = 'DROP' THEN 1 ELSE 0 END) * 100.0 /
        NULLIF(COUNT(*), 0), 2
    ) AS drop_rate,
    ROUND(AVG(al.wait_sec), 1) AS avg_wait_sec
FROM vicidial_log vl
LEFT JOIN vicidial_agent_log al ON vl.uniqueid = al.uniqueid
WHERE vl.call_date >= CURDATE()
  AND vl.campaign_id = 'SALESCAMP'
GROUP BY HOUR(vl.call_date)
ORDER BY hour;
```

If you see hours where the dial level is pegged at the max and wait time is still high, your max level is too low. If you see hours where the dial level is at the max and drops are high, the adaptive algorithm is struggling — lower the max or decrease intensity.

### Step 4: Week-Over-Week Comparison

After a week of the new settings, compare against your baseline:

```sql
-- This week vs. last week, same campaign
SELECT
    'This Week' AS period,
    COUNT(*) AS total_calls,
    ROUND(AVG(vl.length_in_sec), 0) AS avg_talk,
    ROUND(SUM(CASE WHEN vl.status='DROP' THEN 1 ELSE 0 END)*100.0/COUNT(*),2) AS drop_pct,
    ROUND(COUNT(*) / COUNT(DISTINCT DATE(vl.call_date)) / COUNT(DISTINCT vl.user), 1) AS calls_per_agent_per_day
FROM vicidial_log vl
WHERE vl.call_date >= DATE_SUB(CURDATE(), INTERVAL 7 DAY)
  AND vl.campaign_id = 'SALESCAMP'
UNION ALL
SELECT
    'Last Week',
    COUNT(*),
    ROUND(AVG(vl.length_in_sec), 0),
    ROUND(SUM(CASE WHEN vl.status='DROP' THEN 1 ELSE 0 END)*100.0/COUNT(*),2),
    ROUND(COUNT(*) / COUNT(DISTINCT DATE(vl.call_date)) / COUNT(DISTINCT vl.user), 1)
FROM vicidial_log vl
WHERE vl.call_date >= DATE_SUB(CURDATE(), INTERVAL 14 DAY)
  AND vl.call_date < DATE_SUB(CURDATE(), INTERVAL 7 DAY)
  AND vl.campaign_id = 'SALESCAMP';
```

The key metrics to improve are:
- **Calls per agent per day** — more calls means more opportunities
- **Average wait time** — lower means less wasted payroll
- **Drop rate** — must stay below 3% (target 2% or lower)

## How ViciStack Helps

Dial level tuning is not a set-it-and-forget-it task. Connect rates shift as lists age, seasonal patterns change caller behavior, and carrier performance fluctuates. An operation running 25+ agents at $150/agent/month can't afford the productivity loss of even a slightly mis-tuned dialer.

ViciStack provides:

- **Continuous adaptive tuning** — We monitor your dial level performance hourly and adjust parameters based on live data, not monthly reviews
- **Campaign-specific profiles** — Different verticals need different approaches. We build and maintain dial profiles for your specific campaign mix.
- **Drop rate compliance monitoring** — Real-time alerts if any campaign approaches the 2.5% warning threshold, with automatic intensity reduction
- **A/B testing of dial parameters** — We systematically test intensity, timeout, and hopper settings to find your optimal configuration
- **Time-of-day scheduling** — Automatic parameter adjustments based on your historical connect rate patterns by hour

All for $150/agent/month, flat. No per-minute. No per-call. No surprise invoices.

**Get a free analysis of your current dial level configuration** — we'll show you exactly where you're leaving agent productivity on the table: [Request Your Free ViciStack Analysis](https://vicistack.com/proof/)

## Related Resources

- [VICIdial Asterisk CDR Analysis for Connect Rate Optimization](/blog/vicidial-asterisk-cdr-analysis) — CDR data drives dial level decisions with hard connect rate numbers
- [VICIdial Pause Codes and Agent Accountability Systems](/blog/vicidial-pause-codes-accountability) — Agent availability directly impacts how the adaptive dialer performs
- [VICIdial Voicemail Drop Configuration and Compliance Guide](/blog/vicidial-voicemail-drop) — AMD settings interact with dial level and connect rate calculations

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-auto-dial-level-tuning).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
