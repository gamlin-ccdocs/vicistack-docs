# VICIdial Real-Time Agent Dashboard Customization Guide

If you're running a call center with 25 or more agents on VICIdial, the default real-time report is the screen you stare at more than any other. It tells you who's talking, who's paused, who's waiting, and — if you know how to read between the lines — where your operation is bleeding money. But the stock interface only scratches the surface of what's possible.

This guide walks through building custom real-time dashboards that give supervisors the data they actually need, from wallboard displays to Grafana-powered visualizations pulling directly from the VICIdial MySQL backend.

## The Default Real-Time Report: What It Does and Where It Falls Short

The built-in real-time report at `vicidial/realtime_report.php` provides a snapshot of agent activity across campaigns. Out of the box, you see:

- **Agent name and extension** — who's logged in
- **Status** — INCALL, READY, PAUSED, DEAD, etc.
- **Campaign** — which campaign the agent is assigned to
- **Calls today** — raw call count
- **Talk time** — cumulative seconds in conversation
- **Pause time** — cumulative seconds in pause status
- **Wait time** — cumulative time in READY waiting for a call

For a 10-seat operation, this is workable. You can eyeball the screen, spot the agent who's been paused for 20 minutes, and walk over. But at 25+ agents, problems emerge fast.

### Limitation 1: No At-a-Glance KPI Summary

The default report shows individual agent rows but doesn't aggregate. You can't glance at the screen and instantly know your average talk time, total calls per hour, or how many agents are productive right now. You have to mentally tally rows while the data refreshes underneath you.

### Limitation 2: Flat Table Layout

Every agent occupies one row regardless of status. In a 50-agent room, your INCALL agents (the ones making money) are mixed in with PAUSED and DEAD agents. There's no visual grouping, no color-coded priority, no separation of "healthy" from "needs attention."

### Limitation 3: Refresh Behavior

The default page uses a meta-refresh tag that reloads the entire page at a fixed interval. This causes a visible flash, resets scroll position, and provides no smooth transition. On a wallboard TV, this flicker is distracting and looks unprofessional.

### Limitation 4: No Historical Context

The real-time report is purely instantaneous. It can't show you trends — whether an agent's calls per hour are declining through their shift, or whether a campaign's wait time has been climbing for the past 30 minutes.

## Building Custom Real-Time Frames

VICIdial's admin interface supports custom real-time frames through the `realtime_report.php` file's URL parameters and through entirely custom PHP pages that query the same backend tables.

### URL Parameter Customization

The built-in report accepts several parameters that many admins never discover:

```
/vicidial/realtime_report.php?DB=0&group=SALESCAMP&RR=4&Session=your_session
```

Key parameters:

| Parameter | Purpose | Example |
|-----------|---------|---------|
| `group` | Filter to a specific campaign | `group=SALESCAMP` |
| `RR` | Refresh interval in seconds | `RR=4` |
| `DB` | Debug mode (0=off, 1=on) | `DB=0` |
| `show_parks` | Show parked calls | `show_parks=1` |
| `NOLOGin` | Use without agent login | `NOLOGin=Y` |

### Custom PHP Dashboard Pages

For true customization, create a new PHP file in the VICIdial web directory that queries the backend directly. Here's the skeleton of a custom real-time dashboard:

```php
<?php
// custom_wallboard.php — place in /var/www/html/vicidial/
require_once("dbconnect_mysqli.php");

// Authenticate via existing VICIdial session or IP whitelist
$allowed_ips = array('10.0.1.50', '10.0.1.51'); // supervisor stations
if (!in_array($_SERVER['REMOTE_ADDR'], $allowed_ips)) {
    die("Access denied");
}

// Query live agent data
$query = "SELECT
    va.user,
    vu.full_name,
    va.status,
    va.campaign_id,
    va.callerid,
    va.calls_today,
    UNIX_TIMESTAMP(NOW()) - UNIX_TIMESTAMP(va.last_state_change) AS seconds_in_state
FROM vicidial_live_agents va
JOIN vicidial_users vu ON va.user = vu.user
WHERE va.campaign_id IN ('SALESCAMP','RETENTION')
ORDER BY va.status, vu.full_name";

$result = mysqli_query($link, $query);

// Build agent arrays grouped by status
$agents_incall = [];
$agents_ready = [];
$agents_paused = [];

while ($row = mysqli_fetch_assoc($result)) {
    switch ($row['status']) {
        case 'INCALL':
            $agents_incall[] = $row;
            break;
        case 'READY':
            $agents_ready[] = $row;
            break;
        case 'PAUSED':
            $agents_paused[] = $row;
            break;
    }
}
?>
```

