# VICIdial Custom Report Building with MySQL Views and Queries

VICIdial's built-in reporting covers the basics, but if you're running a serious call center with 25+ agents, you've almost certainly hit the wall. The admin reports are slow, the export options are limited, and when your client asks for a custom metric that doesn't exist in the GUI, you're stuck building it yourself.

The good news: VICIdial stores everything in MySQL. Every call, every disposition, every agent state change, every pause code -- it's all there in well-structured tables. Once you understand the schema, you can build reports that are faster, more flexible, and far more powerful than anything the web interface offers.

This guide walks through the VICIdial database schema, the key tables you need to know, and practical MySQL queries and views that will transform your reporting capabilities.

## VICIdial Database Schema Overview

VICIdial uses a single MySQL database called `asterisk` (the default name, though some installations rename it). The schema contains over 200 tables, but for reporting purposes, you only need to understand about 15 of them.

Connect to the database:

```bash
mysql -u cron -p asterisk
```

The default cron user typically has read access to all tables. If you're building a dedicated reporting user (recommended), grant SELECT-only permissions:

```sql
CREATE USER 'reports'@'localhost' IDENTIFIED BY 'your_secure_password';
GRANT SELECT ON asterisk.* TO 'reports'@'localhost';
FLUSH PRIVILEGES;
```

### The Core Reporting Tables

Here's a map of the tables that matter most for custom reports:

| Table | What It Stores | Key Use |
|-------|---------------|---------|
| `vicidial_log` | Outbound call records | Call outcomes, talk time, wait time |
| `vicidial_closer_log` | Inbound/transfer call records | Queue times, handle times |
| `vicidial_agent_log` | Agent state changes | Login time, pause time, productivity |
| `vicidial_list` | Lead/contact records | Lead data, status, source |
| `vicidial_users` | Agent profiles | Names, groups, permissions |
| `vicidial_campaigns` | Campaign configuration | Dial method, settings |
| `vicidial_inbound_groups` | Inbound group config | Queue settings, routing |
| `vicidial_carrier_log` | SIP trunk records | Carrier-level call data |
| `vicidial_did_log` | DID routing log | Inbound call routing |
| `vicidial_lead_recycle` | Recycled leads | Retry scheduling |
| `vicidial_campaign_stats` | Aggregated campaign stats | Pre-calculated metrics |
| `recording_log` | Call recordings | Recording file paths, durations |
| `vicidial_user_log` | User login/logout events | Session tracking |
| `vicidial_pause_codes` | Pause reason definitions | Break types |

## Deep Dive: vicidial_log

The `vicidial_log` table is the backbone of outbound reporting. Every outbound call attempt gets a row here.

```sql
DESCRIBE vicidial_log;
```

Key columns:

| Column | Type | Description |
|--------|------|-------------|
| `uniqueid` | VARCHAR(20) | Asterisk unique call ID |
| `lead_id` | INT(9) | Links to vicidial_list |
| `list_id` | BIGINT(14) | List the lead belongs to |
| `campaign_id` | VARCHAR(8) | Campaign that placed the call |
| `call_date` | DATETIME | When the call was placed |
| `start_epoch` | INT(10) | Unix timestamp of call start |
| `end_epoch` | INT(10) | Unix timestamp of call end |
| `length_in_sec` | INT(10) | Total call duration |
| `status` | VARCHAR(6) | Disposition code |
| `phone_number` | VARCHAR(18) | Number dialed |
| `user` | VARCHAR(20) | Agent who handled the call |
| `term_reason` | VARCHAR(6) | How the call ended |
| `alt_dial` | VARCHAR(6) | Which phone number was dialed (MAIN, ALT, ADDR3) |
| `called_count` | SMALLINT(5) | How many times this lead has been called |

A basic query to see today's call volume by campaign:

```sql
SELECT
    campaign_id,
    COUNT(*) AS total_calls,
    SUM(CASE WHEN status IN ('SALE','XFER','CALLBK') THEN 1 ELSE 0 END) AS connects,
    ROUND(SUM(CASE WHEN status IN ('SALE','XFER','CALLBK') THEN 1 ELSE 0 END) / COUNT(*) * 100, 2) AS connect_pct,
    ROUND(AVG(length_in_sec), 1) AS avg_duration
FROM vicidial_log
WHERE call_date >= CURDATE()
GROUP BY campaign_id
ORDER BY total_calls DESC;
```

