# VICIdial Whisper and Barge: Real-Time Agent Coaching That Actually Works

**The deep-dive technical guide to VICIdial's real-time coaching features. ChanSpy under the hood, whisper audio routing, barge-in conference bridging, every audio bug you'll hit, and how to build a coaching workflow that supervisors will actually use instead of just talking about using.**

---

## How It Actually Works Under the Hood

Every time a supervisor clicks "Monitor" in VICIdial's real-time report, a specific chain of events fires inside Asterisk. Understanding this chain is the difference between "it works" and "I can fix it when it doesn't."

VICIdial uses Asterisk's `ChanSpy()` application for all three coaching modes — [silent monitor](/blog/vicidial-agent-coaching/), whisper, and barge-in. ChanSpy was designed to let one channel listen to (and optionally speak into) another channel. VICIdial wraps it with its own logic through the manager API, but the audio plumbing is pure Asterisk.

Here's what happens when a supervisor initiates monitoring:

1. The supervisor clicks an agent row in `realtime_report.php` and selects a monitoring mode
2. VICIdial's PHP layer makes an AGI/Manager API call to originate a new channel to the supervisor's phone
3. The supervisor's phone rings; they answer
4. VICIdial bridges the supervisor's channel into a `ChanSpy()` application targeting the agent's active channel
5. ChanSpy handles the audio mixing based on the mode flags

The ChanSpy invocation varies by mode:

```
; Silent Monitor — supervisor hears both sides, nobody hears supervisor
ChanSpy(SIP/agent_phone,qw)

; Whisper — supervisor audio goes to agent only, not caller
ChanSpy(SIP/agent_phone,qwW)

; Barge-In — supervisor joins as 3-way conference, all parties hear each other
ChanSpy(SIP/agent_phone,qwB)
```

The flags:
- `q` — quiet mode, no beep when spy connects
- `w` — whisper mode, spy audio goes to spied channel only
- `W` — private whisper, supervisor audio goes to agent only (not bridged caller)
- `B` — barge mode, supervisor joins the bridge

Here's the thing most documentation gets wrong: the `w` and `W` flags have different behavior depending on the Asterisk version. In Asterisk 16+ (which VICIdial 3.14+ runs on), `W` is what you want for whisper — it sends the supervisor's audio only to the agent's earpiece. The `w` flag alone may leak audio to the caller in some bridge configurations.

### The Audio Path in Detail

For a typical VICIdial call with SIP trunking:

```
Caller → SIP Trunk → Asterisk → Agent Phone
                         ↑
                    ChanSpy Channel → Supervisor Phone
```

In silent monitor mode, ChanSpy taps the audio stream from the agent's channel. Both directions of audio (caller→agent and agent→caller) are mixed and sent to the supervisor. The supervisor's microphone audio goes nowhere.

In whisper mode, the supervisor's audio is injected into the agent's receive path only. The caller's bridge leg never sees it. This is possible because Asterisk maintains separate audio streams for each leg of a bridged call.

In barge-in mode, Asterisk creates a three-way conference bridge. The original two-party call is torn down and rebuilt as a ConfBridge with three legs. This is why barge-in sometimes has a brief audio interruption — the call is being rebridged.

---

## Setting Up the Permissions

Before anything works, the supervisor account needs the right flags. In the VICIdial admin panel, go to **Admin > Users** and edit the supervisor's user record.

### Required User Settings

| Setting | Required Value | Why |
|---------|---------------|-----|
| User Level | 7+ (Manager) | Levels below 7 don't see the monitoring controls |
| Agent API Access | Enabled | The monitoring functions use the agent API internally |
| Monitor Allowed | Y | Enables silent monitor and whisper |
| Barge-In Allowed | Y | Enables barge-in (separate permission from monitor) |
| Manager Shift Enforcement | Match to their actual schedule | Prevents monitoring outside authorized hours |

You can verify the permissions are set correctly:

```sql
SELECT
    user,
    full_name,
    user_level,
    vdc_agent_api_access,
    monitor_allowed,
    barge_allowed
FROM vicidial_users
WHERE user_level >= 7
ORDER BY user_level DESC, user;
```

### Supervisor Phone Setup

The supervisor needs a registered phone in VICIdial. This can be:

- **Hardware SIP phone** on the floor (Polycom, Yealink, etc.)
- **Softphone** (MicroSIP, Zoiper, Bria) on the supervisor's workstation
- **WebRTC phone** if you've set up [WebRTC in VICIdial](https://vicistack.com/blog/vicidial-webrtc-setup/)

