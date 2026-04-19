# VICIdial Realtime Report Guide

**Last updated: March 2026 | Reading time: ~24 minutes**

The VICIdial real-time report is the most important screen in your entire call center operation. It's where supervisors live all day. It shows you who's on a call, who's paused, who's been in disposition for 4 minutes when it should take 30 seconds, and whether your campaigns are dialing.

The problem is that most managers who stare at this screen for 8 hours a day only understand about 40% of what it's showing them. They watch the agent status colors and glance at the dial count, but they miss the indicators that tell them something is going wrong before it becomes a crisis.

This is every field on the real-time report, what it means, and what the numbers are telling you about your operation.

---

## Accessing the Real-Time Report

There are two main real-time report scripts in VICIdial:

1. **AST_timeonVDADall.php** — The classic full-page report showing all campaigns and agents. This is what most people call "the real-time report."

2. **realtime_report.php** — A newer wrapper that adds configurable options and a cleaner interface.

Access either one from: **Admin Panel > Reports > Real-Time Main Report**

Or bookmark directly: `https://YOUR-SERVER/vicidial/AST_timeonVDADall.php`

### Report Options

Click the **Report Options** gear icon to configure:

- **Refresh Rate**: How often the page auto-reloads (default 4 seconds). Going below 4 seconds adds database load. For 50+ agent operations, 6-8 seconds is better.
- **Campaign Filter**: Show all campaigns or select specific ones
- **Group Filter**: Filter to specific [user groups](/blog/vicidial-multi-tenant/)
- **Show Monitoring**: Display the listen/whisper/barge buttons next to agents
- **Display Format**: Choose between condensed and expanded views

---

## Agent Status Section

This is the main grid showing every logged-in agent. Each row represents one agent with these columns:

### Status (The Color-Coded Column)

This is the most-watched field. The status and background color tell you exactly what the agent is doing right now:

| Status | Color | Meaning |
|--------|-------|---------|
| **READY** | Green | Agent is waiting for the next call. The dialer is actively looking for calls to send them. |
| **INCALL** | Yellow/Orange | Agent is on a live call with a customer. |
| **PAUSED** | Red/Pink | Agent has paused themselves (or been paused by the system). Not receiving calls. |
| **DISPO** | Blue/Light Blue | Agent is on the disposition screen, selecting a call outcome. |
| **DEAD** | Gray/Dark | Call has ended but agent hasn't started dispositioning yet. Short DEAD times are normal; long ones mean the agent is sitting there. |
| **QUEUE** | Purple | Call is ringing the agent's phone (between dial and answer). |
| **CLOSER** | Teal | Agent is receiving an inbound (closer) call. |
| **3-WAY** | Orange | Agent has initiated a 3-way conference or transfer. |

### Time in Status

The number next to the status shows how many seconds the agent has been in their current state. This is arguably more useful than the status itself.

**Rules of thumb for healthy operations:**
- READY < 30s: Agents shouldn't wait long between calls. If READY times are consistently over 30 seconds, your dial level is too low or your hopper is empty
- INCALL varies: Talk time depends on your campaign type. But if an agent has been INCALL for 45 minutes on an outbound sales campaign, something is wrong
- DISPO < 45s: Disposition should be fast. If agents regularly take 2+ minutes to dispo, they're either confused about disposition codes or using dispo time as an unofficial break
- DEAD < 10s: The gap between call end and dispo screen should be brief. Long DEAD times indicate network lag or agent dawdling
- PAUSED: Depends on pause code. A 15-minute BREAK is fine. A 45-minute BREAK is a conversation

### Calls Today

How many calls this agent has handled today (talk dispositions, not just dials). Compare agents side by side — if one agent has 47 calls and their neighbor has 28, the neighbor is either taking longer calls, pausing more, or gaming the system.

### Talk Time / Wait Time / Pause Time

Cumulative totals for the current session:
- **Talk**: Total seconds the agent spent on calls with customers
- **Wait**: Total seconds the agent spent in READY status waiting for calls
- **Pause**: Total seconds the agent spent in PAUSED status

The ratio of Talk to (Wait + Pause) is your agent utilization rate. For outbound campaigns, a well-tuned operation hits 45-55 minutes of talk time per hour. Below 35 minutes per hour means you're leaving money on the table.

---

## Campaign Statistics Section

Above the agent grid, you'll see per-campaign summary statistics. These are the vital signs of your campaign:

### Calls Waiting