This gives you full control over layout, grouping, and styling. The key table is `vicidial_live_agents`, which VICIdial updates in real time as agent states change.

## Key Metrics to Display on Your Wallboard

Not every metric deserves wallboard space. On a TV visible to agents and supervisors, focus on numbers that drive behavior.

### 1. Calls Per Hour (CPH) — By Agent and Campaign Average

Calls per hour is the single most actionable metric for outbound campaigns. Calculate it from `vicidial_agent_log`:

```sql
SELECT
    user,
    COUNT(*) AS calls,
    ROUND(COUNT(*) / (TIMESTAMPDIFF(MINUTE, MIN(event_time), NOW()) / 60), 1) AS cph
FROM vicidial_agent_log
WHERE event_time >= CURDATE()
  AND campaign_id = 'SALESCAMP'
  AND talk_sec > 0
GROUP BY user
ORDER BY cph DESC;
```

Display the campaign average prominently. Agents self-correct when they can see how they stack up against the group average.

### 2. Talk Time vs. Pause Time Ratio

This ratio instantly reveals productivity. An agent with 3 hours of talk time and 45 minutes of pause is performing very differently from one with 1 hour of talk and 2.5 hours of pause.

```sql
SELECT
    vla.user,
    vu.full_name,
    val_sum.talk_seconds,
    val_sum.pause_seconds,
    ROUND(val_sum.talk_seconds / NULLIF(val_sum.pause_seconds, 0), 2) AS talk_pause_ratio
FROM vicidial_live_agents vla
JOIN vicidial_users vu ON vla.user = vu.user
LEFT JOIN (
    SELECT user, SUM(talk_sec) AS talk_seconds, SUM(pause_sec) AS pause_seconds
    FROM vicidial_agent_log
    WHERE event_time >= CURDATE()
    GROUP BY user
) val_sum ON vla.user = val_sum.user
WHERE vla.campaign_id = 'SALESCAMP'
ORDER BY talk_pause_ratio DESC;
```

### 3. Wait Time (Time in READY)

High wait time means agents are sitting idle waiting for calls. This is either a list exhaustion problem, a dial level problem, or a trunk capacity problem. Show this prominently so supervisors investigate immediately when it climbs.

```sql
SELECT
    vla.campaign_id,
    ROUND(AVG(UNIX_TIMESTAMP(NOW()) - UNIX_TIMESTAMP(vla.last_state_change)), 0) AS avg_wait_seconds,
    MAX(UNIX_TIMESTAMP(NOW()) - UNIX_TIMESTAMP(vla.last_state_change)) AS max_wait_seconds,
    COUNT(*) AS agents_in_campaign
FROM vicidial_live_agents vla
WHERE vla.status = 'READY'
GROUP BY vla.campaign_id;
```

### 4. Active Pause Codes

Don't just show that agents are paused — show why. The `pause_code` field in `vicidial_live_agents` contains the active pause code:

```sql
SELECT
    va.user,
    vu.full_name,
    va.pause_code,
    vpc.pause_code_name,
    UNIX_TIMESTAMP(NOW()) - UNIX_TIMESTAMP(va.last_state_change) AS seconds_paused
FROM vicidial_live_agents va
JOIN vicidial_users vu ON va.user = vu.user
LEFT JOIN vicidial_pause_codes vpc
    ON va.pause_code = vpc.pause_code
    AND va.campaign_id = vpc.campaign_id
WHERE va.status = 'PAUSED'
ORDER BY seconds_paused DESC;
```

