# VICIdial Pause Codes and Agent Accountability Systems

In any call center with more than a handful of agents, pause time is where productivity goes to die. An agent paused for a legitimate one-minute bathroom break is doing exactly what they should. An agent who has been in "BREAK" status for 47 minutes while simultaneously browsing their phone is costing you roughly $4 per minute in lost dialing capacity. Multiply that by 5 agents doing it across a 50-seat floor, and you're burning $1,200/hour in dead weight.

VICIdial's pause code system gives you the foundation to track, categorize, and hold agents accountable for every second they're not on the phone. But the default setup barely scratches the surface. This guide covers how to build a complete accountability system around pause codes — from custom code design to real-time supervisor alerts to monthly scorecards.

## Default Pause Codes and Their Meanings

VICIdial ships with a minimal set of system-level pause codes. When an agent clicks "PAUSE" without selecting a specific code, the system records a blank pause code or the campaign's default. The system-level codes are:

| Code | Description | Type |
|------|-------------|------|
| LOGIN | Agent just logged in, hasn't unpaused yet | System |
| LAGGED | Agent connection lagged/timed out | System |
| DEAD | Dead call / agent channel went dead | System |
| DISPO | Agent is in disposition mode | System |

These system codes are automatically applied — agents don't choose them. The DISPO pause is particularly important to understand: every time an agent finishes a call and enters the disposition screen, VICIdial records pause time with code DISPO until the agent selects a disposition and goes back to READY. Long DISPO times often indicate agents who are stalling between calls.

Beyond these system codes, VICIdial creates some default campaign-level codes during installation, but they vary by version. In many installations, you'll find BREAK, LUNCH, and BATHROOM as pre-configured options.

## Creating Custom Pause Codes for Your Operation

Custom pause codes are configured per campaign (or per in-group for inbound). Navigate to **Admin > Pause Codes** or configure them within the campaign settings.

### Recommended Pause Code Structure for 25+ Agent Teams

Design your codes around the question: "What is this agent doing right now, and is it acceptable?"

**Productive Pauses (Acceptable — Expected)**

| Code | Name | Max Duration | Notes |
|------|------|-------------|-------|
| TRAIN | Training/Meeting | 60 min | Scheduled events only |
| COACH | 1-on-1 Coaching | 30 min | Supervisor-initiated |
| TECH | Technical Issue | 15 min | Workstation/headset/software |
| SYSDN | System Down | Unlimited | Mass event, supervisor confirms |

**Personal Pauses (Acceptable — Time-Limited)**

| Code | Name | Max Duration | Notes |
|------|------|-------------|-------|
| BRK | Break | 15 min | Scheduled breaks per shift |
| LUNCH | Lunch | 30 min | One per shift |
| BATH | Bathroom | 10 min | No pre-approval needed |
| PERS | Personal | 5 min | Quick personal matter |

**Compliance/Admin Pauses (Tracked — Necessary)**

| Code | Name | Max Duration | Notes |
|------|------|-------------|-------|
| ADMIN | Admin Task | 20 min | Data entry, paperwork |
| ESCAL | Escalation | 30 min | Working an escalated issue |
| QA | QA Review | 15 min | Listening to own calls |

### Configuration via VICIdial Admin

To add pause codes, go to **Admin > Pause Codes** and click "Add a New Pause Code." Fill in:

- **Pause Code:** Short code (up to 6 chars), e.g., `BRK`
- **Pause Code Name:** Descriptive name, e.g., `Scheduled Break`
- **Billable:** Yes/No (affects payroll reports if you use them)
- **Campaign:** The campaign this code applies to, or `--ALL--` for system-wide

If you are setting up multiple campaigns at once, repeat through the admin GUI for each campaign or use the VICIdial API to create them in bulk. A recommended starting set for most outbound operations:

| Code | Name | Billable |
|------|------|----------|
| BRK | Scheduled Break | NO |
| LUNCH | Lunch Break | NO |
| BATH | Bathroom | NO |
| PERS | Personal | NO |
| TECH | Technical Issue | YES |
| TRAIN | Training/Meeting | YES |
| COACH | 1-on-1 Coaching | YES |
| ADMIN | Admin Task | YES |
| ESCAL | Escalation Work | YES |
| QA | QA Review | YES |
| SYSDN | System Down | YES |

### Force Pause Code Selection

Critical setting: require agents to select a pause code every time they pause. Without this, agents will hit the pause button and the system logs a blank code, making your reporting useless.