The number of calls currently in queue waiting for an available agent. For outbound campaigns, this should be 0 most of the time (calls connect to agents, not queues). For inbound campaigns, this is your queue depth.

**Inbound alert threshold**: If Calls Waiting exceeds (Agent Count * 0.5), you need more agents or your agents are taking too long per call.

### Agents Logged In / Agents in Call / Agents Ready / Agents Paused

Self-explanatory counts, but the ratios matter:

- **Agents in Call / Agents Logged In** = utilization rate. Target: 55-65% for outbound
- **Agents Paused / Agents Logged In** = pause rate. Should stay under 15% during business hours. Over 20% means too many agents on break simultaneously
- **Agents Ready / Agents Logged In** = idle rate. Some idle is necessary (agents need to be available when calls connect). Over 30% idle means your dial level is too low

### Calls Today / Drops Today

- **Calls Today**: Total dial attempts for this campaign today
- **Drops Today**: Calls that connected but had no agent available (the dreaded "dead air" or "Sorry, please hold"). TCPA and FCC regulations require drops to stay under specific thresholds

**Drop percentage** = Drops Today / Calls Today. The FCC safe harbor threshold is 3%. If your [drop rate](/blog/vicidial-auto-dial-level-tuning/) exceeds 3%, you're in compliance danger AND wasting leads. Lower your dial level in the campaign settings through the Admin GUI.

### Average Hold Seconds

Average time callers wait before connecting to an agent. For outbound, this should be near-zero (agents are waiting for calls, not the other way around). For inbound, under 30 seconds is good, under 60 seconds is acceptable, over 90 seconds and you're losing callers to abandonment.

### Hopper Level

The number of leads pre-loaded in the dial hopper, ready to be dialed. This is the fuel gauge for your campaign.

- **Healthy**: Hopper level > (Agent Count * 2)
- **Warning**: Hopper level < Agent Count
- **Critical**: Hopper level = 0 (campaign will stop dialing)

If the hopper hits zero, check: list status (must be ACTIVE), timezone restrictions (leads may be outside dialable hours), lead filter (too restrictive), and the VDHopper cron job (must be running). See our [dial hopper guide](/blog/vicidial-dial-hopper-guide/) for the full diagnostic.

---

## Monitoring Features

The real-time report isn't just for looking — it's for listening. VICIdial's built-in monitoring lets supervisors interact with live calls directly from this screen.

### Listen (Silent Monitor)

Click the headphone icon next to an agent to silently listen to their active call. Neither the agent nor the customer can hear you. Use this for quality assurance and training.

**Setup requirement**: Monitoring uses a separate SIP channel. Your phone extension must be registered and associated with a user that has monitoring permissions enabled.

### Whisper (Coach)

Click the whisper icon to speak to the agent without the customer hearing. Useful for coaching during live calls — "offer the extended warranty" or "slow down, you're talking too fast."

**Careful**: If the customer can hear the whisper, your audio routing is misconfigured. The whisper audio should only route to the agent leg of the MeetMe conference, not the customer leg. This requires correct conference room configuration.

### Barge-In

Click the barge icon to enter the call as a full participant — both the agent and customer can hear you. Use this sparingly. It's for escalations and emergencies.

### Agent Control from Real-Time Report

Depending on your user permissions, you can also:
- **Pause an agent** remotely (sends them to PAUSED state)
- **Log out an agent** remotely (disconnects their session)
- **Change agent's in-group selection** (redirect which inbound queues they handle)

---

## Reading the Report Like a Pro

Here's how experienced call center managers use the real-time report. Not the what — the how.

### The 10-Second Scan

Every 30 seconds, do a fast scan:
1. **Check drop rate**: Is it creeping above 3%? Lower dial level.
2. **Check hopper**: Is it above zero? If not, something's wrong with lead supply.
3. **Check agent colors**: More than 30% red/pink? Too many agents paused.
4. **Check calls waiting** (inbound): Any queue buildup? Pull agents from outbound or call breaks back.

### The Red Agent Audit

Every 15 minutes, look at every PAUSED agent:
1. What pause code are they on?
2. How long have they been paused?
3. Is it legitimate (BREAK, LUNCH, TRAINING) or suspicious (PERSONAL for 40 minutes)?

Map your findings to your pause code structure. If you don't have meaningful pause codes set up, read our [pause code accountability guide](/blog/vicidial-pause-codes-accountability/).

### The Slow Agent Check