This lets supervisors differentiate between an agent on a legitimate break (pause code `LUNCH`) and one who's been in `BREAK` for 45 minutes.

### 5. Calls in Queue / Drops in Last 30 Minutes

For inbound blended campaigns, show the current queue depth. For outbound, show recent drops — this is a compliance number that should always be visible:

```sql
-- Drops in last 30 minutes
SELECT
    campaign_id,
    COUNT(*) AS drops_30min
FROM vicidial_log
WHERE call_date >= DATE_SUB(NOW(), INTERVAL 30 MINUTE)
  AND status = 'DROP'
GROUP BY campaign_id;
```

## AJAX-Based Auto-Refresh Configuration

Replace the jarring full-page meta-refresh with smooth AJAX updates. This approach fetches only the data (as JSON) and updates the DOM without a page reload.

### Backend: JSON Data Endpoint

Create a PHP file that returns agent data as JSON:

```php
<?php
// wallboard_data.php
header('Content-Type: application/json');
require_once("dbconnect_mysqli.php");

$campaign = mysqli_real_escape_string($link, $_GET['campaign'] ?? 'SALESCAMP');

$query = "SELECT
    va.user,
    vu.full_name,
    va.status,
    va.pause_code AS sub_status,
    va.calls_today,
    UNIX_TIMESTAMP(NOW()) - UNIX_TIMESTAMP(va.last_state_change) AS seconds_in_state,
    va.campaign_id
FROM vicidial_live_agents va
JOIN vicidial_users vu ON va.user = vu.user
WHERE va.campaign_id = '$campaign'
ORDER BY va.status, vu.full_name";

$result = mysqli_query($link, $query);
$agents = [];
while ($row = mysqli_fetch_assoc($result)) {
    $agents[] = $row;
}

// Add campaign-level summary
$summary_query = "SELECT
    COUNT(*) AS total_agents,
    SUM(CASE WHEN status='INCALL' THEN 1 ELSE 0 END) AS agents_incall,
    SUM(CASE WHEN status='READY' THEN 1 ELSE 0 END) AS agents_ready,
    SUM(CASE WHEN status='PAUSED' THEN 1 ELSE 0 END) AS agents_paused,
    ROUND(AVG(calls_today), 1) AS avg_calls
FROM vicidial_live_agents
WHERE campaign_id = '$campaign'";

$summary_result = mysqli_query($link, $summary_query);
$summary = mysqli_fetch_assoc($summary_result);

echo json_encode([
    'timestamp' => time(),
    'campaign' => $campaign,
    'summary' => $summary,
    'agents' => $agents
]);
?>
```

### Frontend: AJAX Polling

