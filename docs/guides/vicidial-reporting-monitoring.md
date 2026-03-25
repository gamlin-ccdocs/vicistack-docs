# VICIdial Reporting & Real-Time Monitoring Guide

**Everything you need to extract actionable data from VICIdial — real-time monitoring, historical reports, the MySQL tables that matter, custom queries for every KPI, and how to build dashboards that actually help you run your operation.**

---

VICIdial stores everything. Every dial, every agent state change, every call disposition, every second of talk time — it's all in MySQL tables with millions of rows of operational data. The problem isn't data availability. The problem is that VICIdial's built-in reporting interface was designed in the mid-2000s, and it shows.

The built-in reports work. They're functional, they're accurate, and they cover the basics. But if you're trying to run a data-driven operation — one where managers make real-time decisions based on campaign performance, where supervisors catch problems before they become crises, and where executives see clear KPI dashboards instead of wall-of-text HTML reports — you need to go beyond the defaults.

This guide covers VICIdial's reporting system from the ground up: what's built in, what's in the database, how to query it, and how to build real-time dashboards that turn VICIdial's raw data into operational intelligence. For the analytics dashboard overview, see our [analytics feature page](/features/analytics-dashboard/).

---

## VICIdial's Built-In Reporting: What You Get Out of the Box

VICIdial includes a substantial set of built-in reports accessible from the admin interface under **Admin → Reports** (or via direct URL at `/vicidial/admin.php` → Reports tab). Here's what's available and when each report is useful.

### Real-Time Reports

These are the reports you keep open on a wallboard or supervisor screen throughout the day.

#### Real-Time Campaign Summary

**URL:** `/vicidial/realtime_report.php`

This is the primary real-time monitoring screen. It shows, for each active campaign:

- **Agents logged in:** Count by status (INCALL, READY, PAUSED, DEAD)
- **Calls ringing:** Currently ringing but not yet answered
- **Calls in IVR:** In the AMD or IVR queue waiting for classification
- **Calls waiting for agents:** Connected calls in queue waiting for an available agent
- **Dial level:** Current adaptive dial level
- **Dropped today:** Count and percentage of abandoned calls (critical for [TCPA compliance](/blog/vicidial-tcpa-compliance/))
- **Hopper count:** Number of leads loaded in the hopper and ready to dial

**How to use it:** This should be on a wallboard visible to every supervisor. The critical metrics to watch are:

- **Calls waiting > 0 for more than 5 seconds:** You don't have enough agents or your dial level is too high. Agents should be picking up connects within 1-2 seconds.
- **Drop percentage approaching 2.5%:** Reduce dial level immediately. See our [TCPA compliance guide](/blog/vicidial-tcpa-compliance/) for the 3% rule.
- **Hopper count dropping below 50:** The hopper is running dry, meaning the dialer will slow down or stop. Check your list availability, call time settings, and [time zone filtering](/settings/timezone-filtering/).
- **Agents in PAUSED state > 20%:** You're paying agents to not take calls. Investigate why they're pausing — break schedules, call avoidance, or system issues.

#### Agent Time Detail (Real-Time)

**URL:** `/vicidial/realtime_report.php` (agent detail section)

Shows each logged-in agent's current state and how long they've been in that state. The columns that matter:

| Column | What It Shows | What to Watch For |
|---|---|---|
| Status | Current agent state | PAUSED for > 5 minutes without a scheduled break |
| In Status | Time in current state | READY for > 2 minutes (no calls — dialer problem?) |
| Calls Today | Total calls handled | Agents significantly below average |
| Talk Time | Cumulative talk time | Unusually short or long averages |
| Wait Time | Cumulative wait time | High wait = underutilized agents |
| Pause Time | Cumulative pause time | High pause = productivity issue |

**Pro tip:** Sort by "In Status" descending to immediately spot agents who have been idle or paused for an unusual amount of time. An agent in PAUSED for 15 minutes during a non-break period is either having a system issue or avoiding calls.

#### AST_manager Interface

VICIdial's real-time data is served by the `AST_manager` process, which runs on each Asterisk server and communicates with the VICIdial web interface via the Manager API (TCP port 5038). The real-time reports poll this interface every few seconds (configurable).

If your real-time reports are showing stale data or not updating, check:

```bash
# Verify AST_manager is running on each dialer server
ps aux | grep AST_manager

# Check the manager connection
asterisk -rx "manager show connected"

# Verify the manager credentials in VICIdial's server settings
# Admin → Servers → [server] → Manager credentials
```

The `AST_manager_send.pl` and `AST_manager_listen.pl` scripts are the backbone of VICIdial's real-time data pipeline. If these processes crash or hang, real-time monitoring goes dark. Monitor them with your process monitoring tool (systemd, monit, or similar).

### Historical Reports

Historical reports pull from MySQL tables and show aggregated data over specified date ranges.

#### Outbound Calling Report

