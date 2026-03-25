# VICIdial Lead Recycling Strategies That Actually Work

## Most of Your Conversions Are Hiding in Leads You Already Gave Up On

Here is a number that should bother you: the average outbound call center converts between 2% and 5% of its leads on the first dial pass. That means 95-98% of the leads you paid for --- the ones that cost you $8, $15, $25 each depending on your vertical --- get dialed once or twice, catch a No Answer or Busy signal, and then sit in your `vicidial_list` table collecting dust while you load the next batch of fresh records.

This is the most expensive mistake in outbound dialing, and it is shockingly common.

The math is simple. If you buy 10,000 leads at $15 each, that is $150,000. Your first pass converts 350 leads. Your cost per acquisition is $428. Now, if a properly designed [lead recycling](/glossary/lead-recycling/) strategy recovers even 15% more conversions from those same 10,000 leads --- an additional 52 sales --- your CPA drops to $372 without spending a single dollar on new data. At 30% recovery, you pull 105 additional sales and your CPA falls to $329. Same leads. Same agents. Same infrastructure. The only difference is whether you built a recycling framework or just let those leads rot.

Most VICIdial operators understand recycling conceptually. They know the feature exists under **Admin > Campaigns > Lead Recycling**. Some of them have even turned it on. But the gap between "recycling is enabled" and "recycling is systematically producing recovered conversions" is enormous. It is the difference between setting a 3600-second delay on NA leads and calling it done, versus building a multi-pass framework with disposition-weighted rules, time-delay optimization, priority injection for high-value prospects, and a clear retirement threshold that prevents over-dialing.

This guide covers the complete recycling strategy: how VICIdial's recycling engine actually works at the database level, disposition-based recycling rules that match real-world contact patterns, multi-pass frameworks for working leads across days and weeks, list_mix and [dial_status_filter](/settings/dial-status-filter/) configurations that control exactly which recycled leads enter the [hopper](/glossary/hopper/), priority injection techniques for high-value leads, and the diminishing returns math that tells you exactly when to stop. Every recommendation is specific to VICIdial's architecture and tested across real production campaigns.

Stop throwing away leads after one pass.

## How VICIdial Lead Recycling Actually Works

Before you can build an effective recycling strategy, you need to understand the mechanics. VICIdial's recycling system operates at two distinct levels, and confusing them is the source of most recycling misconfigurations.

### Intra-Day Recycling: The Built-In Engine

VICIdial's native [lead recycling](/settings/lead-recycling/) feature is configured per-campaign under **Admin > Campaigns > [Campaign Name] > Lead Recycling**. For each [disposition](/glossary/disposition/) you want to recycle, you set two parameters:

- **Attempt Delay**: The number of seconds before the lead becomes eligible for the hopper again (minimum 120 seconds, maximum 43,199 seconds)
- **Attempt Maximum**: How many times the lead can be recycled for this disposition (1 to 10)

When a call receives a recyclable disposition --- say NA (No Answer) --- the recycling engine marks the lead with a timestamp. After the delay period expires, the lead becomes eligible for the hopper again, as long as it has not exceeded the attempt maximum for that disposition. The lead does not need a list reset. It re-enters the dial queue automatically.

The key database interaction happens in the `vicidial_lead_recycle` table, which stores the campaign-level recycling rules, and the lead's `called_since_last_reset` flag, which the recycling engine temporarily overrides for eligible leads.

**The critical limitation**: The maximum delay is 43,199 seconds --- just under 12 hours. This means intra-day recycling is exactly that: intra-day. It does not reliably span midnight boundaries. If a lead gets dispositioned NA at 4:00 PM and your recycling delay is 28,800 seconds (8 hours), the lead becomes eligible at midnight. If your campaign is not running at midnight, that recycled lead never gets dialed. It just sits there until the next list reset.

This is not a bug. It is a design constraint. VICIdial's recycling was built for same-day retry patterns, not multi-day cadences. For anything spanning more than a single shift, you need a different mechanism --- and that is where multi-pass recycling comes in.

### Multi-Day Recycling: List Resets and Dial Statuses

Multi-day recycling is not a single feature. It is a combination of three VICIdial mechanisms working together:

1. **[Dial Statuses](/glossary/dial-status/)**: The campaign setting that determines which lead statuses are eligible for the hopper. If NA is checked in your dial statuses, any lead with status NA that has not been called since the last reset will enter the hopper.

2. **List Resets**: Resetting the `called_since_last_reset` flag on a list (or subset of a list) makes those leads eligible for the hopper again on the next pass.

3. **Lead Filters**: SQL WHERE clauses that restrict which leads enter the hopper, even if their status is technically dialable. This is how you enforce attempt caps, age restrictions, and other recycling rules across passes.

The combination works like this: you run a dialing day, leads get dispositioned, the day ends. Overnight, a cron job resets the `called_since_last_reset` flag for leads that match your recycling criteria. The next morning, those leads are eligible for the hopper again because their status (NA, B, A) is in the campaign's dial status list and their flag has been reset.

```sql
-- Nightly reset for recyclable leads with attempt cap
UPDATE vicidial_list
SET called_since_last_reset = 'N'
WHERE list_id IN (1001, 1002, 1003)
  AND status IN ('NA', 'B', 'A', 'AA', 'LMVM', 'DECMKR')
  AND called_count < 12
  AND modify_date < DATE_SUB(NOW(), INTERVAL 4 HOUR);
```

