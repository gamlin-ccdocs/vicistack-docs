# VICIdial Real-Time Agent Coaching Setup Guide

**The complete, practical guide to VICIdial's real-time coaching tools — silent monitor, whisper, barge-in, agent screen monitoring, performance-based coaching workflows, and how to measure whether your coaching actually moves the numbers.**

---

Most call centers have the same coaching problem: they know they should be doing more of it, they have the tools sitting right there in VICIdial, and they still end up coaching reactively — pulling an agent aside after a bad call, reviewing a random recording once a week, or waiting until someone's numbers are so bad they're about to get fired.

That's not coaching. That's damage control.

Real coaching — the kind that actually improves conversion rates, reduces handle times, and retains agents — is systematic. It's data-driven. It happens in real time, not three days after the call. And VICIdial has every tool you need to do it. [Silent monitoring](/glossary/silent-monitor/), [whisper coaching](/glossary/whisper-coaching/), [barge-in](/glossary/barge-in/), agent screen monitoring, real-time status tracking, performance dashboards — it's all built in or easily configured.

The problem isn't the toolset. The problem is that most operations never set it up properly, never build a repeatable coaching workflow around it, and never measure whether the coaching is actually working.

This guide fixes that. We'll walk through every VICIdial coaching capability, show you how to configure each one, build a coaching workflow that identifies who needs help and when, and give you the queries and dashboards to prove it's working. If you're looking to layer AI-powered analysis on top of these fundamentals, see our [AI coaching feature](/features/ai-coaching/) — but start here first.

---

## Understanding VICIdial's Three Coaching Modes

VICIdial provides three real-time call intervention modes, all accessible from the real-time report or the manager interface. Each serves a different purpose in the coaching workflow.

### Silent Monitor (Listen Mode)

[Silent monitoring](/glossary/silent-monitor/) lets a supervisor listen to a live call without the agent or the caller knowing. The supervisor hears both sides of the conversation in real time. No one on the call hears the supervisor.

**When to use it:**

- Routine quality monitoring — listening to calls to evaluate script adherence, tone, objection handling
- Evaluating new agents during their first days on the floor
- Investigating customer complaints about specific agents
- Gathering examples of good calls for training purposes
- Pre-coaching — listening to an agent's calls before a coaching session to identify specific behaviors to address

**How it works technically:** VICIdial uses Asterisk's ChanSpy application under the hood. When a supervisor initiates silent monitor, the system creates a ChanSpy channel pointed at the agent's active call channel. The audio is mixed and sent to the supervisor's phone or softphone. The agent's channel is not modified, so there's no telltale beep or audio artifact.

### Whisper (Coach Mode)

[Whisper coaching](/glossary/whisper-coaching/) lets a supervisor speak to the agent during a live call without the caller hearing. The supervisor hears both sides. The agent hears the supervisor and the caller. The caller only hears the agent.

**When to use it:**

- Real-time coaching during a call — feeding the agent rebuttals, product information, or pricing
- Guiding a new agent through their first live calls
- Helping an agent recover a call that's going sideways
- Walking an agent through a complex compliance disclosure
- Assisting with upsell or cross-sell opportunities in real time

**How it works technically:** Whisper uses Asterisk's ChanSpy with the whisper flag enabled. The supervisor's audio is injected into the agent's channel only, not the bridged caller channel. This means the caller has zero indication a third party is on the line.

**Important limitation:** Whisper requires the agent to be comfortable with receiving real-time instructions while simultaneously talking to a caller. This is a skill that takes practice. Don't throw agents into whisper coaching cold — let them know in advance and start with simple, short prompts ("ask about the warranty," "slow down") before attempting full rebuttal feeding.

### Barge-In (3-Way Conference)

[Barge-in](/glossary/barge-in/) puts the supervisor directly on the call as a three-way conference. All three parties — supervisor, agent, and caller — can hear each other and speak.

**When to use it:**

- Escalation support — the agent needs a manager to close a deal or handle a complaint
- Compliance situations — a supervisor needs to deliver a required disclosure
- Saving a call from an agent who's completely lost
- Taking over a call from an agent who needs to be pulled off the floor
- Handling a hostile or threatening caller

**When NOT to use it:** Barge-in should be rare. If supervisors are barging into calls regularly, your training program has a problem. Barge-in is the emergency brake, not the steering wheel. Over-using it damages agent confidence and creates a culture where agents lean on supervisors instead of developing their own skills.

---

## Configuring Silent Monitor, Whisper, and Barge-In

### Prerequisites

Before setting up coaching modes, verify the following:

**1. Manager user account with appropriate permissions**

The supervisor or manager account needs specific permissions in VICIdial. Go to **Admin → Users → [manager user]** and verify:

- **User Level:** 7 or higher (7 = manager, 8 = admin, 9 = superadmin). Level 6 and below cannot monitor.
- **Agent API access:** Enabled (required for the monitoring functions)
- **Monitor allowed:** Yes
- **Barge-in allowed:** Yes (separate from monitor permission)

```sql
-- Verify user permissions for coaching capabilities
SELECT
    user,
    user_level,
    vdc_agent_api_access
FROM vicidial_users
WHERE user_level >= 7;
```

**2. Asterisk Manager Interface (AMI) configured**

The monitoring functions work through Asterisk's AMI. Verify the manager connection is working on each dialer server:

```bash
# Check AMI is listening
ss -tlnp | grep 5038

# Verify manager credentials from VICIdial
asterisk -rx "manager show connected"

# Test AMI connectivity from the web server
telnet dialer-server-ip 5038
```

