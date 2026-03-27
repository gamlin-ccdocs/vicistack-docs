# Workforce Management for Call Centers: Erlang C, Schedule Adherence, and the Forecasting Math That Keeps You Staffed

**Last updated: March 2026 | Reading time: ~27 minutes**

A 2025 benchmark spanning 38 countries found that 99% of WFM practitioners said workforce management is essential to business success. That is practically unanimous -- and yet most call centers under 100 agents do WFM by gut feel.

The floor supervisor eyeballs the queue, sends someone to lunch when it looks slow, and scrambles for bodies when the hold times spike. It works until it doesn't, and when it doesn't, you are either overstaffed (burning payroll on agents who sit in READY doing nothing) or understaffed (burning customers who hang up after 4 minutes on hold).

Organizations that implement proper WFM see 15-25% reduction in labor costs while simultaneously improving service levels. That is not a tradeoff -- that is getting both sides of the equation right at the same time.

This guide covers the actual [math behind](/blog/contact-rate-optimization/) [call center staffing](/blog/call-center-staffing-formula/): Erlang C calculations, shrinkage factors, schedule adherence tracking, forecasting from historical data, and the VICIdial-specific tools to make it work. No six-figure WFM platform required.

## Erlang C: The Staffing Math That Actually Works

The Erlang C formula was invented by Danish mathematician Agner Erlang in 1917 to solve telephone traffic problems. Over a century later, it is still the standard for call center staffing calculations. Not because nobody has tried to improve it, but because it works.

### The Core Concept

Erlang C answers one question: given a specific call volume and average handle time, how many agents do you need to meet a target service level?

The inputs:
- **Call volume** -- number of calls per time interval (usually 30 minutes)
- **Average Handle Time (AHT)** -- talk time plus after-call work, in seconds
- **Service Level target** -- percentage of calls answered within a time threshold (e.g., 80% of calls answered within 20 seconds)

The output:
- **Minimum agents required** to meet that service level

### Worked Example

Let us walk through a real calculation.

**Inputs:**
- 200 calls per hour (100 per 30-minute interval)
- Average Handle Time: 180 seconds (3 minutes)
- Service Level target: 80% of calls answered within 20 seconds

**Step 1: Calculate Traffic Intensity (Erlangs)**

```
Traffic Intensity = (Calls per interval × AHT) / Interval duration in seconds
Traffic Intensity = (100 × 180) / 1800 = 10 Erlangs
```

10 Erlangs means you need the equivalent of 10 agents continuously busy just to handle the call volume. But that gives you zero buffer -- every agent is on a call every second, and any new call goes to queue.

**Step 2: Iterate Agent Count Until Service Level Is Met**

You test increasing numbers of agents against the Erlang C formula until the calculated service level meets your target:

| Agents | Service Level | Prob of Waiting | Avg Speed of Answer | Occupancy |
|:---:|:---:|:---:|:---:|:---:|
| 11 | 39% | 67% | 33 sec | 91% |
| 12 | 64% | 43% | 16 sec | 83% |
| 13 | 80% | 28% | 8 sec | 77% |
| 14 | 89% | 17% | 5 sec | 71% |
| 15 | 94% | 10% | 3 sec | 67% |

At 13 agents, you hit 80% service level -- your target. But look at the occupancy column. At 11 agents, occupancy is 91%. That means agents are on calls 91% of the time with almost no breathing room. Sustained occupancy above 85% leads to [burnout](/blog/call-center-agent-burnout/), increased handle times, higher error rates, and turnover.

At 13 agents, occupancy drops to 77% -- a sustainable range. At 14 agents, you exceed your service level target with comfortable occupancy.

**Step 3: Add Shrinkage**

Agents are not available 100% of their scheduled time. They take breaks, attend meetings, do training, call in sick, and go on vacation. The percentage of scheduled time that does not produce agent availability is called shrinkage.

Industry average shrinkage is 30%, broken down roughly as:

| Shrinkage Category | Percentage |
|---|:---:|
| Breaks (lunch + micro) | 10-12% |
| After-call work | 5-8% |
| Meetings and coaching | 3-5% |
| Training | 2-4% |
| Absenteeism (sick, PTO) | 5-8% |
| System downtime | 1-2% |
| **Total** | **26-39%** |

Apply shrinkage to your raw agent count:

```
Agents needed = Raw agents / (1 - Shrinkage rate)
Agents needed = 13 / (1 - 0.30) = 13 / 0.70 = 18.6 ≈ 19 agents
```

You need 19 scheduled agents to have 13 actually available and working the phones at any given time.

### Quick Erlang C Calculator Script

If you want to run these calculations yourself without a spreadsheet, here is a Python implementation:

```python
#!/usr/bin/env python3
"""erlang_c.py - Call center staffing calculator"""

import math
from functools import lru_cache

@lru_cache(maxsize=1024)
def erlang_c(agents, traffic):
    """Calculate Erlang C probability of waiting."""
    if agents <= traffic:
        return 1.0  # system is overloaded

    # Erlang B (probability of blocking)
    inv_b = 1.0
    for i in range(1, agents + 1):
        inv_b = 1.0 + inv_b * i / traffic
    erlang_b = 1.0 / inv_b

    # Erlang C = Erlang B / (1 - rho * (1 - Erlang B))
    rho = traffic / agents
    ec = erlang_b / (1.0 - rho * (1.0 - erlang_b))
    return min(ec, 1.0)

def calculate_service_level(agents, traffic, target_time, aht):
    """Calculate service level for given parameters."""
    pw = erlang_c(agents, traffic)
    rho = traffic / agents
    sl = 1.0 - pw * math.exp(-(agents - traffic) * target_time / aht)
    return max(0.0, min(1.0, sl))

def find_agents_needed(calls_per_interval, aht_seconds, interval_seconds,
                       target_sl, target_time, shrinkage=0.30,
                       max_occupancy=0.85):
    """Find minimum agents to meet service level and occupancy targets."""
    traffic = (calls_per_interval * aht_seconds) / interval_seconds

    for agents in range(int(traffic) + 1, int(traffic) + 100):
        sl = calculate_service_level(agents, traffic, target_time, aht_seconds)
        occupancy = traffic / agents

        if sl >= target_sl and occupancy <= max_occupancy:
            raw_agents = agents
            scheduled = math.ceil(raw_agents / (1 - shrinkage))
            return {
                "traffic_erlangs": round(traffic, 1),
                "raw_agents": raw_agents,
                "service_level": round(sl * 100, 1),
                "occupancy": round(occupancy * 100, 1),
                "prob_waiting": round(erlang_c(agents, traffic) * 100, 1),
                "shrinkage": shrinkage,
                "scheduled_agents": scheduled
            }

    return None

# Example: 100 calls per 30 min, 3 min AHT, 80/20 service level
result = find_agents_needed(
    calls_per_interval=100,
    aht_seconds=180,
    interval_seconds=1800,
    target_sl=0.80,
    target_time=20,
    shrinkage=0.30
)

if result:
    print(f"Traffic intensity:   {result['traffic_erlangs']} Erlangs")
    print(f"Raw agents needed:   {result['raw_agents']}")
    print(f"Service level:       {result['service_level']}%")
    print(f"Occupancy:           {result['occupancy']}%")
    print(f"Prob of waiting:     {result['prob_waiting']}%")
    print(f"With {result['shrinkage']*100:.0f}% shrinkage: {result['scheduled_agents']} scheduled agents")
```

Run this for every 30-minute interval in your day. The output tells you exactly how many agents you need scheduled for each time slot.

## Forecasting: Predicting Tomorrow's Call Volume

Erlang C tells you how many agents you need *if you know the call volume*. Forecasting tells you what the call volume will be. Get the forecast wrong and your staffing will be wrong regardless of how perfect your Erlang C math is.

