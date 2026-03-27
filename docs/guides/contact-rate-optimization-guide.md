# Contact Rate Optimization Guide: DNC Scrubbing, Best Dial Windows, and the Numbers That Actually Move Revenue

**Last updated: March 2026 | Reading time: ~28 minutes**

Your contact rate is the single metric that determines whether your dialer operation makes money or burns it.

Not your close rate. Not your talk time. Not your fancy CRM integration. The contact rate -- the percentage of leads your agents actually reach and have a real conversation with -- sits upstream of every other number that matters. If you can not get [humans on the phone](/blog/contact-rate-optimization/), nothing else in your pipeline works.

And the reality in 2026 is brutal. Hiya analyzed 46.75 billion unknown calls in 2023 and found that 28% were tagged as suspected spam or fraud. Carriers are blocking aggressively. STIR/SHAKEN attestation matters more than ever. The days of loading a list and pressing go are over.

Most outbound operations run 15-25% daily contact rates on fresh, healthy lists with properly managed DIDs. That means 75-85% of your dialing effort produces nothing. The difference between a 15% contact rate and a 25% contact rate -- on the same list -- is the difference between profitability and payroll that does not cover itself.

This guide covers the operational playbook for moving that number: DNC scrubbing that actually protects you, time-of-day analysis that finds your best windows, DID management that keeps your numbers clean, and the VICIdial configuration to tie it all together.

## Measuring Your Baseline Before You Touch Anything

You can not optimize what you have not measured. Before changing a single setting, pull three numbers from your current operation.

### The Three Contact Rate Metrics

**Raw Answer Rate** -- calls answered divided by total calls placed, times 100. This includes answering machines, wrong numbers, and hang-ups. It tells you how many calls get picked up, period.

```
Raw Answer Rate = (Calls Answered / Total Calls Placed) × 100
```

**Right-Party Contact Rate** -- live conversations with the intended person divided by total leads attempted, times 100. This strips out wrong numbers, gatekeepers, and wrong-party answers. It tells you how well your data matches reality.

```
Right-Party Contact Rate = (Right-Party Conversations / Total Leads Attempted) × 100
```

**Qualified Contact Rate** -- meaningful conversations where the agent got through the qualifying script divided by total leads attempted. This is the number that ties directly to revenue.

```
Qualified Contact Rate = (Qualified Conversations / Total Leads Attempted) × 100
```

In VICIdial, you can pull these from the campaign status report. The raw answer rate comes from the [outbound calling](/blog/vicidial-vs-gohighlevel/) report. Right-party and qualified contacts come from your disposition analysis:

```sql
SELECT
    campaign_id,
    COUNT(*) AS total_calls,
    SUM(CASE WHEN status IN ('SALE','CALLBK','XFER','LRERR') THEN 1 ELSE 0 END) AS right_party,
    SUM(CASE WHEN status IN ('SALE','XFER') THEN 1 ELSE 0 END) AS qualified,
    ROUND(SUM(CASE WHEN status NOT IN ('NA','B','DC','N','NP','AFTHRS')
        THEN 1 ELSE 0 END) / COUNT(*) * 100, 1) AS raw_answer_pct,
    ROUND(SUM(CASE WHEN status IN ('SALE','CALLBK','XFER','LRERR')
        THEN 1 ELSE 0 END) / COUNT(*) * 100, 1) AS rpc_pct
FROM vicidial_log
WHERE call_date >= DATE_SUB(NOW(), INTERVAL 7 DAY)
GROUP BY campaign_id
ORDER BY rpc_pct DESC;
```

Run this for the last 7 days. Write the numbers down. You will compare against them after every change.

### Per-DID Performance Baseline

Most operations never look at contact rate by individual phone number. This is a mistake. A single spam-labeled DID in your rotation can drag your entire campaign contact rate down 20-50% overnight.

```sql
SELECT
    LEFT(phone_number, 10) AS outbound_did,
    COUNT(*) AS total_dials,
    SUM(CASE WHEN status NOT IN ('NA','B','DC','N','NP','AFTHRS')
        THEN 1 ELSE 0 END) AS answered,
    ROUND(SUM(CASE WHEN status NOT IN ('NA','B','DC','N','NP','AFTHRS')
        THEN 1 ELSE 0 END) / COUNT(*) * 100, 1) AS answer_rate
FROM vicidial_log
WHERE call_date >= DATE_SUB(NOW(), INTERVAL 7 DAY)
GROUP BY outbound_did
HAVING total_dials > 50
ORDER BY answer_rate ASC;
```