If AMI isn't responding, check `/etc/asterisk/manager.conf` for the correct credentials and permitted network ranges.

**3. SIP/IAX phones for supervisors**

Supervisors need a phone or softphone registered to the VICIdial Asterisk system. The monitoring audio is delivered to this phone. This can be:

- A hardware SIP phone at the supervisor's desk
- A software phone (Zoiper, MicroSIP, Linphone) on their computer
- A WebRTC phone if you're running VICIdial's WebRTC configuration

The supervisor's phone extension must be configured in VICIdial under **Admin → Phones**. The phone must be registered and working before initiating monitoring.

### Starting a Monitoring Session

#### From the Real-Time Report

The fastest way to start monitoring is from VICIdial's real-time report at `/vicidial/realtime_report.php`:

1. Locate the agent you want to monitor in the agent detail section
2. Click the agent's row or the monitoring icon next to their name
3. Select the monitoring mode:
   - **MONITOR** — silent listen
   - **BARGE** — barge-in (3-way)
   - **WHISPER** — whisper to agent only
4. Your supervisor phone will ring. Answer it to begin the session.

The monitoring session continues until the supervisor hangs up or the call ends. If the call ends and the agent takes another call, the monitoring session does NOT automatically continue to the next call — you'll need to initiate a new session.

#### From the Manager Interface (API)

For more control, use VICIdial's non-agent API to initiate monitoring programmatically:

```
http://your-vicidial-server/agc/api.php?source=test
  &function=blind_monitor
  &user=manager_user
  &pass=manager_pass
  &phone_login=manager_phone_extension
  &session_id=active_session_id
  &agent_user=agent_to_monitor
  &stage=MONITOR
```

Change `stage` to `WHISPER` or `BARGE` for those modes.

**API parameters:**

| Parameter | Description |
|---|---|
| `phone_login` | The supervisor's phone extension |
| `session_id` | Active manager session ID |
| `agent_user` | Agent ID to monitor |
| `stage` | MONITOR, WHISPER, or BARGE |

This API approach is useful if you're building a custom supervisory interface or integrating coaching tools into a separate application.

#### Switching Modes During a Session

You can switch between monitoring modes mid-call. If you start in silent monitor and realize the agent needs help, you can escalate to whisper without ending the session. From the real-time report, click the mode change option while monitoring is active.

The escalation path typically goes: **Silent Monitor → Whisper → Barge-In**. Going in reverse (barge-in back to silent) is also possible but less common — once the caller knows a supervisor is on the line, dropping to silent feels awkward.

### Asterisk Configuration for Monitoring

VICIdial handles most of the Asterisk configuration automatically, but if monitoring isn't working, check these Asterisk settings:

```
; In /etc/asterisk/features.conf (or featuremap section)
; ChanSpy must be enabled

; In /etc/asterisk/extensions.conf
; VICIdial adds the monitoring context automatically
; But verify the following context exists:
[monitor-agent]
exten => _X.,1,ChanSpy(SIP/${EXTEN},qBw)
```

The ChanSpy flags control behavior:

- `q` — quiet mode (no beep when entering the channel)
- `B` — bridge mode (required for whisper)
- `w` — whisper mode (supervisor audio goes to agent only)

If agents hear a beep when monitoring starts, the `q` flag is missing. This immediately defeats the purpose of silent monitoring. Fix this in the dialplan or VICIdial's Asterisk configuration templates.

### Recording Monitored Calls

VICIdial records calls independently of monitoring sessions. The call recording continues regardless of whether a supervisor is monitoring. However, the default recording captures only the agent-caller audio — the supervisor's whisper or barge-in audio may or may not be included depending on your recording configuration.

To ensure monitored sessions are fully captured (including supervisor audio during barge-in):

- Verify recording is set to **server-side** (MixMon or Monitor application), not client-side
- Barge-in audio is captured because the supervisor joins the call bridge
- Whisper audio may not be captured in the main recording because it's a separate audio stream injected into the agent channel

For [quality control](/glossary/quality-control/) purposes, note which calls were coached during the monitoring session so you can compare coached vs. uncoached performance.

---

## Agent Screen Monitoring

Audio monitoring is half the picture. VICIdial also supports screen monitoring — letting supervisors see what's on an agent's screen in real time. This is critical for identifying:

- Agents not following the script or call flow displayed on screen
- Data entry errors (wrong dispositions, incomplete lead records)
- Call avoidance behavior (agents slow-rolling disposition or deliberately staying in wrap-up)
- Technical issues (frozen screens, error messages, connectivity problems)

### VICIdial's Built-In Screen Monitoring

VICIdial's agent screen is a web application, which means screen monitoring can be accomplished through several methods:

**1. Agent Screen Preview (built-in)**

VICIdial includes an agent screen preview feature accessible from the real-time report. When viewing an agent in the real-time detail, the supervisor can open a read-only view of what the agent's screen shows — the current lead information, call status, script, and disposition options.

This isn't a pixel-perfect screen mirror. It's a reconstructed view of the agent's VICIdial interface state based on database data. It shows:

- Current lead information displayed to the agent
- Active script and script position
- Call status and timer
- Available dispositions
- Agent's current status and queue position

**2. Web-Based Screen Sharing**

For full screen monitoring (including non-VICIdial applications the agent might have open), deploy a lightweight screen-sharing solution:

- **Apache Guacamole** — open-source, web-based remote desktop gateway. Supervisors can view agent desktops through a browser without installing client software.
- **VNC with a web viewer** — noVNC provides browser-based VNC access. Each agent workstation runs a VNC server, and supervisors connect through the web viewer.