## Deep Dive: vicidial_closer_log

The `vicidial_closer_log` mirrors `vicidial_log` but for inbound and transferred calls. If you're running blended campaigns or have inbound queues, this table is essential.

Key additional columns beyond what `vicidial_log` has:

| Column | Type | Description |
|--------|------|-------------|
| `campaign_id` | VARCHAR(20) | The inbound group (not the outbound campaign) |
| `queue_seconds` | INT(10) | Time the caller waited in queue |
| `xfercallid` | INT(10) | Links to the original outbound call |
| `queue_position` | INT(4) | Caller's position in queue when answered |

Query to check queue performance:

```sql
SELECT
    campaign_id AS inbound_group,
    COUNT(*) AS total_calls,
    ROUND(AVG(queue_seconds), 1) AS avg_queue_sec,
    MAX(queue_seconds) AS max_queue_sec,
    SUM(CASE WHEN queue_seconds <= 30 THEN 1 ELSE 0 END) AS answered_within_30s,
    ROUND(SUM(CASE WHEN queue_seconds <= 30 THEN 1 ELSE 0 END) / COUNT(*) * 100, 1) AS service_level_pct
FROM vicidial_closer_log
WHERE call_date >= CURDATE()
  AND status NOT IN ('DROP','TIMEOT','AFTHRS')
GROUP BY campaign_id;
```

## Deep Dive: vicidial_agent_log

The `vicidial_agent_log` is where agent productivity data lives. Every time an agent changes state (available, in-call, paused, wrapping up), a row is written.

Key columns:

| Column | Type | Description |
|--------|------|-------------|
| `agent_log_id` | INT(9) | Primary key |
| `user` | VARCHAR(20) | Agent login ID |
| `event_time` | DATETIME | When the state change occurred |
| `campaign_id` | VARCHAR(8) | Campaign the agent was logged into |
| `pause_epoch` | INT(10) | When the agent entered pause |
| `pause_sec` | SMALLINT(5) | Seconds spent paused |
| `wait_epoch` | INT(10) | When the agent became available |
| `wait_sec` | SMALLINT(5) | Seconds waiting for a call |
| `talk_epoch` | INT(10) | When the agent started talking |
| `talk_sec` | SMALLINT(5) | Seconds in conversation |
| `dispo_epoch` | INT(10) | When the agent entered disposition |
| `dispo_sec` | SMALLINT(5) | Seconds in wrap-up |
| `status` | VARCHAR(6) | Disposition assigned |
| `sub_status` | VARCHAR(6) | Pause code (if paused) |

## Creating MySQL Views for Common Reports

MySQL views let you save complex queries as virtual tables. They're perfect for VICIdial reporting because they give you a clean interface over the messy underlying data, and you can query them from any tool that connects to MySQL.

### View 1: Agent Daily Summary

This view aggregates agent performance by day:

```sql
CREATE OR REPLACE VIEW v_agent_daily_summary AS
SELECT
    DATE(event_time) AS report_date,
    a.user AS agent_id,
    u.full_name AS agent_name,
    a.campaign_id,
    COUNT(*) AS total_calls,
    SUM(a.talk_sec) AS total_talk_sec,
    SUM(a.pause_sec) AS total_pause_sec,
    SUM(a.wait_sec) AS total_wait_sec,
    SUM(a.dispo_sec) AS total_dispo_sec,
    ROUND(SUM(a.talk_sec) / NULLIF(SUM(a.talk_sec + a.pause_sec + a.wait_sec + a.dispo_sec), 0) * 100, 1) AS talk_pct,
    SUM(CASE WHEN a.status IN ('SALE','XFER') THEN 1 ELSE 0 END) AS conversions,
    ROUND(SUM(CASE WHEN a.status IN ('SALE','XFER') THEN 1 ELSE 0 END) / NULLIF(COUNT(*), 0) * 100, 2) AS conversion_rate
FROM vicidial_agent_log a
LEFT JOIN vicidial_users u ON a.user = u.user
WHERE a.talk_sec > 0
GROUP BY DATE(event_time), a.user, u.full_name, a.campaign_id;
```

Query it simply:

