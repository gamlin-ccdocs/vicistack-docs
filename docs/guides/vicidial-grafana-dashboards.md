# Building VICIdial Dashboards with Grafana & Metabase

**The complete, hands-on guide to building production-grade VICIdial dashboards with Grafana and Metabase — MySQL data source configuration, SQL queries for every KPI that matters, real-time agent status panels, campaign performance charts, threshold-based alerting, and exportable JSON templates you can deploy today.**

---

VICIdial's built-in reporting gets the job done. It's accurate, it covers the fundamentals, and it has served operations of all sizes for years. But if you've ever tried to put VICIdial's default HTML reports on a wallboard, hand them to an executive, or use them to make split-second decisions during a live campaign — you already know the limitations.

The data is all there. VICIdial logs everything into MySQL: every dial attempt, every agent state change, every second of talk time, every disposition. The problem is presentation. VICIdial's reporting interface was built for functional accuracy, not for visual clarity, real-time responsiveness, or the kind of at-a-glance dashboards that modern call center operations demand.

That's where Grafana and Metabase come in. Grafana gives you real-time, auto-refreshing dashboards with configurable panels, threshold-based color coding, and alerting — purpose-built for operational monitoring. Metabase gives your managers and executives a self-service analytics layer where they can ask questions about VICIdial data without writing SQL.

This guide walks through the full setup: installing and configuring both tools, connecting them to VICIdial's MySQL database, building every panel a call center needs, setting up alerting for compliance and operational thresholds, and deploying ready-to-use dashboard templates. If you haven't already reviewed VICIdial's native reporting capabilities and key database tables, start with our [VICIdial Reporting & Real-Time Monitoring Guide](/blog/vicidial-reporting-monitoring/) — this article builds directly on that foundation.

---

## Why VICIdial's Built-In Reports Aren't Enough

Before we get into the build, let's be specific about what the built-in reports lack and what external dashboards solve.

**VICIdial's default reports:**

- Render as static HTML tables that require manual page refreshes
- Show one campaign at a time in most views
- Don't support visual time-series charts, gauges, or heatmaps
- Have no built-in alerting or threshold notifications
- Require admin-level access to the VICIdial interface
- Can't be customized without modifying VICIdial's PHP source code

**What Grafana/Metabase dashboards add:**

- Auto-refreshing panels (configurable down to 5-second intervals)
- Multi-campaign, multi-metric views on a single screen
- Color-coded thresholds — green/yellow/red at a glance
- Time-series charts that show trends, not just snapshots
- Alerting via email, Slack, PagerDuty, or webhooks when metrics breach thresholds
- Role-based access — give managers dashboard access without VICIdial admin credentials
- TV/wallboard display mode purpose-built for operations floors

None of this means you should abandon VICIdial's built-in reports. They're still useful for detailed drill-downs and ad-hoc lookups. External dashboards are for the metrics you watch continuously — the ones that should be on a screen, in front of a supervisor, updating every few seconds.

---

## Grafana Setup: Installation and MySQL Data Source

### Installing Grafana on AlmaLinux/Rocky Linux (ViciBox Base)

If you're running ViciBox (the standard VICIdial server OS), you're on AlmaLinux/Rocky Linux 9. Install Grafana from the official RPM repository:

```bash
# Add the Grafana repository
cat <<'EOF' | sudo tee /etc/yum.repos.d/grafana.repo
[grafana]
name=grafana
baseurl=https://rpm.grafana.com
repo_gpgcheck=1
enabled=1
gpgcheck=1
gpgkey=https://rpm.grafana.com/gpg.key
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
EOF

# Install and start Grafana
sudo dnf install -y grafana
sudo systemctl enable --now grafana-server

# Open the firewall port
sudo firewall-cmd --permanent --add-port=3000/tcp
sudo firewall-cmd --reload
```

Grafana is now running at `http://your-server:3000`. Default login is `admin` / `admin` — change it immediately.

**Where to install Grafana:** Ideally, install it on a dedicated monitoring server or on your VICIdial web server — not on a dialer server. Dialer servers need every CPU cycle for Asterisk and the dialing engine. If you're running a [cluster deployment](/blog/vicidial-cluster-guide/), the web server is the best candidate since it already handles the admin interface and reporting queries.

### Creating a Read-Only MySQL User

Never point Grafana at VICIdial's database using the root or `cron` MySQL user. Create a dedicated read-only account:

```sql
-- Run this on your VICIdial MySQL server
CREATE USER 'grafana_ro'@'%' IDENTIFIED BY 'a_strong_random_password_here';
GRANT SELECT ON asterisk.* TO 'grafana_ro'@'%';
FLUSH PRIVILEGES;
```

If Grafana is on the same server as MySQL, replace `'%'` with `'localhost'` for tighter security. If it's on a separate server, replace `'%'` with the Grafana server's IP address.

**Security note:** The `SELECT` grant on `asterisk.*` gives Grafana read access to all VICIdial tables, including `vicidial_list` which contains lead PII (names, phone numbers, addresses). If you need to restrict access to only operational metrics tables, grant SELECT on specific tables instead:

```sql
GRANT SELECT ON asterisk.vicidial_log TO 'grafana_ro'@'%';
GRANT SELECT ON asterisk.vicidial_closer_log TO 'grafana_ro'@'%';
GRANT SELECT ON asterisk.vicidial_agent_log TO 'grafana_ro'@'%';
GRANT SELECT ON asterisk.vicidial_live_agents TO 'grafana_ro'@'%';
GRANT SELECT ON asterisk.vicidial_auto_calls TO 'grafana_ro'@'%';
GRANT SELECT ON asterisk.vicidial_hopper TO 'grafana_ro'@'%';
GRANT SELECT ON asterisk.vicidial_campaign_stats TO 'grafana_ro'@'%';
GRANT SELECT ON asterisk.vicidial_campaigns TO 'grafana_ro'@'%';
FLUSH PRIVILEGES;
```

### Configuring the MySQL Data Source in Grafana

1. Log into Grafana at `http://your-server:3000`
2. Navigate to **Connections** (gear icon) → **Data sources** → **Add data source**
3. Select **MySQL**
4. Configure the connection:

```
Name: VICIdial MySQL
Host: your-vicidial-db-ip:3306
Database: asterisk
User: grafana_ro
Password: [the password you set above]
Session timezone: (leave as default or set to your server timezone)
Min time interval: 10s
```

5. Click **Save & Test** — you should see "Database Connection OK"

If the connection fails, check:

- MySQL is listening on the correct interface (not just `127.0.0.1` if Grafana is on another server). Check `bind-address` in `/etc/my.cnf`.
- Firewall allows port 3306 from the Grafana server: `sudo firewall-cmd --add-rich-rule='rule family="ipv4" source address="GRAFANA_IP" port port="3306" protocol="tcp" accept' --permanent && sudo firewall-cmd --reload`
- The MySQL user has the correct host grants: `SELECT user, host FROM mysql.user WHERE user = 'grafana_ro';`

For detailed MySQL tuning to support both VICIdial operations and reporting queries simultaneously, see our [VICIdial MySQL Optimization](/blog/vicidial-mysql-optimization/) guide.

### Setting Up Dashboard Variables

Before building individual panels, configure dashboard variables that let you filter by campaign, date range, and agent. This makes your dashboards reusable across campaigns.

In your Grafana dashboard, click the gear icon → **Variables** → **Add variable**:

**Variable: campaign_id**

```
Name: campaign_id
Type: Query
Data source: VICIdial MySQL
Query: SELECT DISTINCT campaign_id FROM vicidial_campaigns WHERE active = 'Y' ORDER BY campaign_id;
Multi-value: Yes
Include All option: Yes
```

**Variable: user_group**

```
Name: user_group
Type: Query
Data source: VICIdial MySQL
Query: SELECT DISTINCT user_group FROM vicidial_user_groups ORDER BY user_group;
Multi-value: Yes
Include All option: Yes
```

