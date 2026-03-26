# VICIdial WebSocket Real-Time Dashboard with Grafana + Node.js

**Build a VICIdial wallboard that actually updates in real time. This guide covers the full stack: a Node.js WebSocket server that polls MySQL and pushes snapshots every 2 seconds, Grafana panels with color-coded thresholds, the SQL queries worth running, alerting, and production deployment with systemd and Docker Compose.**

---

## Why Your VICIdial Floor Needs a Real Wallboard

VICIdial's built-in `realtime_report.php` does its job. It shows agent states, active calls, and campaign stats. But it has problems that compound when you're running a real operations floor:

1. **It polls on page refresh.** Every browser tab running the realtime report fires its own set of MySQL queries on every refresh cycle. Ten managers staring at the page = ten times the query load on your dialer database.

2. **No historical context.** You see what's happening now. You don't see what happened 30 minutes ago unless you switch to a different report. By the time you notice a drop rate problem, you've been over the FTC's 3% Safe Harbor threshold for an hour.

3. **No alerting.** Nobody gets a Slack message when the inbound queue backs up to 40 calls or when a carrier's ASR falls off a cliff. You find out when agents start complaining, which is always too late.

4. **It doesn't look good on a TV.** Sounds superficial, but call center wallboards exist for a reason. When agents can see their numbers, they perform better. The built-in HTML table doesn't render well on a 65-inch screen across the room.

The fix is a two-layer architecture: a thin Node.js WebSocket server that polls MySQL once and pushes to all viewers simultaneously, paired with Grafana for the visual layer. One query per metric per cycle, regardless of how many people are watching.

---

## Architecture Overview

```
┌─────────────────┐     poll every     ┌──────────────────────┐     push every     ┌─────────────────┐
│  VICIdial MySQL  │ ←── 2-3 seconds ──│  Node.js WS Server   │ ── 2-3 seconds ──→ │  HTML Wallboard  │
│  (asterisk DB)   │                    │  (port 8089)         │                    │  (big TV)        │
└─────────────────┘                    └──────────────────────┘                    └─────────────────┘
        │
        │  direct query every 5-10 sec
        ▼
┌─────────────────┐
│     Grafana      │ ← panels, time-series, alerts, kiosk mode
│   (port 3000)    │
└─────────────────┘
```

**Node.js WebSocket server** -- polls the MEMORY-engine tables (`vicidial_live_agents`, `vicidial_auto_calls`) and `vicidial_campaign_stats` using a 3-connection pool. Broadcasts a JSON snapshot to every connected browser. One poll cycle, unlimited viewers.

**Grafana** -- queries MySQL directly on its own 5-10 second refresh timer. Handles time-series panels, historical charts, alerting, and kiosk mode for wallboard TVs and manager laptops.

**Why both?** Grafana handles charts, alerts, and threshold colors. The WebSocket wallboard gives you a raw, fast, custom HTML display updating every 2 seconds. Most shops run Grafana on manager screens and the HTML wallboard on the floor TVs. If you only want one, skip the WebSocket server and use Grafana with 5-second auto-refresh.

---

## The Node.js WebSocket Server

This is the production-ready server. It includes health checks, graceful shutdown, connection heartbeats, optional token auth, and error counting. Save it as `/opt/vicidial-dashboard/server.js`.