The `modify_date` check ensures you only reset leads that have not been recently worked. The `called_count` cap prevents infinite recycling. These two guardrails --- time delay and attempt cap --- are the foundation of every multi-day recycling framework.

## Disposition-Based Recycling Rules That Match Reality

Not all dispositions deserve the same recycling treatment. A Busy signal means something fundamentally different from a No Answer, which means something fundamentally different from an agent who reached a voicemail and left a message. Your recycling rules need to reflect these differences, or you will over-dial some leads and under-dial others.

### The Disposition Recycling Matrix

Here is a disposition-by-disposition breakdown of recycling logic that reflects actual contact patterns across outbound campaigns:

| Disposition | Meaning | Intra-Day Delay | Intra-Day Max | Multi-Day Treatment | Total Cap |
|---|---|---|---|---|---|
| B (Busy) | Line is occupied | 300 sec (5 min) | 5 | Reset daily, retry 2-3x/day | 15 attempts |
| NA (No Answer) | Ring, no pickup | 3,600 sec (1 hr) | 3 | Reset daily, vary time-of-day | 12 attempts |
| A (Answering Machine) | Agent heard VM greeting | 7,200 sec (2 hr) | 2 | Reset every other day | 8 attempts |
| AA (Auto-Detected VM) | AMD flagged as machine | 10,800 sec (3 hr) | 2 | Reset every other day | 8 attempts |
| ADC (Carrier No Answer) | Network-level failure | 1,800 sec (30 min) | 3 | Reset daily | 10 attempts |
| LMVM (Left Voicemail) | Agent left a message | 14,400 sec (4 hr) | 1 | Reset after 48 hours | 4 attempts |
| DECMKR (Decision Maker Unavail) | Wrong person answered | 7,200 sec (2 hr) | 2 | Reset daily, vary time-of-day | 10 attempts |
| HUAGNT (Hung Up on Agent) | Immediate hangup | 21,600 sec (6 hr) | 1 | Reset after 72 hours | 3 attempts |
| DROP (Dropped Call) | No agent available | 600 sec (10 min) | 4 | Reset daily | 12 attempts |

### Why These Specific Delays

The delays are not arbitrary. They are based on the behavioral patterns behind each disposition.

**Busy (5 minutes)**: A busy signal means the line is literally in use. The person is on the phone. Five minutes is enough for most calls to end. Aggressive retry here makes sense because the prospect is clearly near their phone and active.

**No Answer (1 hour)**: No answer means the prospect is unavailable --- at work, driving, in a meeting, ignoring unknown numbers. Retrying in 5 minutes just means you ring the same empty room. A 1-hour offset puts you into a different activity window. Three intra-day attempts at hourly intervals cover morning, midday, and afternoon.

**Answering Machine (2 hours)**: If you are hitting voicemail, the person either screens calls or is genuinely unavailable. A 2-hour delay puts meaningful distance between attempts. Limiting to 2 intra-day retries avoids filling someone's voicemail box, which generates complaints.

**Left Voicemail (4 hours)**: This is the most mishandled disposition. If your agent left a message, the prospect now knows you called and has your number. They may call back. Hammering them with another call 30 minutes later undermines the voicemail's purpose and comes across as aggressive. Four hours gives the prospect time to listen and respond. One intra-day retry is enough --- if they did not call back after the first message, a second voicemail the same day is counterproductive.

**Hung Up on Agent (6 hours)**: The prospect answered and immediately hung up. They know who you are and made a deliberate choice. Calling back within an hour is the fastest way to get a DNC request or spam report. A 6-hour delay and single intra-day retry respects the signal while leaving one door open --- maybe they were just busy and the hangup was reflexive.

### Setting Up Disposition-Based Recycling in VICIdial

Navigate to **Admin > Campaigns > [Campaign Name] > Lead Recycling**. For each status you want to recycle, add a row with the delay and maximum from the matrix above.

Then, ensure your campaign's Dial Statuses include all recyclable statuses. Under **Admin > Campaigns > [Campaign Name] > Dial Statuses**, check: NEW, B, NA, A, AA, ADC, LMVM, DECMKR, HUAGNT, and DROP.

For custom dispositions like LMVM and DECMKR, you need to create them first under **Admin > Campaign > Campaign Statuses** with Dialable set to Y and Human Answer set to the appropriate value. See our [list management guide](/blog/vicidial-list-management/) for the full custom status framework.

> **Your recycling rules should be as deliberate as your lead sourcing.** If you are using VICIdial's default recycling settings --- or worse, no recycling at all --- you are leaving 15-30% of your potential conversions on the table. [Request a free ViciStack recycling audit](/free-audit/) and we will map your current settings against the framework above.

## Multi-Pass Recycling Frameworks

Single-pass dialing is like fishing a lake once and concluding there are no fish. The contacts are there. You just have not reached them at the right time yet. A multi-pass framework structures your recycling into distinct phases, each with its own rules, priorities, and expectations.

### The Three-Pass Framework

This framework has been tested across insurance, solar, home services, and debt settlement campaigns. Adapt the specific numbers to your vertical, but the structure applies broadly.

**Pass 1: Fresh Leads (Days 1-3)**

This is your highest-value window. Leads are at peak intent. Your recycling should be aggressive within compliance limits.