Now every panel query can reference `$campaign_id` and `$user_group` as filters, and your supervisors can switch between campaigns from a dropdown at the top of the dashboard.

---

## Building KPI Panels: The SQL Queries That Matter

Every panel below includes the exact SQL query, the recommended Grafana visualization type, and the refresh interval. These queries are production-tested against VICIdial databases handling 100,000+ calls per day.

### Panel 1: Calls Per Hour (Time Series)

**Visualization:** Time Series
**Refresh:** 30s
**Purpose:** Shows dialing velocity and answer rates over time. The single most useful chart for understanding campaign momentum throughout the day.

```sql
SELECT
    $__timeGroup(call_date, '1h') as time,
    COUNT(*) as Dials,
    COUNT(CASE WHEN status NOT IN ('NA','B','DC','N','AFTHRS') THEN 1 END) as Answered,
    COUNT(CASE WHEN status IN ('SALE','SET','XFER') THEN 1 END) as Conversions
FROM vicidial_log
WHERE $__timeFilter(call_date)
    AND campaign_id IN ($campaign_id)
GROUP BY $__timeGroup(call_date, '1h')
ORDER BY time;
```

The `$__timeGroup` and `$__timeFilter` are Grafana macros that automatically map to the dashboard's selected time range. This means your supervisors can zoom into any window — the last hour, the last 8 hours, today, this week — and the panel adapts.

**Grafana panel settings:**
- Set the Y-axis to show count values
- Use different colors for Dials (blue), Answered (green), Conversions (orange)
- Add a dashed line for Conversions since the scale is typically much smaller

This panel directly tracks your [calls per hour](/glossary/calls-per-hour/) metric, one of the fundamental indicators of dialer efficiency.

### Panel 2: Connect Rate Gauge

**Visualization:** Gauge
**Refresh:** 60s
**Purpose:** Shows the current connect rate as a single number with color-coded thresholds.

```sql
SELECT
    ROUND(
        COUNT(CASE WHEN status NOT IN ('NA','B','DC','N','AFTHRS','AB','ADC','XDROP')
                   AND user != '' THEN 1 END) /
        NULLIF(COUNT(*), 0) * 100, 2
    ) as connect_rate
FROM vicidial_log
WHERE call_date >= CURDATE()
    AND campaign_id IN ($campaign_id);
```

**Threshold configuration:**
- Green: > 15% (healthy connect rate for most B2C outbound)
- Yellow: 10-15% (acceptable, but check DID health)
- Red: < 10% (list quality issues, DID flagging, or time-of-day problem)

These thresholds vary by vertical. A debt collection campaign might see 8-12% connect rates as normal, while a warm lead follow-up campaign should hit 25%+. Adjust the thresholds to match your operation.

### Panel 3: Drop Rate (Compliance Critical)

**Visualization:** Gauge
**Refresh:** 60s
**Purpose:** The most legally important metric on your dashboard. Shows the 30-day rolling [abandon rate](/glossary/abandon-rate/) per campaign.

```sql
SELECT
    ROUND(
        COUNT(CASE WHEN status = 'DROP' THEN 1 END) /
        NULLIF(COUNT(CASE WHEN status NOT IN ('NA','B','DC','N','AFTHRS','XDROP') THEN 1 END), 0) * 100, 2
    ) as drop_rate_30d
FROM vicidial_log
WHERE call_date >= DATE_SUB(NOW(), INTERVAL 30 DAY)
    AND campaign_id IN ($campaign_id);
```

**Threshold configuration:**
- Green: < 2.0% (comfortable margin)
- Yellow: 2.0-2.5% (caution — tighten dial levels)
- Red: > 2.5% (danger zone — immediate action required)

The FTC's 3% rule on abandoned calls is measured over a 30-day campaign period. You want your red threshold well below 3% because by the time you hit 2.5%, one bad hour can push you over. This gauge should be the single largest panel on your wallboard.

### Panel 4: Real-Time Agent Status Distribution

**Visualization:** Pie Chart or Bar Gauge
**Refresh:** 5s
**Purpose:** Shows how many agents are in each state right now. This is the panel supervisors watch second-by-second.

```sql
SELECT
    status as "Status",
    COUNT(*) as "Count"
FROM vicidial_live_agents
WHERE campaign_id IN ($campaign_id)
GROUP BY status
ORDER BY COUNT(*) DESC;
```

**Color mapping:**
- INCALL: Green (productive)
- READY: Blue (waiting for calls — dialer should be sending them)
- PAUSED: Orange (not taking calls — acceptable during breaks, concerning otherwise)
- CLOSER: Teal (handling inbound/transferred call)
- DEAD: Red (session problem — agent needs to re-login)

A healthy outbound campaign should show most agents in INCALL, a small percentage in READY (transitioning between calls), and minimal PAUSED time outside scheduled breaks. If READY count is high, your dial level is too low or your hopper is empty. If PAUSED is consistently above 15-20%, investigate pause code usage.

### Panel 5: Agent Leaderboard (Table)

**Visualization:** Table
**Refresh:** 30s
**Purpose:** Real-time ranking of agents by production metrics. Put this on the floor wallboard — healthy competition drives performance.

```sql
SELECT
    val.user as "Agent",
    COUNT(*) as "Calls",
    COUNT(CASE WHEN val.status IN ('SALE','SET','XFER') THEN 1 END) as "Sales",
    ROUND(AVG(val.talk_sec), 0) as "Avg Talk (s)",
    ROUND(SUM(val.talk_sec) / 60, 0) as "Talk Min",
    ROUND(SUM(val.pause_sec) / 60, 0) as "Pause Min",
    ROUND(
        SUM(val.talk_sec) /
        NULLIF(SUM(val.talk_sec + val.wait_sec + val.pause_sec + val.dispo_sec), 0) * 100, 1
    ) as "Utilization %"
FROM vicidial_agent_log val
WHERE val.event_time >= CURDATE()
    AND val.campaign_id IN ($campaign_id)
GROUP BY val.user
ORDER BY COUNT(CASE WHEN val.status IN ('SALE','SET','XFER') THEN 1 END) DESC;
```

The "Utilization %" column represents agent [occupancy rate](/glossary/occupancy-rate/) — the percentage of logged-in time spent actually handling calls. Top-performing outbound agents typically hit 45-55%. Below 35% means either the dialer isn't feeding them fast enough or they're pausing excessively.

**Grafana table overrides:** Use the column override feature to add a bar gauge to the "Utilization %" column and color-code it: green above 45%, yellow 35-45%, red below 35%.

### Panel 6: Conversion Rate by Campaign (Bar Chart)

**Visualization:** Bar Chart
**Refresh:** 60s
**Purpose:** Cross-campaign conversion rate comparison for today. Lets managers instantly spot which campaigns are producing and which need attention.

```sql
SELECT
    campaign_id as "Campaign",
    COUNT(*) as "Dials",
    COUNT(CASE WHEN user != '' AND status NOT IN ('NA','B','DC','N','DROP','AFTHRS','XDROP')
               THEN 1 END) as "Contacts",
    COUNT(CASE WHEN status IN ('SALE','SET','XFER') THEN 1 END) as "Conversions",
    ROUND(
        COUNT(CASE WHEN status IN ('SALE','SET','XFER') THEN 1 END) /
        NULLIF(COUNT(CASE WHEN user != '' AND status NOT IN ('NA','B','DC','N','DROP','AFTHRS','XDROP')
                   THEN 1 END), 0) * 100, 2
    ) as "Contact-to-Conv %"
FROM vicidial_log
WHERE call_date >= CURDATE()
GROUP BY campaign_id
HAVING COUNT(*) > 10
ORDER BY "Contact-to-Conv %" DESC;
```

