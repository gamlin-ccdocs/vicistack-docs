# VICIdial Agent Efficiency Metrics: Stop Guessing Who's Actually Working

**You have 30 agents. Some are crushing it. Some are hiding in pause codes. You probably can't tell which is which without opening 4 different reports. Let's fix that.**

Agent efficiency isn't one number. It's a stack of metrics that, together, tell you who's productive, who's struggling, and who's gaming the system.

## The Only 5 Metrics That Matter

You could track 50 things. You should track 5. Everything else is noise until these 5 are dialed in.

### 1. Agent Talk Time vs. Total Login Time

This is the big one. What percentage of an agent's shift are they actually on calls?

```
Talk Time Ratio = (Total Talk Time / Total Login Time) × 100
```

**Benchmarks:**
- Outbound predictive: 40-55% talk time is good. Above 55% is excellent.
- Inbound queue: 55-70% depending on call volume.
- Below 35%: something is wrong — long pauses, system issues, or avoidance behavior.

**Pull it from VICIdial:**

```
Admin > Reports > Agent Performance Detail
Date range: Last 7 days
Look at: Talk (sec) vs. Login (sec)
```

The `vicidial_agent_log` table has per-status-change granularity if you need it:

```sql
SELECT user,
  SUM(CASE WHEN status='INCALL' THEN pause_sec ELSE 0 END) as talk_secs,
  SUM(pause_sec) as total_secs
FROM vicidial_agent_log
WHERE event_time >= DATE_SUB(NOW(), INTERVAL 7 DAY)
GROUP BY user;
```

### 2. Calls Per Hour

Simple and hard to fake.

```
Calls Per Hour = Total Calls Handled / Hours Logged In
```

**Benchmarks (outbound):**
- Predictive dialing: 15-25 calls/hour
- Progressive/ratio: 8-15 calls/hour
- Preview/manual: 5-10 calls/hour

**Benchmarks (inbound):**
- Depends entirely on average handle time. At 4-minute AHT, theoretical max is 15 calls/hour. Realistic is 10-12.

If one agent is at 8 calls/hour while the team averages 18, that's a conversation — either they're getting longer calls (check AHT) or they're spending too long in disposition/pause.

### 3. After-Call Work Time (ACW)

The gap between hanging up and being ready for the next call. This is where efficiency hides.

```
ACW = Time in DISPO status
```

**Benchmarks:**
- Under 30 seconds: fast, possibly rushing
- 30-60 seconds: normal
- Over 90 seconds: investigate — are they wrapping up the call or checking their phone?

In VICIdial, this is the DISPO pause time. Pull it from the Agent Time Detail report:

```
Admin > Reports > Agent Time Detail
Look at: Dispo column
```

If your average ACW is over 60 seconds across the floor, look at your disposition form. Maybe there are too many fields. Maybe the CRM integration is slow. Sometimes the fix is technical, not behavioral.

### 4. Pause Code Distribution

This is where gaming lives. An agent with 2 hours of "Break" in a 6-hour shift is either taking very long breaks or using the wrong pause code to hide.

**Set up meaningful pause codes:**

| Code | Name | Expected % of Shift |
|------|------|-------------------|
| BREAK | Break | 8-10% |
| LUNCH | Lunch | 6-8% |
| TRAIN | Training | 0-5% |
| TECH | Tech Issue | 0-3% |
| ADMIN | Administrative | 0-5% |
| BATH | Bathroom | 2-4% |