```html
<!DOCTYPE html>
<html>
<head>
    <title>VICIdial Wallboard</title>
    <style>
        body {
            background: #1a1a2e;
            color: #eee;
            font-family: 'Courier New', monospace;
            margin: 0;
            padding: 20px;
        }
        .summary-bar {
            display: flex;
            justify-content: space-around;
            padding: 20px;
            background: #16213e;
            border-radius: 8px;
            margin-bottom: 20px;
            font-size: 1.4em;
        }
        .summary-item { text-align: center; }
        .summary-value { font-size: 2em; font-weight: bold; }
        .incall .summary-value { color: #00ff88; }
        .paused .summary-value { color: #ff6b6b; }
        .ready .summary-value { color: #4ecdc4; }

        .agent-grid {
            display: grid;
            grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
            gap: 10px;
        }
        .agent-card {
            padding: 15px;
            border-radius: 6px;
            transition: all 0.3s ease;
        }
        .agent-card.INCALL { background: #0a3d1f; border-left: 4px solid #00ff88; }
        .agent-card.PAUSED { background: #3d0a0a; border-left: 4px solid #ff6b6b; }
        .agent-card.READY  { background: #0a2d3d; border-left: 4px solid #4ecdc4; }
    </style>
</head>
<body>
    <div class="summary-bar" id="summary"></div>
    <div class="agent-grid" id="agents"></div>

    <script>
    const CAMPAIGN = 'SALESCAMP';
    const REFRESH_MS = 4000; // 4 seconds

    function formatTime(seconds) {
        const m = Math.floor(seconds / 60);
        const s = seconds % 60;
        return m + ':' + String(s).padStart(2, '0');
    }

    async function fetchData() {
        try {
            const resp = await fetch(`wallboard_data.php?campaign=${CAMPAIGN}`);
            const data = await resp.json();
            renderSummary(data.summary);
            renderAgents(data.agents);
        } catch (e) {
            console.error('Fetch failed:', e);
        }
    }

    function renderSummary(s) {
        document.getElementById('summary').innerHTML = `
            <div class="summary-item incall">
                <div>In Call</div>
                <div class="summary-value">${s.agents_incall}</div>
            </div>
            <div class="summary-item ready">
                <div>Ready</div>
                <div class="summary-value">${s.agents_ready}</div>
            </div>
            <div class="summary-item paused">
                <div>Paused</div>
                <div class="summary-value">${s.agents_paused}</div>
            </div>
            <div class="summary-item">
                <div>Avg Calls</div>
                <div class="summary-value">${s.avg_calls}</div>
            </div>
        `;
    }

    function renderAgents(agents) {
        const html = agents.map(a => `
            <div class="agent-card ${a.status}">
                <strong>${a.full_name}</strong><br>
                ${a.status}${a.sub_status ? ' (' + a.sub_status + ')' : ''}<br>
                In state: ${formatTime(parseInt(a.seconds_in_state))}<br>
                Calls: ${a.calls_today}
            </div>
        `).join('');
        document.getElementById('agents').innerHTML = html;
    }

    fetchData();
    setInterval(fetchData, REFRESH_MS);
    </script>
</body>
</html>
```

The AJAX approach gives you flicker-free updates, smooth CSS transitions when agent states change, and no scroll-position reset. Set `REFRESH_MS` to 3000-5000 for a good balance between responsiveness and database load.

## MySQL Queries for Live Dashboard Data

Beyond the basic queries above, here are advanced queries that power production wallboards.

### Campaign Performance Summary — Live

```sql
SELECT
    vla.campaign_id,
    COUNT(DISTINCT vla.user) AS logged_in_agents,
    SUM(CASE WHEN vla.status = 'INCALL' THEN 1 ELSE 0 END) AS on_call,
    (SELECT COUNT(*)
     FROM vicidial_log vl
     WHERE vl.campaign_id = vla.campaign_id
       AND vl.call_date >= CURDATE()) AS calls_today,
    (SELECT COUNT(*)
     FROM vicidial_log vl
     WHERE vl.campaign_id = vla.campaign_id
       AND vl.call_date >= DATE_SUB(NOW(), INTERVAL 1 HOUR)) AS calls_last_hour,
    (SELECT COUNT(*)
     FROM vicidial_log vl
     WHERE vl.campaign_id = vla.campaign_id
       AND vl.call_date >= CURDATE()
       AND vl.status = 'DROP') AS drops_today
FROM vicidial_live_agents vla
GROUP BY vla.campaign_id;
```

### Agent Leaderboard — Today

```sql
SELECT
    val.user,
    vu.full_name AS agent_name,
    COUNT(*) AS total_calls,
    SUM(CASE WHEN val.status IN ('SALE','UPSELL') THEN 1 ELSE 0 END) AS conversions,
    ROUND(AVG(val.talk_sec), 0) AS avg_talk_sec,
    ROUND(SUM(val.talk_sec) / 3600, 2) AS talk_hours,
    ROUND(SUM(val.pause_sec) / 3600, 2) AS pause_hours
FROM vicidial_agent_log val
JOIN vicidial_users vu ON val.user = vu.user
WHERE val.event_time >= CURDATE()
  AND val.campaign_id = 'SALESCAMP'
GROUP BY val.user
ORDER BY conversions DESC, total_calls DESC;
```

