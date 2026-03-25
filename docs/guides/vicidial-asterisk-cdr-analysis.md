# VICIdial Asterisk CDR Analysis for Connect Rate Optimization

Your VICIdial system generates a CDR (Call Detail Record) for every single call attempt. In a 50-agent outbound operation dialing aggressively, that's tens of thousands of records per day. Most call center operators treat the CDR as a logging artifact — something that exists for billing disputes and compliance audits. That's leaving money on the table.

The CDR is the most granular data source you have for understanding what happens between the moment a call leaves your system and the moment it either connects to a human or doesn't. When you analyze it systematically, you can identify carriers that silently fail, time windows where connect rates spike, and call patterns that distinguish a live answer from dead air or a fast busy.

This guide covers the CDR table structure, the queries that matter, and how to turn raw CDR data into connect rate gains.

## CDR Table Structure in VICIdial's Asterisk Database

VICIdial stores CDR data in the `cdr` table within the `asteriskcdrdb` database. This is separate from VICIdial's own `vicidial_log` table in the `asterisk` database. Both contain call data, but the CDR table has Asterisk-level detail that VICIdial's application layer doesn't expose.

Connect to the CDR database:

```bash
mysql -u cron -p asteriskcdrdb
```

The core columns you'll work with:

| Column | Type | Description |
|--------|------|-------------|
| `calldate` | datetime | When the call was initiated |
| `clid` | varchar | Caller ID sent with the call |
| `src` | varchar | Source channel/extension |
| `dst` | varchar | Destination number dialed |
| `dcontext` | varchar | Dialplan context |
| `channel` | varchar | Asterisk channel used (reveals trunk/carrier) |
| `dstchannel` | varchar | Destination channel |
| `lastapp` | varchar | Last Asterisk application executed |
| `lastdata` | varchar | Arguments to the last application |
| `duration` | int | Total call duration in seconds (ring + talk) |
| `billsec` | int | Billable seconds (talk time only — after answer) |
| `disposition` | varchar | Call outcome: ANSWERED, NO ANSWER, BUSY, FAILED |
| `amaflags` | int | AMA flag for billing categorization |
| `accountcode` | varchar | Account code (often campaign ID) |
| `uniqueid` | varchar | Unique call identifier |
| `userfield` | varchar | Custom field (VICIdial sometimes stores data here) |

The two most important columns for connect rate analysis are `disposition` and `billsec`. Together they tell you: did the call connect, and if so, for how long?

## Key Fields Deep Dive

### disposition — The Outcome

Asterisk sets disposition to one of four values:

- **ANSWERED** — The remote side sent a 200 OK / answer signal. This includes human answers, voicemail pickups, IVR systems, and answering machines.
- **NO ANSWER** — Ring timeout expired with no answer. Your `dial` timeout in the dialplan controls how long Asterisk waits.
- **BUSY** — Remote side returned a 486 Busy Here or equivalent.
- **FAILED** — Call could not be placed. Trunk error, invalid number, network failure, or all circuits busy.

A high FAILED rate on a specific trunk is a carrier problem. A high NO ANSWER rate at specific times is a scheduling optimization opportunity.

### billsec — The Real Duration

`billsec` is the number of seconds after the call was answered. `duration` includes ring time plus talk time. The delta between them tells you how long the phone rang:

```sql
SELECT
    duration - billsec AS ring_seconds,
    billsec AS talk_seconds,
    disposition
FROM cdr
WHERE calldate >= '2026-03-19 00:00:00'
  AND disposition = 'ANSWERED'
LIMIT 20;
```

Calls with `billsec` between 1-3 seconds that are marked ANSWERED are almost certainly voicemail greetings, IVRs, or answering machines that the AMD didn't catch. Calls with `billsec` of 0 and disposition ANSWERED are false positives — the carrier sent an answer signal but the call didn't actually connect.

### channel — The Carrier Fingerprint

The `channel` field reveals which trunk carried the call. In a typical VICIdial setup with SIP trunks:

```
SIP/carrier1-00001a2b
SIP/vicitrunk02-00003c4d
PJSIP/outbound-0000ef01
```

The trunk name before the hyphen identifies the carrier. This lets you compare connect rates, answer rates, and call quality across carriers using the same dialed numbers.

### lastapp — What Asterisk Did

The `lastapp` field shows the final Asterisk application that handled the call:

- **Dial** — Normal outbound call
- **Hangup** — Call was explicitly hung up
- **AGI** — Call went through an AGI script (common for AMD)
- **Playback** — Audio was played (voicemail drop, message blasting)
- **Park** — Call was parked

Most outbound calls will show `Dial` as lastapp. If you see `Hangup` with `lastdata` showing specific hangup causes, that's diagnostic data about why calls terminated.

## SQL Queries for Connect Rate Analysis

### Basic Connect Rate by Campaign

```sql
SELECT
    accountcode AS campaign,
    COUNT(*) AS total_attempts,
    SUM(CASE WHEN disposition = 'ANSWERED' THEN 1 ELSE 0 END) AS answered,
    SUM(CASE WHEN disposition = 'NO ANSWER' THEN 1 ELSE 0 END) AS no_answer,
    SUM(CASE WHEN disposition = 'BUSY' THEN 1 ELSE 0 END) AS busy,
    SUM(CASE WHEN disposition = 'FAILED' THEN 1 ELSE 0 END) AS failed,
    ROUND(SUM(CASE WHEN disposition = 'ANSWERED' THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 2) AS connect_rate_pct
FROM cdr
WHERE calldate >= '2026-03-18 00:00:00'
  AND calldate < '2026-03-19 00:00:00'
GROUP BY accountcode
ORDER BY connect_rate_pct DESC;
```

### True Human Connect Rate (Filtering Short Calls)

A raw ANSWERED rate overstates true connections because voicemail pickups and IVRs count as answers. Filter by `billsec` to isolate likely human conversations:

```sql
SELECT
    accountcode AS campaign,
    COUNT(*) AS total_attempts,
    SUM(CASE WHEN disposition = 'ANSWERED' AND billsec >= 15 THEN 1 ELSE 0 END) AS human_connects,
    SUM(CASE WHEN disposition = 'ANSWERED' AND billsec BETWEEN 1 AND 14 THEN 1 ELSE 0 END) AS short_answers,
    SUM(CASE WHEN disposition = 'ANSWERED' AND billsec = 0 THEN 1 ELSE 0 END) AS zero_sec_answers,
    ROUND(
        SUM(CASE WHEN disposition = 'ANSWERED' AND billsec >= 15 THEN 1 ELSE 0 END) * 100.0 / COUNT(*),
        2
    ) AS human_connect_rate_pct
FROM cdr
WHERE calldate >= CURDATE()
GROUP BY accountcode
ORDER BY human_connect_rate_pct DESC;
```

The 15-second threshold is a reasonable starting point. For B2B campaigns where conversations tend to be shorter at the introduction phase, you might lower this to 10 seconds. For consumer campaigns where agents have longer scripts, 20 seconds might be more appropriate. Calibrate by reviewing a sample of calls at the boundary.

## Identifying Dead Air, Short Calls, and Abandoned Calls

### Dead Air Detection

Dead air calls are ones where the system connected but nobody spoke — either the agent wasn't bridged fast enough, or AMD incorrectly classified the call. These show up as ANSWERED calls with very low billsec:

```sql
SELECT
    calldate,
    dst AS number_dialed,
    channel,
    billsec,
    duration,
    duration - billsec AS ring_time
FROM cdr
WHERE calldate >= CURDATE()
  AND disposition = 'ANSWERED'
  AND billsec BETWEEN 1 AND 4
ORDER BY calldate DESC
LIMIT 100;
```

If you're seeing a high percentage of 1-4 second ANSWERED calls, investigate:

1. **AMD timing** — If AMD takes too long to classify, the human hangs up during the "hello? hello?" phase.
2. **Agent bridge delay** — The time between AMD classifying "HUMAN" and the call reaching an available agent. If all agents are busy, the caller hears silence and hangs up.
3. **Carrier latency** — Some carriers introduce audio delay. The caller answers, says hello, hears nothing (because audio hasn't bridged), and hangs up.

### Short Call Distribution Analysis

Understanding the distribution of call lengths reveals patterns:

```sql
SELECT
    CASE
        WHEN billsec = 0 THEN '0 sec (no audio)'
        WHEN billsec BETWEEN 1 AND 3 THEN '1-3 sec (dead air/hangup)'
        WHEN billsec BETWEEN 4 AND 10 THEN '4-10 sec (quick reject)'
        WHEN billsec BETWEEN 11 AND 30 THEN '11-30 sec (short conversation)'
        WHEN billsec BETWEEN 31 AND 120 THEN '31-120 sec (pitch attempt)'
        WHEN billsec BETWEEN 121 AND 300 THEN '2-5 min (engaged call)'
        WHEN billsec > 300 THEN '5+ min (deep conversation)'
    END AS call_bucket,
    COUNT(*) AS call_count,
    ROUND(COUNT(*) * 100.0 / (SELECT COUNT(*) FROM cdr WHERE calldate >= CURDATE() AND disposition = 'ANSWERED'), 1) AS pct
FROM cdr
WHERE calldate >= CURDATE()
  AND disposition = 'ANSWERED'
GROUP BY call_bucket
ORDER BY MIN(billsec);
```

A healthy outbound campaign should have the majority of answered calls in the "short conversation" through "deep conversation" buckets. If more than 20% of your ANSWERED calls are in the 0-3 second bucket, you have a systemic connection quality problem.

### Abandoned Call Tracking

Abandoned calls (where the system dropped the call before an agent picked up) can be cross-referenced between CDR and VICIdial's drop log:

```sql
-- Calls that lasted under 2 seconds post-answer with no agent bridge
SELECT
    c.calldate,
    c.dst,
    c.channel,
    c.billsec,
    c.disposition
FROM asteriskcdrdb.cdr c
WHERE c.calldate >= CURDATE()
  AND c.disposition = 'ANSWERED'
  AND c.billsec <= 2
  AND c.dstchannel = ''
ORDER BY c.calldate DESC;
```

An empty `dstchannel` combined with a short billsec typically means the call was answered but never bridged to an agent — it was either dropped due to no available agents or killed by the system.

## Time-of-Day Analysis for Optimal Dialing Windows

Connect rates vary dramatically by hour. Analyzing CDR data by time reveals when your specific list demographics are most likely to answer.

### Hourly Connect Rate Heatmap Data

```sql
SELECT
    HOUR(calldate) AS hour_of_day,
    DAYOFWEEK(calldate) AS day_of_week,
    COUNT(*) AS attempts,
    SUM(CASE WHEN disposition = 'ANSWERED' AND billsec >= 15 THEN 1 ELSE 0 END) AS human_connects,
    ROUND(
        SUM(CASE WHEN disposition = 'ANSWERED' AND billsec >= 15 THEN 1 ELSE 0 END) * 100.0 / COUNT(*),
        2
    ) AS connect_rate
FROM cdr
WHERE calldate >= DATE_SUB(CURDATE(), INTERVAL 7 DAY)
  AND accountcode = 'SALESCAMP'
GROUP BY HOUR(calldate), DAYOFWEEK(calldate)
ORDER BY day_of_week, hour_of_day;
```

This gives you a 7-day view of connect rates by hour and day. Typical patterns for consumer lists:

- **B2C:** Peak connect rates between 10 AM-12 PM and 4 PM-7 PM local time. Worst rates before 9 AM and after 8:30 PM.
- **B2B:** Best connect rates Tuesday-Thursday, 9 AM-11 AM and 1:30 PM-3:30 PM. Monday mornings and Friday afternoons are dead zones.

### Per-Area-Code Timing Analysis

Different regions have different answer patterns. Extract the area code from the dialed number and analyze:

```sql
SELECT
    LEFT(dst, 3) AS area_code,
    HOUR(calldate) AS hour_of_day,
    COUNT(*) AS attempts,
    ROUND(
        SUM(CASE WHEN disposition = 'ANSWERED' AND billsec >= 15 THEN 1 ELSE 0 END) * 100.0 / COUNT(*),
        2
    ) AS connect_rate
FROM cdr
WHERE calldate >= DATE_SUB(CURDATE(), INTERVAL 14 DAY)
  AND accountcode = 'SALESCAMP'
  AND LENGTH(dst) >= 10
GROUP BY LEFT(dst, 3), HOUR(calldate)
HAVING attempts >= 50
ORDER BY area_code, hour_of_day;
```

The `HAVING attempts >= 50` filter ensures statistical relevance. With this data, you can build campaign-specific call schedules that prioritize area codes during their peak answer hours.

## Carrier Performance Comparison from CDR Data

If you're running multiple SIP trunks (and at 25+ agents, you should be for redundancy and A/B testing), the CDR tells you which carriers are actually delivering.

### Connect Rate by Carrier

```sql
SELECT
    SUBSTRING_INDEX(channel, '-', 1) AS carrier,
    COUNT(*) AS total_calls,
    SUM(CASE WHEN disposition = 'ANSWERED' THEN 1 ELSE 0 END) AS answered,
    SUM(CASE WHEN disposition = 'FAILED' THEN 1 ELSE 0 END) AS failed,
    ROUND(SUM(CASE WHEN disposition = 'ANSWERED' THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 2) AS answer_rate,
    ROUND(SUM(CASE WHEN disposition = 'FAILED' THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 2) AS fail_rate,
    ROUND(AVG(CASE WHEN disposition = 'ANSWERED' THEN billsec ELSE NULL END), 1) AS avg_talk_sec
FROM cdr
WHERE calldate >= DATE_SUB(CURDATE(), INTERVAL 7 DAY)
GROUP BY SUBSTRING_INDEX(channel, '-', 1)
ORDER BY answer_rate DESC;
```

### Post-Connect Duration by Carrier

Two carriers might have similar raw answer rates, but one delivers longer conversations (indicating better audio quality and caller experience):

```sql
SELECT
    SUBSTRING_INDEX(channel, '-', 1) AS carrier,
    COUNT(*) AS answered_calls,
    ROUND(AVG(billsec), 1) AS avg_billsec,
    ROUND(STDDEV(billsec), 1) AS stddev_billsec,
    SUM(CASE WHEN billsec >= 60 THEN 1 ELSE 0 END) AS calls_over_60s,
    ROUND(SUM(CASE WHEN billsec >= 60 THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 1) AS pct_over_60s
FROM cdr
WHERE calldate >= DATE_SUB(CURDATE(), INTERVAL 7 DAY)
  AND disposition = 'ANSWERED'
  AND billsec > 0
GROUP BY SUBSTRING_INDEX(channel, '-', 1)
ORDER BY avg_billsec DESC;
```

The `pct_over_60s` metric is particularly telling. A carrier with a 20% answer rate but 70% of those calls lasting over 60 seconds is far more valuable than one with a 25% answer rate where only 30% of calls exceed 60 seconds. The first carrier is delivering real conversations; the second is delivering a lot of answering machines.

### Carrier Failure Analysis

When a carrier returns FAILED disposition, the `lastdata` and Asterisk logs contain the SIP response code. Common codes and their meaning:

| SIP Code | Meaning | Action |
|----------|---------|--------|
| 403 | Forbidden | Carrier blocking your caller ID or IP |
| 404 | Not Found | Invalid number format for this carrier |
| 480 | Temporarily Unavailable | Carrier congestion |
| 486 | Busy Here | Legitimate busy |
| 487 | Request Terminated | Call cancelled (your side hung up during ring) |
| 503 | Service Unavailable | Carrier trunk exhaustion |

```sql
SELECT
    SUBSTRING_INDEX(channel, '-', 1) AS carrier,
    lastdata,
    COUNT(*) AS occurrences
FROM cdr
WHERE calldate >= CURDATE()
  AND disposition = 'FAILED'
GROUP BY SUBSTRING_INDEX(channel, '-', 1), lastdata
ORDER BY occurrences DESC
LIMIT 30;
```

A spike in 403 responses from a carrier means they may have flagged your numbers. A spike in 503 means you're overrunning their capacity. Both require immediate action.

## Building a CDR Analysis Routine

For ongoing optimization, run these analyses on a schedule:

1. **Daily:** Connect rate by hour, carrier fail rate, short-call percentage
2. **Weekly:** Carrier comparison, area code timing patterns, call duration distribution
3. **Monthly:** Trend analysis — are connect rates improving or declining? Is a carrier degrading over time?

Automate the daily checks with a cron job that dumps results to a file or sends them to Slack:

```bash
#!/bin/bash
# /opt/scripts/daily_cdr_report.sh
MYSQL_CMD="mysql -u cron -p'yourpass' asteriskcdrdb -N -B"

echo "=== Daily CDR Report $(date +%Y-%m-%d) ==="
echo ""
echo "--- Connect Rate by Campaign ---"
$MYSQL_CMD -e "
SELECT accountcode, COUNT(*) AS attempts,
    ROUND(SUM(CASE WHEN disposition='ANSWERED' AND billsec>=15 THEN 1 ELSE 0 END)*100.0/COUNT(*),2) AS human_connect_pct
FROM cdr WHERE calldate >= CURDATE() - INTERVAL 1 DAY AND calldate < CURDATE()
GROUP BY accountcode ORDER BY human_connect_pct DESC;
"

echo ""
echo "--- Carrier Fail Rates ---"
$MYSQL_CMD -e "
SELECT SUBSTRING_INDEX(channel,'-',1) AS carrier,
    COUNT(*) AS calls,
    ROUND(SUM(CASE WHEN disposition='FAILED' THEN 1 ELSE 0 END)*100.0/COUNT(*),2) AS fail_pct
FROM cdr WHERE calldate >= CURDATE() - INTERVAL 1 DAY AND calldate < CURDATE()
GROUP BY carrier ORDER BY fail_pct DESC;
"
```

## How ViciStack Helps

CDR analysis is one of the highest-leverage optimizations in any VICIdial operation, but it requires someone who lives in this data every day. Most operations are too busy managing agents and running campaigns to build and maintain a CDR analysis pipeline.

At ViciStack, CDR analysis is central to what we do:

- **Automated carrier scoring** — We continuously rank your trunks by human connect rate, not just raw answer rate, and route traffic accordingly
- **Time-of-day optimization** — We build custom dialing schedules based on your specific list demographics and CDR patterns
- **Dead air elimination** — We tune AMD sensitivity, agent bridge timing, and trunk selection to minimize the 1-4 second answered calls that burn through your list
- **Carrier negotiation leverage** — When CDR data proves a carrier is underperforming, we provide the data you need for rate negotiations or replacement decisions

This is included in ViciStack's $150/agent/month flat rate. No per-minute fees, no surprises.

**See what your CDR data reveals about your connect rates** — get a free analysis in under 5 minutes: [Request Your Free ViciStack Analysis](https://vicistack.com/proof/)

## Related Resources

- [VICIdial Real-Time Agent Dashboard Customization Guide](/blog/vicidial-realtime-agent-dashboard) — Visualize CDR-derived metrics in real-time wallboards
- [VICIdial Auto-Dial Level Tuning by Campaign Type](/blog/vicidial-auto-dial-level-tuning) — Use connect rate data to inform dial level decisions
- [VICIdial Voicemail Drop Configuration and Compliance Guide](/blog/vicidial-voicemail-drop) — AMD accuracy directly impacts CDR-based connect rate calculations

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-asterisk-cdr-analysis).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