For remote agent teams, Guacamole is the best option. It works over standard HTTPS, doesn't require VPN, and supports view-only connections that prevent supervisors from accidentally interacting with the agent's screen.

### Real-Time Agent Status Tracking

Beyond screen monitoring, VICIdial's real-time status tracking gives you a second-by-second view of what every agent is doing. The `vicidial_live_agents` table is the source of truth:

```sql
-- Current agent status with time in state
SELECT
    vla.user,
    vla.campaign_id,
    vla.status,
    vla.pause_code,
    TIMESTAMPDIFF(SECOND, vla.last_state_change, NOW()) as seconds_in_state,
    vla.callerid,
    vla.lead_id,
    vl.phone_number,
    vl.first_name,
    vl.last_name
FROM vicidial_live_agents vla
LEFT JOIN vicidial_list vl ON vla.lead_id = vl.lead_id
ORDER BY vla.status, seconds_in_state DESC;
```

**What to watch for in real-time status:**

| Status Pattern | What It Means | Action |
|---|---|---|
| PAUSED > 10 min, no pause code | Agent may be avoiding calls | Investigate immediately |
| PAUSED with BREAK code > 20 min | Extended break | Check break schedule |
| READY > 3 min | No calls coming in | Dialer issue — check hopper and dial level |
| INCALL > avg talk time * 2 | Unusually long call | May need assistance — silent monitor |
| DISPO > 2 min | Slow disposition | May need training on dispo workflow |

Track these patterns over time. An agent who consistently spends 90+ seconds in DISPO when the team average is 30 seconds is either being thorough with notes (good) or stalling before the next call (bad). Silent monitoring during their dispo phase will tell you which.

---

## Performance Dashboards for Coaching

Effective coaching starts with data. You need to identify which agents need coaching and what specific behaviors to address — before you ever listen to a call. VICIdial's database gives you everything you need to build coaching-focused dashboards.

For a deeper dive into VICIdial's reporting infrastructure and how to set up Grafana dashboards, see our [VICIdial Reporting & Real-Time Monitoring Guide](/blog/vicidial-reporting-monitoring/).

### Key Coaching Metrics

These are the metrics that tell you who needs coaching and what kind of coaching they need.

#### Calls Per Hour

The most basic productivity metric. Measures how many calls an agent handles in an hour of logged-in time.

```sql
SELECT
    val.user,
    COUNT(*) as total_calls,
    ROUND(
        TIMESTAMPDIFF(MINUTE,
            MIN(val.event_time),
            MAX(val.event_time)
        ) / 60.0, 2
    ) as hours_logged,
    ROUND(
        COUNT(*) /
        NULLIF(TIMESTAMPDIFF(MINUTE,
            MIN(val.event_time),
            MAX(val.event_time)
        ) / 60.0, 0), 1
    ) as calls_per_hour
FROM vicidial_agent_log val
WHERE val.event_time >= CURDATE()
    AND val.campaign_id = 'YOUR_CAMPAIGN'
GROUP BY val.user
HAVING hours_logged > 1
ORDER BY calls_per_hour ASC;
```

Sort ascending to find your lowest performers first. An agent handling 8 calls per hour when the team average is 15 has a problem — either excessive pausing, long dispo times, or system issues. This query tells you where to look; monitoring tells you why.

#### Conversion Rate by Agent

The metric that matters most for revenue-generating campaigns:

```sql
SELECT
    vl.user,
    COUNT(*) as total_calls,
    COUNT(CASE WHEN vl.status IN ('SALE','SET','XFER') THEN 1 END) as conversions,
    ROUND(
        COUNT(CASE WHEN vl.status IN ('SALE','SET','XFER') THEN 1 END) /
        NULLIF(COUNT(*), 0) * 100, 2
    ) as conversion_rate,
    ROUND(AVG(vl.length_in_sec), 1) as avg_talk_sec,
    ROUND(STDDEV(vl.length_in_sec), 1) as stddev_talk_sec
FROM vicidial_log vl
WHERE vl.call_date >= DATE_SUB(NOW(), INTERVAL 7 DAY)
    AND vl.user != ''
    AND vl.campaign_id = 'YOUR_CAMPAIGN'
GROUP BY vl.user
HAVING total_calls > 50
ORDER BY conversion_rate ASC;
```

The `stddev_talk_sec` column is underrated. An agent with high standard deviation in talk time is inconsistent — some calls way too short, some way too long. Coaching should focus on consistency: what does a good call look like, and how do we replicate that pattern?

#### Talk Time Distribution

Average talk time hides a lot. Two agents with the same 180-second average could have very different distributions — one might be consistently 170-190 seconds, while the other bounces between 60 and 400 seconds. The consistent agent is likely following a process. The inconsistent one is winging it.

```sql
SELECT
    user,
    COUNT(*) as total_calls,
    ROUND(AVG(length_in_sec), 1) as avg_sec,
    ROUND(MIN(length_in_sec), 1) as min_sec,
    ROUND(MAX(length_in_sec), 1) as max_sec,
    ROUND(STDDEV(length_in_sec), 1) as stddev_sec,
    COUNT(CASE WHEN length_in_sec < 30 THEN 1 END) as under_30s,
    COUNT(CASE WHEN length_in_sec BETWEEN 30 AND 120 THEN 1 END) as '30_to_120s',
    COUNT(CASE WHEN length_in_sec BETWEEN 121 AND 300 THEN 1 END) as '121_to_300s',
    COUNT(CASE WHEN length_in_sec > 300 THEN 1 END) as over_300s
FROM vicidial_log
WHERE call_date >= DATE_SUB(NOW(), INTERVAL 7 DAY)
    AND user != ''
    AND length_in_sec > 0
    AND campaign_id = 'YOUR_CAMPAIGN'
GROUP BY user
HAVING total_calls > 50
ORDER BY stddev_sec DESC;
```