Sort ascending. The DIDs at the bottom -- the ones with 3-5% answer rates when your average is 18% -- are almost certainly spam-labeled. Pull them from rotation immediately.

## DNC Scrubbing: The Compliance Layer That Also Saves Money

DNC scrubbing is not just a compliance checkbox. It is a contact rate optimization tool. Every call you place to a DNC-registered number is a wasted dial attempt, a wasted agent second of wait time, and a potential $43,792 fine per violation under the TCPA.

### The Three-Layer DNC Scrub

**Layer 1: Federal DNC Registry**

The FTC maintains the National [Do Not Call](/blog/vicidial-dnc-management/) Registry. You are required to scrub against it before every campaign. The registry is updated daily, and numbers can be added at any time.

In VICIdial, configure your DNC scrub in the Admin panel:

```
Admin > System Settings > DNC Scrub:  AREACODE
Filter DNC Numbers:                    Y
DNC Check Enabled:                     Y
```

VICIdial checks the internal DNC table at dial time. But you also need to run the list through the federal registry before import. The API is available at `telemarketing.donotcall.gov` and most list providers offer pre-scrubbed data.

**Layer 2: State DNC Lists**

Thirty-two states maintain their own DNC registries, and some have requirements stricter than the federal list. Indiana, Pennsylvania, and Texas are the most aggressive enforcers. Your scrub process needs to include state lists for every state you are dialing into.

| State | Registry | Additional Rules |
|-------|----------|-----------------|
| Indiana | Separate from federal | 24-hour processing requirement |
| Pennsylvania | Separate state list | Written consent required for some verticals |
| Texas | Separate state list | No calls before 9AM or after 9PM local |
| California | Uses federal list | CCPA data deletion adds DNC-like requirements |
| Florida | Separate state list | 18-month re-scrub requirement |

**Layer 3: Internal DNC**

Every opt-out request from any channel -- phone, email, web form, social media -- must go into your internal DNC list within the timeframe required by the FCC (as of April 2025, opt-out requests must be honored within 10 business days).

In VICIdial, agents add numbers to the internal DNC via the agent interface or disposition codes. But you need to make sure your [web forms](/blog/vicidial-agent-screen-customization/) and email unsubscribes also feed into the same table:

```bash
# Add a number to VICIdial internal DNC via the non-agent API
curl "https://your-vicidial-server/vicidial/non_agent_api.php?\
source=api&user=apiuser&pass=apipass&\
function=add_dnc_phone&phone_number=5551234567"
```

### Phone Number Validation

Before you even get to DNC scrubbing, validate the numbers on your list. Disconnected numbers, landlines in a mobile-only campaign, and wrong-format numbers all waste dial attempts.

Run your list through a carrier lookup API (Telnyx, Twilio, or Bandwidth all offer this) and strip:

- Disconnected numbers
- Landlines (if your campaign targets mobile)
- VoIP numbers (if your vertical has low VoIP engagement)
- Numbers that have been ported in the last 30 days (often reassigned)

A clean list scrub typically removes 15-25% of records. That sounds like you are shrinking your list, but what you are actually doing is removing every record that would have produced a wasted dial. Your contact rate per dial goes up because you stopped dialing dead numbers.

## Time-of-Day Dialing: Finding Your Best Windows

The time you place a call affects whether it gets answered more than almost any other variable. But the specific best hours depend on your campaign type, your lead source, and your geography. There is no universal "best time to call."

### Building a Time-of-Day Matrix

Pull your historical answer rates by hour for each campaign:

```sql
SELECT
    HOUR(call_date) AS dial_hour,
    COUNT(*) AS total_dials,
    SUM(CASE WHEN status NOT IN ('NA','B','DC','N','NP','AFTHRS')
        THEN 1 ELSE 0 END) AS answered,
    ROUND(SUM(CASE WHEN status NOT IN ('NA','B','DC','N','NP','AFTHRS')
        THEN 1 ELSE 0 END) / COUNT(*) * 100, 1) AS answer_rate,
    SUM(CASE WHEN status IN ('SALE','CALLBK','XFER')
        THEN 1 ELSE 0 END) AS conversions
FROM vicidial_log
WHERE call_date >= DATE_SUB(NOW(), INTERVAL 30 DAY)
    AND campaign_id = 'YOUR_CAMPAIGN'
GROUP BY HOUR(call_date)
ORDER BY dial_hour;
```