**URL:** `/vicidial/AST_VDADstats.php`

The most commonly used historical report. Shows campaign performance metrics over a date range:

- Total dials, answers, drops, human connects
- Average talk time, wait time, wrap time
- Dispositions breakdown
- Agent performance summaries

**Limitations:** The default interface shows one campaign at a time and the date range selector is basic. For cross-campaign comparisons or complex date analysis, you'll want custom queries (covered below).

#### Agent Performance Report

**URL:** `/vicidial/AST_agent_performance_detail.php`

Shows per-agent metrics:

- Calls handled
- Talk time (total, average)
- Pause time (total, by pause code)
- Wait time
- Dispositions breakdown
- Login duration

This report is essential for agent performance reviews and identifying coaching opportunities. An agent with significantly shorter average talk time might be rushing through calls. An agent with high pause time might need schedule adjustments or additional support.

#### Call Export / Recording Access

**URL:** `/vicidial/call_report_export.php`

Exports call-level data to CSV for external analysis. Each row represents one call with: date, phone number, agent, campaign, disposition, talk time, recording location, and custom fields.

This is VICIdial's data export mechanism. If you're feeding data to external BI tools, CRM systems, or compliance auditing platforms, this export is your starting point (though direct MySQL queries are more efficient for automated integrations).

---

## The VICIdial Database: Tables That Matter for Reporting

VICIdial's MySQL database is where the real reporting power lives. The built-in reports query these tables with canned queries — but you can write your own queries to answer any question about your operation.

Here are the key tables and what they contain:

### vicidial_log

**The most important table for outbound reporting.** Every outbound call attempt gets a row in `vicidial_log`.

| Column | Description |
|---|---|
| `uniqueid` | Unique call identifier |
| `lead_id` | Lead record ID |
| `list_id` | List the lead belongs to |
| `campaign_id` | Campaign that placed the call |
| `call_date` | Timestamp of the call |
| `start_epoch` | Unix timestamp of call start |
| `end_epoch` | Unix timestamp of call end |
| `length_in_sec` | Total call duration |
| `status` | Final disposition (NA, B, DROP, SALE, etc.) |
| `phone_code` | Country code |
| `phone_number` | Number dialed |
| `user` | Agent who handled the call (empty if not answered/dropped) |
| `term_reason` | How the call ended (AGENT, CALLER, QUEUETIMEOUT, etc.) |
| `alt_dial` | Whether this was an alt number dial |

**Common query: Daily campaign summary**

```sql
SELECT
    campaign_id,
    DATE(call_date) as call_day,
    COUNT(*) as total_dials,
    COUNT(CASE WHEN status NOT IN ('NA','B','DC','N','AFTHRS') THEN 1 END) as answered,
    COUNT(CASE WHEN status = 'DROP' THEN 1 END) as drops,
    COUNT(CASE WHEN user != '' AND status NOT IN ('NA','B','DC','N','DROP','AFTHRS','XDROP') THEN 1 END) as agent_handled,
    ROUND(AVG(CASE WHEN length_in_sec > 0 THEN length_in_sec END), 1) as avg_call_length,
    ROUND(COUNT(CASE WHEN status NOT IN ('NA','B','DC','N','AFTHRS') THEN 1 END) /
          NULLIF(COUNT(*), 0) * 100, 2) as answer_rate_pct
FROM vicidial_log
WHERE call_date >= '2026-03-01' AND call_date < '2026-03-19'
GROUP BY campaign_id, DATE(call_date)
ORDER BY campaign_id, call_day;
```

### vicidial_closer_log

**The inbound counterpart to vicidial_log.** Every inbound call (or call transferred to an in-group) gets a row here.

| Column | Description |
|---|---|
| `closecallid` | Unique identifier |
| `lead_id` | Lead record ID |
| `campaign_id` | In-group that received the call |
| `call_date` | Timestamp |
| `length_in_sec` | Call duration |
| `status` | Final disposition |
| `user` | Agent who handled the call |
| `queue_seconds` | Time spent waiting in queue |
| `term_reason` | How the call ended |

**Common query: Inbound queue performance**

```sql
SELECT
    campaign_id as ingroup,
    DATE(call_date) as call_day,
    COUNT(*) as total_calls,
    ROUND(AVG(queue_seconds), 1) as avg_queue_time,
    MAX(queue_seconds) as max_queue_time,
    COUNT(CASE WHEN queue_seconds <= 20 THEN 1 END) as answered_within_20s,
    ROUND(COUNT(CASE WHEN queue_seconds <= 20 THEN 1 END) / NULLIF(COUNT(*), 0) * 100, 1)
        as service_level_pct,
    COUNT(CASE WHEN status = 'ABANDON' THEN 1 END) as abandoned
FROM vicidial_closer_log
WHERE call_date >= '2026-03-01' AND call_date < '2026-03-19'
GROUP BY campaign_id, DATE(call_date)
ORDER BY campaign_id, call_day;
```