Agents at the top of this list (highest standard deviation) are your primary coaching targets for process consistency. Listen to their short calls and long calls — you'll likely find they're skipping steps on short calls and rambling or losing control on long ones.

#### Agent Utilization (Productive Time)

[Agent utilization](/glossary/agent-utilization/) measures how much of an agent's logged-in time is spent productively (talking to prospects or in dispo) versus waiting or pausing:

```sql
SELECT
    user,
    ROUND(SUM(talk_sec) / 60, 1) as talk_min,
    ROUND(SUM(dispo_sec) / 60, 1) as dispo_min,
    ROUND(SUM(wait_sec) / 60, 1) as wait_min,
    ROUND(SUM(pause_sec) / 60, 1) as pause_min,
    ROUND(
        SUM(talk_sec + dispo_sec) /
        NULLIF(SUM(talk_sec + dispo_sec + wait_sec + pause_sec), 0) * 100, 1
    ) as productive_pct,
    ROUND(
        SUM(pause_sec) /
        NULLIF(SUM(talk_sec + dispo_sec + wait_sec + pause_sec), 0) * 100, 1
    ) as pause_pct
FROM vicidial_agent_log
WHERE event_time >= CURDATE()
    AND campaign_id = 'YOUR_CAMPAIGN'
GROUP BY user
ORDER BY pause_pct DESC;
```

An agent with 35% pause time when the team average is 15% is either taking too many breaks, slow-rolling between calls, or having legitimate system issues. Sorting by `pause_pct` descending puts your biggest coaching opportunities at the top.

#### Disposition Accuracy Check

Agents who consistently mis-disposition calls corrupt your data and your list management. This query identifies agents whose disposition patterns deviate significantly from the team norm:

```sql
SELECT
    user,
    COUNT(*) as total_calls,
    ROUND(COUNT(CASE WHEN status = 'SALE' THEN 1 END) / NULLIF(COUNT(*), 0) * 100, 1) as sale_pct,
    ROUND(COUNT(CASE WHEN status = 'NI' THEN 1 END) / NULLIF(COUNT(*), 0) * 100, 1) as ni_pct,
    ROUND(COUNT(CASE WHEN status = 'CALLBK' THEN 1 END) / NULLIF(COUNT(*), 0) * 100, 1) as callback_pct,
    ROUND(COUNT(CASE WHEN status = 'DNC' THEN 1 END) / NULLIF(COUNT(*), 0) * 100, 1) as dnc_pct,
    ROUND(COUNT(CASE WHEN status = 'NP' THEN 1 END) / NULLIF(COUNT(*), 0) * 100, 1) as np_pct
FROM vicidial_log
WHERE call_date >= DATE_SUB(NOW(), INTERVAL 7 DAY)
    AND user != ''
    AND campaign_id = 'YOUR_CAMPAIGN'
GROUP BY user
HAVING total_calls > 50
ORDER BY user;
```

Compare each agent's disposition distribution against the team average. An agent with 40% "Not Interested" when the team average is 25% might be giving up too quickly. An agent with unusually high DNC dispositions might be using DNC as a way to remove leads they don't want to call back. Both are coaching opportunities.

### Building a Coaching Dashboard in Grafana

Combine the queries above into a Grafana dashboard specifically designed for coaching decisions. The recommended layout:

**Row 1: Team Overview**
- Stat panel: Total agents logged in, average utilization, team conversion rate
- Gauge: Current drop rate (for context — high drop rate affects agent performance)

**Row 2: Agent Performance Table**
- Table panel showing all agents with: calls today, conversion rate (7-day), avg talk time, utilization %, pause %, calls/hour
- Color-code cells: green for above team average, red for significantly below

**Row 3: Coaching Priority List**
- Table filtered to agents more than one standard deviation below the team mean on key metrics
- Sort by the metric that matters most for your campaign (conversion rate for sales, handle time for service)

**Row 4: Trend Charts**
- Time series: Agent conversion rates over the past 30 days (identify declining performance)
- Time series: Agent utilization over the past 30 days (identify emerging behavior issues)

Set the dashboard refresh to 60 seconds. Supervisors should have this open alongside the real-time report. The real-time report tells you what's happening right now. The coaching dashboard tells you who needs help and why.

> **Not sure where your coaching gaps are?** [Request a free ViciStack audit](/free-audit/) and we'll analyze your agent performance data, identify your highest-impact coaching opportunities, and show you exactly where better coaching will move the revenue needle.

---

## Building a Coaching Workflow That Actually Works

Tools are useless without a process. Here's how to build a coaching workflow that systematically improves agent performance instead of reacting to problems after they've already cost you money.

### Step 1: Identify Coaching Targets

Run the coaching dashboard queries weekly (at minimum). Identify agents who fall into these categories:

**Priority 1 — Immediate coaching needed:**
- Conversion rate more than 2 standard deviations below team average
- Calls per hour less than 60% of team average
- Pause time more than double the team average
- Disposition patterns that suggest gaming (unusually high DNC, callback, or not-interested rates)

**Priority 2 — Coaching opportunity:**
- Conversion rate 1-2 standard deviations below team average
- Inconsistent talk times (high standard deviation)
- Utilization declining over the past 2 weeks
- New agents in their first 30 days (everyone needs coaching here)