You will typically see a pattern like this for B2C campaigns:

| Hour (Local) | Answer Rate | Conversion Rate | Notes |
|:---:|:---:|:---:|---|
| 8 AM | 12-15% | Low | People rushing to work, annoyed |
| 9 AM | 16-19% | Medium | Settled in but busy |
| 10 AM | 19-23% | Medium-High | Morning productivity lull |
| 11 AM | 18-22% | Medium | Pre-lunch, decent window |
| 12 PM | 14-17% | Low | Lunch break, screens calls |
| 1 PM | 15-18% | Medium | Post-lunch, groggy |
| 2 PM | 17-20% | Medium | Afternoon settling in |
| 3 PM | 16-19% | Medium | School pickup starts affecting parents |
| 4 PM | 20-24% | High | End of workday, checking phones |
| 5 PM | 22-26% | High | Commute/home, more available |
| 6 PM | 23-27% | Highest | Dinner prep, actually home |
| 7 PM | 20-24% | High | Evening, relaxed |
| 8 PM | 15-19% | Medium | Getting late, some pushback |

The sweet spots for most B2C operations are 10-11 AM and 4-7 PM local time. But these numbers shift by vertical. Solar leads answer better in the afternoon. Insurance leads peak mid-morning. Debt collection sees the best pickup rates in early evening.

### VICIdial Timezone Dialing Configuration

VICIdial handles timezone-aware dialing through the `local_call_time` setting. Set this per campaign:

```
Campaign > Detail > Local Call Time: 9am-9pm
Campaign > Detail > Use Timezone Awareness: Y
Campaign > Detail > Timezone Selection: LOCAL
```

To target specific time windows within the legal dialing hours, use the hopper priority system. Set leads in your peak time zones to higher hopper priority during their optimal hours:

```
Admin > Call Times > Add Call Time
    Call Time ID:    PEAK_WINDOW
    Call Time Name:  Peak Answer Window
    Start Time:      1000
    End Time:        1100
    Start Time 2:    1600
    End Time 2:      1900
```

### Multi-Timezone Campaign Strategy

If you are dialing nationally across US time zones, stagger your campaign start across zones. At 10 AM Eastern, it is 7 AM Pacific -- outside legal dialing hours in most states. VICIdial handles this automatically when timezone awareness is enabled, but you should also manually verify that your hopper is not starving agents during zone transitions.

Check hopper status by timezone:

```bash
# Check hopper distribution across timezones
mysql -u cron -p vicidial -e "
SELECT
    gmt_offset_now,
    COUNT(*) AS leads_in_hopper
FROM vicidial_hopper h
JOIN vicidial_list l ON h.lead_id = l.lead_id
WHERE h.campaign_id = 'YOUR_CAMPAIGN'
GROUP BY gmt_offset_now
ORDER BY gmt_offset_now;"
```

If you see zero leads for a timezone that should be active, your call time configuration is filtering them out or your list data has bad timezone assignments.

## DID Rotation and Caller ID Reputation Management

This is where most operations lose the contact rate game in 2026. You can have perfect data, perfect timing, and a perfectly tuned dialer -- and still get 8% answer rates because your outbound numbers are [flagged as spam](/blog/vicidial-did-management/).

### The DID Lifecycle

Every phone number you dial from has a reputation. New DIDs start neutral. As you use them, carriers and analytics companies (Hiya, TNS, First Orion, Nomorobo) build a profile. After a certain volume of complaints or call patterns that look like robocalling, the number gets labeled.

Once a DID is labeled "Spam Likely" or "Scam Likely," your answer rate on that number drops 20-50% overnight. In many cases, the carrier blocks the call entirely -- the consumer's phone never rings.

### Rotation Rules

The core principle: spread your call volume across enough DIDs that no single number triggers spam heuristics.