### Pulling Historical Data from VICIdial

Start with 8-12 weeks of historical data broken into 30-minute intervals:

```sql
SELECT
    DATE(call_date) AS call_day,
    DAYOFWEEK(call_date) AS day_of_week,
    FLOOR(HOUR(call_date) * 2 + MINUTE(call_date) / 30) AS interval_id,
    CONCAT(
        LPAD(HOUR(call_date), 2, '0'), ':',
        IF(MINUTE(call_date) < 30, '00', '30')
    ) AS interval_start,
    COUNT(*) AS call_count,
    AVG(length_in_sec) AS avg_talk_time,
    AVG(length_in_sec + 30) AS est_aht  # add 30s for after-call work
FROM vicidial_closer_log
WHERE call_date >= DATE_SUB(NOW(), INTERVAL 12 WEEK)
GROUP BY call_day, interval_id
ORDER BY call_day, interval_id;
```

For outbound campaigns, pull from `vicidial_log` instead and focus on the connected calls:

```sql
SELECT
    DATE(call_date) AS call_day,
    DAYOFWEEK(call_date) AS day_of_week,
    FLOOR(HOUR(call_date) * 2 + MINUTE(call_date) / 30) AS interval_id,
    COUNT(*) AS total_dials,
    SUM(CASE WHEN status NOT IN ('NA','B','DC','N','NP','AFTHRS')
        THEN 1 ELSE 0 END) AS connected_calls,
    AVG(CASE WHEN length_in_sec > 0 THEN length_in_sec ELSE NULL END) AS avg_talk_time
FROM vicidial_log
WHERE call_date >= DATE_SUB(NOW(), INTERVAL 12 WEEK)
GROUP BY call_day, interval_id
ORDER BY call_day, interval_id;
```

### Building the Forecast

The simplest effective forecasting method is a **weighted moving average** that accounts for day-of-week patterns:

1. Calculate the average call volume for each 30-minute interval, grouped by day of week
2. Weight recent weeks more heavily (last 4 weeks get 60% weight, prior 8 weeks get 40%)
3. Apply known adjustments for holidays, marketing campaigns, or seasonal patterns

```python
#!/usr/bin/env python3
"""forecast.py - Call volume forecast from historical data"""

import json
from collections import defaultdict

def build_forecast(historical_data, forecast_weeks=1):
    """Build a weighted forecast from historical interval data.

    historical_data: list of dicts with day_of_week, interval_id, call_count, week_num
    """
    # Group by (day_of_week, interval_id)
    intervals = defaultdict(list)
    max_week = max(d["week_num"] for d in historical_data)

    for d in historical_data:
        key = (d["day_of_week"], d["interval_id"])
        weeks_ago = max_week - d["week_num"]
        weight = 1.5 if weeks_ago < 4 else 1.0
        intervals[key].append({
            "calls": d["call_count"],
            "weight": weight
        })

    forecast = {}
    for (dow, interval), entries in intervals.items():
        total_weight = sum(e["weight"] for e in entries)
        weighted_avg = sum(e["calls"] * e["weight"] for e in entries) / total_weight
        forecast[(dow, interval)] = {
            "predicted_calls": round(weighted_avg, 1),
            "data_points": len(entries),
            "confidence": "high" if len(entries) >= 8 else "medium" if len(entries) >= 4 else "low"
        }

    return forecast
```

### Forecast Accuracy Targets

Good WFM operations hit these accuracy bands:

| Forecast Level | Target Accuracy | Acceptable |
|---|:---:|:---:|
| Daily total | Within 5% | Within 10% |
| 30-minute interval | Within 10% | Within 15% |
| Weekly total | Within 3% | Within 7% |

If your interval forecasts are off by more than 15% consistently, your historical data window is either too short, contaminated by anomalies, or your business has a pattern your model is not capturing.

Track forecast accuracy every day:

```sql
SELECT
    DATE(call_date) AS forecast_day,
    FLOOR(HOUR(call_date) * 2 + MINUTE(call_date) / 30) AS interval_id,
    COUNT(*) AS actual_calls,
    f.predicted_calls,
    ROUND((COUNT(*) - f.predicted_calls) / f.predicted_calls * 100, 1) AS variance_pct
FROM vicidial_closer_log v
JOIN forecast_table f ON DATE(v.call_date) = f.forecast_date
    AND FLOOR(HOUR(v.call_date) * 2 + MINUTE(v.call_date) / 30) = f.interval_id
WHERE DATE(v.call_date) = CURDATE()
GROUP BY forecast_day, interval_id;
```

## Schedule Adherence: Making Sure the Plan Survives Contact With Reality

You can have a perfect forecast and perfect Erlang C staffing, and still blow your service level if agents don't follow the schedule. Schedule adherence measures whether agents are doing what the schedule says they should be doing, when the schedule says they should be doing it.

### The Adherence Formula

```
Adherence % = (Scheduled Time - Non-Adherent Time) / Scheduled Time × 100
```

Non-adherent time includes:
- Late logins (scheduled at 8:00, logged in at 8:12)
- Early logoffs (left at 4:45 instead of 5:00)
- Extended breaks (15-minute break turned into 25 minutes)
- Unauthorized auxiliary/pause time

**Example:** An agent scheduled for 480 minutes (8 hours) who arrives 15 minutes late, takes an extra 10 minutes on breaks, and logs off 5 minutes early has 30 minutes of non-adherent time:

```
Adherence = (480 - 30) / 480 × 100 = 93.75%
```

### Adherence vs. Conformance

These get confused constantly. They measure different things:

- **Adherence** = doing the right thing at the right time. Were you logged in when you were scheduled to be? Were you on break when you were scheduled for break?
- **Conformance** = doing the right amount of total work. Did you work 8 hours total? (You might have come in late and stayed late -- conformance would be fine, adherence would not.)

You need both. An agent who works 8 hours but shifts their schedule by 30 minutes might have 100% conformance but 85% adherence -- and that 30-minute gap is exactly when you were understaffed.

### Tracking Adherence in VICIdial

VICIdial tracks agent status changes in the `vicidial_agent_log` table. Pull adherence data with:

```sql
SELECT
    user AS agent_id,
    DATE(event_time) AS work_date,
    MIN(event_time) AS first_login,
    MAX(event_time) AS last_event,
    SUM(CASE WHEN status = 'READY' THEN pause_sec ELSE 0 END) AS ready_time,
    SUM(CASE WHEN status = 'INCALL' THEN pause_sec ELSE 0 END) AS talk_time,
    SUM(CASE WHEN status = 'PAUSED' THEN pause_sec ELSE 0 END) AS pause_time,
    SUM(pause_sec) AS total_time,
    ROUND(SUM(CASE WHEN status IN ('READY','INCALL')
        THEN pause_sec ELSE 0 END) / SUM(pause_sec) * 100, 1) AS productive_pct
FROM vicidial_agent_log
WHERE event_time >= DATE_SUB(NOW(), INTERVAL 1 DAY)
GROUP BY user, DATE(event_time)
ORDER BY productive_pct ASC;
```

Agents at the bottom of the productive_pct ranking are your adherence problems. Look at their pause code usage to understand why:

```sql
SELECT
    user AS agent_id,
    sub_status AS pause_code,
    COUNT(*) AS pause_count,
    SUM(pause_sec) AS total_pause_seconds,
    ROUND(AVG(pause_sec), 0) AS avg_pause_seconds
FROM vicidial_agent_log
WHERE event_time >= DATE_SUB(NOW(), INTERVAL 7 DAY)
    AND status = 'PAUSED'
GROUP BY user, sub_status
ORDER BY user, total_pause_seconds DESC;
```

If an agent's "BREAK" [pause codes](/blog/vicidial-pause-codes-accountability/) average 22 minutes when breaks are scheduled for 15 minutes, that is a 7-minute adherence leak per break. Over 3 breaks per day, 5 days per week, that is 105 minutes of lost capacity per agent per week.