**Priority 3 — Reinforcement coaching:**
- Top performers — identify what they're doing right and document it for training
- Average performers who could move to above-average with targeted intervention
- Agents returning from extended absence

```sql
-- Identify Priority 1 coaching targets
-- Agents with conversion rate > 2 stddev below team average
SELECT
    sub.user,
    sub.total_calls,
    sub.conversions,
    sub.conversion_rate,
    team.team_avg_rate,
    team.team_stddev,
    ROUND((sub.conversion_rate - team.team_avg_rate) / NULLIF(team.team_stddev, 0), 2) as z_score
FROM (
    SELECT
        user,
        COUNT(*) as total_calls,
        COUNT(CASE WHEN status IN ('SALE','SET','XFER') THEN 1 END) as conversions,
        ROUND(COUNT(CASE WHEN status IN ('SALE','SET','XFER') THEN 1 END) /
              NULLIF(COUNT(*), 0) * 100, 2) as conversion_rate
    FROM vicidial_log
    WHERE call_date >= DATE_SUB(NOW(), INTERVAL 7 DAY)
        AND user != ''
        AND campaign_id = 'YOUR_CAMPAIGN'
    GROUP BY user
    HAVING total_calls > 50
) sub
CROSS JOIN (
    SELECT
        ROUND(AVG(agent_rate), 2) as team_avg_rate,
        ROUND(STDDEV(agent_rate), 2) as team_stddev
    FROM (
        SELECT
            user,
            ROUND(COUNT(CASE WHEN status IN ('SALE','SET','XFER') THEN 1 END) /
                  NULLIF(COUNT(*), 0) * 100, 2) as agent_rate
        FROM vicidial_log
        WHERE call_date >= DATE_SUB(NOW(), INTERVAL 7 DAY)
            AND user != ''
            AND campaign_id = 'YOUR_CAMPAIGN'
        GROUP BY user
        HAVING COUNT(*) > 50
    ) rates
) team
WHERE (sub.conversion_rate - team.team_avg_rate) / NULLIF(team.team_stddev, 0) < -2
ORDER BY z_score ASC;
```

Agents with a z-score below -2 are your Priority 1 coaching targets. They're performing significantly below the team, and the gap is too large to be random variation.

### Step 2: Pre-Coaching Preparation

Before sitting down with an agent, do the work:

**1. Pull their metrics for the past 2 weeks.** Get specific: conversion rate, calls per hour, average talk time, pause percentage, disposition distribution. Compare to team averages and to their own historical performance. Is this a new decline or a consistent pattern?

**2. Silent monitor 5-10 of their calls.** Don't just listen to one bad call. Listen to enough calls to identify patterns. Common patterns you'll find:

- **Rushing the opener** — agent doesn't build rapport, jumps straight to the pitch
- **Weak objection handling** — agent accepts the first "no" instead of working through it
- **Talking too much** — agent doesn't ask questions, doesn't let the prospect talk
- **Poor closing** — agent presents well but never actually asks for the sale
- **Script non-compliance** — agent is off-script, missing required disclosures or key talking points
- **Dead air** — agent can't find information, long pauses while searching the system

**3. Pull 2-3 specific call recordings as examples.** Have concrete examples ready. "Your conversion rate is low" is not coaching. "On this call at 2:14 PM on Tuesday, you got the objection 'I need to think about it' and you said 'okay, thanks for your time' — let's talk about how to handle that differently" is coaching.

**4. Check if there are systemic issues.** Before assuming it's a skill problem, verify:
- Is the agent getting the same lead quality as other agents? (Check list assignment and lead recycling)
- Is the agent's dialer connection working properly? (Check audio quality, latency)
- Did something change in the campaign recently? (New list, new script, pricing change)

### Step 3: The Coaching Session

Keep it structured. A coaching session should be 15-30 minutes and follow this format:

**1. Data review (3 minutes).** Show the agent their metrics compared to team averages. No judgment — just numbers. "Here's where you are, here's where the team is."

**2. Call review (10-15 minutes).** Play the specific recordings you identified. Pause at key moments. Ask the agent what they were thinking. Often, agents know what went wrong but don't know how to fix it.

**3. Skill building (5-10 minutes).** Focus on ONE specific behavior to change. Not five things — one thing. Role-play the scenario. Have the agent practice the new approach with you. For objection handling, go through the specific objection they're struggling with until they can deliver the rebuttal naturally.

**4. Action items (2 minutes).** One measurable goal for the next week. "Increase your conversion rate from 3.2% to 4.0%" is measurable. "Do better on calls" is not.

### Step 4: Post-Coaching Monitoring

This is where most coaching programs fail. The session happens, the agent nods along, goes back on the floor, and nothing changes because nobody follows up.

**Within 24 hours of the coaching session:**
- Silent monitor 3-5 of the agent's calls
- Check if the specific behavior discussed in coaching is being applied
- If yes, send a quick positive message (Slack, email, whatever your team uses)
- If no, consider a brief follow-up conversation — the behavior change didn't stick

**At 3 days:**
- Pull updated metrics and compare to pre-coaching baseline
- Silent monitor 2-3 more calls
- Are you seeing the behavior change?

**At 7 days:**
- Full metrics comparison: pre-coaching week vs. post-coaching week
- If metrics improved, acknowledge it publicly (team leaderboards, shout-outs)
- If metrics didn't improve, schedule a follow-up coaching session focused on the same behavior

**At 30 days:**
- Final assessment: did coaching produce a sustained improvement?
- Document the outcome (improved, partially improved, no change)
- Use this data to evaluate and refine your coaching approach

### Step 5: Coaching for New Agents (Onboarding)