```sql
SELECT * FROM v_agent_daily_summary
WHERE report_date = CURDATE()
ORDER BY conversion_rate DESC;
```

### View 2: Campaign Hourly Metrics

Track campaign performance hour by hour to identify peak times:

```sql
CREATE OR REPLACE VIEW v_campaign_hourly AS
SELECT
    DATE(call_date) AS report_date,
    HOUR(call_date) AS hour_of_day,
    campaign_id,
    COUNT(*) AS attempts,
    SUM(CASE WHEN user != 'VDAD' THEN 1 ELSE 0 END) AS agent_handled,
    SUM(CASE WHEN status = 'DROP' THEN 1 ELSE 0 END) AS drops,
    SUM(CASE WHEN status IN ('A','AA','AM','AL','N','NI','NP','DC','DNC','DNCL','DNCC') THEN 1 ELSE 0 END) AS non_contacts,
    SUM(CASE WHEN status NOT IN ('A','AA','AM','AL','N','NI','NP','DC','DNC','DNCL','DNCC','DROP','B','NA','ADC') THEN 1 ELSE 0 END) AS contacts,
    ROUND(AVG(CASE WHEN length_in_sec > 0 THEN length_in_sec ELSE NULL END), 1) AS avg_talk_sec,
    ROUND(SUM(CASE WHEN status = 'DROP' THEN 1 ELSE 0 END) / NULLIF(COUNT(*), 0) * 100, 2) AS drop_rate
FROM vicidial_log
GROUP BY DATE(call_date), HOUR(call_date), campaign_id;
```

### View 3: List Penetration Report

Know exactly how deep into each list you've dialed:

```sql
CREATE OR REPLACE VIEW v_list_penetration AS
SELECT
    l.list_id,
    vl.list_name,
    vl.campaign_id,
    COUNT(*) AS total_leads,
    SUM(CASE WHEN l.called_count > 0 THEN 1 ELSE 0 END) AS dialed_leads,
    ROUND(SUM(CASE WHEN l.called_count > 0 THEN 1 ELSE 0 END) / NULLIF(COUNT(*), 0) * 100, 1) AS penetration_pct,
    SUM(CASE WHEN l.status IN ('NEW','QUEUE') THEN 1 ELSE 0 END) AS dialable_remaining,
    AVG(l.called_count) AS avg_call_attempts,
    SUM(CASE WHEN l.status IN ('SALE','XFER') THEN 1 ELSE 0 END) AS conversions
FROM vicidial_list l
JOIN vicidial_lists vl ON l.list_id = vl.list_id
GROUP BY l.list_id, vl.list_name, vl.campaign_id;
```

### View 4: AMD Performance Tracker

For operations relying on Answering Machine Detection, this view tracks AMD accuracy:

```sql
CREATE OR REPLACE VIEW v_amd_performance AS
SELECT
    DATE(call_date) AS report_date,
    campaign_id,
    COUNT(*) AS total_calls,
    SUM(CASE WHEN status = 'AA' THEN 1 ELSE 0 END) AS amd_detected,
    SUM(CASE WHEN status = 'A' THEN 1 ELSE 0 END) AS amd_human_auto,
    SUM(CASE WHEN status = 'AL' THEN 1 ELSE 0 END) AS amd_left_message,
    SUM(CASE WHEN status = 'AM' THEN 1 ELSE 0 END) AS amd_machine,
    ROUND(SUM(CASE WHEN status IN ('AA','AM','AL') THEN 1 ELSE 0 END) / NULLIF(COUNT(*), 0) * 100, 1) AS amd_rate,
    SUM(CASE WHEN length_in_sec BETWEEN 1 AND 3 AND status NOT IN ('AA','AM','AL','B','NA','DC') THEN 1 ELSE 0 END) AS possible_false_positives
FROM vicidial_log
GROUP BY DATE(call_date), campaign_id;
```

The `possible_false_positives` column catches calls that were very short (1-3 seconds) but weren't dispositioned as machines -- a signal that AMD may be dropping live humans.

## Agent Performance Report Queries

### Detailed Agent Scorecard

This query builds a comprehensive agent scorecard suitable for team leads:

```sql
SELECT
    u.full_name AS agent,
    a.user AS agent_id,
    SEC_TO_TIME(SUM(a.talk_sec)) AS total_talk_time,
    SEC_TO_TIME(SUM(a.pause_sec)) AS total_pause_time,
    SEC_TO_TIME(SUM(a.wait_sec)) AS total_wait_time,
    SEC_TO_TIME(SUM(a.dispo_sec)) AS total_dispo_time,
    SEC_TO_TIME(SUM(a.talk_sec + a.pause_sec + a.wait_sec + a.dispo_sec)) AS total_logged_time,
    COUNT(CASE WHEN a.talk_sec > 0 THEN 1 END) AS calls_handled,
    ROUND(AVG(CASE WHEN a.talk_sec > 0 THEN a.talk_sec END), 0) AS avg_handle_sec,
    SUM(CASE WHEN a.status IN ('SALE','XFER') THEN 1 ELSE 0 END) AS conversions,
    ROUND(
        SUM(a.talk_sec) / NULLIF(SUM(a.talk_sec + a.pause_sec + a.wait_sec + a.dispo_sec), 0) * 100,
    1) AS utilization_pct,
    -- Calls per hour calculation
    ROUND(
        COUNT(CASE WHEN a.talk_sec > 0 THEN 1 END) /
        NULLIF(SUM(a.talk_sec + a.pause_sec + a.wait_sec + a.dispo_sec) / 3600, 0),
    1) AS calls_per_hour
FROM vicidial_agent_log a
JOIN vicidial_users u ON a.user = u.user
WHERE DATE(a.event_time) = CURDATE()
GROUP BY u.full_name, a.user
ORDER BY conversions DESC, utilization_pct DESC;
```

### Pause Code Breakdown

Identify which pause codes agents are using and how much time is going to each:

```sql
SELECT
    a.user AS agent_id,
    u.full_name AS agent,
    a.sub_status AS pause_code,
    COALESCE(p.pause_code_name, 'UNKNOWN') AS pause_name,
    COUNT(*) AS pause_count,
    SEC_TO_TIME(SUM(a.pause_sec)) AS total_pause_time,
    ROUND(AVG(a.pause_sec), 0) AS avg_pause_sec
FROM vicidial_agent_log a
JOIN vicidial_users u ON a.user = u.user
LEFT JOIN vicidial_pause_codes p
    ON a.sub_status = p.pause_code
    AND a.campaign_id = p.campaign_id
WHERE DATE(a.event_time) = CURDATE()
  AND a.pause_sec > 0
GROUP BY a.user, u.full_name, a.sub_status, p.pause_code_name
ORDER BY SUM(a.pause_sec) DESC;
```

### Agent Login/Logout Timeline

Track when agents log in and out to verify schedule adherence:

```sql
SELECT
    user,
    event,
    event_date,
    event_epoch,
    campaign_id
FROM vicidial_user_log
WHERE event IN ('LOGIN','LOGOUT')
  AND DATE(event_date) = CURDATE()
ORDER BY user, event_epoch;
```

## Campaign Performance Dashboards

### Real-Time Campaign Dashboard Query

This query gives you a snapshot of where each campaign stands right now:

```sql
SELECT
    vl.campaign_id,
    -- Today's metrics from vicidial_log
    COUNT(*) AS calls_today,
    SUM(CASE WHEN vl.status NOT IN ('A','AA','AM','AL','N','NI','NP','DC','DNC','B','NA','ADC','DROP','DNCL','DNCC') THEN 1 ELSE 0 END) AS contacts_today,
    ROUND(
        SUM(CASE WHEN vl.status NOT IN ('A','AA','AM','AL','N','NI','NP','DC','DNC','B','NA','ADC','DROP','DNCL','DNCC') THEN 1 ELSE 0 END)
        / NULLIF(COUNT(*), 0) * 100,
    2) AS contact_rate,
    SUM(CASE WHEN vl.status = 'DROP' THEN 1 ELSE 0 END) AS drops,
    ROUND(SUM(CASE WHEN vl.status = 'DROP' THEN 1 ELSE 0 END) / NULLIF(COUNT(*), 0) * 100, 2) AS drop_rate,
    SUM(CASE WHEN vl.status IN ('SALE','XFER') THEN 1 ELSE 0 END) AS sales,
    ROUND(AVG(vl.length_in_sec), 0) AS avg_call_sec,
    SEC_TO_TIME(SUM(CASE WHEN vl.length_in_sec > 30 THEN vl.length_in_sec ELSE 0 END)) AS total_talk_time
FROM vicidial_log vl
WHERE vl.call_date >= CURDATE()
GROUP BY vl.campaign_id
ORDER BY calls_today DESC;
```

