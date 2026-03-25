# VICIdial Timezone-Aware Dialing and TCPA Safe Hours Compliance

One TCPA violation costs $500 per call. Willful violations cost $1,500 per call. A single dialing session that calls 200 leads outside of safe hours could generate $100,000 to $300,000 in liability. This is not theoretical -- class action TCPA lawsuits are a thriving industry, and plaintiff attorneys actively look for call centers that dial outside legal windows.

VICIdial has built-in timezone detection and call time enforcement. But "built-in" does not mean "configured correctly out of the box." Most VICIdial installations we audit have gaps in their timezone configuration -- missing state-specific rules, incorrect DST handling, or call_time definitions that do not account for every edge case.

This guide covers every aspect of timezone-compliant dialing in VICIdial: federal and state TCPA requirements, VICIdial's timezone detection system, configuring `call_time` and `state_call_time`, handling daylight saving time transitions, state-specific restrictions, and building an audit trail that protects you when (not if) someone challenges your compliance.

## TCPA Calling Window Requirements

### Federal Rules (FCC)

The Telephone Consumer Protection Act, as implemented by FCC regulations (47 CFR 64.1200), establishes the baseline:

- **Permitted calling hours**: 8:00 AM to 9:00 PM **in the called party's local time**
- **This applies to**: Telemarketing calls, prerecorded messages, and autodial calls
- **The critical detail**: The time zone is the *recipient's* time zone, not the caller's

This means if your call center is in Phoenix (MST, no DST) and you are dialing New York (EST), you must track New York's local time, including when New York shifts to EDT in spring.

### State-Specific Rules

Several states have calling windows that are more restrictive than the federal 8 AM - 9 PM rule. These state rules override the federal standard:

| State | Permitted Hours (Local Time) | Notes |
|-------|----------------------------|-------|
| **Federal (default)** | 8:00 AM - 9:00 PM | Baseline |
| **Oklahoma** | 8:00 AM - 8:00 PM | 1 hour earlier cutoff |
| **Oregon** | 9:00 AM - 9:00 PM | 1 hour later start (proposed) |
| **Washington** | 8:00 AM - 8:00 PM | 1 hour earlier cutoff |
| **Indiana** | Complex -- see below | Split timezone state |
| **Florida** | 8:00 AM - 9:00 PM | Same as federal, but state-enforced |
| **Texas** | 8:00 AM - 9:00 PM (noon on Sunday) | Sunday restriction |
| **Louisiana** | 8:00 AM - 9:00 PM | Prohibits calls on certain holidays |

**Important**: State laws change. This table reflects common restrictions as of this writing. Always verify current requirements with legal counsel. The FTC's Telemarketing Sales Rule (TSR) also applies to outbound telemarketing and mirrors the 8 AM - 9 PM window.

### The Indiana Problem

Indiana is the poster child for timezone complexity. Until 2006, most of Indiana did not observe daylight saving time. Today, the state is split:

- **Most of Indiana**: Eastern time (observes DST)
- **Northwest Indiana** (near Chicago): Central time (observes DST)
- **Southwest Indiana** (near Evansville): Central time (observes DST)

This means Indiana zip codes can be in either Eastern or Central time. VICIdial's postal code-to-timezone lookup handles this correctly if you have the right timezone data loaded, but verify it. A wrong timezone assignment for Gary, Indiana (Central time) vs. Indianapolis (Eastern time) means you might call Gary an hour early.

### The Arizona Problem

Arizona does not observe daylight saving time, except for the Navajo Nation, which does. For most call centers:

- **Standard time**: Arizona is MST year-round (same as Mountain Standard)
- **During DST months**: Arizona is the same as Pacific Daylight Time (PDT), not MDT
- **Navajo Nation**: Observes DST (MDT during summer)

VICIdial typically handles Arizona as a fixed offset, but verify your timezone data accounts for the DST non-participation.

## VICIdial's Built-in Timezone Detection

VICIdial determines each lead's timezone using the postal code (zip code) stored in the `vicidial_list` table. Here is how the system works:

### Postal Code to GMT Offset

VICIdial maintains a `vicidial_postal_codes` table that maps zip codes to GMT offsets:

```sql
-- Check the postal code table
SELECT postal_code, state, GMT_offset, DST, DST_range, country
FROM vicidial_postal_codes
WHERE postal_code = '10001'  -- Manhattan, NY
LIMIT 1;
```

Expected output:

```
postal_code: 10001
state: NY
GMT_offset: -5.00
DST: Y
DST_range: SSM-FSN
country: USA
```

- **GMT_offset**: The standard (non-DST) offset from GMT. New York is GMT-5 (Eastern Standard Time).
- **DST**: Whether this postal code observes daylight saving time (Y/N).
- **DST_range**: When DST is in effect. `SSM-FSN` means Second Sunday of March to First Sunday of November.

### How VICIdial Calculates Local Time

When the hopper evaluates a lead for dialing eligibility:

1. Read the lead's postal code from `vicidial_list`
2. Look up GMT_offset and DST flag from `vicidial_postal_codes`
3. Calculate the lead's current local time:
   - If DST=Y and current date is within DST_range: local_time = server_GMT + GMT_offset + 1
   - If DST=N or outside DST_range: local_time = server_GMT + GMT_offset
4. Compare local_time against the campaign's `call_time` definition
5. Only add the lead to the hopper if local_time is within the permitted window

### The gmt_offset_now Field

VICIdial also maintains a real-time field `gmt_offset_now` in `vicidial_list` that reflects the lead's current GMT offset (including DST adjustment). This is updated by the `AST_VDhopper.pl` script:

```sql
-- Check a lead's current timezone data
SELECT lead_id, phone_number, postal_code, state,
       gmt_offset_now, called_since_last_reset
FROM vicidial_list
WHERE lead_id = 12345;
```

The `gmt_offset_now` value is what the hopper actually uses for time filtering. If this field is wrong, leads will be dialed at the wrong time regardless of your `call_time` settings.

### Verifying Timezone Data Accuracy

Run this audit query to check for problems:

```sql
-- Find leads with missing or suspicious timezone data
SELECT COUNT(*) AS total,
       SUM(CASE WHEN postal_code = '' OR postal_code IS NULL THEN 1 ELSE 0 END) AS no_zip,
       SUM(CASE WHEN gmt_offset_now IS NULL OR gmt_offset_now = 0 THEN 1 ELSE 0 END) AS no_offset,
       SUM(CASE WHEN state = '' OR state IS NULL THEN 1 ELSE 0 END) AS no_state
FROM vicidial_list
WHERE list_id IN (SELECT list_id FROM vicidial_lists WHERE active = 'Y');
```

If `no_zip` is greater than zero, those leads have **no timezone protection**. VICIdial will either skip them entirely (safe) or assign them a default timezone (potentially unsafe, depending on configuration). Fix this by enriching your lead data before loading.

## Configuring call_time and state_call_time

VICIdial uses two systems for call time enforcement: `call_time` (the primary window) and `state_call_time` (per-state overrides).

### Defining a call_time

In VICIdial Admin, go to **Admin > Call Times** to define calling windows:

```
Call Time ID: TCPA_STANDARD
Call Time Name: TCPA Compliant 8am-9pm
Default Start: 800     (8:00 AM)
Default Stop: 2100     (9:00 PM)

Sunday Start: 800
Sunday Stop: 2100

Monday Start: 800
Monday Stop: 2100

Tuesday Start: 800
Tuesday Stop: 2100

Wednesday Start: 800
Wednesday Stop: 2100

Thursday Start: 800
Thursday Stop: 2100

Friday Start: 800
Friday Stop: 2100

Saturday Start: 800
Saturday Stop: 2100
```

Each day can have different hours. Some centers choose not to dial on Sundays:

```
Sunday Start: 0
Sunday Stop: 0     (No dialing on Sunday)
```

### Assigning call_time to Campaigns

In **Campaigns > [Your Campaign] > Detail**:

```
Local Call Time: TCPA_STANDARD
```

This tells VICIdial: "Only dial leads whose local time falls within this call_time definition."

### Configuring state_call_time

For states with restrictions tighter than your base `call_time`, use **State Call Times**. In VICIdial Admin, go to **Admin > Call Times > State Call Times**:

```
State Call Time ID: TCPA_STATES
State Call Time Name: State-Specific TCPA Overrides

Oklahoma:
  Start: 800
  Stop: 2000     (8:00 PM, not 9:00 PM)

Washington:
  Start: 800
  Stop: 2000     (8:00 PM)

Texas (Sunday):
  Sunday Start: 1200    (Noon)
  Sunday Stop: 2100     (9:00 PM)
```

Assign the state_call_time to your campaign:

```
Campaigns > [Campaign] > Detail:
  State Call Time: TCPA_STATES
```

VICIdial evaluates both `call_time` AND `state_call_time`. The more restrictive rule wins. So if your `call_time` says 8 AM - 9 PM, but `state_call_time` for Oklahoma says 8 AM - 8 PM, Oklahoma leads will stop dialing at 8 PM local time.

### Verifying Configuration

Test your call_time configuration with this SQL query:

```sql
-- Simulate timezone filtering for your active campaign
-- This shows how many leads are currently dialable by timezone
SELECT gmt_offset_now,
       COUNT(*) AS leads,
       TIME_FORMAT(
         ADDTIME(NOW(), SEC_TO_TIME(gmt_offset_now * 3600)),
         '%H:%i'
       ) AS local_time_now,
       CASE
         WHEN TIME_FORMAT(ADDTIME(NOW(), SEC_TO_TIME(gmt_offset_now * 3600)), '%H%i')
              BETWEEN '0800' AND '2100'
         THEN 'DIALABLE'
         ELSE 'BLOCKED'
       END AS status
FROM vicidial_list
WHERE list_id IN (
  SELECT list_id FROM vicidial_lists
  WHERE campaign_id = 'YOUR_CAMPAIGN' AND active = 'Y'
)
AND status IN ('NEW', 'CALLBK', 'A', 'B', 'NA')
GROUP BY gmt_offset_now
ORDER BY gmt_offset_now;
```

## Handling DST Transitions

Daylight saving time transitions are the most dangerous time for TCPA compliance. On the "spring forward" Sunday, 2:00 AM becomes 3:00 AM, meaning there are only 23 hours in the day. On the "fall back" Sunday, 1:00 AM happens twice.

### How VICIdial Handles DST

VICIdial's timezone system uses the `DST` and `DST_range` fields in `vicidial_postal_codes` to determine if a lead's area is currently in DST. The `AST_VDhopper.pl` script recalculates `gmt_offset_now` for each lead during hopper refills.

The DST transition happens at 2:00 AM local time, but VICIdial's hopper recalculation may not update instantly for all leads. This creates a window where some leads might have stale timezone data.

### DST Safety Buffer

Add a safety buffer around DST transitions:

```
# Instead of 8:00 AM - 9:00 PM, use:
Call Time: 8:15 AM - 8:45 PM

# This gives a 15-minute buffer on each end
# Cost: ~30 minutes of reduced dialing per day
# Benefit: protection during DST transitions and clock skew
```

Many compliance-conscious centers use 8:05 AM - 8:55 PM as a standard buffer even outside DST transitions. The lost dialing time is trivial compared to the risk.

### DST Transition Checklist

Run this verification the week before each DST transition:

```bash
#!/bin/bash
# /opt/vicistack/dst_check.sh
# Run before DST transitions (March and November)

echo "=== DST Transition Verification ==="
echo "Current server time: $(date)"
echo "Server timezone: $(cat /etc/timezone 2>/dev/null || timedatectl | grep 'Time zone')"

echo ""
echo "=== VICIdial Postal Code DST Flags ==="
mysql asterisk -e "
SELECT DST, COUNT(*) AS zip_codes
FROM vicidial_postal_codes
WHERE country = 'USA'
GROUP BY DST;
"

echo ""
echo "=== Sample DST=N States (should be AZ, HI) ==="
mysql asterisk -e "
SELECT DISTINCT state
FROM vicidial_postal_codes
WHERE DST = 'N' AND country = 'USA';
"

echo ""
echo "=== Leads with DST=Y that should be N (Arizona check) ==="
mysql asterisk -e "
SELECT vl.lead_id, vl.phone_number, vl.postal_code, vl.state, vl.gmt_offset_now,
       vpc.DST
FROM vicidial_list vl
JOIN vicidial_postal_codes vpc ON vl.postal_code = vpc.postal_code
WHERE vl.state = 'AZ' AND vpc.DST = 'Y'
LIMIT 10;
"

echo ""
echo "=== Current Active Campaigns Call Times ==="
mysql asterisk -e "
SELECT campaign_id, local_call_time
FROM vicidial_campaigns
WHERE active = 'Y';
"
```

### Forcing a Timezone Recalculation

After a DST transition, or if you suspect timezone data is stale, force a recalculation:

```bash
# Restart the hopper script (it recalculates on startup)
# On the VICIdial server:
screen -r� AST_VDhopper
# Or restart the VICIdial services:
/usr/share/astguiclient/ADMIN_keepalive_ALL.pl --cu3way
```

Or manually update `gmt_offset_now` for a specific list:

```sql
-- Recalculate gmt_offset_now for all leads in list 1001
UPDATE vicidial_list vl
JOIN vicidial_postal_codes vpc ON vl.postal_code = vpc.postal_code
SET vl.gmt_offset_now = CASE
  WHEN vpc.DST = 'Y'
    AND CURDATE() BETWEEN
      -- Approximate DST start: second Sunday of March
      DATE_ADD(MAKEDATE(YEAR(CURDATE()), 1), INTERVAL (13 - DAYOFWEEK(MAKEDATE(YEAR(CURDATE()), 1)) + 7 * 1) DAY + INTERVAL 2 MONTH)
      AND
      -- Approximate DST end: first Sunday of November
      DATE_ADD(MAKEDATE(YEAR(CURDATE()), 1), INTERVAL (7 - DAYOFWEEK(MAKEDATE(YEAR(CURDATE()), 1)) + 7 * 0) DAY + INTERVAL 10 MONTH)
  THEN vpc.GMT_offset + 1
  ELSE vpc.GMT_offset
END
WHERE vl.list_id = 1001;
```

Note: This SQL approximation is for illustration. VICIdial's own DST calculation logic is more precise. Prefer restarting the hopper script over manual SQL updates.

## State-Specific Restrictions: Edge Cases

### Oklahoma

Oklahoma's Do Not Call statute (15 O.S. Section 775B.4) restricts telemarketing calls to 8:00 AM - 8:00 PM Central Time. This is straightforward to implement in `state_call_time`:

```
Oklahoma:
  Start: 800
  Stop: 2000
```

### Indiana Split Timezone

Indiana has counties in both Eastern and Central time. The VICIdial `vicidial_postal_codes` table should correctly map Indiana zip codes to their respective time zones:

```sql
-- Verify Indiana timezone assignments
SELECT postal_code, state, GMT_offset, DST
FROM vicidial_postal_codes
WHERE state = 'IN'
AND GMT_offset = -6.00   -- Central time
ORDER BY postal_code;
```

Central Time Indiana zip codes include areas around Gary (464xx), Hammond, and Evansville (476xx-477xx). If your `vicidial_postal_codes` table does not differentiate these, all Indiana leads will be assigned Eastern time, and you will call Central time Indiana leads an hour early.

### Hawaii and Alaska

Hawaii does not observe DST and is GMT-10. Alaska is GMT-9 with DST. Common mistake: dialing Hawaii at 8 AM EST (which is 3 AM HST). Ensure your `call_time` enforcement works for these extreme offsets:

```sql
-- Check that Hawaiian leads are properly classified
SELECT postal_code, state, GMT_offset, DST
FROM vicidial_postal_codes
WHERE state = 'HI'
LIMIT 5;

-- Should show GMT_offset = -10.00, DST = N
```

