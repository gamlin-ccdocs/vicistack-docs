# VICIdial Agent Performance Tracking

**Last updated: March 2026 | Reading time: ~24 minutes**

You have 40 agents logged into VICIdial right now. Some of them are generating revenue. Some of them are burning your money. And unless you're tracking the right metrics, you have no idea which is which.

Most call center managers track exactly two things: calls per hour and conversion rate. Those matter, but they're lagging indicators — by the time you see a bad conversion rate, the damage is done. The leading indicators — the numbers that predict performance before it shows up in revenue — are buried in VICIdial's reporting system, and almost nobody looks at them.

Here's how to pull the right reports, interpret the data, and build an accountability system that tells you exactly who's working and who's pretending to work.

---

## The Reports That Matter

VICIdial has dozens of built-in reports. Most of them are noise. Five of them drive management decisions.

### 1. Agent Time Detail Report

**Where**: Reports > Agent Time Detail
**Script**: `AST_agent_time_detail.php`

This is the most important report for agent accountability. It breaks down every agent's time into six categories:

| Metric | What It Measures |
|--------|-----------------|
| **WAIT** | Seconds the agent spent in READY state waiting for a call |
| **TALK** | Seconds the agent was on the phone (agent screen in INCALL state) |
| **DISPO** | Seconds the agent spent on the disposition screen |
| **PAUSE** | Seconds the agent was paused (not available for calls) |
| **DEAD** | Seconds after the customer hung up but before disposition screen appeared |
| **CUSTOMER** | Seconds the agent was in a live call with the customer (TALK minus DEAD) |

The critical distinction: **TALK includes DEAD time**. An agent who appears to have 45 minutes of talk time might have 30 minutes of customer conversation and 15 minutes of post-hangup dead air where they're just sitting in the talk screen. CUSTOMER is the real number.

**What to look for**:

**Talk Time Ratio** = CUSTOMER / (WAIT + TALK + DISPO + PAUSE)

This tells you what percentage of logged-in time the agent spent talking to customers. Benchmarks:
- Top performers: 50-60%
- Average performers: 40-50%
- Underperformers: Below 35%
- Gaming the system: Below 25%

**Disposition Time** should be under 30-45 seconds for most campaigns. If agents consistently take 2+ minutes in dispo, they're either:
1. Confused about which disposition code to use (training issue)
2. Using dispo time as a mini-break (accountability issue)
3. Filling out required fields in a CRM that's slow to load (system issue)