```
Maximum calls per DID per day:     50
Maximum calls per DID per hour:    10-15
Cool-down period after labeling:   14-30 days (or retire permanently)
DID-to-agent ratio:                3-5 DIDs per active agent
```

In VICIdial, configure CID rotation at the campaign level:

```
Campaign > Detail > Manual Dial CID: AGENT_PHONE
Campaign > Detail > Use CID Group: Y
Campaign > Detail > CID Group: your_cid_group
```

Create a CID group with your DID pool:

```
Admin > CID Groups > Add CID Group
    CID Group ID:     ROTATION_POOL
    CID Group Name:   Main Rotation Pool
    CID Group Type:   AREACODE
```

The `AREACODE` type enables local presence dialing -- VICIdial matches the outbound CID to the area code of the number being dialed. This alone can increase answer rates. Research from Software Advice found prospects are nearly 4x more likely to answer calls from local numbers.

### Reputation Monitoring

Check your DIDs weekly at minimum. Use these free and paid tools:

- **Free Caller Registry** (freecallerregistry.com) -- check [spam labeling](/blog/vicidial-caller-id-reputation/) status
- **Hiya** -- carrier-level reputation lookup
- **TNS Call Guardian** -- AT&T's spam scoring system
- **First Orion** -- T-Mobile labeling

Automate the check with a script that queries your DID pool:

```bash
#!/bin/bash
# did-health-check.sh - Check all active DIDs for spam labeling
DIDS=$(mysql -u cron -pPASS vicidial -N -e \
  "SELECT DISTINCT caller_id FROM vicidial_campaign_cid_areacodes WHERE active='Y'")

for did in $DIDS; do
    echo "Checking $did..."
    # Query Free Caller Registry API (replace with your actual API endpoint)
    result=$(curl -s "https://api.freecallerregistry.com/lookup?number=$did&apikey=YOUR_KEY")
    status=$(echo "$result" | jq -r '.spamStatus')
    if [ "$status" != "clean" ]; then
        echo "  WARNING: $did flagged as $status"
        # Auto-disable in VICIdial
        mysql -u cron -pPASS vicidial -e \
          "UPDATE vicidial_campaign_cid_areacodes SET active='N' WHERE caller_id='$did'"
    fi
done
```

### STIR/SHAKEN Attestation

As of 2026, STIR/SHAKEN attestation is not optional. Carriers increasingly block or label calls without full attestation. There are three levels:

| Level | Meaning | Impact on Contact Rate |
|:---:|---|---|
| A (Full) | Carrier verified the caller, their right to use the number, and the number itself | Best answer rates, passes through most filters |
| B (Partial) | Carrier verified the caller and their network origin, but not the specific number | Moderate impact, some filtering |
| C (Gateway) | Carrier only verified where the call entered their network | Frequently blocked or labeled |

You need A-level attestation on every outbound DID. Talk to your SIP trunk provider (Telnyx, Bandwidth, Twilio, VoIP.ms) about their attestation policies. If your provider can not guarantee A-level on your numbers, switch providers. The contact rate difference between A-level and C-level attestation is 30-60% in many markets.

## Attempt Cadence: How Many Times and How Far Apart

Most leads require multiple contact attempts before you reach them. The question is how many attempts, how far apart, and across which channels.

### The Multi-Attempt Framework

Industry data shows:

| Attempt | Cumulative Contact Rate | Incremental Gain |
|:---:|:---:|:---:|
| 1st | 15-20% | -- |
| 2nd | 25-30% | +10% |
| 3rd | 32-38% | +7% |
| 4th | 37-42% | +5% |
| 5th | 40-45% | +3% |
| 6th | 42-47% | +2% |
| 7th+ | 43-48% | <2% |

Diminishing returns kick in hard after attempt 5-6. Most operations should cap at 6-8 total attempts [per lead](/blog/call-center-cost-per-lead-benchmarks/), spread across 7-[14 days](/blog/vicidial-roi-case-study/).

### Configuring Attempt Cadence in VICIdial

VICIdial controls re-dial timing through the list recycling settings:

```
Campaign > Detail > Dial Timeout:        26 seconds
Campaign > Detail > Max Attempts:         8
Campaign > Auto-Recycle > NA Recycle:     Y
Campaign > Auto-Recycle > NA Attempt Delay: 3600  (1 hour first retry)
Campaign > Auto-Recycle > NA Max Attempts:  3
Campaign > Auto-Recycle > B Recycle:      Y
Campaign > Auto-Recycle > B Attempt Delay:  7200 (2 hours for busy)
Campaign > Auto-Recycle > B Max Attempts:   3
```