### US Territories

Puerto Rico (GMT-4, no DST), Guam (GMT+10), US Virgin Islands (GMT-4), American Samoa (GMT-11): if your lists contain these postal codes, verify they are in your `vicidial_postal_codes` table with correct offsets.

## Documenting Compliance for Audits

When a TCPA complaint or lawsuit arrives, you need to prove that you had compliant systems in place AND that they were functioning correctly at the time of the alleged violation.

### What to Document

1. **Call time configuration**: Screenshot or export your `call_time` and `state_call_time` settings
2. **Timezone data accuracy**: Periodic audit of `vicidial_postal_codes` table
3. **Hopper logs**: VICIdial logs which leads were added to the hopper and when
4. **Call detail records**: `vicidial_log` records the exact time of every dial attempt
5. **System time accuracy**: NTP sync verification

### Automated Compliance Audit Script

Run this daily and archive the output:

```bash
#!/bin/bash
# /opt/vicistack/compliance_audit.sh
# Run daily via cron. Archive output for legal review.

AUDIT_DIR="/var/log/vicistack/compliance"
mkdir -p ${AUDIT_DIR}
AUDIT_FILE="${AUDIT_DIR}/audit_$(date +%Y%m%d_%H%M).txt"

{
echo "=========================================="
echo "TCPA Compliance Audit - $(date)"
echo "=========================================="

echo ""
echo "--- Server Time Verification ---"
echo "System time: $(date)"
echo "NTP sync status:"
chronyc tracking 2>/dev/null || ntpstat 2>/dev/null || echo "NTP status unavailable"

echo ""
echo "--- Call Time Definitions ---"
mysql asterisk -e "
SELECT call_time_id, call_time_name,
       ct_default_start, ct_default_stop,
       ct_sunday_start, ct_sunday_stop
FROM vicidial_call_times;"

echo ""
echo "--- State Call Time Overrides ---"
mysql asterisk -e "
SELECT state_call_time_id, state_call_time_state,
       sct_default_start, sct_default_stop
FROM vicidial_state_call_times
WHERE state_call_time_state != '';"

echo ""
echo "--- Active Campaign Call Time Assignments ---"
mysql asterisk -e "
SELECT campaign_id, campaign_name, local_call_time
FROM vicidial_campaigns
WHERE active = 'Y';"

echo ""
echo "--- Calls Made Outside 8am-9pm Local Time (Past 24h) ---"
echo "(These should be zero. Any results indicate a compliance issue.)"
mysql asterisk -e "
SELECT vl.uniqueid, vl.call_date, vl.phone_number, vl.campaign_id,
       vl.user, vlist.state, vlist.postal_code, vlist.gmt_offset_now,
       TIME_FORMAT(
         ADDTIME(vl.call_date, SEC_TO_TIME(vlist.gmt_offset_now * 3600)),
         '%H:%i'
       ) AS local_call_time
FROM vicidial_log vl
JOIN vicidial_list vlist ON vl.lead_id = vlist.lead_id
WHERE vl.call_date >= DATE_SUB(NOW(), INTERVAL 24 HOUR)
AND (
  TIME_FORMAT(ADDTIME(vl.call_date, SEC_TO_TIME(vlist.gmt_offset_now * 3600)), '%H%i') < '0800'
  OR
  TIME_FORMAT(ADDTIME(vl.call_date, SEC_TO_TIME(vlist.gmt_offset_now * 3600)), '%H%i') > '2100'
)
LIMIT 50;"

echo ""
echo "--- Leads with Missing Timezone Data ---"
mysql asterisk -e "
SELECT list_id, COUNT(*) AS leads_no_tz
FROM vicidial_list
WHERE (postal_code = '' OR postal_code IS NULL)
AND status IN ('NEW', 'CALLBK', 'A', 'B', 'NA')
AND list_id IN (SELECT list_id FROM vicidial_lists WHERE active = 'Y')
GROUP BY list_id;"

echo ""
echo "=========================================="
echo "Audit complete."
echo "=========================================="

} > ${AUDIT_FILE} 2>&1

# Compress audits older than 30 days
find ${AUDIT_DIR} -name "audit_*.txt" -mtime +30 -exec gzip {} \;

# Alert if any out-of-window calls were found
OUT_OF_WINDOW=$(grep -c "rows in set" ${AUDIT_FILE} 2>/dev/null)
if echo "${AUDIT_FILE}" | grep -q "Calls Made Outside"; then
    # More precise check would parse the SQL output
    echo "Compliance audit saved to ${AUDIT_FILE}"
fi
```