**Dead Time** should be under 10 seconds. Extended dead time means the agent is sitting in the talk screen after the customer hung up. Some of this is inevitable (they're finishing a note), but over 15 seconds consistently indicates agents stalling before disposition.

### 2. Agent Performance Detail Report

**Where**: Reports > Agent Performance Detail
**Script**: `AST_agent_performance_detail.php`

This report focuses on call outcomes rather than time allocation:

| Metric | What It Measures |
|--------|-----------------|
| **Calls** | Total calls handled |
| **Talk Time** | Total seconds in INCALL state |
| **Avg Talk** | Average talk time per call |
| **Pause Time** | Total pause time |
| **Wait Time** | Total wait time |
| **Dispositions** | Breakdown by disposition code |

The disposition breakdown is where this report gets interesting. You can see exactly how each agent is dispositioning their calls:

- High SALE/APPT percentage = productive agent
- High NI (Not Interested) percentage = might need script coaching
- High DNC ([Do Not Call](/blog/vicidial-dnc-management/)) percentage = normal for [cold calling](/blog/cold-calling-scripts-templates/)
- High CALLBK (Callback) percentage = could be productive or could be putting off difficult calls
- High HANGUP percentage = agent might be hanging up on people

**Compare disposition patterns between agents**. If Agent A gets 12% SALE on the same list/campaign where Agent B gets 4%, the difference is almost certainly script delivery, objection handling, or attitude — not luck.

### 3. Outbound Calling Report

**Where**: Reports > [Outbound Calling](/blog/vicidial-vs-gohighlevel/) Report
**Script**: `AST_OUTBOUNDcallsReport.php`

Campaign-level view showing:
- Total dials, connects, and drops
- Average talk time per call
- Dials per agent per hour
- Lead-to-conversion ratios

This report helps you compare agent productivity at the campaign level. Sort by agent to see who's making the most dials, who's getting the most connects, and who's converting at the highest rate.

### 4. Agent Time Sheet

**Where**: Reports > Agent Time Sheet
**Script**: `AST_agent_time_sheet.php`

A per-agent timeline showing every status change throughout the day. This is the detective tool. When a supervisor says "Agent X was paused for 45 minutes and claims they were on a scheduled break," pull the time sheet and verify.

The time sheet shows:
- Login/logout times
- Every pause with pause code and duration
- Every call with start time, duration, and disposition
- Gaps in the timeline (network disconnections, browser crashes)

### 5. Pause Code Report

**Where**: Reports > Agent Pause Detail Report

Aggregates pause code usage across all agents. Shows:
- Total time per pause code (BREAK, LUNCH, PERSONAL, TRAINING, etc.)
- Average pause duration per code
- Frequency of each pause code per agent

If agents are spending 2 hours per shift on PERSONAL pauses when your policy allows 30 minutes, this report shows it in black and white.

---

## The Metrics That Predict Revenue

Here are the numbers that correlate with revenue in outbound sales centers, based on data from VICIdial deployments we've managed:

### 1. Dials Per Agent Per Hour (DPAH)

How many calls the system dials on behalf of each agent per hour. This is largely a function of campaign configuration (dial level, hopper performance) rather than agent behavior, but agents influence it through pause time and disposition speed.

**Benchmark**: 60-120 dials per agent per hour for predictive dialing. Below 50 means something is wrong with your campaign config, not your agents.

### 2. Contacts Per Hour (CPH)

How many live conversations (not voicemails, not disconnected numbers) each agent has per hour. This is DPAH multiplied by your [contact rate](/blog/contact-rate-optimization/).

**Benchmark**: 8-15 contacts per hour for B2C [cold calling](/blog/how-we-built-ai-voice-agents/). 3-8 for B2B.

### 3. Talk Time Per Hour

Minutes of actual customer conversation per agent hour. This is the metric that most directly predicts revenue because you can't sell if you're not talking.

**Benchmark**: 35-45 minutes of talk time per hour for well-run outbound operations. Below 30 means excessive pausing, slow disposition, or campaign issues.

### 4. Conversion Rate Per Contact

Sales (or appointments, or qualified leads) divided by live contacts. This is the metric that separates your best agents from your worst.

**Benchmark**: Varies wildly by industry. Insurance leads: 8-15%. Debt settlement: 3-8%. Solar: 5-12%. Home improvement: 10-20%.

The spread between your best and worst agents will typically be 3-4x. If your top agent converts at 15% and your bottom agent converts at 4%, the bottom agent isn't 11 points worse — they're earning a quarter of the revenue for the same seat cost.

### 5. Revenue Per Agent Hour (RPAH)

The number that matters most. Calculate it as:

```
RPAH = CPH × Conversion Rate × Average Deal Value
```

Example: 12 contacts/hour × 10% conversion × $300 average deal = $360/hour in gross revenue per agent.

VICIdial doesn't calculate RPAH natively, but you can derive it from the Agent Performance Detail Report (calls and dispositions) combined with your CRM data (deal values).

---

## Building Agent Scorecards

Raw reports are useful for managers. Scorecards are useful for agents. A well-designed scorecard tells each agent exactly where they stand compared to their peers and what they need to improve.

### Weekly Scorecard Template

Pull data from the Agent Performance Detail and Agent Time Detail reports for the week. For each agent, calculate:

| Metric | Weight | Agent Score | Team Average |
|--------|--------|-------------|-------------|
| Talk Time Per Hour | 25% | 42 min | 38 min |
| Contacts Per Hour | 20% | 11.2 | 9.8 |
| Conversion Rate | 30% | 12.4% | 9.1% |
| Pause Rate | 15% | 11% | 14% |
| Avg Disposition Time | 10% | 28 sec | 42 sec |

**Composite Score** = Weighted sum, normalized to 0-100. Agents above 80 are your A-players. Agents below 50 need immediate intervention.

### How to Pull the Data

VICIdial's reports are designed for browser viewing, not programmatic consumption. To build automated scorecards, you have two options:

**Option 1: Database queries** (read-only access to the MySQL database)

Talk time per agent:
```sql
SELECT user,
 SUM(talk_epoch - wait_epoch) AS total_talk_seconds,
 COUNT(*) AS total_calls,
 ROUND(SUM(talk_epoch - wait_epoch) / COUNT(*), 1) AS avg_talk_seconds
FROM vicidial_agent_log
WHERE event_time >= DATE_SUB(NOW(), INTERVAL 7 DAY)
 AND talk_epoch > 0
GROUP BY user
ORDER BY total_talk_seconds DESC;
```

Disposition breakdown per agent:
```sql
SELECT user, status, COUNT(*) AS count
FROM vicidial_agent_log
WHERE event_time >= DATE_SUB(NOW(), INTERVAL 7 DAY)
GROUP BY user, status
ORDER BY user, count DESC;
```

Pause time per agent:
```sql
SELECT user,
 SUM(pause_epoch - wait_epoch) AS total_pause_seconds,
 ROUND(SUM(pause_epoch - wait_epoch) / 3600, 1) AS pause_hours
FROM vicidial_agent_log
WHERE event_time >= DATE_SUB(NOW(), INTERVAL 7 DAY)
 AND pause_epoch > 0
 AND sub_status != ''
GROUP BY user
ORDER BY total_pause_seconds DESC;
```

**Option 2: API exports**

Use the [Non-Agent API](/blog/vicidial-api-integration/)'s `agent_stats_export` function for programmatic access:

```bash
curl -s "https://YOUR-SERVER/vicidial/non_agent_api.php?\
source=scorecard&\
user=apiuser&\
pass=API_PASSWORD&\
function=agent_stats_export&\
datetime_start=2026-03-19+00:00:00&\
datetime_end=2026-03-26+23:59:59&\
campaign_id=ALL&\
header=YES"
```

This returns agent stats in a format you can parse with a script and feed into a spreadsheet, BI tool, or custom dashboard.

---

## Coaching Based on Data

Numbers without action are just decoration. Here's how to use performance data for actual coaching:

### The 4 Agent Archetypes

**Type 1: High Contacts, High Conversion (Stars)**
- Talk time ratio: 55%+
- Conversion rate: Top 20%
- Action: Leave them alone. Study their approach. Record their calls for training material.

**Type 2: High Contacts, Low Conversion (Busy but Broken)**
- Talk time ratio: 50%+
- Conversion rate: Bottom 30%
- Action: Script coaching. Listen to 5 of their calls. They're getting in front of people but can't close. Usually a script delivery or objection handling problem. These agents have the hardest fix because they think activity equals productivity.

**Type 3: Low Contacts, High Conversion (Selective but Skilled)**
- Talk time ratio: Below 40%
- Conversion rate: Top 30%
- Action: Reduce pause time. These agents close when they talk, but they're not talking enough. Often gaming the system — long dispo times, frequent pauses, or cherrypicking callbacks. Address the behavior, not the skill.

**Type 4: Low Contacts, Low Conversion (Needs Help or Needs to Go)**
- Talk time ratio: Below 35%
- Conversion rate: Bottom 30%
- Action: Two-week improvement plan with daily monitoring. Pair with a Type 1 agent for ride-along listening. If no improvement in two weeks, reassign to inbound or let go.

### The Weekly Review Cadence

Monday morning: Pull the previous week's scorecard data. Identify:
- Top 3 agents (recognition)
- Bottom 3 agents (coaching conversations)
- Biggest week-over-week changes (positive and negative)

Tuesday-Thursday: Run coaching sessions for the bottom 3. Listen to 3-5 of their calls. Identify specific, actionable improvements. "You're rushing the opening — slow down and ask how their day is going" beats "you need to improve your conversion rate."

Friday: Spot-check real-time report adherence throughout the day. Are agents sticking to scheduled break times? Is pause rate under control?

---

## Common Performance Problems and Root Causes

### "Agents have low talk time but the campaign seems fine"

Pull the Agent Time Detail. Break it down:
- **High PAUSE**: Agents are pausing too much. Audit [pause codes](/blog/vicidial-pause-codes-accountability/).
- **High DISPO**: Agents are slow to disposition. Simplify your dispo codes or address dawdling.
- **High WAIT**: The dialer isn't sending calls fast enough. Check dial level, hopper, and available leads.
- **High DEAD**: Agents are sitting in the talk screen after hangup. Set `dead_max` in campaign settings to auto-dispo after a timeout.

### "One agent has way more calls but the same conversion rate"

Check if they're cherry-picking. An agent who dispositions callbacks instead of calling them, or who has unusual disposition patterns (lots of NI/NA), might be skipping difficult leads and focusing on easy ones. Pull their disposition breakdown and compare to team average.

### "New agents have terrible numbers for the first two weeks"

This is normal. New agents need 1-2 weeks to hit their stride on script delivery, objection handling, and system navigation. Track their weekly improvement trajectory rather than absolute numbers. If they're improving 15-20% week-over-week, they'll catch up. If they're flat, intervene.

### "Conversion rate dropped across the whole team"

Usually not an agent problem. Check:
- List quality: Is the current list exhausted or low-quality?
- Lead freshness: Are you dialing old leads that have been recycled 5 times?
- Carrier issues: Are calls connecting but with bad audio? (Check [MOS scores](/blog/voip-mos-score-testing-tools/))
- Compliance changes: Did a new regulation affect your calling window or messaging?

---

## Automating Performance Tracking

Manual report pulling works for small operations. At 50+ agents, you need automation.

### Daily Automated Email Report

Create a cron job that queries the database and sends a summary email:

```bash
#!/bin/bash
# /usr/local/bin/daily-agent-report.sh

YESTERDAY=$(date -d "yesterday" +%Y-%m-%d)
REPORT_FILE="/tmp/agent-report-${YESTERDAY}.txt"

echo "Agent Performance Summary for ${YESTERDAY}" > "$REPORT_FILE"
echo "============================================" >> "$REPORT_FILE"
echo "" >> "$REPORT_FILE"

mysql -u root -p'YOUR_PASSWORD' asterisk -e "
SELECT
 val.user AS agent,
 COUNT(*) AS calls,
 ROUND(SUM(val.talk_epoch - val.wait_epoch)/60, 1) AS talk_minutes,
 ROUND(SUM(val.pause_epoch - val.wait_epoch)/60, 1) AS pause_minutes,
 ROUND(SUM(val.dispo_epoch - val.talk_epoch)/COUNT(*), 0) AS avg_dispo_secs,
 ROUND(
 SUM(CASE WHEN val.status IN ('SALE','XFER','APPT') THEN 1 ELSE 0 END) * 100.0 / COUNT(*),
 1
 ) AS conversion_pct
FROM vicidial_agent_log val
WHERE DATE(val.event_time) = '${YESTERDAY}'
 AND val.talk_epoch > 0
GROUP BY val.user
ORDER BY talk_minutes DESC;
" >> "$REPORT_FILE"

mail -s "VICIdial Agent Report: ${YESTERDAY}" managers@yourcompany.com < "$REPORT_FILE"
```

Cron it:

```
30 6 * * * /usr/local/bin/daily-agent-report.sh
```

### Grafana Agent Dashboard

If you're running [Grafana with VICIdial](/blog/vicidial-grafana-dashboards/), add agent performance panels:
- Talk time distribution (histogram across agents)
- Conversion rate leaderboard (bar chart)
- Pause time trend (line chart per agent over 7 days)
- Agent utilization heatmap (hour-by-hour, day-by-day)

These visualizations make weekly reviews faster and more impactful — showing agents a graph of their performance trend is more motivating than reading them a number.

---

## The Performance Tracking That Pays

Every VICIdial installation has the same data. The difference between high-performing centers and underperforming ones isn't the technology — it's whether anyone uses the data to make decisions.

---

## Handling Performance Data Objections

Agents will push back on tracking. Here's how to address the common objections:

**"You're micromanaging us."**

Frame it as transparency, not surveillance. "We're tracking the same metrics for every agent so management decisions are based on data, not feelings. The numbers protect you from unfair treatment."

**"My conversion rate is low because I get bad leads."**

Pull the data. If every agent on the same campaign and list has similar contact rates but Agent X has lower conversion, the leads aren't the problem. If Agent X has genuinely lower contact rates (fewer human answers per dial), there might be a system issue — check their phone registration, audio quality, and whether calls are routing correctly.

**"I need longer dispo time to fill out the CRM."**

Time the actual CRM workflow. If it takes 20 seconds to fill out the CRM fields but the agent averages 90 seconds in dispo, there's 70 seconds of non-productive time. Either the CRM is slow (fix it) or the agent is stalling (address it).

**"My talk time is low because I'm efficient."**

Partially valid. Some of the best closers have shorter average talk times because they qualify quickly and move on. But short talk time combined with low conversion means the agent is rushing through calls and not engaging. Check call recordings to determine which scenario applies.

---

**Related reading:**
- [VICIdial Agent Performance Tracking: The Reports That Tell You Who's Actually Working](https://vicistack.com/blog/vicidial-agent-performance-tracking/)
- [VICIdial Real-Time Agent Coaching Setup Guide](https://vicistack.com/blog/vicidial-agent-coaching/)
- [VICIdial Agent Efficiency Metrics: Stop Guessing Who's Actually Working](https://vicistack.com/blog/vicidial-agent-efficiency-metrics/)
- [VICIdial Agent Screen Customization Guide](https://vicistack.com/blog/vicidial-agent-screen-customization/)
## The Baseline Problem

You can't evaluate performance without a baseline. Before implementing any tracking changes:

1. Run all reports for 2 full weeks with no changes
2. Calculate averages for each metric across the team
3. Identify the natural spread (standard deviation) for each metric
4. Set targets at the top-25th-percentile of current performance, not at the theoretical maximum

This prevents the "but we've always done it this way" objection — you have two weeks of data showing what "always" looks like in numbers.

The [ViciStack engagement](https://vicistack.com/) includes setting up automated performance tracking, building agent scorecards, and training your management team on the coaching cadence described in this article. Most centers see a 15-30% improvement in agent utilization within the first two weeks — not because the system changed, but because the management approach did.

The agents who were gaming the system get caught. The agents who needed coaching get it. And the agents who were already good get recognized. That's how you increase conversions by 50% — not with magic, but with management.

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-agent-performance-tracking).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