This shows both the raw conversion rate (dials to conversions) and the contact-to-conversion rate (human contacts to conversions). The second metric tells you about agent effectiveness — once a lead picks up, how often do your agents close? A campaign with high contact rates but low contact-to-conversion rates has a scripting or training problem, not a dialing problem.

### Panel 7: Hopper Status (Stat Panel)

**Visualization:** Stat
**Refresh:** 10s
**Purpose:** Early warning for list exhaustion. When the hopper runs dry, the dialer stops.

```sql
SELECT
    campaign_id as "Campaign",
    COUNT(*) as "Leads in Hopper"
FROM vicidial_hopper
WHERE campaign_id IN ($campaign_id)
GROUP BY campaign_id;
```

**Threshold configuration:**
- Green: > 200 leads (healthy buffer for most operations)
- Yellow: 50-200 leads (monitor closely, may need to add lists)
- Red: < 50 leads (dialer is about to stall — take action immediately)

Adjust these thresholds based on your agent count and dial level. A 100-agent campaign at dial level 3.0 burns through ~300 leads per minute. At that rate, 50 leads in the hopper gives you roughly 10 seconds before the dialer stalls.

### Panel 8: Trunk Utilization (Gauge)

**Visualization:** Gauge
**Refresh:** 10s
**Purpose:** Monitor SIP trunk capacity to prevent call failures.

```sql
SELECT
    COUNT(*) as active_channels
FROM vicidial_auto_calls;
```

Set the gauge maximum to your total trunk capacity (e.g., 300 simultaneous channels). Thresholds:
- Green: < 70% of capacity
- Yellow: 70-85% of capacity
- Red: > 85% of capacity (risk of SIP 503 errors on new calls)

### Panel 9: Average Handle Time Trend (Time Series)

**Visualization:** Time Series
**Refresh:** 60s
**Purpose:** Track [average handle time](/glossary/average-handle-time/) trends throughout the day. AHT spikes can indicate system issues, script changes, or agent behavior problems.

```sql
SELECT
    $__timeGroup(event_time, '30m') as time,
    ROUND(AVG(talk_sec + dispo_sec), 1) as "Avg Handle Time (s)",
    ROUND(AVG(talk_sec), 1) as "Avg Talk Time (s)",
    ROUND(AVG(dispo_sec), 1) as "Avg Dispo Time (s)"
FROM vicidial_agent_log
WHERE $__timeFilter(event_time)
    AND campaign_id IN ($campaign_id)
    AND talk_sec > 0
GROUP BY $__timeGroup(event_time, '30m')
ORDER BY time;
```

Handle time = talk time + disposition time. If you see dispo time climbing, agents may be struggling with the disposition screen or gaming the system by sitting in wrap-up to avoid calls. If talk time spikes, check if there's a script or product change causing longer conversations.

### Panel 10: Hourly Conversion Heatmap

**Visualization:** Heatmap or State Timeline
**Refresh:** 300s (5 minutes)
**Purpose:** Shows which hours and days produce the best conversion rates. Critical for optimizing agent schedules and call time windows.

```sql
SELECT
    DAYNAME(call_date) as day_name,
    HOUR(call_date) as hour,
    ROUND(
        COUNT(CASE WHEN status IN ('SALE','SET','XFER') THEN 1 END) /
        NULLIF(COUNT(CASE WHEN user != '' AND status NOT IN ('NA','B','DC','N','DROP','AFTHRS','XDROP')
                   THEN 1 END), 0) * 100, 2
    ) as conversion_rate
FROM vicidial_log
WHERE call_date >= DATE_SUB(NOW(), INTERVAL 30 DAY)
    AND campaign_id IN ($campaign_id)
    AND user != ''
GROUP BY DAYNAME(call_date), HOUR(call_date)
ORDER BY DAYOFWEEK(call_date), hour;
```

This heatmap is gold for staffing decisions. If Tuesday 10-11 AM consistently converts at 8% while Friday 4-5 PM converts at 2%, you know exactly where to concentrate your best agents and where to schedule training sessions instead of live dialing.

> **Struggling to turn VICIdial data into actionable operations improvements?** We build custom monitoring stacks for VICIdial operations of all sizes. [Request a free infrastructure audit](/free-audit/) and we'll show you exactly where your dashboards, alerting, and data pipeline should be.

---

## Complete Grafana Dashboard JSON Template

Instead of building every panel manually, import this JSON template into Grafana. Go to **Dashboards** → **New** → **Import** → paste the JSON below.

This template creates a complete VICIdial operations dashboard with all the panels described above, pre-configured variables, and sensible refresh intervals.