### vicidial_agent_log

**Agent state transitions.** Every time an agent changes state (login, logout, ready, incall, paused, etc.), a row is written here.

| Column | Description |
|---|---|
| `agent_log_id` | Unique identifier |
| `user` | Agent ID |
| `server_ip` | Server handling the agent |
| `event_time` | Timestamp of state change |
| `campaign_id` | Campaign agent is logged into |
| `pause_epoch` | When agent entered pause state |
| `pause_sec` | Duration of pause |
| `wait_epoch` | When agent entered wait/ready state |
| `wait_sec` | Duration of wait |
| `talk_epoch` | When agent entered talk state |
| `talk_sec` | Duration of talk |
| `dispo_epoch` | When agent entered dispo state |
| `dispo_sec` | Duration of disposition wrap-up |
| `status` | Disposition for this call |
| `sub_status` | Pause code (if paused) |

**Common query: Agent productivity breakdown**

```sql
SELECT
    user,
    campaign_id,
    COUNT(*) as total_calls,
    ROUND(SUM(talk_sec) / 60, 1) as total_talk_min,
    ROUND(AVG(talk_sec), 1) as avg_talk_sec,
    ROUND(SUM(wait_sec) / 60, 1) as total_wait_min,
    ROUND(SUM(pause_sec) / 60, 1) as total_pause_min,
    ROUND(SUM(dispo_sec) / 60, 1) as total_dispo_min,
    ROUND(SUM(talk_sec) / NULLIF(SUM(talk_sec + wait_sec + pause_sec + dispo_sec), 0) * 100, 1)
        as talk_pct
FROM vicidial_agent_log
WHERE event_time >= '2026-03-18' AND event_time < '2026-03-19'
GROUP BY user, campaign_id
ORDER BY talk_pct DESC;
```

The `talk_pct` (talk time as a percentage of total logged-in time) is arguably the single most important agent productivity metric. Top-performing outbound agents typically hit 45-55% talk time. Below 35% suggests a dialer tuning problem (agents waiting too long) or an agent behavior problem (excessive pausing).

### vicidial_list

**The lead table.** Contains every lead loaded into VICIdial.

| Column | Description |
|---|---|
| `lead_id` | Unique lead identifier |
| `list_id` | List the lead belongs to |
| `status` | Current lead status (NEW, SALE, CB, DNC, etc.) |
| `phone_number` | Primary phone number |
| `first_name`, `last_name` | Contact name |
| `called_count` | Number of dial attempts |
| `last_local_call_time` | Last time this lead was called |
| `gmt_offset_now` | Current GMT offset for timezone calculation |

**Common query: List penetration analysis**

```sql
SELECT
    list_id,
    COUNT(*) as total_leads,
    COUNT(CASE WHEN called_count = 0 THEN 1 END) as never_called,
    COUNT(CASE WHEN called_count BETWEEN 1 AND 3 THEN 1 END) as called_1_to_3,
    COUNT(CASE WHEN called_count > 3 THEN 1 END) as called_more_than_3,
    COUNT(CASE WHEN status IN ('SALE','SET','XFER') THEN 1 END) as converted,
    ROUND(COUNT(CASE WHEN status IN ('SALE','SET','XFER') THEN 1 END) /
          NULLIF(COUNT(*), 0) * 100, 2) as conversion_rate_pct
FROM vicidial_list
WHERE list_id IN (10001, 10002, 10003)
GROUP BY list_id;
```

### vicidial_campaign_stats

**Aggregated campaign statistics.** VICIdial periodically updates this table with running campaign totals. Useful for quick lookups without aggregating raw log data.

### vicidial_live_agents

**Real-time agent status.** This table is continuously updated and shows the current state of every logged-in agent. The real-time reports read from this table.

| Column | Description |
|---|---|
| `user` | Agent ID |
| `server_ip` | Server handling agent |
| `campaign_id` | Current campaign |
| `status` | Current state (READY, INCALL, PAUSED, etc.) |
| `callerid` | Current call ID (if on a call) |
| `lead_id` | Current lead (if on a call) |
| `last_state_change` | Timestamp of last state change |
| `pause_code` | Current pause code (if paused) |

**Quick query: Who's doing what right now?**

```sql
SELECT
    user,
    campaign_id,
    status,
    pause_code,
    TIMESTAMPDIFF(SECOND, last_state_change, NOW()) as seconds_in_state,
    callerid
FROM vicidial_live_agents
ORDER BY status, seconds_in_state DESC;
```

---

## Essential KPI Queries

Here are the queries for every KPI a call center manager needs. These are ready to run against your VICIdial MySQL database.

### Contact Rate (by campaign, by day)

The percentage of dials that result in a human connection:

```sql
SELECT
    campaign_id,
    DATE(call_date) as call_day,
    COUNT(*) as total_dials,
    COUNT(CASE WHEN status NOT IN ('NA','B','DC','N','AFTHRS','AB','ADC','XDROP')
               AND user != '' THEN 1 END) as human_contacts,
    ROUND(COUNT(CASE WHEN status NOT IN ('NA','B','DC','N','AFTHRS','AB','ADC','XDROP')
               AND user != '' THEN 1 END) /
          NULLIF(COUNT(*), 0) * 100, 2) as contact_rate_pct
FROM vicidial_log
WHERE call_date >= DATE_SUB(NOW(), INTERVAL 7 DAY)
GROUP BY campaign_id, DATE(call_date)
ORDER BY campaign_id, call_day;
```

### Conversion Rate (dials to sales/sets)

```sql
SELECT
    campaign_id,
    COUNT(*) as total_dials,
    COUNT(CASE WHEN status IN ('SALE','SET','XFER') THEN 1 END) as conversions,
    ROUND(COUNT(CASE WHEN status IN ('SALE','SET','XFER') THEN 1 END) /
          NULLIF(COUNT(*), 0) * 100, 3) as conversion_rate_pct,
    COUNT(CASE WHEN user != '' AND status NOT IN ('NA','B','DC','N','DROP','AFTHRS')
               THEN 1 END) as agent_contacts,
    ROUND(COUNT(CASE WHEN status IN ('SALE','SET','XFER') THEN 1 END) /
          NULLIF(COUNT(CASE WHEN user != '' AND status NOT IN ('NA','B','DC','N','DROP','AFTHRS')
               THEN 1 END), 0) * 100, 2) as contact_to_conversion_pct
FROM vicidial_log
WHERE call_date >= DATE_SUB(NOW(), INTERVAL 30 DAY)
GROUP BY campaign_id;
```

### Agent Leaderboard

```sql
SELECT
    vl.user,
    COUNT(*) as total_calls,
    COUNT(CASE WHEN vl.status IN ('SALE','SET','XFER') THEN 1 END) as conversions,
    ROUND(COUNT(CASE WHEN vl.status IN ('SALE','SET','XFER') THEN 1 END) /
          NULLIF(COUNT(*), 0) * 100, 2) as conversion_rate,
    ROUND(AVG(vl.length_in_sec), 1) as avg_call_length,
    ROUND(SUM(val.talk_sec) / 3600, 2) as total_talk_hours,
    ROUND(SUM(val.talk_sec) / NULLIF(SUM(val.talk_sec + val.wait_sec + val.pause_sec + val.dispo_sec), 0) * 100, 1)
        as utilization_pct
FROM vicidial_log vl
JOIN vicidial_agent_log val ON vl.uniqueid = val.uniqueid
WHERE vl.call_date >= DATE_SUB(NOW(), INTERVAL 7 DAY)
    AND vl.user != ''
GROUP BY vl.user
ORDER BY conversions DESC
LIMIT 20;
```

### Hourly Performance Heatmap

Identify your best and worst calling hours:

```sql
SELECT
    HOUR(call_date) as call_hour,
    DAYNAME(call_date) as day_of_week,
    COUNT(*) as dials,
    COUNT(CASE WHEN status NOT IN ('NA','B','DC','N','AFTHRS') THEN 1 END) as answered,
    ROUND(COUNT(CASE WHEN status NOT IN ('NA','B','DC','N','AFTHRS') THEN 1 END) /
          NULLIF(COUNT(*), 0) * 100, 2) as answer_rate,
    COUNT(CASE WHEN status IN ('SALE','SET','XFER') THEN 1 END) as conversions
FROM vicidial_log
WHERE call_date >= DATE_SUB(NOW(), INTERVAL 30 DAY)
GROUP BY HOUR(call_date), DAYNAME(call_date)
ORDER BY DAYOFWEEK(call_date), call_hour;
```

This query is gold for optimizing agent schedules. If your data shows Monday 5-6 PM has a 12% contact rate and 3.5% conversion rate while Tuesday 10-11 AM has 5% contact rate and 1.2% conversion rate, you know exactly where to concentrate your staffing.

### DID Performance (Caller ID Health)

Track which outbound DIDs are performing well and which are getting flagged:

```sql
SELECT
    phone_number as outbound_did,
    COUNT(*) as total_dials,
    COUNT(CASE WHEN status NOT IN ('NA','B','DC','N','AFTHRS') THEN 1 END) as answered,
    ROUND(COUNT(CASE WHEN status NOT IN ('NA','B','DC','N','AFTHRS') THEN 1 END) /
          NULLIF(COUNT(*), 0) * 100, 2) as answer_rate,
    COUNT(CASE WHEN status = 'NA' THEN 1 END) as no_answer,
    ROUND(COUNT(CASE WHEN status = 'NA' THEN 1 END) /
          NULLIF(COUNT(*), 0) * 100, 2) as no_answer_rate
FROM vicidial_log
WHERE call_date >= DATE_SUB(NOW(), INTERVAL 7 DAY)
    AND campaign_id = 'YOUR_CAMPAIGN'
GROUP BY phone_number
HAVING total_dials > 50
ORDER BY answer_rate ASC;
```