- **Lists**: Fresh leads in their original upload lists
- **[Lead Order](/settings/lead-order/)**: DOWN (newest first) or DOWN RANK if using [lead scoring](/glossary/lead-scoring/)
- **Intra-day recycling**: Full matrix from the previous section
- **Daily resets**: Every night, reset all recyclable statuses with `called_count < 8`
- **Expected contact rate**: 15-25% on quality leads
- **Expected conversion rate**: 3-6% of total leads

```sql
-- Pass 1 nightly reset: aggressive, all recyclable statuses
UPDATE vicidial_list
SET called_since_last_reset = 'N'
WHERE list_id IN (1001, 1002, 1003)
  AND status IN ('NA', 'B', 'A', 'AA', 'ADC', 'LMVM', 'DECMKR', 'HUAGNT', 'DROP')
  AND called_count < 8
  AND modify_date < DATE_SUB(NOW(), INTERVAL 2 HOUR);
```

**Pass 2: Warm Recycle (Days 4-14)**

Leads that survived Pass 1 without converting or reaching a terminal disposition. These are harder to reach but not dead. Your approach should shift from volume to timing.

- **Lists**: Move remaining recyclable leads to dedicated Pass 2 lists
- **Lead Order**: DOWN COUNT (lowest call count first, giving underworked leads priority)
- **Intra-day recycling**: Longer delays --- 2 hours for B, 4 hours for NA, 6 hours for A
- **Daily resets**: Every other night, reset recyclable statuses with `called_count < 15`
- **Time-of-day variation**: Use [lead filters](/settings/lead-filter/) to dial at different times than Pass 1
- **Expected contact rate**: 8-15%
- **Expected incremental conversion**: 1-2% of original lead count

```sql
-- Move leads from Pass 1 to Pass 2 lists after Day 3
UPDATE vicidial_list
SET list_id = CASE
    WHEN list_id = 1001 THEN 2001
    WHEN list_id = 1002 THEN 2002
    WHEN list_id = 1003 THEN 2003
  END
WHERE list_id IN (1001, 1002, 1003)
  AND entry_date < DATE_SUB(NOW(), INTERVAL 3 DAY)
  AND status IN ('NA', 'B', 'A', 'AA', 'ADC', 'LMVM', 'DECMKR', 'HUAGNT', 'DROP')
  AND status NOT IN ('SALE', 'DNC', 'DC', 'DNQUAL', 'WRNUMB', 'INVNUM', 'APPT', 'CALLBK');
```

```sql
-- Pass 2 reset: every other night, higher attempt cap
UPDATE vicidial_list
SET called_since_last_reset = 'N'
WHERE list_id IN (2001, 2002, 2003)
  AND status IN ('NA', 'B', 'A', 'AA', 'ADC', 'DROP')
  AND called_count < 15
  AND modify_date < DATE_SUB(NOW(), INTERVAL 6 HOUR);
```

The key difference in Pass 2 is the exclusion of LMVM and DECMKR from the reset. By this point, if you have left multiple voicemails and spoken to household members multiple times without reaching the decision maker, continued dialing on those dispositions is low-yield and high-annoyance. Let those leads rest.

**Pass 3: Aged Leads (Days 15-45)**

These leads have been worked hard and have not converted. The contact rate will be lower, but the leads that do connect at this stage often convert at a surprisingly high rate --- the timing finally aligned with their need.

- **Lists**: Move remaining recyclable leads to Pass 3 lists
- **Lead Order**: RANDOM (eliminates any remaining time-of-day patterns from previous passes)
- **Intra-day recycling**: Disabled. One attempt per day maximum.
- **Resets**: Twice per week (Monday and Thursday), only NA and B statuses, `called_count < 20`
- **Different caller ID**: Use a different outbound caller ID than Passes 1 and 2 to avoid number fatigue
- **Different script**: Acknowledge the multiple contacts. "I know we have been trying to reach you" works better than pretending this is a first contact.
- **Expected contact rate**: 4-8%
- **Expected incremental conversion**: 0.5-1% of original lead count

```sql
-- Pass 3 reset: twice weekly, minimal statuses
UPDATE vicidial_list
SET called_since_last_reset = 'N'
WHERE list_id IN (3001, 3002, 3003)
  AND status IN ('NA', 'B')
  AND called_count < 20
  AND modify_date < DATE_SUB(NOW(), INTERVAL 48 HOUR);
```

### Why Three Passes Instead of Two or Five

Three passes is not arbitrary. It maps to three distinct phases of lead behavior:

1. **Fresh interest window** (Pass 1): The prospect is actively seeking information. Speed and frequency matter.
2. **Scheduling mismatch window** (Pass 2): The prospect is reachable but you have not hit the right time yet. Varied timing matters.
3. **Circumstance change window** (Pass 3): The prospect's situation or availability has shifted. Patience and different approach matter.

Adding a fourth or fifth pass rarely produces enough incremental conversion to justify the agent time, caller ID wear, and compliance risk. The math on diminishing returns is covered later in this guide.

### Automating Pass Transitions with Cron

Schedule your pass transitions as nightly cron jobs so leads flow through the framework automatically:

```bash
# Run at 11 PM nightly
# Step 1: Move Day 3+ leads from Pass 1 to Pass 2
0 23 * * * mysql -u vicidialuser -p'password' asterisk -e "UPDATE vicidial_list SET list_id = list_id + 1000 WHERE list_id IN (1001,1002,1003) AND entry_date < DATE_SUB(NOW(), INTERVAL 3 DAY) AND status IN ('NA','B','A','AA','ADC','LMVM','DECMKR','HUAGNT','DROP');"

# Step 2: Move Day 14+ leads from Pass 2 to Pass 3
5 23 * * * mysql -u vicidialuser -p'password' asterisk -e "UPDATE vicidial_list SET list_id = list_id + 1000 WHERE list_id IN (2001,2002,2003) AND entry_date < DATE_SUB(NOW(), INTERVAL 14 DAY) AND status IN ('NA','B','A','AA','ADC','DROP');"

# Step 3: Retire Day 45+ leads to archive lists
10 23 * * * mysql -u vicidialuser -p'password' asterisk -e "UPDATE vicidial_list SET list_id = list_id + 1000 WHERE list_id IN (3001,3002,3003) AND entry_date < DATE_SUB(NOW(), INTERVAL 45 DAY) AND status NOT IN ('SALE','DNC','CALLBK','APPT');"

# Step 4: Reset appropriate leads for next day
15 23 * * * mysql -u vicidialuser -p'password' asterisk -e "UPDATE vicidial_list SET called_since_last_reset='N' WHERE list_id IN (1001,1002,1003) AND status IN ('NA','B','A','AA','ADC','LMVM','DECMKR','HUAGNT','DROP') AND called_count < 8;"
20 23 * * * mysql -u vicidialuser -p'password' asterisk -e "UPDATE vicidial_list SET called_since_last_reset='N' WHERE list_id IN (2001,2002,2003) AND status IN ('NA','B','A','AA','ADC','DROP') AND called_count < 15;"
```

## Using list_mix and dial_status_filter for Recycling Control

VICIdial includes two powerful but underused features that give you fine-grained control over how recycled leads enter the hopper alongside fresh leads: List Mix and the [Dial Status Filter](/settings/dial-status-filter/).

### List Mix: Blending Fresh and Recycled Leads

List Mix allows a single campaign to dial from multiple lists with a controlled ratio. Instead of activating three lists and hoping the hopper pulls proportionally from each, List Mix lets you explicitly define the blend.

Configure List Mix under **Admin > Campaigns > [Campaign Name] > List Mix**. You define a sequence of list entries with these parameters:

- **List ID**: Which list to pull from
- **Mix Percentage**: What proportion of hopper fills should come from this list
- **Status**: Which [dial statuses](/glossary/dial-status/) to include from this list

Here is a practical List Mix configuration for a campaign running Pass 1 and Pass 2 simultaneously:

| Order | List ID | Description | Mix % | Statuses |
|---|---|---|---|---|
| 1 | 1001 | Fresh leads - Source A | 40% | NEW |
| 2 | 1002 | Fresh leads - Source B | 20% | NEW |
| 3 | 2001 | Pass 2 recycle - Source A | 25% | NA, B, A |
| 4 | 2002 | Pass 2 recycle - Source B | 15% | NA, B, A |

This configuration ensures that 60% of your dialing capacity goes to fresh leads while 40% works the recycle pool. Without List Mix, the hopper might load 90% recycled leads (because there are more of them) and starve your fresh leads of dial attempts.

The Mix Percentage does not need to add up to exactly 100%. VICIdial normalizes the values proportionally. But keeping them at round numbers that sum to 100 makes the configuration easier to reason about.

### Dial Status Filter: Precision Recycling

The Dial Status Filter is a campaign-level setting that controls which statuses are eligible for dialing on a given pass. While the standard Dial Statuses checkboxes determine overall eligibility, the Dial Status Filter adds a layer of dynamic control.

Use this in combination with list resets to create time-phased recycling. For example, during your morning shift you might want to focus exclusively on fresh and Busy leads:

Navigate to the campaign's dial statuses and ensure only NEW and B are checked. Then at midday, add NA and A to the dial statuses to start pulling in morning No Answers.

For more granular control, use a lead filter instead:

```sql
-- Morning shift: only fresh and busy leads
status IN ('NEW', 'B') OR (status = 'NA' AND called_count <= 2)
```

```sql
-- Afternoon shift: expand to include morning NA and A leads
status IN ('NEW', 'B', 'NA', 'A') OR (status = 'LMVM' AND modify_date < DATE_SUB(NOW(), INTERVAL 4 HOUR))
```

You can update lead filters mid-shift without restarting the campaign. The hopper refresh process picks up filter changes within a few minutes.

### Combining List Mix with Time-of-Day Filters

The most sophisticated recycling setups combine List Mix with time-of-day lead filters to create a dynamic blend that shifts throughout the day:

**Morning (8 AM - 12 PM)**: Heavy fresh leads, light recycling

| List | Mix % | Filter |
|---|---|---|
| Fresh | 70% | status = 'NEW' |
| Recycle | 30% | status IN ('B', 'NA') AND called_count < 5 |

**Afternoon (12 PM - 5 PM)**: Balanced mix, catching morning misses

| List | Mix % | Filter |
|---|---|---|
| Fresh | 50% | status = 'NEW' |
| Recycle | 50% | status IN ('B', 'NA', 'A') AND called_count < 10 |

**Evening (5 PM - 9 PM)**: Heavy recycling, residential availability peaks

| List | Mix % | Filter |
|---|---|---|
| Fresh | 30% | status = 'NEW' |
| Recycle | 70% | status IN ('NA', 'A', 'LMVM', 'DECMKR') AND called_count < 12 |