In campaign settings, set:

- **Agent Pause Codes Active:** `FORCE`

This forces the agent to select from the dropdown before the pause takes effect. If set to just `Y`, agents can pause without selecting a code. `FORCE` is the only setting that produces reliable data.

To verify the current setting across all campaigns, run this read-only query:

```sql
SELECT campaign_id, agent_pause_codes_active
FROM vicidial_campaigns
WHERE active = 'Y';
```

If any campaign shows `Y` or `N` instead of `FORCE`, go to **Admin > Campaigns > [campaign name] > Detail** and change **Agent Pause Codes Active** to `FORCE`. Do this through the admin GUI so the change is logged in the vicidial_admin_log and validated against other campaign settings.

## Pause Code Reporting and Analysis

### Daily Pause Summary by Agent

This query gives you each agent's total time in each pause code for the current day:

```sql
SELECT
    val.user,
    vu.full_name AS agent_name,
    val.sub_status AS pause_code,
    vpc.pause_code_name,
    COUNT(*) AS pause_instances,
    SUM(val.pause_sec) AS total_pause_seconds,
    ROUND(SUM(val.pause_sec) / 60, 1) AS total_pause_minutes,
    MAX(val.pause_sec) AS longest_pause_seconds
FROM vicidial_agent_log val
JOIN vicidial_users vu ON val.user = vu.user
LEFT JOIN vicidial_pause_codes vpc
    ON val.sub_status = vpc.pause_code
    AND val.campaign_id = vpc.campaign_id
WHERE val.event_time >= CURDATE()
  AND val.pause_sec > 0
  AND val.sub_status != ''
GROUP BY val.user, val.sub_status
ORDER BY val.user, total_pause_seconds DESC;
```

### Excessive Pause Detection

Find agents who exceeded the maximum allowed pause time for each code:

```sql
-- Agents with any single pause instance over 20 minutes
SELECT
    val.user,
    vu.full_name AS agent_name,
    val.sub_status AS pause_code,
    val.pause_sec,
    ROUND(val.pause_sec / 60, 1) AS pause_minutes,
    val.event_time
FROM vicidial_agent_log val
JOIN vicidial_users vu ON val.user = vu.user
WHERE val.event_time >= CURDATE()
  AND val.pause_sec > 1200  -- 20 minutes
  AND val.sub_status NOT IN ('LUNCH', 'TRAIN', 'SYSDN')
ORDER BY val.pause_sec DESC;
```

### Campaign-Level Pause Analysis

Understand total productive vs. non-productive time at the campaign level:

```sql
SELECT
    val.campaign_id,
    ROUND(SUM(val.talk_sec) / 3600, 1) AS total_talk_hours,
    ROUND(SUM(val.pause_sec) / 3600, 1) AS total_pause_hours,
    ROUND(SUM(val.wait_sec) / 3600, 1) AS total_wait_hours,
    ROUND(SUM(val.dispo_sec) / 3600, 1) AS total_dispo_hours,
    ROUND(
        SUM(val.talk_sec) * 100.0 /
        NULLIF(SUM(val.talk_sec) + SUM(val.pause_sec) + SUM(val.wait_sec) + SUM(val.dispo_sec), 0),
        1
    ) AS talk_pct,
    ROUND(
        SUM(val.pause_sec) * 100.0 /
        NULLIF(SUM(val.talk_sec) + SUM(val.pause_sec) + SUM(val.wait_sec) + SUM(val.dispo_sec), 0),
        1
    ) AS pause_pct
FROM vicidial_agent_log val
WHERE val.event_time >= CURDATE()
GROUP BY val.campaign_id;
```

Healthy outbound campaigns typically show 55-70% talk time, 10-20% pause time, 5-15% wait time, and 5-10% dispo time. If pause percentage exceeds 25%, you have an accountability problem.

### DISPO Time Analysis — The Hidden Pause