**Red flags:**
- Total pause time > 30% of shift
- Any single pause code > 15% (unless it's lunch)
- Frequent short pauses (5-10 seconds) — agent hitting pause to avoid the next call
- "Tech Issue" code used suspiciously often by one agent

VICIdial tracks every pause code change in `vicidial_agent_log`. The Agent Time Detail report breaks it down by code.

### 5. Conversion Rate (Sales) or First Call Resolution (Support)

Efficiency without effectiveness is pointless. An agent doing 25 calls/hour with a 2% close rate is less valuable than one doing 15 calls/hour with a 6% close rate.

For sales:
```
Conversion Rate = (Sales / Calls Handled) × 100
```

For support:
```
FCR = (Issues Resolved on First Call / Total Issues) × 100
```

Pull conversions from VICIdial sale dispositions:

```
Admin > Reports > Agent Performance Detail
Look at: Sales/Total Calls
```

## Building the Agent Scorecard

Don't dump 5 metrics on managers and hope they figure it out. Build a simple scorecard that ranks agents on a composite score.

**Weighting (outbound sales):**

| Metric | Weight |
|--------|--------|
| Conversion Rate | 35% |
| Calls Per Hour | 25% |
| Talk Time Ratio | 20% |
| ACW Time | 10% |
| Pause Time | 10% |

**Weighting (inbound support):**

| Metric | Weight |
|--------|--------|
| FCR / CSAT | 35% |
| Calls Per Hour | 20% |
| Talk Time Ratio | 20% |
| ACW Time | 15% |
| Pause Time | 10% |

Normalize each metric to a 0-100 scale based on your team's range, multiply by weight, sum it up. Now you have one number per agent per week. Trend it over time.

## The Reports You Should Run Weekly

### 1. Agent Performance Summary

```
Admin > Reports > Agent Performance Detail
Group by: User
Date range: Last 7 days
```

This gives you calls, talk time, pause time, dispo time, and sales per agent. Export to CSV and build your scorecard in a spreadsheet.

### 2. Pause Code Breakdown

```
Admin > Reports > Agent Time Detail
Group by: User + Pause Code
Date range: Last 7 days
```

Flag anyone with total pause > 30% or unusual patterns.

### 3. Hourly Performance

```
Admin > Reports > Agent Performance Detail
Group by: Hour
Date range: Yesterday
```

Identify when your team is hot and when they slump. Most floors have a post-lunch dip from 1-3pm. If it's severe, that's when you schedule coaching, contests, or energy breaks.

## Mistakes We See

**Tracking too many metrics.** If you have a dashboard with 20 KPIs, nobody reads it. Pick 5. Make them visible. Act on them.

**Comparing agents unfairly.** An agent on a cold list and an agent on warm transfers are not in the same game. Segment your metrics by campaign or lead source.

**Punishing without coaching.** Metrics should drive coaching conversations, not just write-ups. "Your ACW is 90 seconds, team average is 45 — let's figure out what's slowing you down" is more productive than "your ACW is too high, fix it."

**Ignoring system factors.** If agents are in DEAD status for 15 seconds between calls because the dialer isn't feeding them fast enough, that's not an agent problem. Check your dial level and hopper settings before blaming people.

## Quick Wins

1. **Set max pause timers.** VICIdial lets you force agents out of pause after X minutes. Set breaks to 15 minutes. Lunch to 30. Agents who need more time can re-pause — but now you have visibility.

2. **Display real-time stats on a wallboard.** Nothing motivates like a public leaderboard. Calls per hour, updated every 60 seconds, on a big screen the whole floor can see. The bottom performers will pick up the pace without anyone saying a word.

3. **Auto-disposition after timeout.** If an agent sits in DISPO for more than 60 seconds, auto-disposition the call as "No Sale" and move them to the next one. This alone can add 5-10 calls per agent per shift.

4. **Review the bottom 3 agents weekly.** Not to fire them — to coach them. Pull their recordings. Listen to 3 calls. Find the pattern. Is it a skill gap or a will gap? Different problems, different fixes.

## What Good Looks Like

A well-tuned outbound sales floor running VICIdial predictive:

- Talk time ratio: 45-55%
- Calls per hour: 18-22
- ACW: 30-45 seconds
- Total pause: < 20% of shift
- Conversion rate: varies by vertical, but trending up month over month

If you're not there, the gap between where you are and where you could be is probably worth six figures per year in either recovered productivity or additional revenue.

**We help call centers get there.** [ViciStack](https://vicistack.com) does VICIdial optimization — agent efficiency, dialer tuning, the whole stack. If your numbers aren't where they should be, [reach out](https://vicistack.com) and we'll take a look.

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-agent-efficiency-metrics).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