### Trunk Utilization — Current

```sql
SELECT
    server_ip,
    COUNT(*) AS active_channels,
    (SELECT COUNT(*) FROM servers WHERE server_ip = va.server_ip) AS max_trunks
FROM vicidial_auto_calls va
WHERE status IN ('SENT','RINGING','LIVE','XFER')
GROUP BY server_ip;
```

### Hopper Status — Lead Supply Health

```sql
SELECT
    campaign_id,
    COUNT(*) AS leads_in_hopper,
    MIN(priority) AS min_priority,
    MAX(priority) AS max_priority
FROM vicidial_hopper
GROUP BY campaign_id;
```

A hopper count below 2x your agent count is a warning sign. If you have 30 agents in a campaign, you want at least 60 leads in the hopper for the adaptive dialer to work efficiently. Display this on your wallboard with a color-coded threshold.

## Integration with Grafana for Advanced Visualization

For operations that want historical trending, alerting, and multi-screen dashboards, Grafana with a direct MySQL datasource is the gold standard. Here's how to set it up without disrupting your VICIdial installation.

### Step 1: Create a Read-Only MySQL User

Never point Grafana at your VICIdial admin credentials. Create a dedicated read-only user:

```sql
CREATE USER 'grafana_ro'@'10.0.1.%' IDENTIFIED BY 'strong_random_password_here';

GRANT SELECT ON asterisk.vicidial_live_agents TO 'grafana_ro'@'10.0.1.%';
GRANT SELECT ON asterisk.vicidial_agent_log TO 'grafana_ro'@'10.0.1.%';
GRANT SELECT ON asterisk.vicidial_log TO 'grafana_ro'@'10.0.1.%';
GRANT SELECT ON asterisk.vicidial_closer_log TO 'grafana_ro'@'10.0.1.%';
GRANT SELECT ON asterisk.vicidial_auto_calls TO 'grafana_ro'@'10.0.1.%';
GRANT SELECT ON asterisk.vicidial_hopper TO 'grafana_ro'@'10.0.1.%';
GRANT SELECT ON asterisk.vicidial_pause_codes TO 'grafana_ro'@'10.0.1.%';
GRANT SELECT ON asterisk.vicidial_campaign_stats TO 'grafana_ro'@'10.0.1.%';

FLUSH PRIVILEGES;
```

### Step 2: Install Grafana on a Separate Box

Keep Grafana off your VICIdial server. Install it on a monitoring host:

```bash
sudo apt-get install -y apt-transport-https software-properties-common
sudo mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee /etc/apt/sources.list.d/grafana.list
sudo apt-get update && sudo apt-get install grafana
sudo systemctl enable grafana-server && sudo systemctl start grafana-server
```

### Step 3: Add the MySQL Datasource

In Grafana's web interface (default port 3000), go to Configuration > Data Sources > Add > MySQL:

- **Host:** `10.0.1.10:3306` (your VICIdial DB IP)
- **Database:** `asterisk`
- **User:** `grafana_ro`
- **TLS:** Enable if crossing network segments

### Step 4: Build Dashboard Panels

Here are Grafana-ready queries for the most valuable panels.

**Agents by Status Over Time (Time Series):**

```sql
SELECT
    $__timeGroup(event_time, '1m') AS time,
    SUM(CASE WHEN status = 'INCALL' THEN 1 ELSE 0 END) AS incall,
    SUM(CASE WHEN status = 'READY' THEN 1 ELSE 0 END) AS ready,
    SUM(CASE WHEN status = 'PAUSED' THEN 1 ELSE 0 END) AS paused
FROM vicidial_agent_log
WHERE $__timeFilter(event_time)
  AND campaign_id = 'SALESCAMP'
GROUP BY time
ORDER BY time;
```