```javascript
// /opt/vicidial-dashboard/server.js
'use strict';

const WebSocket = require('ws');
const mysql = require('mysql2/promise');
const http = require('http');

// --- config from environment ---
const PORT = parseInt(process.env.PORT) || 8089;
const POLL_MS = parseInt(process.env.POLL_MS) || 2000;
const DB = {
  host: process.env.VICI_DB_HOST || '127.0.0.1',
  port: parseInt(process.env.VICI_DB_PORT) || 3306,
  user: process.env.VICI_DB_USER || 'cron',
  password: process.env.VICI_DB_PASS || '1234',
  database: 'asterisk',
  connectionLimit: 3,       // never hog MySQL connections
  enableKeepAlive: true,
  keepAliveInitialDelay: 10000,
  waitForConnections: true,
  queueLimit: 5
};
const AUTH_TOKEN = process.env.WS_AUTH_TOKEN || '';

// --- state ---
const pool = mysql.createPool(DB);
const clients = new Set();
let lastSnapshot = null;
let lastPollMs = 0;
let pollErrors = 0;

// --- HTTP health check endpoint ---
const httpServer = http.createServer((req, res) => {
  if (req.url === '/health') {
    const healthy = lastSnapshot
      && (Date.now() - lastSnapshot.timestamp < POLL_MS * 5);
    res.writeHead(healthy ? 200 : 503,
      { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({
      status: healthy ? 'ok' : 'degraded',
      clients: clients.size,
      lastPollMs,
      pollErrors,
      lastUpdate: lastSnapshot?.timestamp || 0
    }));
    return;
  }
  res.writeHead(404);
  res.end();
});

// --- WebSocket server mounts on the HTTP server ---
const wss = new WebSocket.Server({ server: httpServer });

wss.on('connection', (ws, req) => {
  // optional token auth
  if (AUTH_TOKEN) {
    const url = new URL(req.url, 'http://localhost');
    if (url.searchParams.get('token') !== AUTH_TOKEN) {
      ws.close(4001, 'unauthorized');
      return;
    }
  }

  clients.add(ws);
  ws.isAlive = true;
  ws.on('pong', () => { ws.isAlive = true; });

  // send cached snapshot immediately on connect
  if (lastSnapshot) {
    ws.send(JSON.stringify(lastSnapshot));
  }

  ws.on('close', () => clients.delete(ws));
  ws.on('error', () => clients.delete(ws));
});

// heartbeat — kill dead connections every 30s
const heartbeatInterval = setInterval(() => {
  for (const ws of clients) {
    if (!ws.isAlive) { ws.terminate(); clients.delete(ws); continue; }
    ws.isAlive = false;
    ws.ping();
  }
}, 30000);

// --- MySQL polling ---
async function pollOnce() {
  const start = Date.now();
  const conn = await pool.getConnection();
  try {
    const [agents] = await conn.query(`
      SELECT campaign_id,
        SUM(status='INCALL') AS incall,
        SUM(status='READY') AS ready,
        SUM(status='PAUSED') AS paused,
        SUM(status='DISPO') AS dispo,
        COUNT(*) AS total
      FROM vicidial_live_agents
      GROUP BY campaign_id
    `);

    const [agentDetail] = await conn.query(`
      SELECT
        vla.user, vu.full_name, vla.status,
        vla.campaign_id, vla.pause_code,
        vla.calls_today,
        UNIX_TIMESTAMP(vla.last_call_time) AS last_call_epoch,
        UNIX_TIMESTAMP(vla.last_call_finish) AS last_finish_epoch
      FROM vicidial_live_agents vla
      LEFT JOIN vicidial_users vu ON vu.user = vla.user
      ORDER BY vla.campaign_id, vla.status
    `);

    const [activeCalls] = await conn.query(`
      SELECT campaign_id,
        COUNT(*) AS total_lines,
        SUM(status='RINGING') AS ringing,
        SUM(status='LIVE') AS live,
        SUM(status='IVR') AS ivr
      FROM vicidial_auto_calls
      GROUP BY campaign_id
    `);

    const [queue] = await conn.query(`
      SELECT campaign_id,
        COUNT(*) AS waiting,
        TIMESTAMPDIFF(SECOND, MIN(call_time), NOW())
          AS longest_wait_sec
      FROM vicidial_auto_calls
      WHERE status IN ('LIVE','IVR') AND agent_only = ''
      GROUP BY campaign_id
    `);

    const [stats] = await conn.query(`
      SELECT campaign_id, calls_today, answers_today,
             drops_today, drops_today_pct, dialable_leads,
             calls_hour, answers_hour, drops_hour_pct,
             calls_fivemin, answers_fivemin
      FROM vicidial_campaign_stats
    `);

    lastPollMs = Date.now() - start;
    pollErrors = 0;

    lastSnapshot = {
      timestamp: Date.now(),
      pollMs: lastPollMs,
      agents,
      agentDetail,
      activeCalls,
      queue,
      stats
    };

    return lastSnapshot;
  } finally {
    conn.release();
  }
}

function broadcast(snapshot) {
  const payload = JSON.stringify(snapshot);
  for (const ws of clients) {
    if (ws.readyState === WebSocket.OPEN) {
      ws.send(payload);
    }
  }
}

// --- poll loop ---
const pollInterval = setInterval(async () => {
  try {
    const snapshot = await pollOnce();
    broadcast(snapshot);
  } catch (err) {
    pollErrors++;
    console.error(`poll error (#${pollErrors}):`, err.message);
    if (pollErrors > 10) {
      console.error('too many consecutive poll errors, exiting');
      process.exit(1);
    }
  }
}, POLL_MS);

// --- graceful shutdown ---
async function shutdown(signal) {
  console.log(`${signal} received, shutting down`);
  clearInterval(pollInterval);
  clearInterval(heartbeatInterval);
  for (const ws of clients) {
    ws.close(1001, 'server shutting down');
  }
  wss.close();
  httpServer.close();
  await pool.end();
  process.exit(0);
}

process.on('SIGTERM', () => shutdown('SIGTERM'));
process.on('SIGINT', () => shutdown('SIGINT'));