Sort or scan agents by Calls Today count. Identify the bottom 20%:
- Are they pausing too much? (High pause time vs. peers)
- Are they taking too long per call? (High talk time average)
- Are they slow in disposition? (Check DISPO time patterns)

The real-time report gives you the raw data. Pattern recognition is your job.

### The Hourly Campaign Pulse

Every hour, capture these numbers:
- Calls this hour
- Drops this hour
- Average hold time this hour
- Agent count this hour

Compare hour-over-hour. A sudden drop in calls without a corresponding agent decrease means something technical is wrong — hopper issues, carrier problems, or a campaign misconfiguration.

---

## Customizing the Real-Time Report

### Adjusting Refresh Rate

The default 4-second refresh works for most operations. But on large deployments (100+ agents), each refresh hits the database with several queries. If your MySQL server is under load, increasing the refresh to 8 or 10 seconds reduces database impact.

In the Report Options, change the **Refresh Rate** dropdown. Or modify the URL directly:

```
https://YOUR-SERVER/vicidial/AST_timeonVDADall.php?RR=8
```

### Filtering by Campaign

To focus on a single campaign:

```
https://YOUR-SERVER/vicidial/AST_timeonVDADall.php?group=SALES01&RR=6
```

### Multiple Monitor Screens

Most call center operations display the real-time report on a wall-mounted TV or dedicated monitor. For this setup:

1. Create a dedicated read-only user with no admin access
2. Set the user's IP restriction to the monitoring machine's IP
3. Use a kiosk-mode browser (Chrome in kiosk mode: `google-chrome --kiosk URL`)
4. Set the refresh rate to 8-10 seconds

For multiple campaigns on multiple screens, use different URLs with different `group` filters per screen.

---

## Common Real-Time Report Problems

### "My Agents Show as READY but No Calls Are Going Out"

**Check 1**: Hopper level. If zero, no leads are available.
**Check 2**: Campaign status. Must be ACTIVE.
**Check 3**: Dial level. If set to 0, no calls will be placed.
**Check 4**: The `VDauto_dial_CAMPAIGN.pl` cron process. Must be running:

```bash
ps aux | grep VDauto_dial
```

If the cron process isn't running, check if the AST_update scripts are active and the screen or nohup sessions are alive.

### "Agent Status Stuck — Not Updating"

Usually a browser issue or a database connectivity problem.
- Refresh the page manually (Ctrl+Shift+R)
- Check if the agent's session is still valid: look for the agent in `vicidial_live_agents` table
- Verify the `AST_update` cron scripts are running
- Check Apache/httpd error logs for PHP errors

### "Drop Rate Suddenly Spiked"

Drops happen when a call connects but no agent is available within the hold time threshold. Causes:
- Too many agents paused simultaneously
- Dial level too high for available agent count
- Multiple agents just went on break at the same time
- A carrier delivery spike (batch of calls connecting simultaneously)

Immediate fix: Lower the dial level through the Admin GUI until the drop rate stabilizes. Then investigate root cause.

### "The Report Takes Forever to Load"

Real-time report performance is directly tied to MySQL performance. Slow loading means:
- vicidial_live_agents table needs optimization
- Too many concurrent database queries
- MySQL buffer pool undersized
- The report is querying across too many campaigns simultaneously

Try filtering to a single campaign to test. If that loads fast, the issue is query volume across campaigns. If it's still slow, your database needs [maintenance](/blog/vicidial-database-maintenance/).

---

## Beyond the Stock Report

The built-in real-time report is functional but not beautiful. If you need more:

### Grafana Integration

We've documented [building Grafana dashboards for VICIdial](/blog/vicidial-grafana-dashboards/) that pull from the same data the real-time report uses. Grafana gives you historical trending, custom visualizations, and alerting that the stock report can't do.

### Mobile Access

The stock report works on mobile browsers, but it's not optimized for small screens. Some third-party VICIdial panel tools offer mobile-optimized real-time views. Or build a simple mobile dashboard using the Agent API's `agent_status` function to pull the same data.

### API-Driven Custom Dashboards

The [Non-Agent API](/blog/vicidial-api-integration/) has a real-time reporting function that returns agent status data as plain text or JSON. You can build a completely custom dashboard by polling this endpoint:

```bash
curl -s "https://YOUR-SERVER/vicidial/non_agent_api.php?\
source=dashboard&\
user=apiuser&\
pass=API_PASSWORD&\
function=agent_status&\
campaign_id=SALES01&\
header=YES"
```