New agent coaching follows a more intensive schedule. During the first two weeks, supervisors should be monitoring new agents on the majority of their calls. For a complete onboarding process, see our [Call Center Agent Onboarding Checklist](/blog/call-center-agent-onboarding/).

**Days 1-3: Nesting**
- Supervisor monitors every call via whisper
- Active whisper coaching on every call — feeding rebuttals, guiding the flow
- End-of-day review of all calls with the agent
- Goal: agent can get through the script without major supervisor intervention

**Days 4-7: Supervised independence**
- Silent monitor 50% of calls
- Whisper only when the agent is about to lose a call
- Brief mid-day and end-of-day check-ins
- Goal: agent handles routine calls independently, escalates appropriately

**Days 8-14: Spot monitoring**
- Silent monitor 10-20% of calls (random sampling)
- No whisper unless specifically requested by the agent
- Daily metrics review
- Goal: agent is performing within 80% of team average

**After 30 days:**
- Transition to standard coaching cadence (same as tenured agents)
- If agent is still significantly below team average after 30 days of intensive coaching, have a frank conversation about fit

---

## Measuring Coaching Effectiveness

If you can't measure it, you can't improve it. Here's how to prove that your coaching program is actually working — or identify that it isn't so you can fix your approach.

### Before/After Metrics Query

Run this query for each agent who has gone through a coaching session. Replace the dates with the coaching session date:

```sql
-- Compare agent performance: 7 days before coaching vs. 7 days after
SELECT
    'Before Coaching' as period,
    COUNT(*) as total_calls,
    COUNT(CASE WHEN status IN ('SALE','SET','XFER') THEN 1 END) as conversions,
    ROUND(COUNT(CASE WHEN status IN ('SALE','SET','XFER') THEN 1 END) /
          NULLIF(COUNT(*), 0) * 100, 2) as conversion_rate,
    ROUND(AVG(length_in_sec), 1) as avg_talk_sec,
    ROUND(STDDEV(length_in_sec), 1) as stddev_talk_sec
FROM vicidial_log
WHERE user = 'AGENT_ID'
    AND call_date >= '2026-03-04' AND call_date < '2026-03-11'
    AND campaign_id = 'YOUR_CAMPAIGN'

UNION ALL

SELECT
    'After Coaching' as period,
    COUNT(*) as total_calls,
    COUNT(CASE WHEN status IN ('SALE','SET','XFER') THEN 1 END) as conversions,
    ROUND(COUNT(CASE WHEN status IN ('SALE','SET','XFER') THEN 1 END) /
          NULLIF(COUNT(*), 0) * 100, 2) as conversion_rate,
    ROUND(AVG(length_in_sec), 1) as avg_talk_sec,
    ROUND(STDDEV(length_in_sec), 1) as stddev_talk_sec
FROM vicidial_log
WHERE user = 'AGENT_ID'
    AND call_date >= '2026-03-11' AND call_date < '2026-03-18'
    AND campaign_id = 'YOUR_CAMPAIGN';
```

**What to look for:**
- **Conversion rate increase:** Direct measure of coaching impact on the target skill
- **Talk time change:** If coaching focused on building rapport, talk time should increase slightly. If it focused on efficiency, it might decrease.
- **Standard deviation decrease:** If coaching focused on consistency, the spread in talk times should narrow
- **Call volume change:** Did coaching affect the agent's pace? (Sometimes fixing one issue temporarily slows agents down as they focus on the new behavior)

### Team-Wide Coaching Impact

Track the aggregate impact of your coaching program across all agents:

```sql
-- Monthly coaching effectiveness: compare team metrics month-over-month
SELECT
    DATE_FORMAT(call_date, '%Y-%m') as month,
    COUNT(DISTINCT user) as active_agents,
    COUNT(*) as total_calls,
    COUNT(CASE WHEN status IN ('SALE','SET','XFER') THEN 1 END) as total_conversions,
    ROUND(COUNT(CASE WHEN status IN ('SALE','SET','XFER') THEN 1 END) /
          NULLIF(COUNT(*), 0) * 100, 2) as team_conversion_rate,
    ROUND(AVG(length_in_sec), 1) as avg_talk_sec,
    ROUND(STDDEV(length_in_sec), 1) as team_stddev_talk,
    ROUND(
        STDDEV(
            CASE WHEN user != '' THEN
                (SELECT COUNT(CASE WHEN vl2.status IN ('SALE','SET','XFER') THEN 1 END) /
                       NULLIF(COUNT(*), 0) * 100
                 FROM vicidial_log vl2
                 WHERE vl2.user = vicidial_log.user
                   AND DATE_FORMAT(vl2.call_date, '%Y-%m') = DATE_FORMAT(vicidial_log.call_date, '%Y-%m'))
            END
        ), 2
    ) as agent_performance_spread
FROM vicidial_log
WHERE call_date >= DATE_SUB(NOW(), INTERVAL 6 MONTH)
    AND user != ''
    AND campaign_id = 'YOUR_CAMPAIGN'
GROUP BY DATE_FORMAT(call_date, '%Y-%m')
ORDER BY month;
```

A good coaching program should show two trends over time:

1. **Team conversion rate increasing** — the overall average goes up
2. **Performance spread decreasing** — the gap between top and bottom performers narrows (bottom performers improve, bringing the floor up)

If you see the average increasing but the spread staying the same, your coaching is only reaching agents who would have improved anyway. If the spread is decreasing but the average isn't moving, your top performers are getting worse (possibly because supervisors are spending all their time on bottom performers and neglecting skill maintenance for the A-players).

### ROI Calculation