A DID with a significantly lower answer rate than your campaign average is likely flagged as spam. See our [DID management guide](/blog/vicidial-did-management/) for reputation monitoring and rotation strategies.

### Abandon Rate Monitoring (TCPA Compliance)

Already covered in our [TCPA compliance guide](/blog/vicidial-tcpa-compliance/), but here's the critical 30-day rolling query:

```sql
SELECT
    campaign_id,
    COUNT(CASE WHEN status = 'DROP' THEN 1 END) as drops,
    COUNT(CASE WHEN status NOT IN ('NA','B','DC','N','AFTHRS','XDROP') THEN 1 END) as answered,
    ROUND(COUNT(CASE WHEN status = 'DROP' THEN 1 END) /
          NULLIF(COUNT(CASE WHEN status NOT IN ('NA','B','DC','N','AFTHRS','XDROP') THEN 1 END), 0) * 100, 2)
          as drop_rate_30d
FROM vicidial_log
WHERE call_date >= DATE_SUB(NOW(), INTERVAL 30 DAY)
GROUP BY campaign_id
HAVING drop_rate_30d > 2.0
ORDER BY drop_rate_30d DESC;
```

The `HAVING drop_rate_30d > 2.0` clause filters to only show campaigns approaching the danger zone. Set up a cron job to run this query daily and alert if any campaign exceeds 2.5%.

---

## Server Health and Infrastructure Monitoring

Reporting isn't just about calls and agents. VICIdial's performance depends on infrastructure health, and the database gives you visibility into that too.

### Trunk Utilization

Monitor how many simultaneous calls your SIP trunks are handling:

```sql
SELECT
    server_ip,
    campaign_id,
    COUNT(*) as active_channels,
    MAX(TIMESTAMPDIFF(SECOND, call_time, NOW())) as oldest_call_sec
FROM vicidial_auto_calls
GROUP BY server_ip, campaign_id;
```