This approach maximizes your contact rate across the entire day by matching lead types to the times when they are most likely to answer. Fresh leads get priority during business hours when web form submissions are flowing in. Recycled residential leads get priority during evening hours when those people are actually home.

## Priority Injection for High-Value Leads

Not all recycled leads are created equal. A lead that filled out a detailed form, requested a callback, or was previously quoted at a high value deserves faster and more persistent recycling than a lead that came from a bulk list and has never answered.

### Using the Rank Field for Recycling Priority

VICIdial's `rank` field in the `vicidial_list` table accepts any integer value. When your campaign's [Lead Order](/settings/lead-order/) is set to DOWN RANK, higher rank values get dialed first. This applies to both fresh and recycled leads in the hopper.

Set rank values during upload based on lead attributes:

| Lead Attribute | Rank Value | Rationale |
|---|---|---|
| High-value quote request (>$500 potential) | 90-99 | Maximum priority, fastest recycling |
| Callback request / inbound inquiry | 80-89 | Explicit interest, high conversion probability |
| Multi-step form completion | 70-79 | Demonstrated engagement |
| Single-step form / basic inquiry | 50-59 | Standard priority |
| Purchased list - verified | 30-39 | Lower intent, but validated data |
| Purchased list - unverified | 10-19 | Lowest priority, data quality unknown |
| Aged recycle (Pass 3) | 1-9 | Minimal priority, dial only when capacity allows |

### Dynamic Rank Adjustment Based on Disposition History

Here is where recycling gets intelligent. Instead of treating all recycled leads with the same priority, adjust the rank based on what happened during prior contact attempts.

```sql
-- Boost rank for leads where decision maker was reached but unavailable
UPDATE vicidial_list
SET rank = LEAST(rank + 20, 99)
WHERE status = 'DECMKR'
  AND list_id IN (2001, 2002, 2003)
  AND called_count < 15;

-- Boost rank for leads with prior human contact (they answer the phone)
UPDATE vicidial_list
SET rank = LEAST(rank + 15, 99)
WHERE status = 'HUAGNT'
  AND list_id IN (2001, 2002, 2003)
  AND called_count < 10;

-- Reduce rank for leads hitting voicemail repeatedly
UPDATE vicidial_list
SET rank = GREATEST(rank - 10, 1)
WHERE status IN ('A', 'AA')
  AND called_count > 6
  AND list_id IN (2001, 2002, 2003);
```

Run these adjustments as part of your nightly cron sequence, after the pass transition queries but before the list resets. This ensures that when the hopper fills the next morning, high-value recycled leads jump ahead of low-value ones.

### Priority Injection via the Non-Agent API

For real-time priority injection --- such as when a web form submission comes in for a lead that already exists in your recycled pool --- use VICIdial's Non-Agent API to update the lead directly:

```
/vicidial/non_agent_api.php?function=update_lead&lead_id=12345&rank=95&called_since_last_reset=N&source=WEB_REINQUIRY
```

This immediately makes the lead eligible for the hopper with a high priority rank. The lead jumps to the front of the queue on the next hopper refill cycle, typically within 30-60 seconds.

This is particularly powerful for re-engagement scenarios. A prospect who was dispositioned NA three days ago just visited your website again and filled out another form. That behavioral signal means they are back in the market. Injecting them at rank 95 ensures they get dialed within minutes, not hours.

### Callback-Recycling Integration

[Callback automation](/blog/vicidial-callback-automation/) and lead recycling should work together, not as separate systems. When a callback is scheduled (CALLBK disposition), the lead exits the recycling pool --- it is now in the callback queue, which has its own priority in the hopper. But when a scheduled callback fails (the agent calls at the appointed time and gets no answer), the lead should re-enter the recycling pool with elevated priority.

```sql
-- Re-inject failed callbacks into recycling with boosted rank
UPDATE vicidial_list
SET status = 'NA',
    rank = LEAST(rank + 25, 99),
    called_since_last_reset = 'N'
WHERE status = 'CALLBK'
  AND modify_date < DATE_SUB(NOW(), INTERVAL 24 HOUR)
  AND called_count < 15;
```

This query catches callbacks that were scheduled more than 24 hours ago but were never successfully completed. It resets them to NA status with a rank boost so they get recycled promptly. Without this, failed callbacks fall into a black hole --- they are not in the recycling queue because their status is CALLBK, and they are not in the callback queue because the scheduled time has passed.

## When to Recycle and When to Stop: The Diminishing Returns Math

Every additional dial attempt on a recycled lead costs money. Agent time, trunk minutes, carrier fees, caller ID wear --- these are not free. At some point, the cost of additional attempts exceeds the expected revenue from the marginal conversions those attempts produce. Finding that breakeven point is the difference between profitable recycling and expensive busywork.

### The Marginal Conversion Curve

Contact rates and conversion rates decline with each successive pass. The decline is not linear --- it follows a curve that drops steeply after the first few attempts and then flattens out.

Here is a representative curve from a real outbound insurance campaign (50,000 leads, $18 per lead, 30 agents):

| Pass | Attempts | Cumulative Contact Rate | Incremental Conversion Rate | Incremental Conversions | CPA (Incremental) |
|---|---|---|---|---|---|
| 1 (Days 1-3) | 1-8 | 22% | 3.2% | 1,600 | $562 |
| 2 (Days 4-14) | 9-15 | 28% (+6%) | 1.1% | 550 | $327 |
| 3 (Days 15-45) | 16-20 | 30% (+2%) | 0.4% | 200 | $450 |
| 4 (Days 46-90) | 21-25 | 31% (+1%) | 0.1% | 50 | $1,800 |