DISPO time is technically a pause (the agent isn't taking calls), but it's system-initiated. However, agents learn to abuse it — staying in the disposition screen while texting or chatting, then finally selecting a dispo after several minutes.

```sql
SELECT
    user,
    ROUND(AVG(dispo_sec), 1) AS avg_dispo_seconds,
    ROUND(MAX(dispo_sec), 1) AS max_dispo_seconds,
    COUNT(*) AS total_dispos,
    ROUND(SUM(dispo_sec) / 60, 1) AS total_dispo_minutes
FROM vicidial_agent_log
WHERE event_time >= CURDATE()
  AND campaign_id = 'SALESCAMP'
  AND dispo_sec > 0
GROUP BY user
ORDER BY avg_dispo_seconds DESC;
```

Average dispo time should be 5-15 seconds for most outbound campaigns. An agent averaging 45+ seconds per disposition is either struggling with the interface (training issue) or deliberately stalling (accountability issue). Compare against the campaign median to identify outliers.

## Agent Scorecard Based on Pause Behavior

A comprehensive agent scorecard combines pause metrics with productivity metrics to create a single accountability framework.

### Scorecard Query — Daily

```sql
SELECT
    val.user,
    vu.full_name AS agent_name,

    -- Productivity Metrics
    COUNT(DISTINCT CASE WHEN val.talk_sec > 0 THEN val.agent_log_id END) AS total_calls,
    ROUND(SUM(val.talk_sec) / 3600, 2) AS talk_hours,
    ROUND(SUM(val.talk_sec) / NULLIF(COUNT(DISTINCT CASE WHEN val.talk_sec > 0 THEN val.agent_log_id END), 0), 0) AS avg_talk_sec,

    -- Pause Metrics
    ROUND(SUM(val.pause_sec) / 60, 1) AS total_pause_min,
    COUNT(DISTINCT CASE WHEN val.pause_sec > 0 AND val.sub_status NOT IN ('','LOGIN','DISPO','LAGGED','DEAD') THEN val.agent_log_id END) AS pause_events,

    -- Dispo Metrics
    ROUND(AVG(val.dispo_sec), 1) AS avg_dispo_sec,

    -- Efficiency Score (talk time as % of logged time)
    ROUND(
        SUM(val.talk_sec) * 100.0 /
        NULLIF(SUM(val.talk_sec) + SUM(val.pause_sec) + SUM(val.wait_sec) + SUM(val.dispo_sec), 0),
        1
    ) AS efficiency_pct,

    -- Pause Score (lower is better — inverse of pause %)
    ROUND(
        100 - (SUM(val.pause_sec) * 100.0 /
        NULLIF(SUM(val.talk_sec) + SUM(val.pause_sec) + SUM(val.wait_sec) + SUM(val.dispo_sec), 0)),
        1
    ) AS availability_score

FROM vicidial_agent_log val
JOIN vicidial_users vu ON val.user = vu.user
WHERE val.event_time >= CURDATE()
  AND val.campaign_id = 'SALESCAMP'
GROUP BY val.user
ORDER BY efficiency_pct DESC;
```

### Interpreting the Scorecard

| Efficiency % | Rating | Action |
|-------------|--------|--------|
| 65%+ | Excellent | Recognize and reward |
| 55-64% | Good | Standard performance |
| 45-54% | Below Average | Coaching conversation |
| Below 45% | Poor | Immediate intervention |

The `availability_score` (100 minus pause percentage) is a quick number to display on wallboards. Agents see their own number and know immediately where they stand.

### Weekly Trend Tracking

Track whether agents are improving or declining over time:

```sql
SELECT
    val.user,
    vu.full_name AS agent_name,
    DATE(val.event_time) AS work_date,
    ROUND(
        SUM(val.talk_sec) * 100.0 /
        NULLIF(SUM(val.talk_sec) + SUM(val.pause_sec) + SUM(val.wait_sec) + SUM(val.dispo_sec), 0),
        1
    ) AS efficiency_pct,
    ROUND(SUM(val.pause_sec) / 60, 1) AS pause_min
FROM vicidial_agent_log val
JOIN vicidial_users vu ON val.user = vu.user
WHERE val.event_time >= DATE_SUB(CURDATE(), INTERVAL 7 DAY)
  AND val.campaign_id = 'SALESCAMP'
GROUP BY val.user, DATE(val.event_time)
ORDER BY val.user, work_date;
```

An agent whose efficiency drops 5+ points day-over-day is either dealing with a personal issue, a system issue, or a motivation issue. The data tells you something changed — the supervisor's job is to find out what.

## Real-Time Pause Alerts for Supervisors

The most effective accountability happens in real time, not in end-of-day reports. Build alerts that notify supervisors the moment an agent exceeds their pause threshold.

### Simple Cron-Based Alert Script

```bash
#!/bin/bash
# /opt/scripts/pause_alert.sh — run every 2 minutes via cron
# */2 * * * * /opt/scripts/pause_alert.sh

ALERT_THRESHOLD=900  # 15 minutes in seconds
WEBHOOK_URL="https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK"

RESULTS=$(mysql -u cron -p'yourpass' asterisk -N -B -e "
SELECT
    va.user,
    vu.full_name,
    va.pause_code,
    UNIX_TIMESTAMP(NOW()) - UNIX_TIMESTAMP(va.last_state_change) AS seconds_paused,
    va.campaign_id
FROM vicidial_live_agents va
JOIN vicidial_users vu ON va.user = vu.user
WHERE va.status = 'PAUSED'
  AND UNIX_TIMESTAMP(NOW()) - UNIX_TIMESTAMP(va.last_state_change) > ${ALERT_THRESHOLD}
  AND va.pause_code NOT IN ('LUNCH', 'TRAIN', 'SYSDN', 'COACH')
ORDER BY seconds_paused DESC;
")

if [ -n "$RESULTS" ]; then
    MESSAGE="*Pause Alert* — Agents over $(($ALERT_THRESHOLD / 60)) minutes:\n"
    while IFS=$'\t' read -r user name code seconds campaign; do
        MINUTES=$((seconds / 60))
        MESSAGE="${MESSAGE}\n- *${name}* (${user}): ${code:-NO CODE} — ${MINUTES} min [${campaign}]"
    done <<< "$RESULTS"

    curl -s -X POST "$WEBHOOK_URL" \
        -H 'Content-type: application/json' \
        -d "{\"text\": \"${MESSAGE}\"}"
fi
```

### Advanced: Tiered Alert System

Different pause durations trigger different severity levels:

| Duration | Severity | Action |
|----------|----------|--------|
| 10 min (personal code) | Info | Slack message to supervisor |
| 20 min (any code except LUNCH/TRAIN) | Warning | Slack + direct message to agent |
| 30+ min | Critical | Slack + supervisor phone alert |

Implement tiering by running the check at different thresholds:

```sql
-- Critical: Any non-exempt pause over 30 minutes
SELECT va.user, vu.full_name, va.pause_code,
    ROUND((UNIX_TIMESTAMP(NOW()) - UNIX_TIMESTAMP(va.last_state_change)) / 60, 0) AS minutes_paused
FROM vicidial_live_agents va
JOIN vicidial_users vu ON va.user = vu.user
WHERE va.status = 'PAUSED'
  AND UNIX_TIMESTAMP(NOW()) - UNIX_TIMESTAMP(va.last_state_change) > 1800
  AND va.pause_code NOT IN ('LUNCH', 'TRAIN', 'SYSDN');

-- Warning: Personal pauses over 15 minutes
SELECT va.user, vu.full_name, va.pause_code,
    ROUND((UNIX_TIMESTAMP(NOW()) - UNIX_TIMESTAMP(va.last_state_change)) / 60, 0) AS minutes_paused
FROM vicidial_live_agents va
JOIN vicidial_users vu ON va.user = vu.user
WHERE va.status = 'PAUSED'
  AND UNIX_TIMESTAMP(NOW()) - UNIX_TIMESTAMP(va.last_state_change) > 900
  AND va.pause_code IN ('BRK', 'BATH', 'PERS');
```

### Wallboard Integration

Display a "longest current pause" metric on the supervisor wallboard. Nothing motivates an agent to unpause like seeing their name at the top of the screen with a running timer:

```sql
SELECT
    vu.full_name,
    va.pause_code AS code,
    ROUND((UNIX_TIMESTAMP(NOW()) - UNIX_TIMESTAMP(va.last_state_change)) / 60, 1) AS minutes_paused
FROM vicidial_live_agents va
JOIN vicidial_users vu ON va.user = vu.user
WHERE va.status = 'PAUSED'
ORDER BY (UNIX_TIMESTAMP(NOW()) - UNIX_TIMESTAMP(va.last_state_change)) DESC
LIMIT 5;
```

Display this on the wallboard with color coding: green under 5 minutes, yellow 5-15 minutes, red over 15 minutes.

## Best Practices for 25+ Agent Teams

### 1. Set Clear Expectations on Day One

Print or display the pause code policy. Every agent should know:
- Which codes to use for which situations
- Maximum allowed duration for each code
- That pause time is tracked in real time and reviewed daily
- Consequences for excessive or uncoded pauses

### 2. Require Supervisor Approval for Extended Pauses

For pauses expected to exceed 15 minutes (except LUNCH), require agents to radio or message the supervisor. This creates a human checkpoint without being micromanagerial.

### 3. Track Pause Frequency, Not Just Duration

An agent who takes twenty 3-minute breaks is as problematic as one who takes one 60-minute break. Track both:

```sql
SELECT
    user,
    COUNT(*) AS pause_count,
    ROUND(SUM(pause_sec) / 60, 1) AS total_minutes,
    ROUND(AVG(pause_sec), 0) AS avg_seconds_per_pause
FROM vicidial_agent_log
WHERE event_time >= CURDATE()
  AND pause_sec > 0
  AND sub_status NOT IN ('', 'LOGIN', 'DISPO', 'LAGGED', 'DEAD')
GROUP BY user
ORDER BY pause_count DESC;
```

### 4. Use Pause Data in Performance Reviews

When pause data backs up a coaching conversation, it's not opinion — it's fact. "Your efficiency was 42% last week compared to the team average of 61%. You had 14 break pauses averaging 12 minutes each." That's specific, measurable, and defensible.

### 5. Reward Good Pause Behavior

Don't only use pause tracking for punishment. Agents who consistently maintain 60%+ efficiency and use pause codes correctly should be recognized. Post the top 5 efficiency scores weekly. Consider tying bonuses to efficiency metrics alongside conversion metrics.

### 6. Review Pause Codes Quarterly

As your operation evolves, so should your codes. If agents are consistently using `ADMIN` for tasks that should be their own code (e.g., CRM updates), create a specific code for it. If a code is never used, remove it to simplify the agent interface.

### 7. Handle the DISPO Loophole

Some agents learn that DISPO time isn't flagged the same way as explicit pauses. Combat this by:

- Setting `dispo_max` in campaign settings to auto-unpause agents after a maximum dispo duration
- Flagging agents whose average dispo time exceeds 2x the campaign median

To set this, go to **Admin > Campaigns > [campaign name] > Detail** and change the **Dispo Max** field to `60`. After 60 seconds in the disposition screen, VICIdial will automatically move the agent back to READY or PAUSED, preventing indefinite dispo camping.

### 8. Segment Analysis by Shift

If you run multiple shifts, compare pause patterns across shifts. Night shifts often have higher pause percentages — this might be acceptable (lighter call volume, more downtime) or it might indicate weaker supervision:

```sql
SELECT
    CASE
        WHEN HOUR(event_time) BETWEEN 8 AND 15 THEN 'Day Shift'
        WHEN HOUR(event_time) BETWEEN 16 AND 23 THEN 'Evening Shift'
        ELSE 'Night Shift'
    END AS shift,
    ROUND(AVG(
        talk_sec * 100.0 / NULLIF(talk_sec + pause_sec + wait_sec + dispo_sec, 0)
    ), 1) AS avg_efficiency_pct,
    ROUND(AVG(pause_sec) / 60, 1) AS avg_pause_min_per_event
FROM vicidial_agent_log
WHERE event_time >= DATE_SUB(CURDATE(), INTERVAL 7 DAY)
  AND campaign_id = 'SALESCAMP'
GROUP BY shift;
```

## How ViciStack Helps

Building an accountability system is the first step. Making it stick — and continuously improving based on the data — is where most operations struggle. At ViciStack, we've implemented pause code accountability systems for call centers ranging from 25 to 500+ agents.

Our approach includes:

- **Custom pause code design** tailored to your specific operation type (insurance, solar, political, collections, etc.)
- **Automated alert pipelines** that notify the right supervisor at the right time, without alert fatigue
- **Weekly agent scorecards** delivered to managers with trend analysis and outlier flagging
- **Dashboard integration** that puts pause accountability front and center on wallboards
- **Policy templates** that give you the documentation agents sign on day one

All included in $150/agent/month. No per-minute charges. No surprise fees.

**Find out how much pause time is costing your operation** — get a free analysis: [Request Your Free ViciStack Analysis](https://vicistack.com/proof/)

## Related Resources

- [VICIdial Real-Time Agent Dashboard Customization Guide](/blog/vicidial-realtime-agent-dashboard) — Display pause metrics on wallboards for instant visibility
- [VICIdial Auto-Dial Level Tuning by Campaign Type](/blog/vicidial-auto-dial-level-tuning) — Agent availability (reduced pausing) directly impacts optimal dial levels
- [VICIdial Asterisk CDR Analysis for Connect Rate Optimization](/blog/vicidial-asterisk-cdr-analysis) — Connect CDR data with agent behavior patterns

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-pause-codes-accountability).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