If active channels are approaching your trunk limit (typically defined in your SIP provider's configuration), calls will start getting SIP 503 errors. Monitor this during peak hours and scale trunk capacity before you hit limits.

### Hopper Status

The hopper is VICIdial's pre-loaded dialing queue. If it runs dry, the dialer stops:

```sql
SELECT
    campaign_id,
    COUNT(*) as leads_in_hopper,
    MIN(gmt_offset_now) as earliest_tz,
    MAX(gmt_offset_now) as latest_tz,
    COUNT(DISTINCT list_id) as lists_represented
FROM vicidial_hopper
GROUP BY campaign_id;
```

**Hopper best practices:**
- Maintain at least 2x your agent count in the hopper at all times
- For a 50-agent campaign, target 200-500 leads in the hopper
- If hopper count is low, check: list availability, call time restrictions, timezone filtering, lead filter criteria, and whether the list is exhausted

### VICIdial Process Health

VICIdial runs several background processes that must be running for the system to function. Check these regularly:

```bash
# Key processes to monitor on each VICIdial server
ps aux | grep -E "AST_manager|VD_auto_dialer|AST_send|VD_call_log|ADMIN_keepalive"

# Expected processes on a dialer server:
# AST_manager_send.pl — sends commands to Asterisk
# AST_manager_listen.pl — listens for events from Asterisk
# VD_auto_dialer.pl — the actual dialing engine
# AST_VDauto_dial.pl — auto-dial process
# ADMIN_keepalive_ALL.pl — keeps all processes running
```

If any of these processes are missing, VICIdial's `ADMIN_keepalive_ALL.pl` cron job should restart them automatically (runs every minute via cron). If they're consistently dying, check your system logs for OOM killer events, MySQL connection limits, or Asterisk crashes.

### MySQL Performance for Reporting

VICIdial's reporting queries can be heavy on large databases. Monitor MySQL health:

```sql
-- Check table sizes
SELECT
    table_name,
    ROUND(data_length / 1024 / 1024, 2) as data_mb,
    ROUND(index_length / 1024 / 1024, 2) as index_mb,
    table_rows
FROM information_schema.tables
WHERE table_schema = 'asterisk'
    AND table_name IN ('vicidial_log', 'vicidial_closer_log', 'vicidial_agent_log',
                       'vicidial_list', 'recording_log')
ORDER BY data_length DESC;

-- Check for slow queries
SHOW FULL PROCESSLIST;
```

For operations handling more than 50,000 calls per day, consider:
- **Archiving old data:** Move vicidial_log entries older than 90 days to an archive table. VICIdial's built-in archiving scripts handle this.
- **Read replicas:** Run reporting queries against a MySQL read replica instead of the production master. This prevents heavy report queries from impacting dialing performance.
- **Index optimization:** Ensure indexes exist on `call_date`, `campaign_id`, `user`, and `status` columns in the log tables. VICIdial's default schema includes these, but verify after any upgrades.

For [cluster deployments](/blog/vicidial-cluster-guide/), all reporting data lives in the centralized MySQL database on the master server. Dialer servers push data to the master in near-real-time.

---

## Building Custom Dashboards: Grafana Integration

VICIdial's built-in reports are functional but static. For real-time, visual dashboards that auto-refresh and look good on wallboards, Grafana is the best option for VICIdial environments.

### Setting Up Grafana with VICIdial's MySQL Database

**1. Install Grafana** on a separate server or on your VICIdial web server:

```bash
# On AlmaLinux/Rocky Linux 9 (ViciBox base)
sudo dnf install -y grafana
sudo systemctl enable --now grafana-server
```

Access Grafana at `http://your-server:3000` (default credentials: admin/admin).

**2. Add MySQL data source:**

In Grafana → Configuration → Data Sources → Add data source → MySQL:

```
Host: your-vicidial-db-server:3306
Database: asterisk
User: grafana_readonly (create a read-only MySQL user)
Password: [strong password]
```

**Create a read-only MySQL user for Grafana** (never use the VICIdial admin credentials):

```sql
CREATE USER 'grafana_readonly'@'%' IDENTIFIED BY 'your_strong_password';
GRANT SELECT ON asterisk.* TO 'grafana_readonly'@'%';
FLUSH PRIVILEGES;
```

**3. Build dashboard panels.** Here are Grafana panel configurations for key VICIdial metrics:

#### Panel: Real-Time Agent Status

```sql
-- Grafana: Stat panel or Table panel
-- Refresh: 5s
SELECT
    status,
    COUNT(*) as count
FROM vicidial_live_agents
GROUP BY status;
```

#### Panel: Calls Per Hour (Time Series)

```sql
-- Grafana: Time series panel
-- Variable: $campaign_id
SELECT
    DATE_FORMAT(call_date, '%Y-%m-%d %H:00:00') as time,
    COUNT(*) as dials,
    COUNT(CASE WHEN status NOT IN ('NA','B','DC','N','AFTHRS') THEN 1 END) as answered
FROM vicidial_log
WHERE call_date >= DATE_SUB(NOW(), INTERVAL 24 HOUR)
    AND campaign_id = '$campaign_id'
GROUP BY DATE_FORMAT(call_date, '%Y-%m-%d %H:00:00')
ORDER BY time;
```

#### Panel: Drop Rate Gauge

```sql
-- Grafana: Gauge panel
-- Set thresholds: Green < 2%, Yellow 2-2.5%, Red > 2.5%
SELECT
    ROUND(COUNT(CASE WHEN status = 'DROP' THEN 1 END) /
          NULLIF(COUNT(CASE WHEN status NOT IN ('NA','B','DC','N','AFTHRS','XDROP') THEN 1 END), 0) * 100, 2)
          as drop_rate
FROM vicidial_log
WHERE call_date >= DATE_SUB(NOW(), INTERVAL 30 DAY)
    AND campaign_id = '$campaign_id';
```

#### Panel: Agent Leaderboard (Table)

```sql
-- Grafana: Table panel
-- Refresh: 30s
SELECT
    val.user as Agent,
    COUNT(*) as Calls,
    COUNT(CASE WHEN val.status IN ('SALE','SET','XFER') THEN 1 END) as Sales,
    ROUND(SUM(val.talk_sec) / 60, 0) as Talk_Min,
    ROUND(SUM(val.talk_sec) /
          NULLIF(SUM(val.talk_sec + val.wait_sec + val.pause_sec + val.dispo_sec), 0) * 100, 0)
          as Utilization
FROM vicidial_agent_log val
WHERE val.event_time >= CURDATE()
GROUP BY val.user
ORDER BY Sales DESC;
```

### Metabase as an Alternative

If Grafana feels too complex, **Metabase** is an excellent alternative with a simpler UI and "ask a question" natural language query interface. Installation is similar:

```bash
# Docker-based Metabase installation
docker run -d -p 3000:3000 \
  -e "MB_DB_TYPE=mysql" \
  -e "MB_DB_DBNAME=metabase" \
  -e "MB_DB_PORT=3306" \
  -e "MB_DB_USER=metabase" \
  -e "MB_DB_PASS=your_password" \
  -e "MB_DB_HOST=localhost" \
  --name metabase metabase/metabase
```

Connect Metabase to VICIdial's MySQL database the same way as Grafana. Metabase excels at ad-hoc questions — managers can build their own reports without writing SQL, using the visual query builder.

---

## Automated Report Delivery

For operations that need daily or weekly reports emailed to stakeholders, set up automated report generation:

### Cron-Based Daily Report

Create a shell script that runs key queries and emails the results:

```bash
#!/bin/bash
# /usr/local/bin/vicidial_daily_report.sh
# Run via cron: 0 7 * * * /usr/local/bin/vicidial_daily_report.sh

YESTERDAY=$(date -d "yesterday" +%Y-%m-%d)
DB_USER="report_user"
DB_PASS="your_password"
DB_NAME="asterisk"
EMAIL="manager@yourcompany.com"
REPORT_FILE="/tmp/vicidial_daily_${YESTERDAY}.txt"

echo "VICIdial Daily Report - ${YESTERDAY}" > $REPORT_FILE
echo "======================================" >> $REPORT_FILE
echo "" >> $REPORT_FILE

echo "CAMPAIGN SUMMARY" >> $REPORT_FILE
echo "----------------" >> $REPORT_FILE
mysql -u $DB_USER -p$DB_PASS $DB_NAME -e "
SELECT
    campaign_id as Campaign,
    COUNT(*) as Dials,
    COUNT(CASE WHEN status NOT IN ('NA','B','DC','N','AFTHRS') THEN 1 END) as Answered,
    COUNT(CASE WHEN status = 'DROP' THEN 1 END) as Drops,
    COUNT(CASE WHEN status IN ('SALE','SET','XFER') THEN 1 END) as Conversions,
    ROUND(COUNT(CASE WHEN status NOT IN ('NA','B','DC','N','AFTHRS') THEN 1 END) /
          NULLIF(COUNT(*), 0) * 100, 1) as 'Answer%',
    ROUND(COUNT(CASE WHEN status = 'DROP' THEN 1 END) /
          NULLIF(COUNT(CASE WHEN status NOT IN ('NA','B','DC','N','AFTHRS','XDROP') THEN 1 END), 0) * 100, 1)
          as 'Drop%'
FROM vicidial_log
WHERE DATE(call_date) = '${YESTERDAY}'
GROUP BY campaign_id;
" >> $REPORT_FILE

echo "" >> $REPORT_FILE
echo "TOP AGENTS" >> $REPORT_FILE
echo "----------" >> $REPORT_FILE
mysql -u $DB_USER -p$DB_PASS $DB_NAME -e "
SELECT
    user as Agent,
    COUNT(*) as Calls,
    COUNT(CASE WHEN status IN ('SALE','SET','XFER') THEN 1 END) as Sales,
    ROUND(AVG(CASE WHEN length_in_sec > 0 THEN length_in_sec END), 0) as 'Avg_Sec'
FROM vicidial_log
WHERE DATE(call_date) = '${YESTERDAY}'
    AND user != ''
GROUP BY user
ORDER BY Sales DESC
LIMIT 15;
" >> $REPORT_FILE

mail -s "VICIdial Daily Report - ${YESTERDAY}" $EMAIL < $REPORT_FILE
rm $REPORT_FILE
```

### Alerting on Threshold Breaches

Set up monitoring alerts for critical metrics:

```bash
#!/bin/bash
# /usr/local/bin/vicidial_compliance_check.sh
# Run via cron: */15 * * * * /usr/local/bin/vicidial_compliance_check.sh

DB_USER="report_user"
DB_PASS="your_password"
DB_NAME="asterisk"
ALERT_EMAIL="compliance@yourcompany.com"

# Check 30-day rolling drop rate for each campaign
DROP_ALERT=$(mysql -u $DB_USER -p$DB_PASS $DB_NAME -N -e "
SELECT campaign_id, ROUND(COUNT(CASE WHEN status = 'DROP' THEN 1 END) /
    NULLIF(COUNT(CASE WHEN status NOT IN ('NA','B','DC','N','AFTHRS','XDROP') THEN 1 END), 0) * 100, 2)
    as drop_rate
FROM vicidial_log
WHERE call_date >= DATE_SUB(NOW(), INTERVAL 30 DAY)
GROUP BY campaign_id
HAVING drop_rate > 2.5;
")

if [ -n "$DROP_ALERT" ]; then
    echo "WARNING: Campaigns approaching 3% drop rate limit:" | \
    mail -s "URGENT: VICIdial Drop Rate Alert" $ALERT_EMAIL <<< "$DROP_ALERT"
fi
```

---

## Performance Monitoring Best Practices

### What to Monitor Continuously (Wallboard)

Put these on screens visible to supervisors and managers:

1. **Agent status distribution** — how many agents are in READY, INCALL, PAUSED at any moment
2. **Calls waiting in queue** — should be zero or near-zero for outbound; minimal for inbound
3. **Current drop rate** — the big number that keeps you out of legal trouble
4. **Hopper count per campaign** — early warning for list exhaustion
5. **Active trunk count** — are you approaching capacity?

### What to Review Daily

1. **Campaign contact rates** — are they trending up or down? Declining rates suggest DID reputation issues or list quality degradation
2. **Agent utilization percentages** — are agents spending enough time on calls vs. waiting or pausing?
3. **Disposition distribution** — unexpected shifts (sudden increase in DNC dispositions, for example) need investigation
4. **Top and bottom agent performance** — coaching opportunities and recognition
5. **Cost per dial** — track your [SIP costs](/blog/vicidial-cost-2026/) against dial volume

### What to Review Weekly

1. **30-day rolling abandon rate** — the TCPA compliance metric
2. **DID performance comparison** — identify flagged numbers for rotation
3. **List penetration** — how much of each list has been worked and what's remaining
4. **Hourly performance heatmap** — optimize staffing schedules
5. **AMD accuracy** — check the ratio of AMD HUMAN calls that agents immediately disconnect (false positives)

### What to Review Monthly

1. **Cost per lead/appointment trends** — see our [CPL benchmarks](/blog/call-center-cost-per-lead-benchmarks/) for comparison
2. **Agent turnover impact** — how are new agents performing vs. tenured agents?
3. **Infrastructure capacity** — do you need more trunks, more server capacity, or more [cluster nodes](/blog/vicidial-cluster-guide/)?
4. **Compliance audit** — DNC scrub dates, consent records, state registration status

---

## Frequently Asked Questions

### Where are VICIdial's reports in the admin interface?

VICIdial's built-in reports are accessible from the admin interface at Admin → Reports tab, or directly via URLs like `/vicidial/realtime_report.php` (real-time), `/vicidial/AST_VDADstats.php` (outbound stats), and `/vicidial/AST_agent_performance_detail.php` (agent performance). The exact URLs depend on your installation path, but these are the standard locations on default ViciBox installations.

### Can I connect Grafana directly to VICIdial's database?

Yes, and it's the recommended approach for building custom dashboards. Create a read-only MySQL user, point Grafana at VICIdial's `asterisk` database, and build panels using the SQL queries provided in this guide. For high-volume operations (50,000+ calls/day), run Grafana against a MySQL read replica to avoid impacting production performance.

### What's the most important VICIdial report for TCPA compliance?

The 30-day rolling abandon rate by campaign. The TCPA mandates no more than 3% of answered calls be abandoned, measured over a 30-day campaign period. Run the abandon rate query in this guide daily (or set up automated alerting). If any campaign exceeds 2.5%, take immediate corrective action. See our [TCPA compliance checklist](/blog/vicidial-tcpa-compliance/) for the full configuration guide.

### How do I export VICIdial data to Excel or CSV?

Three options: (1) Use VICIdial's built-in Call Export report at `/vicidial/call_report_export.php` which generates CSV files; (2) Run MySQL queries with the `INTO OUTFILE` clause to write results to CSV; (3) Use a tool like Metabase or Grafana that allows CSV/Excel export of query results. For automated daily exports, set up a cron job that runs your queries and emails the results as CSV attachments.

### What VICIdial database tables should I back up for reporting?

At minimum, back up `vicidial_log`, `vicidial_closer_log`, `vicidial_agent_log`, `vicidial_list`, `recording_log`, and any custom field tables (`custom_[list_id]`). These contain your complete operational history. VICIdial's default MySQL backup script (`ADMIN_backup.pl`) handles this, but verify it's running and that backups are stored off-server. For compliance, retain at least 5 years of call log data.

### How can I track cost per lead in VICIdial?

VICIdial doesn't natively track cost per lead because it doesn't know your telco or infrastructure costs. Calculate CPL externally: (total monthly spend on infrastructure + VoIP + DIDs + agent labor) / total conversions from VICIdial. Use the SQL queries in this guide to pull conversion counts and dial volumes, then combine with your cost data in a spreadsheet or BI tool. Our [cost per lead benchmarks](/blog/call-center-cost-per-lead-benchmarks/) article provides industry-specific targets to measure against.

### Why is my VICIdial real-time report showing stale data?

The real-time report relies on the AST_manager processes running on each dialer server. Check: (1) Are `AST_manager_send.pl` and `AST_manager_listen.pl` running? (2) Is the Asterisk Manager Interface (AMI) accessible on port 5038? (3) Is the web server's connection to the database working? (4) Check the `vicidial_live_agents` table directly with a SQL query — if the data there is current but the web report is stale, the issue is in the web server or browser. If the table data is stale, the issue is in the AST_manager processes.

### Can I build real-time alerts for specific events in VICIdial?

Yes, using a combination of database monitoring and scripting. Write a cron script (running every 1-5 minutes) that queries `vicidial_live_agents`, `vicidial_auto_calls`, or `vicidial_log` for conditions you want to alert on — such as drop rate spikes, hopper exhaustion, agents paused too long, or trunk utilization approaching capacity. Send alerts via email, SMS (using an API), or Slack webhook. The automated alerting scripts in this guide provide starting templates.

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-reporting-monitoring).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