Look at the CPA column. Pass 2 is actually more cost-efficient than Pass 1 because the lead cost is already sunk --- you are only paying for agent time and telecom. The incremental cost of Pass 2 is roughly $180,000 in agent wages and trunk costs to produce 550 additional conversions, giving you a $327 CPA on those incremental sales.

Pass 3 is still viable if your product margin supports a $450 CPA. Pass 4 is almost certainly unprofitable --- the incremental CPA of $1,800 means you are paying more per conversion than the lifetime value of most outbound-sold accounts.

### Calculating Your Breakeven Point

To find your recycling breakeven, you need three numbers:

1. **Incremental cost per attempt**: Agent hourly wage / dials per hour + per-minute carrier cost. For most operations, this is $0.80-$1.50 per attempt.

2. **Marginal conversion rate**: The conversion rate of leads contacted for the first time on this pass (not the cumulative rate). This drops with each pass.

3. **Revenue per conversion**: Gross margin on a sale or the value of a qualified appointment.

The formula:

```
Breakeven Attempts = Revenue Per Conversion x Marginal Conversion Rate / Incremental Cost Per Attempt
```

Example: Revenue per conversion is $800. Marginal conversion rate on Pass 3 contacts is 12% (of contacts, not of total leads). Contact rate on Pass 3 leads is 5%. Incremental cost per attempt is $1.20.

```
Expected revenue per attempt = $800 x 0.12 x 0.05 = $4.80
Cost per attempt = $1.20
Profit per attempt = $3.60
```

Still profitable. But on Pass 4, the contact rate drops to 2% and marginal conversion drops to 8%:

```
Expected revenue per attempt = $800 x 0.08 x 0.02 = $1.28
Cost per attempt = $1.20
Profit per attempt = $0.08
```

Barely breakeven. One more pass and you are losing money.

### Hard Stop Rules

Based on the diminishing returns math, implement these hard stops in your recycling framework:

1. **Maximum called_count**: Set an absolute cap. For most verticals, 20 total attempts is the practical ceiling. Enforce this in your lead filter:

```sql
called_count < 20
```

2. **Maximum age from entry**: Leads older than 90 days have degraded data quality (changed numbers, changed circumstances). Enforce via lead filter or pass transitions:

```sql
entry_date > DATE_SUB(NOW(), INTERVAL 90 DAY)
```

3. **Disposition-based retirement**: Some dispositions should be terminal after a certain count regardless of the overall attempt cap:

```sql
-- Retire leads with 3+ voicemail-only contacts
UPDATE vicidial_list
SET status = 'VMTIRED', list_id = 9999
WHERE status IN ('A', 'AA', 'LMVM')
  AND called_count >= 6
  AND lead_id NOT IN (
    SELECT lead_id FROM vicidial_log
    WHERE status IN ('SALE','N','NI','CALLBK','CBHOLD','DNQUAL','DECMKR','APPT')
  );
```

This retires leads that have been called 6+ times and have never reached a human. These are screen-or-disconnect numbers and continued dialing is pure waste.

4. **Contact-without-conversion retirement**: A lead that has been contacted (human answered) 3 or more times without converting is unlikely to convert on the fourth. Consider retiring these or moving them to a low-priority reactivation pool:

```sql
-- Find leads with 3+ human contacts but no conversion
SELECT lead_id, called_count,
  (SELECT COUNT(*) FROM vicidial_log vl
   WHERE vl.lead_id = v.lead_id
   AND vl.status IN ('N','NI','DNQUAL','HUAGNT','DECMKR','NOFUND','CMPTN')) AS human_contacts
FROM vicidial_list v
WHERE list_id IN (2001, 2002, 2003)
HAVING human_contacts >= 3;
```

### The Compliance Ceiling

Diminishing returns is not just a financial calculation. Regulatory and reputational risks escalate with each additional dial attempt:

- **TCPA exposure**: Every call to a number on the DNC list or outside permitted hours is a potential $500-$1,500 violation. More attempts means more exposure.
- **Caller ID flagging**: Carriers and call-labeling services (Nomorobo, Hiya, TNS) track calling patterns. Numbers that repeatedly call the same destination get flagged as spam. The [DID management](/blog/vicidial-did-management/) burden scales directly with recycling volume.
- **Consumer complaints**: The FTC tracks complaints per campaign. Higher complaint volumes trigger investigations. Every additional attempt on a reluctant prospect increases complaint risk.

These factors should push your hard stop threshold lower than the pure financial breakeven. If the math says you can profitably recycle up to 25 attempts, set the cap at 20 to preserve compliance and reputation margin.

> **Getting the recycling-to-retirement balance wrong costs money in both directions.** Stop too early and you leave conversions on the table. Stop too late and you burn caller IDs, annoy prospects, and waste agent time. [Let ViciStack model the optimal recycling curve for your specific vertical and lead sources](/free-audit/).

## Monitoring Recycling Performance

A recycling strategy is only as good as the data you use to tune it. Without per-pass and per-disposition metrics, you are guessing at delays and attempt caps instead of optimizing them.

### Key Recycling Metrics

**1. Incremental Contact Rate Per Pass**