// --- start ---
httpServer.listen(PORT, '0.0.0.0', () => {
  console.log(`vicidial-dashboard ws://0.0.0.0:${PORT}`);
  console.log(`health check http://0.0.0.0:${PORT}/health`);
  console.log(`polling every ${POLL_MS}ms`);
});
```

### package.json

```json
{
  "name": "vicidial-ws-dashboard",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "start": "node server.js",
    "dev": "node --watch server.js"
  },
  "dependencies": {
    "ws": "^8.17.0",
    "mysql2": "^3.9.0"
  }
}
```

### HTML Wallboard Client

The WebSocket server pushes JSON snapshots. Here's a self-contained HTML page that renders them — dark background, big numbers, color-coded status indicators visible from across a call center floor. Save it as `index.html` and serve it from the same box or open it as a local file.

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>VICIdial Real-Time Dashboard</title>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body { background: #1a1a2e; color: #eee; font-family: system-ui, sans-serif; }
    .grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(200px, 1fr)); gap: 16px; padding: 16px; }
    .card { background: #16213e; border-radius: 8px; padding: 24px; text-align: center; }
    .card .value { font-size: 48px; font-weight: 700; margin: 8px 0; }
    .card .label { font-size: 14px; color: #888; text-transform: uppercase; }
    .green { color: #73BF69; }
    .blue  { color: #5794F2; }
    .orange { color: #FF9830; }
    .red   { color: #F2495C; }
    .purple { color: #B877D9; }
    #status { position: fixed; top: 8px; right: 12px; font-size: 11px; color: #555; }
    #agents-table { width: 100%; border-collapse: collapse; margin: 16px; }
    #agents-table th, #agents-table td { padding: 8px 12px; text-align: left; border-bottom: 1px solid #2a2a4a; }
    #agents-table th { color: #888; font-size: 12px; text-transform: uppercase; }
  </style>
</head>
<body>
  <div id="status">connecting...</div>
  <div class="grid" id="cards"></div>
  <table id="agents-table">
    <thead>
      <tr><th>Agent</th><th>Name</th><th>Status</th><th>Campaign</th><th>Calls</th></tr>
    </thead>
    <tbody id="agents-body"></tbody>
  </table>

  <script>
    const WS_URL = 'ws://' + location.hostname + ':8089';
    let ws;
    let reconnectMs = 1000;

    function connect() {
      ws = new WebSocket(WS_URL);
      ws.onopen = () => {
        document.getElementById('status').textContent = 'connected';
        reconnectMs = 1000;
      };
      ws.onmessage = (e) => render(JSON.parse(e.data));
      ws.onclose = () => {
        document.getElementById('status').textContent = 'reconnecting...';
        setTimeout(connect, reconnectMs);
        reconnectMs = Math.min(reconnectMs * 2, 10000);
      };
      ws.onerror = () => ws.close();
    }

    function render(data) {
      let totalIncall = 0, totalReady = 0, totalPaused = 0, totalAgents = 0;
      let totalWaiting = 0, longestWait = 0;
      let callsToday = 0, answersToday = 0, dropPct = 0;

      for (const a of data.agents || []) {
        totalIncall += Number(a.incall) || 0;
        totalReady  += Number(a.ready) || 0;
        totalPaused += Number(a.paused) || 0;
        totalAgents += Number(a.total) || 0;
      }
      for (const q of data.queue || []) {
        totalWaiting += Number(q.waiting) || 0;
        longestWait = Math.max(longestWait, Number(q.longest_wait_sec) || 0);
      }
      for (const s of data.stats || []) {
        callsToday   += Number(s.calls_today) || 0;
        answersToday += Number(s.answers_today) || 0;
        dropPct = Math.max(dropPct, parseFloat(s.drops_today_pct) || 0);
      }

      const cards = [
        { label: 'Agents Talking', value: totalIncall, cls: 'green' },
        { label: 'Agents Ready',   value: totalReady,  cls: 'blue' },
        { label: 'Agents Paused',  value: totalPaused, cls: 'orange' },
        { label: 'Calls Waiting',  value: totalWaiting,
          cls: totalWaiting > 5 ? 'red' : 'blue' },
        { label: 'Longest Wait',   value: longestWait + 's',
          cls: longestWait > 30 ? 'red' : 'green' },
        { label: 'Drop Rate',      value: dropPct + '%',
          cls: dropPct > 3 ? 'red' : dropPct > 2 ? 'orange' : 'green' },
        { label: 'Calls Today',    value: callsToday,   cls: 'blue' },
        { label: 'Answers',        value: answersToday, cls: 'green' }
      ];

      document.getElementById('cards').innerHTML = cards.map(c =>
        `<div class="card">
          <div class="label">${c.label}</div>
          <div class="value ${c.cls}">${c.value}</div>
        </div>`
      ).join('');

      const statusColor = {
        INCALL: 'green', READY: 'blue',
        PAUSED: 'orange', DISPO: 'purple'
      };
      document.getElementById('agents-body').innerHTML =
        (data.agentDetail || []).map(a =>
          `<tr>
            <td>${a.user}</td>
            <td>${a.full_name || ''}</td>
            <td class="${statusColor[a.status] || ''}">${a.status}</td>
            <td>${a.campaign_id}</td>
            <td>${a.calls_today}</td>
          </tr>`
        ).join('');

      document.getElementById('status').textContent =
        `updated ${new Date(data.timestamp).toLocaleTimeString()} (${data.pollMs}ms)`;
    }

    connect();
  </script>
</body>
</html>
```

Reconnection uses exponential backoff capped at 10 seconds. When the server restarts, browsers reconnect automatically and receive the cached snapshot immediately.

### What the queries are doing

Every poll cycle runs five SELECT queries against the `asterisk` database. Here's what each one gives you:

1. **Agent status counts** — groups `vicidial_live_agents` by campaign and status. Tells you how many agents are talking, waiting, paused, or in dispo. This table is MEMORY engine (lives in RAM), so the query executes in under 1ms even with 300 agents.

2. **Agent detail roster** — full list of every logged-in agent with their current status, campaign, pause code, and call count. Joined with `vicidial_users` for display names. This feeds the agent table on the wallboard.

3. **Active call summary** — counts rows in `vicidial_auto_calls` grouped by campaign and status (RINGING, LIVE, IVR). Row count equals the number of calls happening right now. A 50-seat predictive dialer might have 80-150 rows here at peak.

4. **Queue depth** — counts calls waiting for an agent in each campaign, plus the age of the oldest waiting call. This is the number that makes managers nervous, and it should.

5. **Campaign stats** — reads from `vicidial_campaign_stats`, which VICIdial's `AST_update.pl` cron process pre-aggregates every 2-4 seconds. One row per campaign with today's totals, hourly rolling windows, and five-minute snapshots. No need to COUNT(*) against `vicidial_log` — the math is already done.

Total query time: 2-8ms on typical hardware. Connection pool is limited to 3 to avoid competing with VICIdial's own cron processes and agent sessions for MySQL connections.