Coaching costs money — supervisor time is expensive. Calculate whether it's paying off:

```
Coaching ROI = (Revenue gain from improved conversions - Cost of coaching time) / Cost of coaching time

Revenue gain = (Post-coaching conversion rate - Pre-coaching conversion rate) * Total calls * Revenue per conversion

Coaching cost = Supervisor hours * Supervisor hourly cost (fully loaded)
```

**Example:** A supervisor spends 4 hours per week coaching 5 agents (45 minutes of prep + 15 minutes of session per agent). Those 5 agents collectively handle 2,000 calls per week. After coaching, their conversion rate improves from 3.5% to 4.2%. Revenue per conversion is $50.

- Revenue gain per week: (0.042 - 0.035) * 2,000 * $50 = $700
- Coaching cost per week: 4 hours * $35/hour = $140
- ROI: ($700 - $140) / $140 = 400%

That's a 4x return. And this is conservative — it doesn't account for reduced agent turnover (coached agents stay longer), improved customer satisfaction, or the compounding effect of sustained improvement.

---

## Advanced Coaching Techniques

### Peer Coaching and Call Shadowing

Not all coaching needs to come from supervisors. Pair high-performing agents with struggling agents and let them listen to each other's calls. VICIdial's silent monitor isn't limited to manager accounts — you can grant monitoring permissions to senior agents.

Set up a peer coaching program:

1. Create a user level (Level 6 or 7) with monitor permissions but not barge-in
2. Assign top performers as "peer coaches" — 2-3 hours per week of monitoring time
3. Peer coaches listen to their assigned agent's calls and provide feedback
4. Supervisors review peer coach notes and spot-check the coaching quality

This scales your coaching capacity dramatically. One supervisor can coach 5-8 agents per week. Add 4 peer coaches and you've doubled or tripled your coaching coverage.

### AI-Augmented Coaching

The future of call center coaching isn't replacing human coaches — it's giving them better data. AI tools can analyze every call automatically, flagging specific moments that need coaching attention. Instead of randomly sampling calls, supervisors can go straight to the calls and moments that matter.

For details on how AI-powered quality analysis works with VICIdial, see our [AI quality control guide](/blog/ai-call-center-quality-control/) and the [AI coaching feature page](/features/ai-coaching/). The key capabilities:

- **Automatic call scoring** — every call gets a quality score based on script adherence, sentiment, and outcome
- **Keyword and phrase detection** — flag calls where required disclosures were missed or prohibited language was used
- **Sentiment analysis** — identify calls where the customer or agent became frustrated
- **Talk-to-listen ratio** — automatically flag agents who are talking too much and not listening enough

AI doesn't replace the coaching conversation. It replaces the hours of random call listening that supervisors do to find coaching opportunities. See our [AI quality control feature](/features/ai-quality-control/) for implementation details.

### Calibration Sessions

Calibration ensures that all supervisors and coaches are evaluating calls consistently. Without calibration, one supervisor might rate a call as "excellent" while another rates the same call as "needs improvement."

Run monthly calibration sessions:

1. Select 3-5 recorded calls that represent a range of quality levels
2. Have all supervisors and coaches independently score each call using your QA scorecard
3. Compare scores and discuss discrepancies
4. Agree on scoring standards and document edge cases
5. Track inter-rater reliability over time — the goal is consistency, not perfection

### Coaching Documentation

Track every coaching session in a simple log. At minimum, record:

- Agent name and ID
- Date of coaching session
- Specific behavior addressed
- Pre-coaching metric (e.g., "conversion rate 2.8% over past 7 days")
- Action item assigned
- Follow-up date
- Post-coaching metric (filled in at follow-up)
- Outcome (improved / partially improved / no change)

This documentation serves three purposes: it holds coaches accountable for follow-up, it provides data to evaluate the coaching program itself, and it creates a paper trail for HR purposes if performance issues escalate beyond coaching.

> **Want help building a coaching program that moves your numbers?** [Schedule a free ViciStack audit](/free-audit/) — we'll review your current agent performance data, identify your top coaching opportunities, and help you build a repeatable process that delivers measurable ROI.

---

## Common Coaching Pitfalls and How to Avoid Them

### Pitfall 1: Coaching Everyone the Same Way

New agents need different coaching than tenured agents. Agents struggling with product knowledge need different coaching than agents struggling with call control. Don't run the same coaching playbook for everyone.

**Fix:** Categorize coaching needs into buckets — product knowledge, objection handling, call control, compliance, efficiency — and tailor your approach. Use the diagnostic queries above to identify which bucket each agent falls into.

### Pitfall 2: Coaching Without Data

"I listened to a call and it didn't sound great" is not a coaching plan. Without data, coaching becomes subjective, inconsistent, and impossible to measure.

**Fix:** Always start with metrics. Pull the numbers before you pull an agent off the floor. Use the z-score query to identify who actually needs coaching versus who's just having a bad day.

### Pitfall 3: Trying to Fix Everything at Once

Giving an agent a list of 10 things to improve guarantees they'll improve at none of them. Human behavior change works one habit at a time.

**Fix:** Pick the ONE behavior that will have the biggest impact on the target metric. Coach that until it sticks (usually 2-3 weeks). Then move to the next one.

### Pitfall 4: No Follow-Up

The coaching session is 10% of the work. The follow-up — monitoring, measuring, reinforcing — is 90%. Most coaching programs fail here.

**Fix:** Schedule the follow-up before ending the coaching session. Put it in the calendar. Run the before/after query. Close the loop.

### Pitfall 5: Only Coaching Bottom Performers