This is the most important recycling metric. It tells you how many new contacts each pass is generating.

```sql
SELECT
  CASE
    WHEN called_count BETWEEN 1 AND 8 THEN 'Pass 1'
    WHEN called_count BETWEEN 9 AND 15 THEN 'Pass 2'
    WHEN called_count BETWEEN 16 AND 20 THEN 'Pass 3'
    ELSE 'Pass 4+'
  END AS pass_group,
  COUNT(*) AS total_leads,
  COUNT(CASE WHEN status IN ('SALE','N','NI','CALLBK','CBHOLD','DNQUAL','DECMKR','APPT','NOFUND','CMPTN','HUAGNT') THEN 1 END) AS contacted,
  ROUND(COUNT(CASE WHEN status IN ('SALE','N','NI','CALLBK','CBHOLD','DNQUAL','DECMKR','APPT','NOFUND','CMPTN','HUAGNT') THEN 1 END) / COUNT(*) * 100, 2) AS contact_rate_pct
FROM vicidial_list
WHERE list_id IN (1001,1002,1003,2001,2002,2003,3001,3002,3003)
GROUP BY pass_group;
```

**2. Conversion Recovery Rate**

The percentage of total conversions that came from recycled leads (Pass 2+):

```sql
SELECT
  COUNT(CASE WHEN called_count <= 8 THEN 1 END) AS pass1_conversions,
  COUNT(CASE WHEN called_count > 8 THEN 1 END) AS recycled_conversions,
  ROUND(COUNT(CASE WHEN called_count > 8 THEN 1 END) / COUNT(*) * 100, 2) AS recycled_pct
FROM vicidial_list
WHERE status = 'SALE'
  AND entry_date > DATE_SUB(NOW(), INTERVAL 90 DAY);
```

If your recycled_pct is below 15%, your recycling framework is underperforming. If it is above 40%, you may be under-dialing on Pass 1.

**3. Disposition Recycle Yield**

Which dispositions produce the most recovered contacts when recycled:

```sql
SELECT
  prev_status,
  COUNT(*) AS recycled_attempts,
  COUNT(CASE WHEN new_status IN ('SALE','CALLBK','APPT') THEN 1 END) AS conversions,
  ROUND(COUNT(CASE WHEN new_status IN ('SALE','CALLBK','APPT') THEN 1 END) / COUNT(*) * 100, 3) AS conversion_rate
FROM (
  SELECT vl.lead_id,
    LAG(vl.status) OVER (PARTITION BY vl.lead_id ORDER BY vl.call_date) AS prev_status,
    vl.status AS new_status
  FROM vicidial_log vl
  WHERE vl.call_date > DATE_SUB(NOW(), INTERVAL 30 DAY)
) sub
WHERE prev_status IN ('NA','B','A','AA','LMVM','DECMKR','HUAGNT','DROP')
GROUP BY prev_status
ORDER BY conversion_rate DESC;
```

This query shows you which dispositions are worth recycling and which are dead weight. If recycled A (Answering Machine) leads convert at 0.01% while recycled B (Busy) leads convert at 0.8%, you should be recycling Busy leads more aggressively and backing off on Answering Machine leads.

### Weekly Recycling Review

Build a weekly review process around these metrics. Every Monday morning, your operations team should answer:

1. **What was last week's conversion recovery rate?** Is it trending up or down?
2. **Which passes are still productive?** Has Pass 3 dropped below breakeven?
3. **Which dispositions are recycling well?** Should you increase or decrease delay/attempts for any status?
4. **How fast are lists exhausting?** Do you need to adjust attempt caps or add fresh leads?
5. **Are caller IDs getting flagged?** Rising spam flags indicate over-recycling. Check your [predictive dialer settings](/blog/vicidial-predictive-dialer-settings/) and DID rotation strategy.

Recycling is not a set-and-forget configuration. It is a weekly optimization cycle. The centers that treat it as a living system consistently outperform the ones that configure it once and never revisit.

## Putting It All Together: The Complete Recycling Workflow

Here is the end-to-end workflow that ties every concept in this guide into a single operational system:

**Day 0**: Leads uploaded to Pass 1 lists. Rank values set based on lead source and attributes. Lists activated in campaign with DOWN RANK lead order.

**Days 1-3**: Pass 1 dialing. Intra-day recycling active per disposition matrix. Nightly resets for all recyclable statuses with `called_count < 8`. Priority injection via API for re-inquiries and callback requests.

**Day 3 (11 PM)**: Cron job moves remaining recyclable leads to Pass 2 lists. Rank values adjusted based on disposition history. Pass 2 lists activated in campaign with List Mix at 40% allocation.

**Days 4-14**: Pass 2 dialing. Reduced intra-day recycling delays. Resets every other night for core recyclable statuses with `called_count < 15`. Time-of-day variation in lead filters. Failed callbacks re-injected with rank boost.

**Day 14 (11 PM)**: Cron job moves remaining recyclable leads to Pass 3 lists. Rank values reduced. Pass 3 lists activated with 15% List Mix allocation. Different caller ID and script assigned at list level.

**Days 15-45**: Pass 3 dialing. No intra-day recycling. Resets twice weekly for NA and B only, `called_count < 20`. RANDOM lead order.

**Day 45 (11 PM)**: Cron job moves remaining leads to archive lists. Leads set to non-dialable status. Archive lists are inactive --- not assigned to any campaign.