---

## The SQL Queries That Actually Matter

Not every query is worth running every 5 seconds on a wallboard. Here are the ones we use in production, organized by how often you should poll them.

### Tier 1: Poll Every 2-5 Seconds (Real-Time State)

**Agent status summary** — your main wallboard panel:

```sql
SELECT
    campaign_id,
    SUM(status = 'READY')   AS agents_ready,
    SUM(status = 'INCALL')  AS agents_incall,
    SUM(status = 'PAUSED')  AS agents_paused,
    SUM(status = 'QUEUE')   AS agents_in_queue,
    SUM(status = 'CLOSER')  AS agents_closer,
    COUNT(*)                AS agents_total
FROM vicidial_live_agents
GROUP BY campaign_id;
```

**Detailed agent roster with time-in-state** — for the agent table:

```sql
SELECT
    user,
    status,
    campaign_id,
    calls_today,
    last_call_time,
    last_state_change,
    TIMESTAMPDIFF(SECOND, last_state_change, NOW())
      AS seconds_in_state,
    callerid,
    lead_id
FROM vicidial_live_agents
ORDER BY campaign_id, status, seconds_in_state DESC;
```

**Queue depth by campaign** — the metric supervisors care about most:

```sql
SELECT
    campaign_id,
    call_type,
    COUNT(*)                                           AS calls_in_queue,
    MIN(call_time)                                     AS oldest_call,
    TIMESTAMPDIFF(SECOND, MIN(call_time), NOW())       AS longest_wait_sec,
    ROUND(AVG(TIMESTAMPDIFF(SECOND, call_time, NOW())), 1)
                                                       AS avg_wait_sec
FROM vicidial_auto_calls
WHERE status IN ('LIVE', 'CLOSER', 'QUEUE')
GROUP BY campaign_id, call_type;
```

**Queue vs. available agents** — tells you if you're understaffed right now:

```sql
SELECT
    ac.campaign_id,
    COUNT(DISTINCT ac.auto_call_id)  AS calls_waiting,
    (SELECT COUNT(*) FROM vicidial_live_agents la
     WHERE la.campaign_id = ac.campaign_id
       AND la.status = 'READY')      AS agents_ready,
    ROUND(
        COUNT(DISTINCT ac.auto_call_id) /
        GREATEST((SELECT COUNT(*) FROM vicidial_live_agents la
                  WHERE la.campaign_id = ac.campaign_id
                    AND la.status = 'READY'), 1)
    , 2)                             AS queue_to_agent_ratio
FROM vicidial_auto_calls ac
WHERE ac.status IN ('LIVE', 'CLOSER', 'QUEUE')
GROUP BY ac.campaign_id;
```

### Tier 2: Poll Every 10-30 Seconds (Pre-Aggregated Stats)

**Campaign stats** — pre-calculated by VICIdial's backend:

```sql
SELECT
    campaign_id,
    calls_today,
    answers_today,
    drops_today,
    IF(calls_today > 0,
       ROUND(drops_today / calls_today * 100, 2), 0) AS drop_pct,
    IF(calls_today > 0,
       ROUND(answers_today / calls_today * 100, 2), 0) AS answer_pct,
    dialable_leads
FROM vicidial_campaign_stats
ORDER BY calls_today DESC;
```

**System-wide agent counts** — for the big-number panels:

```sql
SELECT
    COUNT(*)                           AS total_logged_in,
    SUM(status = 'INCALL')             AS on_calls,
    SUM(status = 'READY')              AS waiting,
    SUM(status = 'PAUSED')             AS paused,
    SUM(TIMESTAMPDIFF(SECOND, last_state_change, NOW()) > 600
        AND status = 'PAUSED')         AS paused_over_10min
FROM vicidial_live_agents;
```

### Tier 3: Poll Every 60 Seconds (Log Table Queries)

These hit append-only log tables that grow all day. Always filter by date. Never let Grafana run these every 5 seconds.

**Calls per hour today** — the classic volume chart:

```sql
SELECT
    HOUR(call_date)        AS call_hour,
    COUNT(*)               AS total_calls,
    SUM(length_in_sec > 0) AS answered,
    SUM(status = 'SALE')   AS sales,
    IF(COUNT(*) > 0,
       ROUND(SUM(length_in_sec > 0) / COUNT(*) * 100, 1), 0)
                           AS answer_rate_pct
FROM vicidial_log
WHERE call_date >= CURDATE()
GROUP BY HOUR(call_date)
ORDER BY call_hour;
```

**Agent handle time** — AHT = talk_sec + dispo_sec:

```sql
SELECT
    user,
    COUNT(*)                                  AS calls_handled,
    ROUND(AVG(talk_sec), 1)                   AS avg_talk_sec,
    ROUND(AVG(dispo_sec), 1)                  AS avg_dispo_sec,
    ROUND(AVG(talk_sec + dispo_sec), 1)       AS avg_handle_time,
    IF(SUM(talk_sec + wait_sec + dispo_sec + pause_sec) > 0,
       ROUND(SUM(talk_sec) /
             SUM(talk_sec + wait_sec + dispo_sec + pause_sec)
             * 100, 1), 0)                    AS utilization_pct
FROM vicidial_agent_log
WHERE event_time >= CURDATE()
  AND talk_sec > 0
GROUP BY user
ORDER BY avg_handle_time DESC;
```

**Agent leaderboard** — gamification panel for the floor TV:

```sql
SELECT
    al.user,
    COUNT(*)                                  AS total_calls,
    SUM(al.status = 'SALE')                   AS sales,
    ROUND(SUM(al.talk_sec) / 60, 1)           AS talk_minutes,
    ROUND(AVG(al.talk_sec + al.dispo_sec), 1) AS avg_handle_sec,
    IF(COUNT(*) > 0,
       ROUND(SUM(al.status = 'SALE') /
             COUNT(*) * 100, 1), 0)           AS conversion_pct
FROM vicidial_agent_log al
WHERE al.event_time >= CURDATE()
  AND al.talk_sec > 0
GROUP BY al.user
ORDER BY sales DESC, conversion_pct DESC
LIMIT 20;
```

**Inbound service level** — the 80/20 SLA (80% of calls answered within 20 seconds) is the industry standard. Track it:

```sql
SELECT
    campaign_id,
    COUNT(*)                               AS total_calls,
    SUM(queue_seconds <= 20)               AS answered_20s,
    ROUND(SUM(queue_seconds <= 20)
          / COUNT(*) * 100, 1)             AS sla_20s_pct,
    ROUND(AVG(queue_seconds), 1)           AS avg_speed_answer,
    SUM(status IN ('DROP', 'ABANDON'))     AS abandoned,
    ROUND(SUM(status IN ('DROP', 'ABANDON'))
          / COUNT(*) * 100, 1)             AS abandon_pct
FROM vicidial_closer_log
WHERE call_date >= CURDATE()
  AND status NOT IN ('DROP', 'ABANDON')
GROUP BY campaign_id;
```

**Carrier ASR** — catch trunk problems before they cost you hours of downtime:

```sql
SELECT
    SUBSTRING_INDEX(
        SUBSTRING_INDEX(channel, '/', -1), '-', 1
    )                                        AS carrier,
    COUNT(*)                                 AS total_attempts,
    SUM(dialstatus = 'ANSWER')               AS answered,
    SUM(dialstatus = 'CONGESTION')           AS congestion,
    SUM(dialstatus = 'CHANUNAVAIL')          AS chan_unavail,
    ROUND(
        SUM(dialstatus = 'ANSWER') / COUNT(*) * 100, 2
    )                                        AS asr_pct
FROM vicidial_carrier_log
WHERE call_date >= CURDATE()
GROUP BY carrier
ORDER BY total_attempts DESC;
```

### Query Design Rules

1. **Always filter log tables by date.** `WHERE call_date >= CURDATE()` at minimum. Without it, you scan millions of rows and peg disk I/O.

2. **Use `vicidial_campaign_stats` over raw table counts.** Don't recount `vicidial_log` when `calls_today` is already computed.

3. **Keep monitoring connections under 5.** Node.js `connectionLimit: 3` + Grafana `maxOpenConns: 2` = 5 out of a 256-512 pool.

---

## Grafana Setup

### Install Grafana

On AlmaLinux/Rocky Linux (ViciBox base):

```bash
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

sudo dnf install -y grafana
sudo systemctl enable --now grafana-server
sudo firewall-cmd --permanent --add-port=3000/tcp
sudo firewall-cmd --reload
```

### Create a Read-Only MySQL User

Never give Grafana write access to VICIdial's database. Create a SELECT-only user scoped to the specific tables you need:

```sql
CREATE USER 'grafana_ro'@'10.0.0.%' IDENTIFIED BY 'strongpassword';
GRANT SELECT ON asterisk.vicidial_live_agents TO 'grafana_ro'@'10.0.0.%';
GRANT SELECT ON asterisk.vicidial_auto_calls TO 'grafana_ro'@'10.0.0.%';
GRANT SELECT ON asterisk.vicidial_campaign_stats TO 'grafana_ro'@'10.0.0.%';
GRANT SELECT ON asterisk.vicidial_log TO 'grafana_ro'@'10.0.0.%';
GRANT SELECT ON asterisk.vicidial_closer_log TO 'grafana_ro'@'10.0.0.%';
GRANT SELECT ON asterisk.vicidial_agent_log TO 'grafana_ro'@'10.0.0.%';
GRANT SELECT ON asterisk.vicidial_users TO 'grafana_ro'@'10.0.0.%';
GRANT SELECT ON asterisk.vicidial_carrier_log TO 'grafana_ro'@'10.0.0.%';
FLUSH PRIVILEGES;
```

### Data Source Provisioning

Save this to `/etc/grafana/provisioning/datasources/vicidial.yaml`:

```yaml
apiVersion: 1
datasources:
  - name: VICIdial
    type: mysql
    access: proxy
    url: 10.0.0.5:3306
    database: asterisk
    user: grafana_ro
    secureJsonData:
      password: "${VICI_GRAFANA_PW}"
    jsonData:
      maxOpenConns: 2
      maxIdleConns: 1
      connMaxLifetime: 300
      tlsSkipVerify: true
    isDefault: true
    editable: false
```

`maxOpenConns: 2` is critical. Every open connection is one fewer for VICIdial's own processes.

### Dashboard Provisioning

Save to `/etc/grafana/provisioning/dashboards/vicidial.yaml`:

```yaml
apiVersion: 1
providers:
  - name: VICIdial
    orgId: 1
    folder: Call Center
    type: file
    disableDeletion: false
    updateIntervalSeconds: 30
    options:
      path: /var/lib/grafana/dashboards/vicidial
      foldersFromFilesStructure: false
```

### Set Minimum Refresh Interval

In `/etc/grafana/grafana.ini`:

```ini
[dashboards]
min_refresh_interval = 5s
```

---

## Grafana Panel Configurations

Here are the panels that belong on a VICIdial wallboard. Each includes the complete JSON you can paste into Grafana's dashboard JSON model.

### Agents Talking (Stat Panel)

```json
{
  "id": 1,
  "title": "Agents Talking",
  "type": "stat",
  "datasource": { "type": "mysql", "uid": "vicidial" },
  "targets": [
    {
      "rawSql": "SELECT COUNT(*) AS value FROM vicidial_live_agents WHERE status = 'INCALL'",
      "format": "table"
    }
  ],
  "fieldConfig": {
    "defaults": {
      "color": { "mode": "thresholds" },
      "thresholds": {
        "mode": "absolute",
        "steps": [
          { "color": "#EAB839", "value": null },
          { "color": "#73BF69", "value": 5 },
          { "color": "#56A64B", "value": 15 }
        ]
      }
    }
  },
  "options": {
    "reduceOptions": { "calcs": ["lastNotNull"] },
    "colorMode": "background",
    "graphMode": "none"
  },
  "gridPos": { "h": 4, "w": 4, "x": 0, "y": 0 }
}
```

Yellow under 5 agents talking, green at 5+, deep green at 15+.

### Drop Rate Gauge

The 3% FTC Safe Harbor threshold is law, not a suggestion.

```json
{
  "id": 2,
  "title": "Drop Rate %",
  "type": "gauge",
  "datasource": { "type": "mysql", "uid": "vicidial" },
  "targets": [
    {
      "rawSql": "SELECT CAST(drops_today_pct AS DECIMAL(5,2)) AS value FROM vicidial_campaign_stats WHERE campaign_id = '${campaign}'",
      "format": "table"
    }
  ],
  "fieldConfig": {
    "defaults": {
      "color": { "mode": "thresholds" },
      "thresholds": {
        "mode": "absolute",
        "steps": [
          { "color": "#73BF69", "value": null },
          { "color": "#EAB839", "value": 2 },
          { "color": "#F2495C", "value": 3 }
        ]
      },
      "min": 0,
      "max": 10,
      "unit": "percent"
    }
  },
  "options": {
    "showThresholdLabels": true,
    "showThresholdMarkers": true
  },
  "gridPos": { "h": 6, "w": 6, "x": 0, "y": 4 }
}
```

Green under 2%, yellow 2-3%, red 3%+. When this goes red, reduce the dial level.

### Calls Waiting (Stat with Sparkline)

```json
{
  "id": 3,
  "title": "Calls Waiting",
  "type": "stat",
  "datasource": { "type": "mysql", "uid": "vicidial" },
  "targets": [
    {
      "rawSql": "SELECT COUNT(*) AS value FROM vicidial_auto_calls WHERE status IN ('LIVE','IVR') AND agent_only = ''",
      "format": "table"
    }
  ],
  "fieldConfig": {
    "defaults": {
      "color": { "mode": "thresholds" },
      "thresholds": {
        "mode": "absolute",
        "steps": [
          { "color": "#73BF69", "value": null },
          { "color": "#EAB839", "value": 3 },
          { "color": "#FF9830", "value": 8 },
          { "color": "#F2495C", "value": 15 }
        ]
      }
    }
  },
  "options": {
    "colorMode": "background",
    "graphMode": "area"
  },
  "gridPos": { "h": 4, "w": 4, "x": 4, "y": 0 }
}
```

### Agent Status Breakdown (Bar Gauge)

```json
{
  "id": 4,
  "title": "Agent Status Breakdown",
  "type": "bargauge",
  "datasource": { "type": "mysql", "uid": "vicidial" },
  "targets": [
    {
      "rawSql": "SELECT SUM(status='INCALL') AS `Talking`, SUM(status='READY') AS `Ready`, SUM(status='PAUSED') AS `Paused`, SUM(status='DISPO') AS `Dispo` FROM vicidial_live_agents WHERE campaign_id = '${campaign}'",
      "format": "table"
    }
  ],
  "fieldConfig": {
    "overrides": [
      {
        "matcher": { "id": "byName", "options": "Talking" },
        "properties": [{ "id": "color", "value": { "fixedColor": "#73BF69", "mode": "fixed" } }]
      },
      {
        "matcher": { "id": "byName", "options": "Ready" },
        "properties": [{ "id": "color", "value": { "fixedColor": "#5794F2", "mode": "fixed" } }]
      },
      {
        "matcher": { "id": "byName", "options": "Paused" },
        "properties": [{ "id": "color", "value": { "fixedColor": "#FF9830", "mode": "fixed" } }]
      },
      {
        "matcher": { "id": "byName", "options": "Dispo" },
        "properties": [{ "id": "color", "value": { "fixedColor": "#B877D9", "mode": "fixed" } }]
      }
    ]
  },
  "options": {
    "orientation": "horizontal",
    "displayMode": "gradient",
    "showUnfilled": true
  },
  "gridPos": { "h": 6, "w": 12, "x": 12, "y": 0 }
}
```

### Call Volume Time Series