For more granular control, use the [lead recycling](/blog/vicidial-lead-recycling/) system with status-specific retry intervals:

| Disposition | Retry After | Max Retries | Notes |
|---|:---:|:---:|---|
| NA (No Answer) | 1 hour, then 4 hours, then next day | 3 | Vary the time of day |
| B (Busy) | 2 hours | 3 | Often means they screened it |
| AM (Answering Machine) | 4 hours | 2 | Try a different time slot |
| DC (Disconnect) | Never | 0 | Dead number, stop wasting dials |
| NP (No Phone) | Never | 0 | Strip from list |

### Speed to Lead for Inbound-Generated Leads

If your leads come from web forms, landing pages, or inbound inquiries, the clock starts the moment the lead submits. Research consistently shows that [speed to lead](/blog/speed-to-lead-response-time/) is the biggest factor in contact rate for warm leads.

Leads contacted within 5 minutes of submission convert at 8x the rate of leads contacted after 30 minutes. In VICIdial, use the real-time lead insertion API to add new leads directly to the hopper with maximum priority:

```bash
# Insert a new web lead with immediate priority
curl "https://your-vicidial-server/vicidial/non_agent_api.php?\
source=api&user=apiuser&pass=apipass&\
function=add_lead&\
phone_number=5551234567&\
first_name=John&last_name=Doe&\
list_id=YOUR_HOT_LIST&\
campaign_id=YOUR_CAMPAIGN&\
hopper_priority=99&\
duplicate_check=DUPLIST"
```

The `hopper_priority=99` flag pushes this lead to the front of the dialing queue. Combined with a dedicated high-priority campaign for web leads, you can get agents calling back within 60-90 seconds of form submission.

## Putting It Together: The Weekly Optimization Cycle

Contact rate optimization is not a one-time project. It is a weekly operational discipline.

**Monday: DID Health Check**
Run your spam labeling check script across all active DIDs. Retire any flagged numbers. Order replacement DIDs if your pool is shrinking.

**Tuesday: List Quality Audit**
Check disconnected number rates from the previous week. If more than 10% of your dials hit disconnects, your list source has a data quality problem.

**Wednesday: Time-of-Day Analysis**
Pull your hourly answer rate report. Compare against the previous week. Adjust your dialing windows if a time slot degraded.

**Thursday: Attempt Cadence Review**
Check how many leads have exhausted their maximum attempts. Review the conversion rate by attempt number. If attempts 6-8 are producing zero conversions, cap at 5.

**Friday: Campaign Performance Summary**
Compare this week's contact rate against last week. Identify which changes moved the number and which did not.

### The Contact Rate Improvement Checklist

Here is the condensed version for operations that want to move fast:

1. Run three-layer DNC scrub (federal, state, internal) on every list before dialing
2. Validate phone numbers through carrier lookup -- strip disconnects and wrong-type numbers
3. Pull per-DID answer rates -- retire anything under 10% immediately
4. Set DID rotation to 50 calls per number per day maximum
5. Enable local presence dialing through CID groups
6. Verify A-level STIR/SHAKEN attestation on every outbound DID
7. Build timezone-aware dialing windows targeting 10-11 AM and 4-7 PM local time
8. Set attempt cadence to 6-8 attempts over 7-14 days with status-specific retry intervals
9. For web leads, implement real-time API insertion with hopper priority 99 and sub-2-minute callback
10. Measure baseline before changes, re-measure weekly, and kill what does not work

If your contact rate is sitting below 15% and you implement all of the above, you should see it climb into the 22-28% range within 2-3 weeks. That is a 50-80% improvement in the number of conversations your agents have per day -- on the same list, with the same agents.

The teams at [ViciStack](https://vicistack.com/) typically see contact rate improvements of 40-60% within the first two weeks of engagement. We tune the dialer, clean the DID pool, fix the data pipeline, and build the monitoring to keep it there. If you are burning agent hours on a contact rate problem and want it fixed fast, [that is what we do](https://vicistack.com/contact/).

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/contact-rate-optimization-guide).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