---

## The Dashboard That Pays for Itself

The real-time report is free, built into VICIdial, and tells you everything you need to know — if you know how to read it. Most centers we audit are losing 10-20% of potential productivity because their managers watch the report but don't act on what it's showing them.

High pause rates go unchallenged. Slow disposition times become the norm. Drop rates hover at 4-5% without anyone adjusting dial levels. The hopper empties and nobody notices for 20 minutes.

---

## Quick Reference: Report URL Parameters

For power users who want to bookmark specific views or embed reports:

```
# Base URL
https://YOUR-SERVER/vicidial/AST_timeonVDADall.php

# Common parameters
?RR=6 # Refresh rate in seconds
&group=SALES01 # Filter to one campaign
&groups=YES # Show user group breakdown
&show_parks=1 # Show parked calls
&NOLEADSalert=1 # Alert when hopper is empty
&Session=Y # Track session data
```

You can combine these for tailored views per supervisor or per TV screen:

```
# Sales floor TV — 8 second refresh, SALES campaigns only
https://YOUR-SERVER/vicidial/AST_timeonVDADall.php?RR=8&group=SALES01&groups=YES&NOLEADSalert=1

# Support desk — 6 second refresh, inbound focus
https://YOUR-SERVER/vicidial/AST_timeonVDADall.php?RR=6&group=SUPPORT01&groups=YES
```

## Extracting Real-Time Data Programmatically

If you need the real-time data in your own application:

```bash
# Non-Agent API — get campaign real-time stats
curl -s "https://YOUR-SERVER/vicidial/non_agent_api.php?\
source=dashboard&\
user=apiuser&\
pass=API_PASSWORD&\
function=agent_status&\
campaign_id=SALES01&\
stage=csv&\
header=YES"
```

The response includes agent user, status, campaign, calls today, and seconds in current status — everything shown on the real-time report, but in a parseable format your code can consume.

For building real-time wallboards: poll this endpoint every 5-10 seconds, parse the CSV, and render in your preferred frontend framework. We've seen centers build custom React and Vue dashboards this way that display exactly the metrics their operation cares about, without the noise of the stock report.

---

**Related reading:**
- [VICIdial Real-Time Report: What Every Field Means and How to Stop Staring at It Wrong](https://vicistack.com/blog/vicidial-realtime-report-guide/)
- [STIR/SHAKEN for VICIdial: The Complete 2026 Implementation Guide](https://vicistack.com/blog/stir-shaken-vicidial-guide/)
- [VICIdial AMD Configuration: The Only Guide That Doesn't Waste Your Time](https://vicistack.com/blog/vicidial-amd-guide/)
- [VICIdial Caller ID Reputation Monitoring and Recovery Guide](https://vicistack.com/blog/vicidial-caller-id-reputation/)
## The Daily Real-Time Report Checklist

Print this out and tape it next to your monitoring screen:

**Every 30 seconds**: Quick scan. Drop rate under 3%? Hopper above zero? Agent colors mostly green/yellow? Good.

**Every 15 minutes**: Pause audit. Any agent paused longer than their scheduled break? Document and follow up.

**Every hour**: Campaign pulse. Calls this hour vs. last hour. Agent count stable? Any campaign with zero dials? Check cron processes:

```bash
# Quick cron health check from SSH
ps aux | grep VDauto_dial | wc -l
# Should match your active campaign count

ps aux | grep AST_VDauto | wc -l
# Should show at least 2-3 processes per server
```

**At shift change**: Verify new agents are logged in, old shift agents are logged out, and no "ghost" sessions are cluttering the report. Ghost sessions happen when agents close their browser without logging out. Clean them up:

```bash
# Check for stale agent sessions
asterisk -rx "meetme list" | wc -l
# Compare to actual logged-in agents on the real-time report
# Stale conferences indicate orphaned sessions
```

**End of day**: Screenshot the final numbers. Calls today, drops today, conversion count. Compare to yesterday. Spot trends [before they](/blog/tcpa-compliance-2026/) become problems.

The [ViciStack engagement](https://vicistack.com/) includes training your operations team on reading the real-time report like a pro. We show them which numbers to watch, when to intervene, and how to make the campaign adjustments that increase agent talk time by 15-25% in the first week. That's the 50% conversion increase — not from magic, but from reading the screen that was in front of them the whole time.

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-realtime-report-guide).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