```json
{
  "dashboard": {
    "id": null,
    "uid": null,
    "title": "VICIdial Operations Dashboard",
    "tags": ["vicidial", "call-center", "operations"],
    "timezone": "browser",
    "refresh": "30s",
    "time": {
      "from": "now-12h",
      "to": "now"
    },
    "templating": {
      "list": [
        {
          "name": "campaign_id",
          "type": "query",
          "datasource": "VICIdial MySQL",
          "query": "SELECT DISTINCT campaign_id FROM vicidial_campaigns WHERE active = 'Y' ORDER BY campaign_id",
          "multi": true,
          "includeAll": true,
          "current": { "text": "All", "value": "$__all" }
        }
      ]
    },
    "panels": [
      {
        "id": 1,
        "title": "Drop Rate (30-Day Rolling)",
        "type": "gauge",
        "gridPos": { "h": 6, "w": 6, "x": 0, "y": 0 },
        "targets": [
          {
            "rawSql": "SELECT ROUND(COUNT(CASE WHEN status = 'DROP' THEN 1 END) / NULLIF(COUNT(CASE WHEN status NOT IN ('NA','B','DC','N','AFTHRS','XDROP') THEN 1 END), 0) * 100, 2) as drop_rate FROM vicidial_log WHERE call_date >= DATE_SUB(NOW(), INTERVAL 30 DAY) AND campaign_id IN ($campaign_id)",
            "format": "table"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "min": 0,
            "max": 5,
            "thresholds": {
              "steps": [
                { "color": "green", "value": null },
                { "color": "yellow", "value": 2.0 },
                { "color": "red", "value": 2.5 }
              ]
            },
            "unit": "percent"
          }
        }
      },
      {
        "id": 2,
        "title": "Connect Rate (Today)",
        "type": "gauge",
        "gridPos": { "h": 6, "w": 6, "x": 6, "y": 0 },
        "targets": [
          {
            "rawSql": "SELECT ROUND(COUNT(CASE WHEN status NOT IN ('NA','B','DC','N','AFTHRS','AB','ADC','XDROP') AND user != '' THEN 1 END) / NULLIF(COUNT(*), 0) * 100, 2) as connect_rate FROM vicidial_log WHERE call_date >= CURDATE() AND campaign_id IN ($campaign_id)",
            "format": "table"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "min": 0,
            "max": 50,
            "thresholds": {
              "steps": [
                { "color": "red", "value": null },
                { "color": "yellow", "value": 10 },
                { "color": "green", "value": 15 }
              ]
            },
            "unit": "percent"
          }
        }
      },
      {
        "id": 3,
        "title": "Hopper Status",
        "type": "stat",
        "gridPos": { "h": 6, "w": 6, "x": 12, "y": 0 },
        "targets": [
          {
            "rawSql": "SELECT COUNT(*) as leads_in_hopper FROM vicidial_hopper WHERE campaign_id IN ($campaign_id)",
            "format": "table"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "thresholds": {
              "steps": [
                { "color": "red", "value": null },
                { "color": "yellow", "value": 50 },
                { "color": "green", "value": 200 }
              ]
            }
          }
        }
      },
      {
        "id": 4,
        "title": "Active Trunks",
        "type": "gauge",
        "gridPos": { "h": 6, "w": 6, "x": 18, "y": 0 },
        "targets": [
          {
            "rawSql": "SELECT COUNT(*) as active_channels FROM vicidial_auto_calls",
            "format": "table"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "min": 0,
            "max": 300,
            "thresholds": {
              "steps": [
                { "color": "green", "value": null },
                { "color": "yellow", "value": 210 },
                { "color": "red", "value": 255 }
              ]
            }
          }
        }
      },
      {
        "id": 5,
        "title": "Agent Status Distribution",
        "type": "piechart",
        "gridPos": { "h": 8, "w": 8, "x": 0, "y": 6 },
        "targets": [
          {
            "rawSql": "SELECT status as Status, COUNT(*) as Count FROM vicidial_live_agents WHERE campaign_id IN ($campaign_id) GROUP BY status ORDER BY Count DESC",
            "format": "table"
          }
        ]
      },
      {
        "id": 6,
        "title": "Calls Per Hour",
        "type": "timeseries",
        "gridPos": { "h": 8, "w": 16, "x": 8, "y": 6 },
        "targets": [
          {
            "rawSql": "SELECT $__timeGroup(call_date, '1h') as time, COUNT(*) as Dials, COUNT(CASE WHEN status NOT IN ('NA','B','DC','N','AFTHRS') THEN 1 END) as Answered, COUNT(CASE WHEN status IN ('SALE','SET','XFER') THEN 1 END) as Conversions FROM vicidial_log WHERE $__timeFilter(call_date) AND campaign_id IN ($campaign_id) GROUP BY $__timeGroup(call_date, '1h') ORDER BY time",
            "format": "time_series"
          }
        ]
      },
      {
        "id": 7,
        "title": "Agent Leaderboard",
        "type": "table",
        "gridPos": { "h": 10, "w": 24, "x": 0, "y": 14 },
        "targets": [
          {
            "rawSql": "SELECT val.user as Agent, COUNT(*) as Calls, COUNT(CASE WHEN val.status IN ('SALE','SET','XFER') THEN 1 END) as Sales, ROUND(AVG(val.talk_sec), 0) as 'Avg Talk (s)', ROUND(SUM(val.talk_sec) / 60, 0) as 'Talk Min', ROUND(SUM(val.pause_sec) / 60, 0) as 'Pause Min', ROUND(SUM(val.talk_sec) / NULLIF(SUM(val.talk_sec + val.wait_sec + val.pause_sec + val.dispo_sec), 0) * 100, 1) as 'Utilization %' FROM vicidial_agent_log val WHERE val.event_time >= CURDATE() AND val.campaign_id IN ($campaign_id) GROUP BY val.user ORDER BY Sales DESC",
            "format": "table"
          }
        ]
      },
      {
        "id": 8,
        "title": "Average Handle Time Trend",
        "type": "timeseries",
        "gridPos": { "h": 8, "w": 12, "x": 0, "y": 24 },
        "targets": [
          {
            "rawSql": "SELECT $__timeGroup(event_time, '30m') as time, ROUND(AVG(talk_sec + dispo_sec), 1) as 'Avg Handle Time (s)', ROUND(AVG(talk_sec), 1) as 'Avg Talk (s)', ROUND(AVG(dispo_sec), 1) as 'Avg Dispo (s)' FROM vicidial_agent_log WHERE $__timeFilter(event_time) AND campaign_id IN ($campaign_id) AND talk_sec > 0 GROUP BY $__timeGroup(event_time, '30m') ORDER BY time",
            "format": "time_series"
          }
        ]
      },
      {
        "id": 9,
        "title": "Campaign Conversion Rates (Today)",
        "type": "barchart",
        "gridPos": { "h": 8, "w": 12, "x": 12, "y": 24 },
        "targets": [
          {
            "rawSql": "SELECT campaign_id as Campaign, ROUND(COUNT(CASE WHEN status IN ('SALE','SET','XFER') THEN 1 END) / NULLIF(COUNT(CASE WHEN user != '' AND status NOT IN ('NA','B','DC','N','DROP','AFTHRS','XDROP') THEN 1 END), 0) * 100, 2) as 'Contact-to-Conv %' FROM vicidial_log WHERE call_date >= CURDATE() GROUP BY campaign_id HAVING COUNT(*) > 10 ORDER BY 'Contact-to-Conv %' DESC",
            "format": "table"
          }
        ]
      }
    ]
  }
}
```

**To customize this template:**

1. Import it into Grafana
2. Adjust the data source name if yours isn't "VICIdial MySQL"
3. Set the trunk capacity max on Panel 4 to match your actual SIP trunk limits
4. Adjust threshold values on all gauges to match your operation's benchmarks
5. Add or remove `$campaign_id` filters based on whether you want per-campaign or global views

---

## Advanced Grafana Panels for Power Users

The template above covers the essentials. Here are additional panels for operations that want deeper visibility.

### DID/Caller ID Health Tracker

Track which outbound DIDs are performing and which have been flagged as spam:

```sql
-- Grafana: Table panel
-- Refresh: 300s
SELECT
    vc.campaign_cid as "Outbound DID",
    COUNT(vl.uniqueid) as "Dials (7d)",
    COUNT(CASE WHEN vl.status NOT IN ('NA','B','DC','N','AFTHRS') THEN 1 END) as "Answered",
    ROUND(
        COUNT(CASE WHEN vl.status NOT IN ('NA','B','DC','N','AFTHRS') THEN 1 END) /
        NULLIF(COUNT(vl.uniqueid), 0) * 100, 2
    ) as "Answer Rate %",
    ROUND(
        COUNT(CASE WHEN vl.status = 'NA' THEN 1 END) /
        NULLIF(COUNT(vl.uniqueid), 0) * 100, 2
    ) as "No Answer %"
FROM vicidial_log vl
JOIN vicidial_campaigns vc ON vl.campaign_id = vc.campaign_id
WHERE vl.call_date >= DATE_SUB(NOW(), INTERVAL 7 DAY)
    AND vl.campaign_id IN ($campaign_id)
GROUP BY vc.campaign_cid
HAVING COUNT(vl.uniqueid) > 100
ORDER BY ROUND(
    COUNT(CASE WHEN vl.status NOT IN ('NA','B','DC','N','AFTHRS') THEN 1 END) /
    NULLIF(COUNT(vl.uniqueid), 0) * 100, 2
) ASC;
```

Sort by answer rate ascending so flagged DIDs (with abnormally low answer rates) float to the top. A DID answering at 8% when your campaign average is 18% is almost certainly flagged. Rotate it immediately.

### Inbound Queue Performance (for Blended Campaigns)

If you're running blended campaigns with inbound in-groups, monitor queue health:

```sql
-- Grafana: Time Series
-- Refresh: 10s
SELECT
    $__timeGroup(call_date, '15m') as time,
    ROUND(AVG(queue_seconds), 1) as "Avg Queue Time (s)",
    MAX(queue_seconds) as "Max Queue Time (s)",
    ROUND(
        COUNT(CASE WHEN queue_seconds <= 20 THEN 1 END) /
        NULLIF(COUNT(*), 0) * 100, 1
    ) as "SLA % (20s)"
FROM vicidial_closer_log
WHERE $__timeFilter(call_date)
    AND campaign_id IN ($campaign_id)
GROUP BY $__timeGroup(call_date, '15m')
ORDER BY time;
```

The SLA percentage (calls answered within 20 seconds) is the standard inbound service level metric. Industry standard target is 80/20 (80% of calls answered within 20 seconds). If you're consistently below this, you need more agents on the inbound queue.

### Agent State Timeline

Visualize individual agent state transitions throughout the day:

```sql
-- Grafana: State Timeline visualization
-- Refresh: 30s
SELECT
    event_time as time,
    user as "Agent",
    CASE
        WHEN talk_sec > 0 THEN 'INCALL'
        WHEN pause_sec > 0 THEN 'PAUSED'
        WHEN wait_sec > 0 THEN 'READY'
        ELSE 'DISPO'
    END as "State"
FROM vicidial_agent_log
WHERE event_time >= CURDATE()
    AND campaign_id IN ($campaign_id)
ORDER BY user, event_time;
```