```json
{
  "id": 5,
  "title": "Call Volume (Last 8 Hours)",
  "type": "timeseries",
  "datasource": { "type": "mysql", "uid": "vicidial" },
  "targets": [
    {
      "rawSql": "SELECT DATE_FORMAT(call_date, '%Y-%m-%d %H:00:00') AS time, COUNT(*) AS calls, SUM(length_in_sec > 0) AS answers FROM vicidial_log WHERE call_date >= NOW() - INTERVAL 8 HOUR AND campaign_id = '${campaign}' GROUP BY DATE_FORMAT(call_date, '%Y-%m-%d %H:00:00') ORDER BY time",
      "format": "time_series"
    }
  ],
  "fieldConfig": {
    "defaults": {
      "custom": {
        "drawStyle": "line",
        "lineInterpolation": "smooth",
        "fillOpacity": 20,
        "lineWidth": 2
      }
    }
  },
  "gridPos": { "h": 8, "w": 24, "x": 0, "y": 10 }
}
```

### Agent Detail Table

```json
{
  "id": 6,
  "title": "Agent Detail",
  "type": "table",
  "datasource": { "type": "mysql", "uid": "vicidial" },
  "targets": [
    {
      "rawSql": "SELECT vla.user AS Agent, vu.full_name AS Name, vla.status AS Status, vla.campaign_id AS Campaign, vla.calls_today AS 'Calls Today', CASE WHEN vla.status = 'INCALL' THEN TIMESTAMPDIFF(SECOND, vla.last_call_time, NOW()) WHEN vla.status = 'PAUSED' THEN TIMESTAMPDIFF(SECOND, vla.last_state_change, NOW()) WHEN vla.status = 'READY' THEN TIMESTAMPDIFF(SECOND, vla.last_call_finish, NOW()) ELSE 0 END AS 'Seconds In State' FROM vicidial_live_agents vla LEFT JOIN vicidial_users vu ON vu.user = vla.user ORDER BY vla.status, vla.campaign_id",
      "format": "table"
    }
  ],
  "fieldConfig": {
    "overrides": [
      {
        "matcher": { "id": "byName", "options": "Status" },
        "properties": [
          {
            "id": "mappings",
            "value": [
              { "type": "value", "options": { "INCALL": { "color": "#73BF69", "text": "TALKING" } } },
              { "type": "value", "options": { "READY": { "color": "#5794F2", "text": "READY" } } },
              { "type": "value", "options": { "PAUSED": { "color": "#FF9830", "text": "PAUSED" } } },
              { "type": "value", "options": { "DISPO": { "color": "#B877D9", "text": "DISPO" } } }
            ]
          }
        ]
      }
    ]
  },
  "gridPos": { "h": 10, "w": 24, "x": 0, "y": 18 }
}
```

### Campaign Picker Variable

Add to your dashboard's templating section:

```json
{
  "templating": {
    "list": [
      {
        "name": "campaign",
        "type": "query",
        "datasource": { "type": "mysql", "uid": "vicidial" },
        "query": "SELECT campaign_id FROM vicidial_campaigns WHERE active = 'Y' ORDER BY campaign_id",
        "refresh": 2,
        "includeAll": true,
        "multi": true,
        "current": { "text": "All", "value": "$__all" }
      }
    ]
  }
}
```

---

## Alerts That Should Wake Someone Up

Grafana alerts fire when a metric crosses a threshold. These are the ones worth configuring.

### Drop Rate Over 3%

The FTC Safe Harbor is 3%. Use `drops_today_pct` from `vicidial_campaign_stats`. Alert yellow at 2%, red at 3%. Send to every dialer supervisor.

### Queue Depth Spike

Alert when calls waiting exceeds a threshold for 30+ seconds. Adjust the threshold for your staffing model.

```sql
SELECT campaign_id,
    COUNT(*) AS calls_waiting,
    TIMESTAMPDIFF(SECOND, MIN(call_time), NOW()) AS longest_wait_sec
FROM vicidial_auto_calls
WHERE status IN ('LIVE', 'CLOSER', 'QUEUE')
GROUP BY campaign_id
HAVING calls_waiting > 15;
```

### Agent Idle Over 10 Minutes

Agent in READY for 10+ minutes with no calls means the hopper is empty, the dial level is too low, or the session is stuck.

```sql
SELECT user, campaign_id,
    TIMESTAMPDIFF(SECOND, last_state_change, NOW()) AS idle_seconds
FROM vicidial_live_agents
WHERE status = 'READY'
  AND TIMESTAMPDIFF(SECOND, last_state_change, NOW()) > 600;
```

### No Agents Logged In

A campaign that had calls today but currently has zero agents logged in. Shift gap.

```sql
SELECT cs.campaign_id
FROM vicidial_campaign_stats cs
LEFT JOIN vicidial_live_agents la
    ON cs.campaign_id = la.campaign_id
WHERE la.user IS NULL
  AND cs.calls_today > 0;
```

### Carrier ASR Below 30%

ASR under 30% over the last hour (minimum 50 attempts to avoid false positives) means something is broken on the carrier side.

```sql
SELECT
    SUBSTRING_INDEX(
        SUBSTRING_INDEX(channel, '/', -1), '-', 1
    ) AS carrier,
    COUNT(*) AS attempts,
    SUM(dialstatus = 'ANSWER') AS answered,
    ROUND(SUM(dialstatus = 'ANSWER') / COUNT(*) * 100, 2) AS asr_pct
FROM vicidial_carrier_log
WHERE call_date >= DATE_SUB(NOW(), INTERVAL 1 HOUR)
GROUP BY carrier
HAVING attempts > 50 AND asr_pct < 30.0;
```

### Stale Records in vicidial_auto_calls

Zombie records -- calls that ended but rows never got cleaned up. Causes phantom queue depth and "no live calls waiting" issues.