Register the phone under **Admin > Phones**. The phone must be registered to the same Asterisk server that handles the agents being monitored. In a multi-server cluster, this means the supervisor's phone needs to register to the server where the target agent's calls are processed.

For clustered deployments with multiple Asterisk servers:

```
Server A: Agents 1-50, Supervisor Phone A
Server B: Agents 51-100, Supervisor Phone B
```

If Supervisor A tries to monitor Agent 75 (on Server B), VICIdial handles the cross-server routing through its internal IAX trunks. But audio quality degrades because the monitor stream now traverses:

```
Agent Phone → Server B → IAX Trunk → Server A → Supervisor Phone
```

That's an extra network hop, and IAX trunks add latency. For the best monitoring experience, keep supervisors on the same server as their agents. In VICIdial's admin, you can assign server affinity through the **Server IP** field in the phone configuration.

---

## Initiating a Monitor Session

### From the Real-Time Report

The standard way: Open `realtime_report.php` in your browser, find the agent, click their row.

You'll see options including:
- **MONITOR** — silent monitor mode
- **WHISPER** — whisper to agent only
- **BARGE** — 3-way conference join

Click one. Your supervisor phone rings. Answer it. You're in.

### Escalating During a Session

Here's the part most people don't know: you can escalate modes during an active session. Start in silent monitor, listen for a minute, decide the agent needs help, and escalate to whisper without hanging up and re-dialing.

From the real-time report, while already monitoring, click the agent row again and select a different mode. VICIdial will modify the ChanSpy flags on the active channel. There might be a 1-2 second audio glitch during the transition, but the session continues.

The escalation path should be:
1. **Silent Monitor** → Listen, assess the situation
2. **Whisper** → Feed the agent a rebuttal or instruction
3. **Barge** → Only if the agent can't recover and you need to address the caller directly

### From the API

For operations that want to build monitoring into their own supervisor dashboards, VICIdial's non-agent API supports monitoring commands:

```bash
# Initiate silent monitor
curl "https://your-vicidial-server/vicidial/non_agent_api.php?\
source=monitoring&\
user=supervisor1&\
pass=SuperPass123&\
function=blind_monitor&\
agent_user=agent42&\
stage=MONITOR"
```

API response:

```
SUCCESS: blind_monitor - MONITOR session initiated for agent agent42
```

You can also use `stage=WHISPER` or `stage=BARGE` to start directly in those modes.

---

## Audio Troubleshooting

This is where 80% of the pain lives. Monitoring works perfectly in theory. In practice, you'll hit audio issues that range from annoying to session-killing.

### Problem 1: The Beep

**Symptom:** Agent hears a beep when monitoring starts, immediately knows they're being watched.

**Cause:** The ChanSpy `q` (quiet) flag is missing or not being applied.

**Fix:** Check the VICIdial configuration for your server. In `/etc/astguiclient.conf`, verify:

```ini
; This should already be set correctly on a standard VICIdial install
VARserver_ip => 10.0.0.1
```

The `q` flag is hardcoded in VICIdial's monitoring code, so if you're hearing beeps, something is overriding it. Check for custom dialplan entries in `/etc/asterisk/extensions_custom.conf` that might be invoking ChanSpy without the quiet flag:

```bash
# Check for any ChanSpy invocations in custom dialplan
asterisk -rx "dialplan show" | grep -i chanspy
```

If you find a custom ChanSpy entry, make sure it includes the `q` flag.

### Problem 2: One-Way Audio

**Symptom:** Supervisor can hear the agent but not the caller (or vice versa).

**Cause:** [NAT traversal](/blog/vicidial-remote-agent-setup/) issue. The ChanSpy channel's RTP stream is being sent to the wrong IP address.

**Debug with `asterisk -r`:**

```
*CLI> core show channels
Channel              Location    State   Application(Name)
SIP/agent42-000001   s@default   Up      Dial(...)
SIP/supervisor-0002  s@spy       Up      ChanSpy(SIP/agent42,qw)

*CLI> sip show channel SIP/supervisor-0002
-- Audio:  192.168.1.50:10234 <-> 10.0.0.1:18456
```

If the "Audio" line shows a private IP when it should show a public one (or vice versa), you have a NAT issue. The fix depends on your network topology:

For supervisors behind NAT, ensure the SIP phone's STUN settings are correct, or configure `nat=force_rport,comedia` in the supervisor's SIP peer definition.

For pjsip endpoints (Asterisk 16+):

```ini
; In /etc/asterisk/pjsip_custom.conf for the supervisor phone
[supervisor_transport]
type=transport
protocol=udp
bind=0.0.0.0:5060
local_net=10.0.0.0/8
external_media_address=YOUR_PUBLIC_IP
external_signaling_address=YOUR_PUBLIC_IP

[supervisor1]
type=endpoint
transport=supervisor_transport
context=from-internal
direct_media=no    ; CRITICAL for monitoring to work
```