This creates a Gantt-style view showing exactly what each agent was doing and when. It's particularly useful for identifying agents who take excessively long pauses or who consistently drop into extended dispo time between calls.

### Cost Per Lead Tracker

Combine VICIdial conversion data with cost estimates:

```sql
-- Grafana: Stat panel
-- Note: Update cost_per_minute to match your actual VoIP rate
SELECT
    COUNT(CASE WHEN status IN ('SALE','SET','XFER') THEN 1 END) as "Conversions Today",
    ROUND(SUM(length_in_sec) / 60 * 0.012, 2) as "Est. Telco Cost ($)",
    ROUND(
        (SUM(length_in_sec) / 60 * 0.012) /
        NULLIF(COUNT(CASE WHEN status IN ('SALE','SET','XFER') THEN 1 END), 0), 2
    ) as "Telco Cost Per Lead ($)"
FROM vicidial_log
WHERE call_date >= CURDATE()
    AND campaign_id IN ($campaign_id);
```

This gives you the telecom component of [cost per lead](/blog/call-center-cost-per-lead-benchmarks/). Replace `0.012` with your actual per-minute VoIP rate. For total CPL including labor and infrastructure, you'll need to add those costs externally.

---

## Metabase: The Manager-Friendly Alternative

Grafana is powerful but technical. The dashboard JSON, SQL queries, and panel configuration require someone comfortable with databases and code. For managers, team leads, and executives who need to explore VICIdial data without writing SQL, Metabase is the better tool.

### Why Metabase for VICIdial

Metabase provides:

- **Visual query builder:** Drag-and-drop interface for building reports without SQL
- **Natural language queries:** Type "how many sales did we make last week" and get a chart
- **Self-service dashboards:** Managers can build and share their own reports
- **Scheduled email reports:** Automatic delivery of key metrics on a schedule
- **Simpler permission model:** Easy to give different teams access to different data
- **Embedded analytics:** Embed charts in internal tools or client portals

### Installing Metabase

Metabase runs as a Java application or Docker container. Docker is the simplest deployment:

```bash
# Create a directory for Metabase data
sudo mkdir -p /opt/metabase/data

# Run Metabase via Docker
docker run -d \
  --name metabase \
  -p 3001:3000 \
  -v /opt/metabase/data:/metabase-data \
  -e "MB_DB_FILE=/metabase-data/metabase.db" \
  --restart unless-stopped \
  metabase/metabase
```

We're mapping to port 3001 since Grafana likely already occupies 3000 on the same server. Access Metabase at `http://your-server:3001`.

If you're not running Docker on your ViciBox server (and you probably shouldn't be on a production dialer), install Metabase on a separate utility server:

```bash
# Java-based installation (no Docker required)
sudo dnf install -y java-17-openjdk
sudo mkdir -p /opt/metabase
cd /opt/metabase
sudo curl -OL https://downloads.metabase.com/v0.50.0/metabase.jar

# Create a systemd service
cat <<'EOF' | sudo tee /etc/systemd/system/metabase.service
[Unit]
Description=Metabase
After=network.target

[Service]
Type=simple
User=metabase
WorkingDirectory=/opt/metabase
ExecStart=/usr/bin/java -jar /opt/metabase/metabase.jar
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

sudo useradd -r -s /bin/false metabase
sudo chown -R metabase:metabase /opt/metabase
sudo systemctl enable --now metabase
```

### Connecting Metabase to VICIdial

1. Open Metabase at `http://your-server:3001`
2. Complete the setup wizard
3. When prompted to add a database, select **MySQL** and enter:

```
Display name: VICIdial Production
Host: your-vicidial-db-ip
Port: 3306
Database name: asterisk
Username: grafana_ro (reuse the same read-only user)
Password: [password]
```

4. Metabase will scan the database and catalog all tables automatically

### Building Reports Without SQL

This is where Metabase shines. Once connected, managers can:

**Ask a "New Question" via the visual builder:**

1. Click **New** → **Question**
2. Select the `vicidial_log` table
3. Add filters: `call_date` is today, `campaign_id` is "SALES01"
4. Summarize: Count of rows, grouped by `status`
5. Click **Visualize** — instant disposition breakdown chart

**Create a "Sales by Hour" chart:**

1. Select `vicidial_log`
2. Filter: `status` is "SALE", `call_date` is "this week"
3. Summarize: Count, grouped by `call_date` (hourly)
4. Visualize as a line chart

No SQL required. Managers can save these questions to collections, add them to dashboards, and schedule them for email delivery.

### Metabase Scheduled Reports

One of Metabase's best features for call center operations: automated report delivery.