**Day 90+**: Optional reactivation campaign. Leads with 60+ days of rest get moved to a reactivation list with reset status, new rank values, different caller ID, and different offer. This is a separate campaign entirely, not a continuation of the original.

That is the framework. Adapt the specific numbers --- pass durations, attempt caps, rank values, List Mix percentages --- to your vertical, lead quality, and margin structure. But the architecture stays the same: structured passes with declining intensity, disposition-aware recycling rules, priority injection for high-value leads, and hard stops based on diminishing returns math.

Your leads are not dead after one pass. Most of them have not even been reached yet. Build the system that gives them a fair chance.

## Frequently Asked Questions

### What is the maximum number of times VICIdial can recycle a lead in a single day?

VICIdial's intra-day recycling supports a maximum of 10 attempts per disposition. However, the practical limit depends on your delay settings and shift length. If you set a 3,600-second (1-hour) delay on NA leads with a maximum of 3 attempts, and your shift runs 10 hours, you can get at most 3 recycled attempts plus the original dial --- 4 total attempts per day for that disposition. If you set a 300-second delay on B leads with a maximum of 5 attempts, you could theoretically hit all 5 recycled attempts plus the original within a 2-hour window. The attempt maximum per disposition is configurable from 1 to 10 under **Admin > Campaigns > Lead Recycling**.

### How do I prevent recycled leads from overwhelming my fresh lead queue?

Use List Mix to control the ratio between fresh and recycled leads in the [hopper](/glossary/hopper/). Assign fresh leads and recycled leads to separate lists, then configure List Mix to allocate a specific percentage to each. A common starting ratio is 60% fresh / 40% recycled, adjusted based on your fresh lead volume and recycling pool size. Without List Mix, the hopper will pull whichever eligible leads it finds first based on your lead order setting, which typically means recycled leads dominate because they outnumber fresh leads 3-to-1 or more. See the List Mix section of this guide for detailed configuration.

### Should I recycle DNC and Disconnected Number dispositions?

No. DNC leads must never be re-dialed --- doing so violates federal and state telemarketing laws and can result in per-call penalties of $500-$1,500. Disconnected numbers (DC) should also be treated as terminal dispositions because the number is no longer in service. Remove both DNC and DC from your campaign's dial statuses and never include them in list resets or recycling rules. For a comprehensive disposition framework, see our [list management guide](/blog/vicidial-list-management/).

### How do I know if my recycling delays are too short or too long?

Track the contact rate of recycled leads by disposition and compare it to the contact rate of fresh leads. If recycled NA leads have a contact rate of 3% while fresh NA leads (after intra-day recycling) have a contact rate of 8%, your multi-day recycling delays may be too long --- the leads are going stale. If recycled leads are generating a high rate of Answering Machine or Hung Up dispositions, your delays may be too short --- you are calling the same people at the same time of day. The ideal delay produces a contact rate that is 40-60% of the fresh lead contact rate. Adjust delays in 2-hour increments and measure for a full week before making further changes.

### Can I use different caller IDs for recycled leads versus fresh leads?

Yes. VICIdial supports per-list caller ID overrides. In **Admin > Lists > [List ID] > List Modification**, set a different **Campaign CID** for your recycled lists. This is strongly recommended for Pass 3 and beyond. Prospects may have saved or blocked your Pass 1 caller ID after multiple contact attempts. A different number from a different area code gives you a fresh appearance. Rotate your recycled-lead caller IDs regularly and monitor them for spam labeling. Our guide on [predictive dialer settings](/blog/vicidial-predictive-dialer-settings/) covers DID rotation strategies in detail.

### What happens if I reset a list while intra-day recycling is still active?

List resets and intra-day recycling operate on different mechanisms and can conflict. A list reset sets `called_since_last_reset` to N for all leads in the list, which makes them eligible for the hopper regardless of recycling state. If you reset a list mid-shift while recycling delays are still counting down, you effectively override the recycling delays --- those leads become immediately eligible instead of waiting for their scheduled recycle time. For this reason, only run list resets between shifts or during off-hours. Never reset a list while agents are actively dialing from it unless you intentionally want to flush the recycling timers.

### How does lead recycling interact with VICIdial's callback system?

Leads in CALLBK (Callback) status are managed by VICIdial's callback system, not the recycling engine. Scheduled callbacks have the highest priority in the hopper and are dialed at the appointed time regardless of recycling rules. Once a callback is completed (the agent dials and dispositions the lead), the new disposition determines whether the lead re-enters recycling. If the callback results in NA, it goes back into the recycling pool at whatever attempt count it is currently at. The risk is failed callbacks that never get redialed --- see the Callback-Recycling Integration section above for a query that catches these orphaned leads. For full callback setup, see our [callback automation guide](/blog/vicidial-callback-automation/).

### Is there a way to automate the entire multi-pass recycling framework without manual SQL?

Yes. ViciStack's Recycling Engine automates the entire framework described in this guide --- pass transitions, disposition-weighted delays, rank adjustments, attempt caps, time-of-day variation, and retirement thresholds --- through a configuration interface instead of manual cron jobs and SQL queries. It also provides real-time dashboards showing per-pass conversion rates, recycling yield by disposition, and diminishing returns curves. If you are running the multi-pass framework manually and want to eliminate the cron job maintenance and reporting overhead, [schedule a free audit](/free-audit/) to see how ViciStack automates it.

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-lead-recycling).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