### Disposition Breakdown by Campaign

See exactly how calls are being dispositioned:

```sql
SELECT
    campaign_id,
    status,
    sd.status_name,
    COUNT(*) AS call_count,
    ROUND(COUNT(*) / SUM(COUNT(*)) OVER (PARTITION BY campaign_id) * 100, 2) AS pct_of_campaign
FROM vicidial_log vl
LEFT JOIN vicidial_statuses sd ON vl.status = sd.status
WHERE call_date >= CURDATE()
GROUP BY campaign_id, status, sd.status_name
ORDER BY campaign_id, call_count DESC;
```

### Carrier Performance Report

Track which SIP carriers are performing best:

```sql
SELECT
    server_ip,
    dialstatus,
    COUNT(*) AS calls,
    ROUND(AVG(answered_time), 1) AS avg_answer_sec,
    ROUND(COUNT(*) / SUM(COUNT(*)) OVER () * 100, 2) AS pct_of_total
FROM vicidial_carrier_log
WHERE call_date >= CURDATE()
GROUP BY server_ip, dialstatus
ORDER BY server_ip, calls DESC;
```

## Scheduled Report Generation with Cron

Manual queries are useful for ad-hoc analysis, but production reporting should be automated. Here's how to set up scheduled report generation.

### Step 1: Create a Report Script

Save this as `/opt/vicidial-reports/daily_agent_report.sh`:

```bash
#!/bin/bash
# Daily Agent Performance Report
# Generates CSV and emails to management

REPORT_DATE=$(date -d "yesterday" +%Y-%m-%d)
REPORT_DIR="/opt/vicidial-reports/output"
REPORT_FILE="${REPORT_DIR}/agent_report_${REPORT_DATE}.csv"

mkdir -p "$REPORT_DIR"

mysql -u reports -p'your_password' -h localhost asterisk \
  --batch --raw -e "
SELECT
    u.full_name AS 'Agent Name',
    a.user AS 'Agent ID',
    a.campaign_id AS 'Campaign',
    COUNT(CASE WHEN a.talk_sec > 0 THEN 1 END) AS 'Calls Handled',
    SEC_TO_TIME(SUM(a.talk_sec)) AS 'Talk Time',
    SEC_TO_TIME(SUM(a.pause_sec)) AS 'Pause Time',
    ROUND(SUM(a.talk_sec) / NULLIF(SUM(a.talk_sec + a.pause_sec + a.wait_sec + a.dispo_sec), 0) * 100, 1) AS 'Utilization %',
    SUM(CASE WHEN a.status IN ('SALE','XFER') THEN 1 ELSE 0 END) AS 'Conversions',
    ROUND(SUM(CASE WHEN a.status IN ('SALE','XFER') THEN 1 ELSE 0 END) / NULLIF(COUNT(CASE WHEN a.talk_sec > 0 THEN 1 END), 0) * 100, 2) AS 'Conv Rate %'
FROM vicidial_agent_log a
JOIN vicidial_users u ON a.user = u.user
WHERE DATE(a.event_time) = '${REPORT_DATE}'
GROUP BY u.full_name, a.user, a.campaign_id
ORDER BY SUM(CASE WHEN a.status IN ('SALE','XFER') THEN 1 ELSE 0 END) DESC
" > "$REPORT_FILE"

# Email the report
if [ -s "$REPORT_FILE" ]; then
    echo "Agent performance report for ${REPORT_DATE}" | \
    mail -s "Daily Agent Report - ${REPORT_DATE}" \
         -a "$REPORT_FILE" \
         manager@yourcallcenter.com
    echo "$(date): Report sent for ${REPORT_DATE}" >> /var/log/vicidial-reports.log
else
    echo "$(date): No data for ${REPORT_DATE}" >> /var/log/vicidial-reports.log
fi
```

### Step 2: Schedule with Cron

```bash
chmod +x /opt/vicidial-reports/daily_agent_report.sh

# Edit crontab
crontab -e
```

Add these entries:

```cron
# Daily agent report at 6:00 AM
0 6 * * * /opt/vicidial-reports/daily_agent_report.sh

# Hourly campaign snapshot during business hours (9 AM - 9 PM)
0 9-21 * * 1-6 /opt/vicidial-reports/hourly_campaign_snapshot.sh

# Weekly summary every Monday at 7:00 AM
0 7 * * 1 /opt/vicidial-reports/weekly_summary.sh
```

### Step 3: Create an Hourly Snapshot Script

For real-time operations, an hourly snapshot keeps leadership informed:

```bash
#!/bin/bash
# Hourly Campaign Snapshot
HOUR=$(date +%H)
TODAY=$(date +%Y-%m-%d)

mysql -u reports -p'your_password' asterisk --batch -e "
SELECT
    campaign_id AS Campaign,
    COUNT(*) AS 'Calls This Hour',
    SUM(CASE WHEN status NOT IN ('A','AA','AM','AL','N','NI','NP','DC','DNC','B','NA','ADC','DROP') THEN 1 ELSE 0 END) AS Contacts,
    ROUND(SUM(CASE WHEN status = 'DROP' THEN 1 ELSE 0 END) / NULLIF(COUNT(*), 0) * 100, 1) AS 'Drop %'
FROM vicidial_log
WHERE call_date >= '${TODAY} ${HOUR}:00:00'
  AND call_date < '${TODAY} ${HOUR}:59:59'
GROUP BY campaign_id;
" | mail -s "Hourly Snapshot - ${TODAY} ${HOUR}:00" ops-team@yourcallcenter.com
```

## Connecting to BI Tools

### Grafana Setup

Grafana turns your VICIdial data into live dashboards with auto-refresh. Here's how to set it up:

**1. Install the MySQL data source plugin** (usually included by default):

```bash
# Install Grafana if not already present
sudo yum install -y grafana
sudo systemctl enable --now grafana-server
```

**2. Add MySQL as a data source** in Grafana (Settings > Data Sources > Add > MySQL):

- Host: `localhost:3306`
- Database: `asterisk`
- User: `reports` (the read-only user you created)
- TLS/SSL: Disable for localhost

**3. Create a dashboard panel** with a query like:

```sql
SELECT
    $__timeGroupAlias(call_date, '1h') AS time,
    campaign_id AS metric,
    COUNT(*) AS value
FROM vicidial_log
WHERE $__timeFilter(call_date)
GROUP BY time, campaign_id
ORDER BY time;
```

Grafana's `$__timeFilter` macro automatically injects the time range from the dashboard selector.

**4. A real-time agent count panel:**

```sql
SELECT
    NOW() AS time,
    campaign_id AS metric,
    COUNT(DISTINCT user) AS value
FROM vicidial_live_agents
WHERE status IN ('INCALL','QUEUE','READY')
GROUP BY campaign_id;
```

Set this panel to refresh every 10 seconds for a live wallboard.

### Metabase Setup

Metabase is easier to set up than Grafana and better for non-technical users who want to explore data themselves.

```bash
# Quick Docker install
docker run -d -p 3000:3000 \
  --name metabase \
  -e "MB_DB_TYPE=mysql" \
  -e "MB_DB_DBNAME=metabase" \
  -e "MB_DB_PORT=3306" \
  -e "MB_DB_USER=metabase" \
  -e "MB_DB_PASS=your_password" \
  -e "MB_DB_HOST=localhost" \
  metabase/metabase
```

After setup, add the `asterisk` database as a data source. Metabase will auto-detect tables and let users build questions using a point-and-click interface.

Pro tip: Create the MySQL views described earlier in this article. Metabase works much better with views because they hide the complexity of joins and give clean, human-readable column names.

### Security Considerations

When connecting external BI tools to your VICIdial database:

1. **Always use a read-only MySQL user.** Never give BI tools write access.
2. **Restrict to localhost** if the BI tool runs on the same server. If remote, use SSH tunneling:

```bash
ssh -L 3306:localhost:3306 user@vicidial-server
```

3. **Create views that exclude sensitive data.** Don't expose SSNs, full credit card numbers, or other PII through BI tools:

```sql
CREATE OR REPLACE VIEW v_leads_safe AS
SELECT
    lead_id, list_id, status, called_count,
    entry_date, modify_date, source_id,
    state, vendor_lead_code,
    -- Mask sensitive fields
    CONCAT(LEFT(phone_number, 3), '***', RIGHT(phone_number, 2)) AS phone_masked,
    CONCAT(LEFT(first_name, 1), '***') AS first_name_masked
FROM vicidial_list;
```

4. **Monitor query performance.** Heavy reporting queries can slow down live dialing. Run expensive queries during off-hours or use a read replica.

## Performance Optimization Tips

VICIdial databases can grow massive. A center running 50 agents for a year can easily have 10+ million rows in `vicidial_log`. Here's how to keep reporting fast:

### Add Indexes for Report Queries

```sql
-- Speed up date-range queries on vicidial_log
ALTER TABLE vicidial_log ADD INDEX idx_call_date_campaign (call_date, campaign_id);

-- Speed up agent log queries
ALTER TABLE vicidial_agent_log ADD INDEX idx_event_time_user (event_time, user);

-- Speed up list penetration queries
ALTER TABLE vicidial_list ADD INDEX idx_list_status_count (list_id, status, called_count);
```

**Warning:** Only add indexes during off-hours. `ALTER TABLE` locks the table on older MySQL versions, which will freeze your dialer.

### Use Summary Tables for Historical Data

For reports spanning months, query summary tables instead of raw data:

```sql
CREATE TABLE report_daily_summary (
    report_date DATE NOT NULL,
    campaign_id VARCHAR(8) NOT NULL,
    total_calls INT DEFAULT 0,
    total_contacts INT DEFAULT 0,
    total_sales INT DEFAULT 0,
    total_drops INT DEFAULT 0,
    total_talk_sec INT DEFAULT 0,
    avg_handle_sec DECIMAL(8,1) DEFAULT 0,
    PRIMARY KEY (report_date, campaign_id)
);

-- Populate nightly via cron
INSERT INTO report_daily_summary
SELECT
    DATE(call_date), campaign_id,
    COUNT(*),
    SUM(CASE WHEN status NOT IN ('A','AA','AM','AL','N','NI','NP','DC','DNC','B','NA','ADC','DROP') THEN 1 ELSE 0 END),
    SUM(CASE WHEN status IN ('SALE','XFER') THEN 1 ELSE 0 END),
    SUM(CASE WHEN status = 'DROP' THEN 1 ELSE 0 END),
    SUM(length_in_sec),
    ROUND(AVG(length_in_sec), 1)
FROM vicidial_log
WHERE DATE(call_date) = DATE_SUB(CURDATE(), INTERVAL 1 DAY)
GROUP BY DATE(call_date), campaign_id
ON DUPLICATE KEY UPDATE
    total_calls = VALUES(total_calls),
    total_contacts = VALUES(total_contacts),
    total_sales = VALUES(total_sales);
```

## How ViciStack Helps

Building custom reports is powerful, but it takes time and MySQL expertise that most call center teams don't have in-house. At ViciStack, we've built and optimized VICIdial reporting for over 100 call centers. Our managed service includes:

- **Pre-built reporting dashboards** covering agent performance, campaign metrics, AMD accuracy, and carrier health -- ready from day one
- **Automated daily and weekly reports** delivered to your inbox with the metrics that matter for your specific operation
- **Database optimization** including proper indexing, query tuning, and summary table management to keep reports fast even at scale
- **Custom report development** for any metric your clients or management need, built and maintained by VICIdial database experts
- **Real-time monitoring** that catches problems (drop rate spikes, AMD drift, carrier failures) before they cost you money

All of this is included in our flat $150/agent/month pricing. No per-minute charges, no surprise fees.

**Get a free analysis of your VICIdial reporting and performance.** We'll review your current setup and show you exactly where you're leaving data (and money) on the table. Request your free analysis at [vicistack.com/proof/](https://vicistack.com/proof/) -- we respond within 5 minutes during business hours.

## Related Articles

- [VICIdial AMD Optimization: Eliminating False Positives](/blog/vicidial-amd-optimization)
- [VICIdial Performance Tuning Guide](/blog/vicidial-performance-tuning)
- [VICIdial CRM Integration Guide](/blog/vicidial-crm-integration)
- [VICIdial Predictive Dialer Optimization](/blog/vicidial-predictive-dialer-optimization)

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-custom-mysql-reports).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