### Adherence Targets

| Performance Level | Adherence % | Action |
|---|:---:|---|
| Top performer | 97%+ | Recognize and reward |
| Meeting expectations | 92-96% | Normal operations |
| Below target | 85-91% | Coaching conversation |
| Unacceptable | Below 85% | Written warning territory |
| Industry benchmark | 90-95% | Where most centers target |

A realistic target for most operations is 92-95%. Pushing for 99% adherence creates a micromanagement culture that drives turnover -- which costs you far more than the 5% of lost adherence time.

## Intraday Management: Adjusting in Real Time

No forecast is perfect. Intraday management is the discipline of watching actual performance against the forecast and making adjustments before service levels crater.

### Real-Time Metrics to Monitor

Track these in VICIdial's real-time reports during operating hours:

```
Admin > Reports > Real-Time Report
```

Use VICIdial's [real-time report](/blog/vicidial-realtime-report-guide/) to track the key numbers:

| Metric | Target | Yellow Alert | Red Alert |
|---|:---:|:---:|:---:|
| Calls in queue | 0-3 | 4-8 | 9+ |
| Longest wait time | Under 20 sec | 20-60 sec | Over 60 sec |
| Agents in READY | 10-15% of total | 5-10% | Under 5% |
| Agents in PAUSED | Under 15% of total | 15-20% | Over 20% |
| Service level (rolling 30 min) | 80%+ | 70-80% | Under 70% |

### Intraday Adjustment Playbook

**Scenario: Call volume 20% above forecast**

1. Cancel non-essential training and meetings -- pull those agents back to the phones
2. Offer overtime to agents who already went home (text them, let them accept via app)
3. Adjust break schedules -- shorten breaks by 5 minutes, stagger them wider
4. If you have a blended inbound/outbound operation, pause outbound campaigns to free agents for inbound

In VICIdial, you can shift agents between campaigns in real-time:

```
Admin > Users > Agent Transfer
    Select agent > Move to Campaign: INBOUND_QUEUE
```

**Scenario: Call volume 20% below forecast**

1. Offer Voluntary Time Off (VTO) to avoid paying agents to sit idle
2. Pull agents into coaching sessions or training that was scheduled for later
3. Run blended outbound dials to keep agents productive
4. Do not send everyone home -- volume can spike back up unpredictably

### Building an Intraday Dashboard

VICIdial's real-time report gives you the raw data. Build a dashboard that compares actual vs. forecast in real-time:

```bash
#!/bin/bash
# intraday-check.sh - Compare actual call volume against forecast
INTERVAL=$(date +%H:%M | awk -F: '{
    if ($2 < 30) printf "%s:00", $1;
    else printf "%s:30", $1;
}')
DOW=$(date +%u)

ACTUAL=$(mysql -u cron -pPASS vicidial -N -e "
    SELECT COUNT(*) FROM vicidial_closer_log
    WHERE call_date >= CONCAT(CURDATE(), ' ', '${INTERVAL}', ':00')
    AND call_date < CONCAT(CURDATE(), ' ', '${INTERVAL}', ':00') + INTERVAL 30 MINUTE")

FORECAST=$(mysql -u cron -pPASS vicidial -N -e "
    SELECT predicted_calls FROM wfm_forecast
    WHERE day_of_week = ${DOW} AND interval_start = '${INTERVAL}'")

if [ -n "$FORECAST" ] && [ "$FORECAST" -gt 0 ]; then
    VARIANCE=$(echo "scale=1; ($ACTUAL - $FORECAST) / $FORECAST * 100" | bc)
    echo "[$INTERVAL] Actual: $ACTUAL | Forecast: $FORECAST | Variance: ${VARIANCE}%"

    # Alert if variance exceeds 15%
    ABS_VAR=$(echo "$VARIANCE" | tr -d '-')
    if (( $(echo "$ABS_VAR > 15" | bc -l) )); then
        echo "  WARNING: Variance exceeds 15% threshold"
    fi
fi
```