**Calls Per Hour Trend (Time Series):**

```sql
SELECT
    $__timeGroup(call_date, '1h') AS time,
    COUNT(*) AS calls
FROM vicidial_log
WHERE $__timeFilter(call_date)
  AND campaign_id = 'SALESCAMP'
GROUP BY time
ORDER BY time;
```

**Drop Rate Gauge:**

```sql
SELECT
    ROUND(
        SUM(CASE WHEN status = 'DROP' THEN 1 ELSE 0 END) * 100.0 / COUNT(*),
        2
    ) AS drop_rate_pct
FROM vicidial_log
WHERE call_date >= CURDATE()
  AND campaign_id = 'SALESCAMP';
```

Set this gauge with thresholds: green below 2%, yellow at 2-3%, red above 3% (the FTC safe harbor limit).

### Step 5: Set Up Alerts

Grafana alerting can notify supervisors via Slack, email, or PagerDuty when:

- Drop rate exceeds 2.5%
- Hopper falls below threshold
- Average wait time exceeds 30 seconds
- More than 50% of agents are paused simultaneously

```yaml
# Example alert rule (Grafana alerting YAML)
groups:
  - name: vicidial-alerts
    rules:
      - alert: HighDropRate
        expr: drop_rate_pct > 2.5
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Drop rate above 2.5% for campaign {{ $labels.campaign_id }}"
      - alert: HopperLow
        expr: hopper_count < 50
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Hopper below 50 leads for {{ $labels.campaign_id }}"
```

## Wallboard Display Best Practices for 25+ Agent Rooms

When you have multiple screens on the call floor, organize them by purpose:

1. **Screen 1 — Agent Status Grid:** The card-based layout from the AJAX example above. Color-coded, large text, visible from across the room.
2. **Screen 2 — Leaderboard:** Top 10 agents by conversions and calls per hour. Rotates every 30 seconds between campaigns.
3. **Screen 3 — Grafana Trends:** Calls per hour trend, drop rate gauge, hopper health. This screen is for supervisors more than agents.
4. **Screen 4 — Campaign KPIs:** Conversion rate, average talk time, contacts per hour at the campaign level. Update every 5 minutes.

Use a Raspberry Pi or similar micro-PC for each display, running Chromium in kiosk mode:

```bash
chromium-browser --kiosk --disable-restore-session-state \
    "http://10.0.1.10/vicidial/custom_wallboard.php?campaign=SALESCAMP"
```

## How ViciStack Helps

Building and maintaining custom dashboards requires MySQL query expertise, PHP development, Grafana administration, and ongoing tuning as your campaigns evolve. At ViciStack, we've built dashboards for operations running 25 to 500+ agents and know exactly which metrics move the needle for each campaign type.

Our managed optimization includes:

- **Pre-built wallboard templates** configured for your specific campaigns and KPIs
- **Grafana dashboards** with alerting tuned to your drop rate targets and hopper thresholds
- **Real-time anomaly detection** that catches problems before they cost you leads
- **Weekly dashboard reviews** where we identify trends and recommend adjustments

All of this is included in the flat $150/agent/month — no per-minute fees, no surprise charges.

**Get a free analysis of your current VICIdial setup** and see exactly where your real-time monitoring gaps are costing you leads: [Request Your Free ViciStack Analysis](https://vicistack.com/proof/)

## Related Resources

- [VICIdial Pause Codes and Agent Accountability Systems](/blog/vicidial-pause-codes-accountability) — Deep dive into the pause code metrics your wallboard should display
- [VICIdial Auto-Dial Level Tuning by Campaign Type](/blog/vicidial-auto-dial-level-tuning) — Understand the metrics that drive dial level decisions
- [VICIdial Asterisk CDR Analysis for Connect Rate Optimization](/blog/vicidial-asterisk-cdr-analysis) — Advanced SQL analysis that feeds into Grafana dashboards

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-realtime-agent-dashboard).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