Cron entry:

```
0 22 * * * /opt/vicistack/compliance_audit.sh
```

### Retention

Keep compliance audit logs for a minimum of **4 years** (the TCPA statute of limitations). Compress and archive monthly:

```bash
# Monthly archive
tar -czf /backup/compliance/compliance_$(date +%Y%m).tar.gz \
  /var/log/vicistack/compliance/audit_$(date +%Y%m)*.txt.gz
```

## Best Practices Summary

1. **Never rely solely on VICIdial defaults.** Configure `call_time` and `state_call_time` explicitly for every active campaign.

2. **Add a time buffer.** Use 8:05 AM - 8:55 PM instead of 8:00 AM - 9:00 PM. The 10 minutes of lost dialing is insignificant compared to one violation.

3. **Validate postal code data on lead import.** Reject or flag leads with missing or invalid zip codes. No zip code means no timezone protection.

4. **Audit weekly.** Run the compliance audit script and review the "calls outside window" section. Any non-zero result needs investigation.

5. **Track DST transitions.** Mark the second Sunday of March and the first Sunday of November on your calendar. Verify timezone data on the Monday after each transition.

6. **Keep state restrictions current.** State telemarketing laws change. Review at least quarterly with legal counsel.

7. **Use NTP.** If your server clock drifts even 5 minutes, you could dial early or late. Ensure NTP is running and synced:

```bash
# Verify NTP sync
chronyc tracking
# Should show: "Leap status: Normal" and "System time: 0.000... seconds"

# Or with ntpd
ntpq -p
# Should show at least one server with * (active sync)
```

8. **Document everything.** If you cannot prove compliance, you were not compliant. Auditors and courts want records, not verbal assurances.

## How ViciStack Helps

TCPA compliance is not a set-it-and-forget-it task. State laws change, DST transitions create risk windows, lead data quality varies, and VICIdial's timezone system has edge cases that are easy to miss. A single audit gap can cost more than a year of operational budget.

ViciStack manages TCPA-compliant timezone dialing for VICIdial call centers:

- **Complete call_time and state_call_time configuration** covering all 50 states plus territories
- **Postal code data validation** on every lead import, with automatic timezone enrichment
- **DST transition management** with pre- and post-transition verification
- **Daily compliance auditing** with automated alerting for any out-of-window dials
- **Quarterly compliance reviews** incorporating state law changes
- **Audit-ready documentation** that satisfies legal teams and regulatory inquiries

At $150/agent/month, compliance management is included -- not an add-on. Compare that to a single TCPA violation at $500-$1,500 per call.

**Get a free TCPA compliance audit for your VICIdial deployment.** We will review your call_time settings, timezone data accuracy, and check for out-of-window dial attempts in your recent call history.

[Request your free ViciStack analysis](https://vicistack.com/proof/) -- response in under 5 minutes.

## Related Articles

- [VICIdial Quality Assurance Scoring with Call Recordings](/blog/vicidial-qa-scoring) -- compliance extends to recording disclosures and agent behavior
- [VICIdial CNAM Lookup Integration for Inbound Routing](/blog/vicidial-cnam-lookup-inbound) -- handle the callbacks generated by your compliant outbound dialing
- [VICIdial Database Partitioning for High-Volume Call Centers](/blog/vicidial-database-partitioning) -- manage the vicidial_log data that documents your compliance

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-timezone-dialing-tcpa).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