```sql
SELECT auto_call_id, server_ip, campaign_id, status,
    TIMESTAMPDIFF(MINUTE, last_update_time, NOW()) AS minutes_stale
FROM vicidial_auto_calls
WHERE TIMESTAMPDIFF(MINUTE, last_update_time, NOW()) > 15;
```

---

## Deployment

### systemd Service File

Save to `/etc/systemd/system/vicidial-dashboard.service`:

```ini
[Unit]
Description=VICIdial Real-Time Dashboard WebSocket Server
After=network.target mysql.service
Wants=mysql.service

[Service]
Type=simple
User=nobody
Group=nogroup
WorkingDirectory=/opt/vicidial-dashboard
ExecStart=/usr/bin/node server.js
Restart=always
RestartSec=5
Environment=NODE_ENV=production
Environment=VICI_DB_HOST=127.0.0.1
Environment=VICI_DB_USER=cron
Environment=VICI_DB_PASS=your_password_here
Environment=POLL_MS=3000
Environment=PORT=8089

# security hardening
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable vicidial-dashboard
systemctl start vicidial-dashboard
systemctl status vicidial-dashboard
```

The server runs as `nobody` with a read-only filesystem mount. It doesn't need to write anything to disk.

### Docker Compose (Separate Monitoring Server)

For 100+ agents, run the dashboard stack on a dedicated VM to keep query load off the dialer.

```yaml
# docker-compose.yml
version: '3.8'

services:
  grafana:
    image: grafana/grafana-oss:11.4.0
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana
      - ./provisioning:/etc/grafana/provisioning
    environment:
      GF_SECURITY_ADMIN_PASSWORD: changeme
      GF_USERS_ALLOW_SIGN_UP: "false"
      GF_DASHBOARDS_MIN_REFRESH_INTERVAL: "5s"
    restart: unless-stopped

  ws-server:
    build: ./vicidial-dashboard
    ports:
      - "8089:8089"
    environment:
      VICI_DB_HOST: 10.0.0.5
      VICI_DB_USER: cron
      VICI_DB_PASS: "${VICI_DB_PASS}"
      POLL_MS: 3000
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8089/health"]
      interval: 30s
      timeout: 5s
      retries: 3

volumes:
  grafana-data:
```

Dockerfile for the WebSocket server:

```dockerfile
# vicidial-dashboard/Dockerfile
FROM node:20-slim
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --production
COPY server.js .
USER node
EXPOSE 8089
CMD ["node", "server.js"]
```

On the VICIdial server, lock MySQL to the dashboard server's IP: `iptables -A INPUT -p tcp --dport 3306 -s 10.0.0.20 -j ACCEPT && iptables -A INPUT -p tcp --dport 3306 -j DROP`.

### Nginx Reverse Proxy with SSL

If Grafana is accessible outside the LAN, put it behind nginx with TLS:

```nginx
server {
    listen 443 ssl http2;
    server_name dashboard.example.com;

    ssl_certificate /etc/letsencrypt/live/dashboard.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/dashboard.example.com/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # WebSocket support for Grafana Live
    location /api/live/ {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
    }
}
```

### Wallboard Kiosk Mode

For the TV on the call floor, set up Grafana's anonymous auth and kiosk mode:

```ini
# grafana.ini
[auth.anonymous]
enabled = true
org_name = CallCenter
org_role = Viewer
hide_version = true
```

Open `http://grafana:3000/d/vicidial-rt/realtime?kiosk` in a full-screen browser. The `?kiosk` query param hides the Grafana toolbar. Set the browser to launch at boot. Done.

---

## Performance Tuning

The real-time MEMORY tables are fast but use table-level locks. Here are safe polling intervals:

| Deployment | Poll Interval | Notes |
|-----------|---------------|-------|
| Under 30 agents | 2 seconds | Tiny tables, negligible load |
| 30-100 agents | 3-5 seconds | `vicidial_auto_calls` might have 200 rows at peak, still trivial |
| 100-300 agents | 5 seconds | Watch `SHOW PROCESSLIST` for lock waits |
| 300+ agents | 10 seconds | Use `vicidial_campaign_stats` (InnoDB, already aggregated) instead of counting MEMORY table rows |

For installs over 200 agents, point Grafana and Node.js at a MySQL read replica instead of the primary. Replication lag is under 1 second for MEMORY tables. See our [VICIdial MySQL Optimization guide](/blog/vicidial-mysql-optimization/) for replica setup details.

---

## What to Build First

**Day 1:** Install Grafana. Add MySQL data source. Build four stat panels (Agents Talking, Ready, Paused, Calls Waiting). Auto-refresh at 5 seconds. Put it on a TV.

**Day 2:** Add the drop rate gauge, agent detail table, call volume time series, and the drop rate alert. That's a production wallboard.

**Week 2:** Deploy the Node.js WebSocket server and the HTML wallboard on a second TV. Add leaderboard and carrier ASR panels. Move to a dedicated monitoring server if you're over 100 agents.

**Month 2:** Add InfluxDB for time-series history. Build DID performance panels. By now you'll wonder how you ran a floor without this.

The difference between a call center that reacts to problems and one that prevents them is about 4 hours of setup and a TV from Costco.

---

*Running VICIdial at scale and need help building dashboards, tuning performance, or migrating to a modern deployment? [Contact ViciStack](https://vicistack.com/contact/) for a free operations review. We'll audit your current setup and show you where the quick wins are — typically a 30-50% improvement in agent productivity within the first two weeks.*

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-grafana-realtime-dashboard).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