1. Create a dashboard with key metrics (today's conversions, agent leaderboard, campaign summary)
2. Click the **Share** icon → **Dashboard subscription**
3. Set the schedule: daily at 7:00 AM, before the shift starts
4. Add recipients: manager@company.com, supervisor@company.com
5. Choose format: email with attached PDF or embedded charts

Your managers get a daily performance snapshot in their inbox before they walk onto the floor. No logins required, no remembering to pull reports.

### When to Use Grafana vs. Metabase

| Use Case | Grafana | Metabase |
|---|---|---|
| Real-time wallboard dashboards | Best choice | Adequate |
| Auto-refresh every 5-10 seconds | Yes | Limited |
| Threshold alerting (Slack, email) | Built-in | Limited |
| Manager self-service reports | Complex | Best choice |
| Ad-hoc data exploration | SQL only | Visual builder |
| Scheduled email reports | Via plugin | Built-in |
| Embedding in other tools | Yes | Yes |
| Learning curve | Steep | Gentle |

**Our recommendation:** Run both. Grafana on the wallboard for real-time operations monitoring. Metabase for everything else — daily reports, ad-hoc analysis, executive dashboards, and scheduled delivery.

---

## Alerting: Automated Threshold Monitoring

Dashboards are only useful when someone is watching. Alerting makes sure critical metrics get attention even when nobody's looking at the screen.

### Grafana Alerting Setup

Grafana has a built-in alerting engine that evaluates queries on a schedule and fires notifications when conditions are met.

**Step 1: Configure notification channels**

Go to **Alerting** → **Notification channels** → **Add channel**:

- **Email:** Configure SMTP in `/etc/grafana/grafana.ini` under `[smtp]`
- **Slack:** Add an incoming webhook URL from your Slack workspace
- **PagerDuty:** Add your PagerDuty integration key for critical alerts

```ini
# /etc/grafana/grafana.ini — SMTP configuration
[smtp]
enabled = true
host = smtp.yourprovider.com:587
user = grafana@yourcompany.com
password = your_smtp_password
from_address = grafana@yourcompany.com
from_name = VICIdial Monitoring
```

Restart Grafana after editing: `sudo systemctl restart grafana-server`

**Step 2: Create alert rules**

### Alert: Drop Rate Exceeds 2.5%

This is the compliance-critical alert. If the 30-day rolling drop rate for any campaign exceeds 2.5%, someone needs to act immediately.

```yaml
Alert name: Drop Rate Warning
Evaluate every: 5m
For: 0m (fire immediately)

Query:
  SELECT
      ROUND(
          COUNT(CASE WHEN status = 'DROP' THEN 1 END) /
          NULLIF(COUNT(CASE WHEN status NOT IN ('NA','B','DC','N','AFTHRS','XDROP') THEN 1 END), 0) * 100, 2
      ) as drop_rate
  FROM vicidial_log
  WHERE call_date >= DATE_SUB(NOW(), INTERVAL 30 DAY)
      AND campaign_id = 'YOUR_CAMPAIGN'

Condition: WHEN last() OF query IS ABOVE 2.5
Notification: Send to [compliance-team-slack-channel]
Message: "URGENT: Campaign YOUR_CAMPAIGN 30-day drop rate has exceeded 2.5%. Current rate: {{ $value }}%. Reduce dial level immediately."
```

### Alert: Hopper Running Low

```yaml
Alert name: Hopper Low
Evaluate every: 1m
For: 2m (allow brief dips during list loading)

Query:
  SELECT COUNT(*) as hopper_count
  FROM vicidial_hopper
  WHERE campaign_id = 'YOUR_CAMPAIGN'

Condition: WHEN last() OF query IS BELOW 50
Notification: Send to [operations-slack-channel]
Message: "WARNING: Campaign YOUR_CAMPAIGN hopper is at {{ $value }} leads. Dialer will stall soon. Load more leads or check list/filter settings."
```

### Alert: Trunk Capacity Critical

```yaml
Alert name: Trunk Capacity
Evaluate every: 30s
For: 1m

Query:
  SELECT COUNT(*) as active_channels
  FROM vicidial_auto_calls

Condition: WHEN last() OF query IS ABOVE 255 (set to 85% of your trunk limit)
Notification: Send to [infrastructure-slack-channel]
Message: "WARNING: Active trunk count is {{ $value }}. Approaching SIP trunk capacity. New calls may fail with 503 errors."
```

### Alert: Agent Idle for Extended Period

This one catches agents who are logged in but not taking calls:

```yaml
Alert name: Agent Extended Idle
Evaluate every: 2m
For: 5m

Query:
  SELECT
      COUNT(*) as idle_agents
  FROM vicidial_live_agents
  WHERE status = 'PAUSED'
      AND TIMESTAMPDIFF(MINUTE, last_state_change, NOW()) > 10
      AND pause_code NOT IN ('BREAK','LUNCH','TRAIN')

Condition: WHEN last() OF query IS ABOVE 3
Notification: Send to [supervisor-slack-channel]
Message: "{{ $value }} agents have been paused for over 10 minutes (excluding breaks). Check for system issues or unauthorized pausing."
```

### Cron-Based Alerting (Non-Grafana)

If you prefer not to use Grafana's alerting engine, shell scripts via cron are equally effective and simpler to maintain:

```bash
#!/bin/bash
# /usr/local/bin/vicidial_alerts.sh
# Cron: */5 * * * * /usr/local/bin/vicidial_alerts.sh

DB_USER="grafana_ro"
DB_PASS="your_password"
DB_NAME="asterisk"
SLACK_WEBHOOK="https://hooks.slack.com/services/YOUR/WEBHOOK/URL"

# Alert 1: Drop rate check
CAMPAIGNS_OVER_LIMIT=$(mysql -u $DB_USER -p"$DB_PASS" $DB_NAME -N -e "
SELECT campaign_id, ROUND(COUNT(CASE WHEN status = 'DROP' THEN 1 END) /
    NULLIF(COUNT(CASE WHEN status NOT IN ('NA','B','DC','N','AFTHRS','XDROP') THEN 1 END), 0) * 100, 2)
    as drop_rate
FROM vicidial_log
WHERE call_date >= DATE_SUB(NOW(), INTERVAL 30 DAY)
GROUP BY campaign_id
HAVING drop_rate > 2.5;
")

if [ -n "$CAMPAIGNS_OVER_LIMIT" ]; then
    curl -s -X POST -H 'Content-type: application/json' \
        --data "{\"text\":\"ALERT: Campaigns exceeding 2.5% drop rate:\n\`\`\`${CAMPAIGNS_OVER_LIMIT}\`\`\`\"}" \
        "$SLACK_WEBHOOK"
fi

# Alert 2: Hopper check
LOW_HOPPERS=$(mysql -u $DB_USER -p"$DB_PASS" $DB_NAME -N -e "
SELECT campaign_id, COUNT(*) as hopper_count
FROM vicidial_hopper
GROUP BY campaign_id
HAVING hopper_count < 50;
")

if [ -n "$LOW_HOPPERS" ]; then
    curl -s -X POST -H 'Content-type: application/json' \
        --data "{\"text\":\"WARNING: Low hopper count:\n\`\`\`${LOW_HOPPERS}\`\`\`\"}" \
        "$SLACK_WEBHOOK"
fi

# Alert 3: Trunk capacity
TRUNK_COUNT=$(mysql -u $DB_USER -p"$DB_PASS" $DB_NAME -N -e "
SELECT COUNT(*) FROM vicidial_auto_calls;
")

if [ "$TRUNK_COUNT" -gt 255 ]; then
    curl -s -X POST -H 'Content-type: application/json' \
        --data "{\"text\":\"CRITICAL: Active trunk count is ${TRUNK_COUNT}. Approaching capacity limit.\"}" \
        "$SLACK_WEBHOOK"
fi
```

This script is dead simple, runs every 5 minutes via cron, and sends Slack notifications for three critical conditions. No Grafana required. Adjust the thresholds and add additional checks as needed for your operation.

---

## Performance Considerations for Dashboard Queries

Running real-time dashboards against a production VICIdial database adds query load. Here's how to avoid impacting your dialing operations.

### Use a MySQL Read Replica

For operations handling more than 50,000 calls per day, point Grafana and Metabase at a MySQL read replica instead of the production master. VICIdial's master database handles real-time dialer operations — adding heavy reporting queries to that workload can cause lock contention and slow down the dialer.

Setting up MySQL replication:

```bash
# On the master (VICIdial production server)
# Add to /etc/my.cnf under [mysqld]:
server-id = 1
log-bin = mysql-bin
binlog-do-db = asterisk
binlog_format = ROW

# Create replication user
mysql -e "CREATE USER 'replicator'@'REPLICA_IP' IDENTIFIED BY 'strong_password';"
mysql -e "GRANT REPLICATION SLAVE ON *.* TO 'replicator'@'REPLICA_IP';"
mysql -e "FLUSH PRIVILEGES;"
```

```bash
# On the replica (dashboard/reporting server)
# Add to /etc/my.cnf under [mysqld]:
server-id = 2
relay-log = relay-bin
read-only = 1
replicate-do-db = asterisk

# Configure and start replication
mysql -e "CHANGE MASTER TO MASTER_HOST='MASTER_IP', MASTER_USER='replicator',
          MASTER_PASSWORD='strong_password', MASTER_LOG_FILE='mysql-bin.000001',
          MASTER_LOG_POS=0;"
mysql -e "START SLAVE;"
mysql -e "SHOW SLAVE STATUS\G"
```

Point your Grafana and Metabase data sources at the replica. Your reporting queries now run on a separate server with zero impact on production dialing. See our [VICIdial MySQL Optimization](/blog/vicidial-mysql-optimization/) guide for complete replication and tuning details.

### Index Optimization for Dashboard Queries

VICIdial's default schema includes indexes on the most common query columns, but dashboard queries often hit different patterns. Verify these indexes exist:

```sql
-- Check existing indexes on key tables
SHOW INDEX FROM vicidial_log;
SHOW INDEX FROM vicidial_agent_log;
SHOW INDEX FROM vicidial_closer_log;

-- Add indexes if missing (run on replica, not production, during low-traffic periods)
ALTER TABLE vicidial_log ADD INDEX idx_calldate_campaign (call_date, campaign_id);
ALTER TABLE vicidial_log ADD INDEX idx_campaign_status (campaign_id, status);
ALTER TABLE vicidial_agent_log ADD INDEX idx_eventtime_campaign (event_time, campaign_id);
ALTER TABLE vicidial_closer_log ADD INDEX idx_calldate_campaign (call_date, campaign_id);
```

**Warning:** Adding indexes to production tables on a busy VICIdial system can cause temporary table locks. Run `ALTER TABLE` commands during off-hours or on the replica only.

### Query Optimization Tips

1. **Always filter by date.** Never run a dashboard query without a `WHERE call_date >= ...` clause. Scanning the entire `vicidial_log` table (which can have hundreds of millions of rows) will lock the server.

2. **Use `CURDATE()` for today's data.** Grafana's `$__timeFilter` macro handles this, but for custom queries, `WHERE call_date >= CURDATE()` is the fastest filter for "today" panels.

3. **Limit real-time queries to live tables.** For 5-10 second refresh panels, query `vicidial_live_agents` and `vicidial_auto_calls` (which are small, constantly-updated tables), not the log tables (which are large and grow continuously).

4. **Aggregate in the database, not in Grafana.** Do all your `GROUP BY`, `ROUND`, and `COUNT` operations in SQL. Pulling raw rows into Grafana and transforming them client-side is dramatically slower.

5. **Set appropriate refresh intervals.** Not every panel needs a 5-second refresh:
   - Agent status, hopper, trunks: 5-10 seconds
   - Calls per hour, conversion rates: 30-60 seconds
   - Daily summaries, leaderboards: 30-60 seconds
   - Historical heatmaps, DID analysis: 300 seconds (5 minutes)

---

## Wallboard Layout Best Practices

Building great panels is half the battle. Arranging them into a wallboard that's useful at a glance — readable from 15 feet away on a TV screen — is the other half.

### Recommended Wallboard Layout

**Top row (critical compliance and capacity):**
- Drop Rate Gauge (large) | Connect Rate Gauge | Hopper Status | Active Trunks

**Middle row (real-time operations):**
- Agent Status Pie Chart | Calls Per Hour Time Series (wide)

**Bottom row (performance):**
- Agent Leaderboard Table (full width)

### Grafana Kiosk Mode

For TV/wallboard displays, use Grafana's kiosk mode to hide the header, sidebar, and controls:

```
http://your-server:3000/d/YOUR_DASHBOARD_UID?orgId=1&refresh=30s&kiosk
```

The `&kiosk` parameter removes all Grafana chrome, leaving only the panels. Add `&kiosk=tv` for a mode that cycles through dashboard pages if you have multiple.

**Auto-login for wallboard displays:** Create a Grafana service account with Viewer permissions, generate a service account token, and use it to authenticate the wallboard browser:

```
http://your-server:3000/d/YOUR_DASHBOARD_UID?orgId=1&refresh=30s&kiosk&auth_token=YOUR_SERVICE_ACCOUNT_TOKEN
```

Or configure anonymous access in `grafana.ini` for dedicated wallboard networks:

```ini
[auth.anonymous]
enabled = true
org_name = Main Org.
org_role = Viewer
```

### Font Size and Readability

Grafana's default font sizes work well on a desktop monitor at arm's length, but they're unreadable on a TV across the room. For wallboard panels:

- **Stat panels:** Set font size to "200%" or higher in panel options
- **Gauge panels:** These scale well by default — just make them large enough (6+ grid units wide)
- **Table panels:** Limit to 5-7 columns maximum. More than that becomes unreadable from a distance
- **Time series:** Use thick line widths (2-3px) and distinct colors that are distinguishable from 15+ feet

> **Want a production-ready monitoring stack for your VICIdial operation without building it yourself?** Our team deploys pre-configured Grafana and Metabase dashboards tailored to your campaigns, agent count, and compliance requirements. [Get a free infrastructure audit](/free-audit/) and we'll scope the build for you.

---

## Integrating Dashboard Data with External Systems

Dashboards are powerful, but some operations need VICIdial data flowing into CRM systems, data warehouses, or external analytics platforms.

### Grafana API for Data Export

Grafana exposes a REST API for programmatic access to dashboard data:

```bash
# Export dashboard JSON
curl -H "Authorization: Bearer YOUR_API_KEY" \
  http://your-server:3000/api/dashboards/uid/YOUR_DASHBOARD_UID

# Query a specific panel's data
curl -H "Authorization: Bearer YOUR_API_KEY" \
  "http://your-server:3000/api/ds/query" \
  -d '{
    "queries": [{
      "datasourceId": 1,
      "rawSql": "SELECT campaign_id, COUNT(*) as dials FROM vicidial_log WHERE call_date >= CURDATE() GROUP BY campaign_id",
      "format": "table"
    }]
  }'
```

### Metabase API for Automated Reporting

Metabase's API lets you programmatically run saved questions and export results:

```bash
# Authenticate
TOKEN=$(curl -s -X POST http://your-server:3001/api/session \
  -H "Content-Type: application/json" \
  -d '{"username":"admin@company.com","password":"your_password"}' \
  | jq -r '.id')

# Run a saved question and get CSV results
curl -H "X-Metabase-Session: $TOKEN" \
  "http://your-server:3001/api/card/1/query/csv" > report.csv
```

This is useful for automated data pipelines — pull VICIdial reporting data from Metabase on a schedule and push it to your data warehouse, CRM, or external reporting platform.

### Pushing VICIdial Metrics to Prometheus/InfluxDB

For operations running a full observability stack (Prometheus + Grafana, or InfluxDB + Grafana), you can write a lightweight exporter that queries VICIdial's MySQL database and exposes metrics:

```bash
#!/bin/bash
# /usr/local/bin/vicidial_metrics_exporter.sh
# Run via cron every minute, writes metrics to a file for node_exporter textfile collector

METRICS_DIR="/var/lib/node_exporter/textfile_collector"
DB_USER="grafana_ro"
DB_PASS="your_password"
DB_NAME="asterisk"

# Active agents by status
mysql -u $DB_USER -p"$DB_PASS" $DB_NAME -N -e "
SELECT status, COUNT(*) FROM vicidial_live_agents GROUP BY status;
" | while read status count; do
    echo "vicidial_agents_by_status{status=\"${status}\"} ${count}"
done > "$METRICS_DIR/vicidial_agents.prom.$$"
mv "$METRICS_DIR/vicidial_agents.prom.$$" "$METRICS_DIR/vicidial_agents.prom"

# Active calls
ACTIVE_CALLS=$(mysql -u $DB_USER -p"$DB_PASS" $DB_NAME -N -e "SELECT COUNT(*) FROM vicidial_auto_calls;")
echo "vicidial_active_calls ${ACTIVE_CALLS}" > "$METRICS_DIR/vicidial_calls.prom.$$"
mv "$METRICS_DIR/vicidial_calls.prom.$$" "$METRICS_DIR/vicidial_calls.prom"

# Hopper counts
mysql -u $DB_USER -p"$DB_PASS" $DB_NAME -N -e "
SELECT campaign_id, COUNT(*) FROM vicidial_hopper GROUP BY campaign_id;
" | while read campaign count; do
    echo "vicidial_hopper_count{campaign=\"${campaign}\"} ${count}"
done > "$METRICS_DIR/vicidial_hopper.prom.$$"
mv "$METRICS_DIR/vicidial_hopper.prom.$$" "$METRICS_DIR/vicidial_hopper.prom"
```

This approach lets you unify VICIdial metrics with server metrics (CPU, memory, disk, network) in a single Grafana dashboard, giving you full-stack visibility from SIP trunks down to disk I/O.

---

## Troubleshooting Common Dashboard Issues

### "No Data" in Grafana Panels

The most common issue when setting up Grafana with VICIdial. Check these in order:

1. **Data source connection:** Go to Data Sources → VICIdial MySQL → Test. If it fails, verify MySQL host, port, credentials, and firewall.

2. **Time range mismatch:** Grafana's time picker might be set to "Last 6 hours" but your query uses `WHERE call_date >= CURDATE()`. Make sure your queries use Grafana's `$__timeFilter()` macro for time-dependent panels.

3. **Campaign filter empty:** If you're using the `$campaign_id` variable and it's set to "All" with `$__all`, the `IN ($campaign_id)` clause might not expand correctly. Test with a hardcoded campaign ID first.

4. **Empty database tables:** If you just installed VICIdial or are in a testing environment, the log tables may be empty. Run `SELECT COUNT(*) FROM vicidial_log;` directly on MySQL to verify data exists.

5. **Time series format:** Time series panels require the first column to be a time column. If your query returns a table format, switch the query format dropdown from "Time series" to "Table" (or vice versa).

### Slow Dashboard Loading

If panels take more than 2-3 seconds to load:

1. **Check query execution time:** Run the panel's SQL query directly in MySQL with `EXPLAIN` to see if it's doing a full table scan:

```sql
EXPLAIN SELECT COUNT(*) FROM vicidial_log
WHERE call_date >= DATE_SUB(NOW(), INTERVAL 30 DAY)
    AND campaign_id = 'SALES01';
```

If you see `type: ALL` (full table scan), you need an index on `(call_date, campaign_id)`.

2. **Reduce time range:** A 30-day time series at 1-hour granularity generates 720 data points, which is fine. But a 30-day range at 1-minute granularity generates 43,200 data points — unnecessary and slow.

3. **Use the read replica:** Move dashboard queries off the production master.

4. **Cache results:** In Grafana panel settings, set "Min interval" to prevent the panel from querying more frequently than needed.

### Grafana Running Out of Memory

Grafana defaults to a modest memory allocation. For dashboards with many panels and frequent refreshes, increase the memory limit:

```ini
# /etc/grafana/grafana.ini
[server]
# Increase if Grafana becomes unresponsive with many panels
router_logging = false

# /etc/sysconfig/grafana-server (environment overrides)
GRAFANA_MEMORY_LIMIT=512m
```

For systemd-managed installations, add a memory limit override:

```bash
sudo systemctl edit grafana-server
```

```ini
[Service]
MemoryMax=1G
```

---

## Security Hardening

Running dashboards that display call center data — including potentially sensitive lead information and operational metrics — requires proper security.

### Network Segmentation

- **Grafana and Metabase should not be internet-accessible.** Run them on an internal network or behind a VPN. If remote access is required, put them behind a reverse proxy (nginx, Apache) with TLS and authentication.
- **MySQL access should be restricted.** The `grafana_ro` user should only be accessible from the Grafana/Metabase server IP, not from any host.

### TLS for Grafana

```bash
# Generate or install TLS certificates
# If using Let's Encrypt with a reverse proxy:
sudo dnf install -y nginx certbot python3-certbot-nginx

# Nginx reverse proxy config for Grafana
cat <<'EOF' | sudo tee /etc/nginx/conf.d/grafana.conf
server {
    listen 443 ssl;
    server_name grafana.internal.yourcompany.com;

    ssl_certificate /etc/letsencrypt/live/grafana.internal.yourcompany.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/grafana.internal.yourcompany.com/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket support for live dashboards
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
EOF

sudo systemctl enable --now nginx
```

### Role-Based Access in Grafana

Create organizations and roles to control who sees what:

- **Viewers:** Supervisors and managers who need to see dashboards but not edit them
- **Editors:** Operations managers who can modify dashboards and create new panels
- **Admins:** IT staff who manage data sources, users, and system configuration

For Metabase, use Collections and Groups to control data access. Create a "Floor Supervisors" group that can see agent performance and campaign metrics but cannot access the `vicidial_list` table (which contains PII).

---

## Frequently Asked Questions

### Can I run Grafana on the same server as VICIdial?

Yes, but with caveats. Grafana itself uses minimal resources (a few hundred MB of RAM, negligible CPU). The concern is the MySQL queries it runs. On a lightly loaded system (under 30 agents, under 20,000 calls/day), running Grafana on the VICIdial web server is perfectly fine. For larger operations, install Grafana on a separate monitoring server and point it at a MySQL read replica. The dialer servers (running Asterisk) should never run anything non-essential — every CPU cycle matters for real-time call processing.

### How often should dashboard panels refresh?

It depends on the panel's purpose. Real-time agent status panels should refresh every 5-10 seconds since supervisor decisions depend on current state. Calls-per-hour time series and conversion rate gauges can refresh every 30-60 seconds — these metrics don't change meaningfully second-to-second. Historical panels like heatmaps and DID analysis should refresh every 5 minutes at most. Over-refreshing wastes database resources and provides no actionable benefit. The golden rule: refresh at the cadence someone would actually look at the panel and make a decision.

### What's the difference between Grafana and Metabase for VICIdial?

Grafana excels at real-time operational dashboards — auto-refreshing panels on wallboards, threshold-based color coding, and alerting. It's designed for the operations team watching metrics second-by-second. Metabase excels at self-service analytics — managers asking ad-hoc questions about the data, building their own reports without SQL, and scheduling automated email delivery of reports. Most VICIdial operations benefit from running both: Grafana on the wallboard for real-time monitoring, Metabase for everything else. If you can only pick one and your primary need is real-time wallboards, choose Grafana. If your primary need is giving managers self-service reporting, choose Metabase.

### How do I display Grafana dashboards on a TV wallboard?

Use Grafana's kiosk mode by appending `?kiosk` to the dashboard URL: `http://your-server:3000/d/DASHBOARD_UID?kiosk&refresh=30s`. This removes the header, sidebar, and all navigation, leaving only the panels. For auto-login on dedicated wallboard displays, configure either anonymous access in `grafana.ini` or use a service account token in the URL. Use a Chromium browser in full-screen mode on the TV's connected device (a Raspberry Pi works perfectly). Set the browser to auto-start on boot with the kiosk URL, and configure the system to disable screen blanking.

### Do these dashboards work with VICIdial clusters?

Yes, seamlessly. In a VICIdial [cluster deployment](/blog/vicidial-cluster-guide/), all operational data (vicidial_log, vicidial_agent_log, vicidial_live_agents, etc.) is stored in the centralized MySQL database on the master server. Grafana and Metabase connect to that single database and see the complete picture across all dialer servers. The `server_ip` column in tables like `vicidial_live_agents` and `vicidial_auto_calls` identifies which dialer server is handling each agent or call, so you can build per-server panels if needed. No special cluster-specific configuration is required for the dashboards — the data is already unified in one database.

### How do I alert on specific VICIdial events in real-time?

Grafana's built-in alerting evaluates queries on a configurable schedule (as frequently as every 10 seconds for Grafana Alerting, or every 1 minute for legacy alerting). Create an alert rule with a query that checks your condition (drop rate > 2.5%, hopper < 50, trunk utilization > 85%), set the evaluation interval, and configure a notification channel (email, Slack, PagerDuty, webhook). For simpler setups, a cron script running every 1-5 minutes that queries MySQL and sends Slack webhook notifications is equally effective and easier to maintain. The cron approach is covered in detail earlier in this guide with ready-to-use scripts.

### What VICIdial tables should I index for dashboard performance?

The most impactful indexes for dashboard queries are compound indexes on the columns you filter and group by most often. The critical ones: `(call_date, campaign_id)` on `vicidial_log`, `(event_time, campaign_id)` on `vicidial_agent_log`, and `(call_date, campaign_id)` on `vicidial_closer_log`. VICIdial's default schema includes basic indexes but may not have these compound indexes. Check with `SHOW INDEX FROM table_name` before adding new ones. Always add indexes during off-hours or on a read replica, as `ALTER TABLE` on a large table can temporarily lock it and disrupt live dialing operations. Our [MySQL optimization guide](/blog/vicidial-mysql-optimization/) covers indexing strategy in depth.

### Can I embed Grafana or Metabase dashboards in other web applications?

Yes, both tools support embedding. Grafana supports iframe embedding — enable it in `grafana.ini` under `[security]` by setting `allow_embedding = true`, then embed individual panels or entire dashboards in any web application using iframes. Metabase supports a more advanced embedding model with signed embed URLs that control which filters are locked, editable, or hidden. This is particularly useful for multi-tenant operations or client-facing portals where you want to show each client their campaign's data without exposing other campaigns. Both tools also offer public dashboard links (shareable URLs without authentication) for situations where that level of access is acceptable.

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-grafana-dashboards).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