The `direct_media=no` setting is crucial. If direct media is enabled, Asterisk will try to route RTP directly between endpoints, bypassing the server. ChanSpy can't tap a stream that doesn't flow through Asterisk.

### Problem 3: Echo on the Supervisor's End

**Symptom:** Supervisor hears echo — their own voice coming back, or the agent's voice doubled.

**Cause:** Audio loopback in the mixing. Usually happens when the supervisor is using a speakerphone or a headset with poor acoustic isolation.

**Fix:** Use a quality headset. Seriously. Half the monitoring audio complaints we see at [ViciStack](https://vicistack.com/) are a $15 headset with bad echo cancellation.

If the echo persists with a good headset, check if DTMF mode is set to SIP INFO instead of RFC2833 on the supervisor's phone. SIP INFO DTMF processing can cause audio artifacts during ChanSpy sessions:

```
*CLI> sip show peer supervisor1
  DTMF Mode:  rfc2833
```

If it shows `info`, switch to `rfc2833`.

### Problem 4: Latency

**Symptom:** There's a 2-5 second delay between what happens on the call and what the supervisor hears.

**Cause:** Multiple hops. The audio is going Caller → Carrier → Asterisk → ChanSpy → Supervisor, and each hop adds latency. In a cluster, add another Asterisk server in the middle.

**Fix:** Minimize hops. Put the supervisor's phone on the same LAN as the Asterisk server handling the call. If you're in a cluster:

```
# Check which server is handling the agent's call
asterisk -rx "core show channels" | grep agent42
```

If the agent is on server B and the supervisor's phone is on server A, consider registering a second softphone on server B for monitoring purposes.

Also check the jitter buffer settings on the supervisor's softphone. For monitoring, you want a shorter jitter buffer (20-40ms) because you're prioritizing low latency over perfect audio. You're listening to assess the call, not recording it for broadcast.

### Problem 5: Monitoring Drops After 60 Seconds

**Symptom:** Monitoring session disconnects after about a minute.

**Cause:** Asterisk's RTP timeout. If ChanSpy doesn't detect RTP activity for 60 seconds, it assumes the session is dead and tears it down.

**Fix:** Check `rtptimeout` and `rtpholdtimeout` in your SIP configuration:

```ini
; /etc/asterisk/sip_general_custom.conf (chan_sip)
rtptimeout=300
rtpholdtimeout=600

; Or for pjsip, in the endpoint section:
rtp_timeout=300
rtp_timeout_hold=600
```

Set `rtptimeout` to 300 seconds (5 minutes) to prevent premature disconnection. Restart the SIP module after:

```bash
asterisk -rx "sip reload"
# or for pjsip:
asterisk -rx "module reload res_pjsip.so"
```

---

## Building a Whisper Coaching Workflow

Technology without process is just expensive noise. Here's a coaching workflow that works with VICIdial's monitoring features.

### Daily Monitoring Schedule

For a team of 20 agents with 2 supervisors:

| Time Block | Supervisor Activity | Mode |
|-----------|-------------------|------|
| 9:00-9:30 | Monitor bottom 3 agents by yesterday's conversion rate | Silent |
| 9:30-10:00 | Monitor any new agents (< 30 days) | Whisper ready |
| 10:00-10:30 | Targeted coaching — listen for specific behavior from last coaching session | Silent |
| 10:30-11:00 | Monitor random agents for [QA scoring](/blog/vicidial-qa-scoring/) | Silent |
| 1:00-1:30 | Afternoon coaching round — bottom 3 agents again | Silent/Whisper |
| 3:00-3:30 | Monitor top performers to identify techniques to teach others | Silent |

That's 3 hours of monitoring per supervisor per day. It sounds like a lot. It's actually the minimum for a 20-agent team if you want coaching to move metrics. Most operations do 30 minutes a week and wonder why nothing changes.

### The Whisper Playbook

Whisper coaching is an art. Done well, it turns a losing call into a sale. Done badly, it confuses the agent, the caller hears a weird pause, and everyone loses. Here are the rules:

**Rule 1: Keep instructions to 5 words or fewer.** The agent is already talking to a real human. They cannot process a full sentence from you while maintaining a conversation. Good whispers:

- "Ask about the timeline"
- "Price objection — use ROI"
- "Slow down, you're rushing"
- "Compliance disclosure now"
- "Ask for the close"

Bad whispers:

- "So what you want to do here is transition into the pricing discussion by first asking about their current solution costs and then framing our pricing as a comparison"

That agent is now frozen, trying to remember what you said, while the caller wonders why they went silent for 8 seconds.

**Rule 2: Timing matters more than content.** Whisper during natural pauses — while the caller is talking, during a brief silence, after a question where the caller is thinking. Never whisper while the agent is mid-sentence. The agent will stumble, and the caller will hear it.

**Rule 3: Pre-arrange signals.** Before the first whisper session, tell the agent: "If you hear me say 'take a breath,' that means pause for 2 seconds and let the caller talk. If you hear me say 'close,' that means ask for the sale right now." Having pre-agreed short codes makes whisper coaching 10x more effective because the agent knows exactly what each instruction means.

**Rule 4: Don't whisper-coach for more than 15 minutes per session.** After 15 minutes, the agent's cognitive load is maxed. They stop hearing you. End the session, send a message with 2-3 specific things they did well, and one thing to work on.

### Escalation Protocol

When do you escalate from whisper to barge? Almost never. But here are the three legitimate reasons:

1. **Compliance emergency.** The agent made a statement that could create legal liability and you need to correct it immediately with the caller present.
2. **Agent is completely lost.** Not struggling — lost. Can't form sentences, caller is getting hostile, agent has gone silent for more than 10 seconds.
3. **High-value save.** The caller is about to hang up, the deal is worth $50K+, and you bringing your authority to the call is the only way to save it.

If you're barging into more than one call per week across a 20-agent team, something else is broken — training, scripting, or hiring.

---

## Measuring Coaching Impact

Coaching without measurement is hope-based management. Here's how to track whether your whisper sessions are actually improving performance.

### Pre/Post Agent Metrics

For each agent who receives coaching, pull a before-and-after comparison:

```sql
SELECT
    'PRE-COACHING' AS period,
    user,
    COUNT(*) AS total_calls,
    AVG(length_in_sec) AS avg_talk_sec,
    SUM(CASE WHEN status = 'SALE' THEN 1 ELSE 0 END) AS sales,
    ROUND(SUM(CASE WHEN status = 'SALE' THEN 1 ELSE 0 END) /
          NULLIF(COUNT(*), 0) * 100, 2) AS conversion_pct
FROM vicidial_log
WHERE user = 'agent42'
    AND call_date BETWEEN '2026-03-01' AND '2026-03-14'
    AND length_in_sec > 0
GROUP BY user

UNION ALL

SELECT
    'POST-COACHING' AS period,
    user,
    COUNT(*) AS total_calls,
    AVG(length_in_sec) AS avg_talk_sec,
    SUM(CASE WHEN status = 'SALE' THEN 1 ELSE 0 END) AS sales,
    ROUND(SUM(CASE WHEN status = 'SALE' THEN 1 ELSE 0 END) /
          NULLIF(COUNT(*), 0) * 100, 2) AS conversion_pct
FROM vicidial_log
WHERE user = 'agent42'
    AND call_date BETWEEN '2026-03-15' AND '2026-03-28'
    AND length_in_sec > 0
GROUP BY user;
```

Sample output:

```
+---------------+--------+-------+----------+-------+----------+
| period        | user   | calls | avg_talk | sales | conv_pct |
+---------------+--------+-------+----------+-------+----------+
| PRE-COACHING  | agent42| 312   | 198.4    | 14    | 4.49     |
| POST-COACHING | agent42| 298   | 215.6    | 22    | 7.38     |
+---------------+--------+-------+----------+-------+----------+
```

That's a 64% conversion improvement. The talk time went up by 17 seconds (which is expected — the agent is now handling objections instead of folding). More time on the phone, more closes. The numbers don't lie.

### Team-Level Trends

Track the overall team conversion rate over time and overlay coaching sessions:

```sql
SELECT
    DATE(call_date) AS day,
    COUNT(*) AS total_calls,
    ROUND(AVG(length_in_sec), 0) AS team_avg_talk,
    ROUND(SUM(CASE WHEN status = 'SALE' THEN 1 ELSE 0 END) /
          NULLIF(COUNT(*), 0) * 100, 2) AS team_conversion
FROM vicidial_log
WHERE call_date >= DATE_SUB(NOW(), INTERVAL 8 WEEK)
    AND campaign_id = 'OUTBOUND_B2B'
    AND length_in_sec > 0
GROUP BY day
ORDER BY day;
```

Plot this in Grafana (see our [Grafana dashboard guide](https://vicistack.com/blog/vicidial-grafana-dashboards/) for setup) and annotate the chart with coaching session dates. After 4-6 weeks, you should see a trend line moving up. If you don't, either the coaching content needs adjustment or the agents aren't retaining the training.

### Monitoring Session Logging

VICIdial logs monitoring sessions, but the data is sparse. Augment it with your own tracking:

```sql
-- Create a coaching log table (run once)
CREATE TABLE IF NOT EXISTS vicistack_coaching_log (
    id INT AUTO_INCREMENT PRIMARY KEY,
    coaching_date DATETIME NOT NULL,
    supervisor_user VARCHAR(20) NOT NULL,
    agent_user VARCHAR(20) NOT NULL,
    mode ENUM('SILENT','WHISPER','BARGE') NOT NULL,
    duration_minutes INT DEFAULT 0,
    focus_area VARCHAR(100),
    notes TEXT,
    pre_conversion DECIMAL(5,2),
    INDEX idx_agent_date (agent_user, coaching_date),
    INDEX idx_supervisor (supervisor_user)
);
```

Have supervisors log each session. Yes, it's extra work. But without data on who was coached, when, and on what, you can't measure whether coaching works. You're just guessing.

---

## Advanced: Automated Monitoring Alerts

Instead of randomly picking agents to monitor, let VICIdial tell you who needs attention right now. Set up a cron job that identifies struggling agents in real time:

```bash
#!/bin/bash
# /usr/local/bin/coaching-alert.sh
# Runs every 30 minutes during business hours

THRESHOLD_CONVERSION=3.0  # % — flag agents below this
MIN_CALLS=15              # minimum calls to avoid noise
HOURS_BACK=4              # look at last 4 hours

mysql -u readonly -pReadOnlyPass123 asterisk -N -e "
SELECT
    user,
    COUNT(*) AS calls,
    ROUND(SUM(CASE WHEN status = 'SALE' THEN 1 ELSE 0 END) /
          COUNT(*) * 100, 2) AS conversion
FROM vicidial_log
WHERE call_date >= DATE_SUB(NOW(), INTERVAL ${HOURS_BACK} HOUR)
    AND campaign_id = 'OUTBOUND_B2B'
    AND length_in_sec > 0
GROUP BY user
HAVING calls >= ${MIN_CALLS}
    AND conversion < ${THRESHOLD_CONVERSION}
ORDER BY conversion ASC;
" | while read user calls conv; do
    echo "[$(date)] COACHING ALERT: $user — $conv% conversion on $calls calls (last ${HOURS_BACK}h)"
done
```

Add this to cron:

```
# Run coaching alerts every 30 minutes, 9 AM to 5 PM, Mon-Fri
*/30 9-17 * * 1-5 /usr/local/bin/coaching-alert.sh >> /var/log/coaching-alerts.log
```

Now your supervisors get a list of who to monitor every 30 minutes, ranked by who's struggling most. No more random monitoring. Every session is targeted.

---

## The Coaching Culture Problem

The hardest part of whisper coaching isn't the technology. It's getting supervisors to actually do it. Every day. Consistently.

Most call center supervisors were promoted from the agent floor because they were good on the phone. That doesn't make them good coaches. Many have never been trained on how to coach, feel awkward listening to live calls, and default to checking email instead of putting on a headset.

Three things that fix this:

1. **Make monitoring time non-negotiable.** Put it on the supervisor's calendar as a blocked 3-hour recurring appointment. If they have meetings during monitoring time, the meetings lose.

2. **Tie supervisor bonuses to agent improvement, not just team averages.** If the supervisor's bonus is based on the team hitting 6% conversion, they'll just rely on the top 3 agents to carry the number. If the bonus is based on the bottom quartile improving by 1 percentage point, they'll actually coach the agents who need it.

3. **Coach the coaches.** Have the operations manager listen to coaching sessions monthly. Not the calls — the coaching sessions themselves. Is the supervisor giving actionable feedback? Are they using whisper effectively or just telling agents to "do better"? The quality of the coaching matters more than the quantity.

VICIdial gives you every tool you need. [Silent monitor, whisper, barge-in](https://vicistack.com/blog/vicidial-agent-coaching/) — they're all built in, they all work, and they cost nothing extra. The only investment is time and discipline. The call centers that make that investment convert 2-3x better than the ones that don't. That's not an exaggeration. We've seen it across dozens of operations.

The agents who get coached stay longer, perform better, and cost less. The ones who don't get coached either plateau or leave. Pick your outcome and build the process to match.

---

*Need help setting up monitoring in a VICIdial cluster or troubleshooting audio issues? [Contact ViciStack](https://vicistack.com/contact/) — we've debugged ChanSpy audio paths in every network topology imaginable.*

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-whisper-coaching).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