Your middle performers have the highest coaching ROI. Moving an agent from the 30th percentile to the 50th percentile is harder than moving an agent from the 50th to the 70th. Your mid-tier agents have the skills — they just need refinement.

**Fix:** Allocate coaching time proportionally: 40% to bottom performers, 40% to middle performers, 20% to top performers (for skill reinforcement and leadership development).

### Pitfall 6: Using Barge-In as a Crutch

If supervisors are barging into calls multiple times per shift, you have a training problem, not a coaching problem. Barge-in should be reserved for genuine escalations.

**Fix:** Track barge-in frequency. If a supervisor is barging into more than 2-3 calls per day, investigate why. If it's always the same agents, those agents need intensive nesting-style coaching (or a reassessment of their readiness for the floor).

---

## Frequently Asked Questions

### How do I enable silent monitoring in VICIdial?

Silent monitoring is available by default for manager-level users (Level 7+). Verify the user has `monitor_allowed` set to Yes in their user profile (**Admin → Users → [user]**). The supervisor also needs a registered phone or softphone connected to VICIdial's Asterisk system. To start monitoring, go to the real-time report, find the agent, and click the MONITOR option. The supervisor's phone will ring — answer it to begin listening. If monitoring fails, check that the AMI (Asterisk Manager Interface) is running and accessible on port 5038, and that the ChanSpy application is available in your Asterisk dialplan.

### Can agents tell when they're being monitored?

Not if it's configured correctly. VICIdial uses Asterisk's ChanSpy in quiet mode, which suppresses the default beep that ChanSpy normally plays when entering a channel. If agents hear a beep, the `q` (quiet) flag is missing from the ChanSpy configuration — check your Asterisk dialplan. That said, experienced agents often know monitoring happens and may assume they're being listened to at any time. This is actually desirable — the possibility of monitoring improves compliance even when no one is actually listening.

### What's the difference between whisper and barge-in?

[Whisper](/glossary/whisper-coaching/) allows the supervisor to speak to the agent without the caller hearing. The caller has no idea a third party is involved. [Barge-in](/glossary/barge-in/) puts the supervisor on the call as a full participant — all three parties can hear and speak to each other. Use whisper for real-time coaching during a call (feeding rebuttals, guidance). Use barge-in for escalations where the supervisor needs to interact directly with the caller (closing a deal, handling a complaint, delivering required disclosures).

### How often should I coach each agent?

New agents (first 30 days): daily monitoring, formal coaching sessions 2-3 times per week. Struggling agents (Priority 1): formal coaching session weekly with daily monitoring follow-up. Average agents: formal coaching session every 2-3 weeks, spot monitoring weekly. Top performers: monthly skill reinforcement sessions. Adjust frequency based on results — if an agent improves rapidly, reduce frequency. If improvement stalls, increase it. The key is consistent follow-up regardless of session frequency.

### How many agents can one supervisor effectively coach?

A full-time supervisor can typically maintain active coaching relationships with 12-15 agents. This assumes about 45 minutes of total time per agent per week (prep, monitoring, session, follow-up). For intensive coaching (new agents, Priority 1 performers), the ratio drops to 5-8 agents because each one needs more monitoring and more frequent sessions. Scale coaching capacity with peer coaching programs — train your top agents to monitor and provide feedback, and you can double your coverage without adding supervisory headcount.

### Can I monitor remote agents with VICIdial?

Yes. VICIdial's monitoring features work the same regardless of agent location — the monitoring happens at the Asterisk level, not the agent's physical location. As long as the remote agent is logged into VICIdial and their SIP phone is registered, you can silent monitor, whisper, and barge-in exactly as you would with an on-site agent. For screen monitoring of remote agents, use a web-based solution like Apache Guacamole or a VNC web viewer. The supervisor accesses everything through a browser — no VPN required if you configure it over HTTPS.

### How do I measure whether my coaching program is working?

Track three metrics over time: (1) **Team conversion rate** — should trend upward as coaching improves agent skills; (2) **Performance spread** — the standard deviation of agent conversion rates should decrease as bottom performers improve; (3) **Coaching ROI** — calculate the revenue gain from improved conversions minus the cost of supervisor time spent coaching. Run the before/after query from this guide for every coached agent and track the results in a coaching log. If you're not seeing measurable improvement after 60 days of consistent coaching, the problem is usually in the coaching method (too many behaviors at once, no follow-up) rather than the agents themselves.

### What metrics should I show agents during a coaching session?

Show them their own numbers compared to the team average — never compared to specific other agents by name. The key metrics to display: conversion rate, calls per hour, average talk time, and pause percentage. Keep it to 3-4 numbers maximum. Then show them the specific metric you're coaching on, with a clear target: "Your conversion rate is 3.2%, team average is 4.8%, your target for next week is 3.8%." Avoid overwhelming agents with data. The goal is to make the coaching concrete and measurable, not to drown them in dashboards. After the session, agents should be able to articulate exactly one thing they're going to do differently and exactly what number they're trying to move.

### Does VICIdial support recording the whisper audio during coached calls?

By default, VICIdial's call recordings capture the agent-caller audio stream. Whisper audio exists on a separate channel injected into the agent's earpiece, and it is not always included in the standard recording. Barge-in audio, however, is captured because the supervisor joins the actual call bridge. If you need to record whisper sessions for QA review of your coaching itself, configure server-side recording with MixMon and verify that the mixed audio includes all channels. Alternatively, have supervisors take timestamped notes during whisper sessions documenting what guidance was provided — this is often more useful than the audio anyway, since whisper coaching tends to be short phrases that lack context without the coaching plan.

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-agent-coaching).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