Run this every 30 minutes via cron. Add email or Slack alerts when variance exceeds your threshold.

## Scheduling: Turning Numbers Into Actual Shifts

The forecast tells you how many agents you need per interval. Scheduling turns that into actual human schedules with shift starts, break times, and off days.

### The Core Scheduling Challenge

You need 19 agents at 10:00 AM and 12 agents at 2:00 PM. You can not schedule 19 people for 10 AM and 12 different people for 2 PM -- agents work full shifts. The art of scheduling is building shift patterns that match the demand curve as closely as possible.

### Shift Types That Improve Coverage

| Shift Type | Hours | Best For |
|---|:---:|---|
| Standard 8-hour | 8:00-4:30 or 9:00-5:30 | Baseline coverage |
| Split shift | 8:00-12:00 + 4:00-8:00 | Covering morning and evening peaks with a midday gap |
| Staggered start | Shifts starting every 30 min from 7:30-9:30 | Smoothing the login surge |
| Part-time 4-hour | 10:00-2:00 or 4:00-8:00 | Peak coverage without full-shift cost |
| Overlap shift | 11:00-7:30 | Bridging the transition between morning and evening crews |

The most common mistake is scheduling everyone to start at the same time. If 50 agents all log in at 9:00 AM, you have zero coverage at 8:45 and a massive surplus at 9:05. Stagger starts across 30-minute windows.

### Break Scheduling

Breaks must be staggered too. If all 50 agents go to lunch at noon, your noon-1 PM interval collapses. Spread lunch breaks across 11:30 AM to 1:30 PM in 15-minute waves.

In VICIdial, you can enforce break windows using the Timeclock system:

```
Admin > Timeclock > Shift Definition
    Shift ID:           MORNING_A
    Start Time:         08:00
    End Time:           16:30
    Lunch Start:        11:30
    Lunch End:          12:00
    Break 1 Start:      10:00
    Break 1 End:        10:15
    Break 2 Start:      14:00
    Break 2 End:        14:15
```

Create multiple shift definitions with staggered break times and assign agents to them.

## The WFM Weekly Cycle

Workforce management is an ongoing cycle, not a one-time setup. Here is the weekly rhythm:

**Monday:** Review last week's forecast accuracy. Where were you off? Update the forecast model with last week's actuals.

**Tuesday:** Build next week's schedule based on updated forecast. Post schedules at least 5 days in advance so agents can plan.

**Wednesday:** Run adherence reports for the current week. Coach agents who are consistently below 90%.

**Thursday:** Review intraday performance for the week. Are there intervals where you are consistently over or under? Adjust next week's schedule.

**Friday:** Run the Erlang C calculator against next week's forecast. Verify that scheduled agents cover required agents for every interval. Flag any gaps for overtime or flex scheduling.

### WFM Metrics Dashboard

Build a weekly dashboard that tracks these numbers:

| Metric | Target | This Week | Trend |
|---|:---:|:---:|:---:|
| Forecast accuracy (daily) | Within 5% | -- | -- |
| Forecast accuracy (interval) | Within 10% | -- | -- |
| Average adherence | 92-95% | -- | -- |
| Service level | 80/20 | -- | -- |
| Average occupancy | 75-85% | -- | -- |
| Shrinkage | Under 35% | -- | -- |
| Agent utilization | 85-90% | -- | -- |

Track trends week over week. A single bad week is noise. Three bad weeks in a row is a pattern that needs investigation.

The operations team at [ViciStack](https://vicistack.com/) builds WFM processes alongside dialer optimization because they are two sides of the same coin. The best dialer configuration in the world does not help if you do not have enough agents logged in to take the calls it connects. If your staffing math feels like guesswork and you want it replaced with actual forecasting, [we can help with that](https://vicistack.com/contact/).

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/workforce-management-call-center).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
