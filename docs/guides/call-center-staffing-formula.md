# The Call Center Staffing Formula: Erlang C vs Real-World Math

**Erlang C gives you a number. Reality gives you a different one. This guide covers both: the math behind call center staffing, why Erlang C is wrong in predictable ways, and how to build a staffing model that accounts for shrinkage, blended campaigns, multi-skill routing, and agents who call in sick on Mondays.**

---

## The Problem Nobody Admits

Every call center manager has been in this meeting. Someone from finance asks "how many agents do we actually need?" and somebody pulls up an Erlang C calculator from a website that looks like it was built in 2003, plugs in three numbers, and says "thirty-two agents." Everyone nods. Nobody asks why Monday at 9 AM has a 12-minute wait time while Thursday at 2 PM has six agents sitting idle.

The Erlang C formula is a beautiful piece of queuing theory. A.K. Erlang published it in 1917 to figure out how many telephone circuits the Copenhagen Telephone Company needed. It works. It just doesn't work the way most people use it.

Here's what Erlang C assumes:

- Calls arrive randomly following a Poisson distribution
- All agents are identical and interchangeable
- No calls abandon (callers wait forever)
- Agents handle one call, then immediately take the next
- Call arrival rate is constant during the measurement period
- There's no wrap-up time between calls
- Nobody takes breaks, calls in sick, or goes to lunch

Exactly zero of these are true in a real call center. But Erlang C is still where you start, because it gives you the mathematical floor -- the minimum number of agents you'd need in a universe where everything works perfectly. Then you adjust for the real world.

---

## Step 1: Get Your Actual Numbers

Before touching any formula, you need three numbers from your actual operation. Not your guesses. Not what the campaign manager told you last quarter. Actual numbers from your database.

### Call Volume Per Half-Hour

If you're running VICIdial, pull inbound call volume from the `vicidial_closer_log`:

```sql
SELECT
    DATE(call_date) AS day,
    CONCAT(LPAD(HOUR(call_date), 2, '0'), ':',
           IF(MINUTE(call_date) < 30, '00', '30')) AS half_hour,
    COUNT(*) AS calls
FROM vicidial_closer_log
WHERE call_date >= DATE_SUB(NOW(), INTERVAL 4 WEEK)
    AND campaign_id = 'INBOUND_SALES'
    AND status NOT IN ('DROP','XDROP')
GROUP BY day, half_hour
ORDER BY day, half_hour;
```

For outbound, use `vicidial_log`:

```sql
SELECT
    DATE(call_date) AS day,
    CONCAT(LPAD(HOUR(call_date), 2, '0'), ':',
           IF(MINUTE(call_date) < 30, '00', '30')) AS half_hour,
    COUNT(*) AS calls,
    AVG(length_in_sec) AS avg_talk_sec
FROM vicidial_log
WHERE call_date >= DATE_SUB(NOW(), INTERVAL 4 WEEK)
    AND campaign_id = 'OUTBOUND_B2B'
    AND length_in_sec > 0
GROUP BY day, half_hour
ORDER BY day, half_hour;
```

Take the average for each half-hour across at least four weeks. Two weeks of data is noisy. Eight weeks is better if your volume is seasonal.

### Average Handle Time (AHT)

AHT is talk time + hold time + after-call work (wrap-up/disposition time). Most people only measure talk time and wonder why they're always short-staffed.

```sql
SELECT
    AVG(length_in_sec) AS avg_talk_sec,
    AVG(TIMESTAMPDIFF(SECOND, end_epoch_time,
        FROM_UNIXTIME(
            (SELECT MIN(UNIX_TIMESTAMP(event_time))
             FROM vicidial_agent_log val2
             WHERE val2.user = val.user
               AND val2.event_time > FROM_UNIXTIME(val.end_epoch_time)
               AND val2.sub_status IN ('READY','INCALL'))
        ))) AS avg_wrapup_sec
FROM vicidial_agent_log val
WHERE event_time >= DATE_SUB(NOW(), INTERVAL 4 WEEK)
    AND campaign_id = 'INBOUND_SALES'
    AND talk_sec > 0;
```

That wrap-up query is ugly. In practice, most VICIdial shops measure AHT by looking at the time between call end and the agent going back to READY status. A simpler approximation:

```sql
SELECT
    user,
    AVG(talk_sec + dispo_sec) AS avg_handle_time,
    STDDEV(talk_sec + dispo_sec) AS handle_time_stddev
FROM vicidial_agent_log
WHERE event_time >= DATE_SUB(NOW(), INTERVAL 4 WEEK)
    AND talk_sec > 0
    AND campaign_id = 'INBOUND_SALES'
GROUP BY user
ORDER BY avg_handle_time DESC;
```

The standard deviation matters here. If your average AHT is 240 seconds but the stddev is 180, you don't have one AHT -- you have a bimodal distribution. Some calls are 60 seconds (quick answers) and some are 500 seconds (complex issues). The Erlang formula can't handle bimodal distributions, and this is one of its major blind spots.

For a typical inbound support campaign, expect an AHT somewhere between 180 and 420 seconds. Sales calls tend to run 240-600 seconds. Collections are usually 120-300 seconds.

### Service Level Target

This is the percentage of calls you want answered within a specified number of seconds. The industry standard is "80/20" -- 80% of calls answered within 20 seconds. But there's nothing sacred about 80/20. You pick the numbers based on your business:

- **80/20:** Standard for most call centers. Reasonable balance.
- **90/10:** Premium support, high-value customers. Expensive.
- **80/30:** Common for non-emergency inbound. Saves money, customers don't complain much.
- **70/60:** Budget operations. Callers start abandoning around 45 seconds.

The difference between 80/20 and 90/10 can be 30-40% more agents for the same volume. That's not a typo. The relationship between service level and staffing is exponential, not linear. Going from 80% answered in 20 seconds to 90% answered in 10 seconds doesn't cost 12.5% more. It costs a lot more.

---

## Step 2: The Erlang C Formula

The Erlang C formula calculates the probability that a call has to wait in queue. Here it is in its full mathematical glory:

```
P(wait > 0) = (A^N / N!) * (N / (N - A)) / [sum(k=0 to N-1) (A^k / k!) + (A^N / N!) * (N / (N - A))]
```

Where:
- A = traffic intensity in Erlangs = (calls per interval * AHT) / interval length
- N = number of agents
- P(wait > 0) = probability a call waits

And the service level formula:

```
SL = 1 - P(wait > 0) * e^(-(N - A) * (target_wait / AHT))
```

Where:
- SL = service level (fraction of calls answered within target wait time)
- target_wait = your acceptable wait time in seconds (e.g., 20)

Nobody computes this by hand. Here's a Python implementation you can actually run:

```python
#!/usr/bin/env python3
"""
erlang_c.py — Staffing calculator using Erlang C
"""
import math
from functools import lru_cache

@lru_cache(maxsize=1024)
def erlang_c(agents: int, traffic: float) -> float:
    """Probability a call waits (Erlang C)."""
    if agents <= traffic:
        return 1.0  # system is at or over capacity

    # Use log-space to avoid factorial overflow
    log_numerator = traffic * math.log(traffic) - math.lgamma(agents + 1)
    log_denominator_sum = 0
    terms = []
    for k in range(agents):
        terms.append(math.exp(k * math.log(traffic) - math.lgamma(k + 1)))

    last_term = math.exp(log_numerator)
    rho = traffic / agents
    last_term_adjusted = last_term / (1 - rho)

    total = sum(terms) + last_term_adjusted
    return last_term_adjusted / total

def service_level(agents: int, calls_per_interval: float,
                  aht_sec: float, interval_sec: float,
                  target_wait_sec: float) -> float:
    """Fraction of calls answered within target_wait_sec."""
    traffic = (calls_per_interval * aht_sec) / interval_sec
    if agents <= traffic:
        return 0.0

    pw = erlang_c(agents, traffic)
    sl = 1 - pw * math.exp(-(agents - traffic) * (target_wait_sec / aht_sec))
    return max(0.0, min(1.0, sl))

def find_agents(calls_per_interval: float, aht_sec: float,
                interval_sec: float = 1800,
                target_sl: float = 0.80,
                target_wait_sec: float = 20) -> dict:
    """Find minimum agents to meet service level target."""
    traffic = (calls_per_interval * aht_sec) / interval_sec
    min_agents = max(1, int(math.ceil(traffic)))

    for n in range(min_agents, min_agents + 200):
        sl = service_level(n, calls_per_interval, aht_sec,
                          interval_sec, target_wait_sec)
        if sl >= target_sl:
            return {
                'agents': n,
                'traffic_erlangs': round(traffic, 2),
                'service_level': round(sl * 100, 1),
                'occupancy': round(traffic / n * 100, 1),
                'calls_per_interval': calls_per_interval,
                'aht_sec': aht_sec,
            }
    return {'agents': -1, 'error': 'Could not find solution in range'}

# Example: 120 calls per half-hour, 240 sec AHT, 80/20 target
if __name__ == '__main__':
    result = find_agents(
        calls_per_interval=120,
        aht_sec=240,
        interval_sec=1800,
        target_sl=0.80,
        target_wait_sec=20
    )
    print(f"Agents needed:     {result['agents']}")
    print(f"Traffic intensity: {result['traffic_erlangs']} Erlangs")
    print(f"Service level:     {result['service_level']}%")
    print(f"Occupancy:         {result['occupancy']}%")
```

Running this with 120 calls per half-hour and a 240-second AHT gives you:

```
Agents needed:     19
Traffic intensity: 16.0 Erlangs
Service level:     82.1%
Occupancy:         84.2%
```

Nineteen agents. Write that down, because it's wrong. It's the correct answer to the wrong question.

---

## Step 3: Why Erlang C Is Wrong (And How Wrong)

Here's a table showing the Erlang C result versus what you'll actually need for a range of scenarios:

| Scenario | Calls/30min | AHT (sec) | Erlang C Says | You Actually Need | Gap |
|----------|------------|-----------|---------------|-------------------|-----|
| Small inbound support | 30 | 300 | 7 | 9-10 | +29-43% |
| Medium sales inbound | 80 | 240 | 14 | 18-20 | +29-43% |
| Large blended center | 200 | 210 | 26 | 33-37 | +27-42% |
| Collections floor | 50 | 180 | 7 | 9-11 | +29-57% |
| High-touch B2B sales | 25 | 600 | 11 | 14-16 | +27-45% |

The gap is consistently 27-57%. That's not noise. It's the accumulated weight of everything Erlang C ignores. Let's quantify each factor.

### Factor 1: Shrinkage (Usually 25-35%)

Shrinkage is the percentage of paid time agents are unavailable to handle calls. It includes:

- Breaks (15 min morning + 15 min afternoon = 30 min/day)
- Lunch (30-60 min/day)
- Training and meetings (2-4 hours/week)
- Coaching sessions (30-60 min/week per agent)
- System issues (computer freezing, VPN dropping, softphone glitching)
- Unscheduled absences (sick days, personal emergencies, no-shows)
- Late arrivals and early departures

Here's how to calculate your actual shrinkage from VICIdial data:

```sql
SELECT
    DATE(event_time) AS day,
    user,
    SUM(CASE WHEN status = 'PAUSED' THEN pause_sec ELSE 0 END) AS pause_seconds,
    SUM(CASE WHEN status = 'READY' OR status = 'INCALL'
        THEN wait_sec + talk_sec ELSE 0 END) AS productive_seconds,
    SUM(pause_sec + wait_sec + talk_sec + dispo_sec) AS total_logged_seconds,
    ROUND(
        SUM(CASE WHEN status = 'PAUSED' THEN pause_sec ELSE 0 END) /
        SUM(pause_sec + wait_sec + talk_sec + dispo_sec) * 100, 1
    ) AS shrinkage_pct
FROM vicidial_agent_log
WHERE event_time >= DATE_SUB(NOW(), INTERVAL 4 WEEK)
GROUP BY day, user
ORDER BY shrinkage_pct DESC;
```

Most operations discover their shrinkage is higher than they thought. The industry average is 30-35% for on-site centers and 35-40% for remote agents (home distractions, internet issues, longer breaks).

To adjust for shrinkage:

```
Adjusted agents = Erlang C agents / (1 - shrinkage_rate)
```

So if Erlang C says 19 and your shrinkage is 32%:

```
Adjusted = 19 / (1 - 0.32) = 19 / 0.68 = 28 agents
```

That's 28 scheduled agents to have 19 actually taking calls at any given moment. Already a 47% increase from the Erlang C number.

### Factor 2: Occupancy Limits (Don't Burn Your Agents)

Occupancy is the percentage of time an agent spends on calls versus waiting. At 84% occupancy (from our example above), agents spend 50 minutes of every hour on the phone. That sounds efficient. It's also a fast track to burnout and turnover.

Sustainable occupancy depends on the type of work:

| Call Type | Max Sustainable Occupancy | Why |
|-----------|--------------------------|-----|
| Inbound support (simple) | 85-88% | Repetitive but low stress |
| Inbound sales | 78-82% | High cognitive load, rejection |
| Outbound cold calling | 75-80% | Highest stress, most rejection |
| Collections | 72-78% | Emotionally draining |
| Blended in/outbound | 80-85% | Variety reduces fatigue |

If your staffing model produces 90%+ occupancy, you're going to have a turnover problem within 60 days. Agents will start calling in sick, taking longer breaks, and quitting. The cost of replacing one agent (recruiting, training, ramp-up time) is typically $3,000-8,000. Burning your agents to save on staffing is one of the most expensive mistakes a call center can make.

### Factor 3: Call Abandonment (Erlang C Ignores It)

Erlang C assumes callers wait forever. They don't. In practice:

- 25% of callers hang up after 30 seconds
- 50% hang up after 60 seconds
- 70% hang up after 120 seconds

This creates a paradox: if you're understaffed, some callers abandon, which reduces the apparent call volume, which makes Erlang C tell you that you need fewer agents. The formula rewards you for bad service.

The fix is to add abandoned calls back into your volume calculation:

```sql
SELECT
    CONCAT(LPAD(HOUR(call_date), 2, '0'), ':',
           IF(MINUTE(call_date) < 30, '00', '30')) AS half_hour,
    COUNT(*) AS total_calls,
    SUM(CASE WHEN status = 'DROP' THEN 1 ELSE 0 END) AS abandoned,
    SUM(CASE WHEN status != 'DROP' THEN 1 ELSE 0 END) AS answered,
    ROUND(SUM(CASE WHEN status = 'DROP' THEN 1 ELSE 0 END) /
          COUNT(*) * 100, 1) AS abandon_pct
FROM vicidial_closer_log
WHERE call_date >= DATE_SUB(NOW(), INTERVAL 4 WEEK)
    AND campaign_id = 'INBOUND_SALES'
GROUP BY half_hour
ORDER BY half_hour;
```

If you see abandon rates above 5% in any half-hour, those intervals are understaffed. Period. Use the total calls (answered + abandoned) as your volume input, not just answered calls.

### Factor 4: Multi-Skill Routing

If you have agents handling multiple queues (billing, sales, tech support), Erlang C breaks down. The formula assumes all agents are interchangeable, but in reality:

- Agent A handles billing and sales
- Agent B handles sales only
- Agent C handles billing, sales, and tech support

This creates partial pooling, which is more efficient than separate queues but less efficient than full pooling. The math for multi-skill routing requires simulation, not closed-form formulas. Erlang C will underestimate staffing needs for multi-skill environments by 8-15% because it assumes perfect pooling efficiency.

### Factor 5: Blended Campaigns

In VICIdial, blended campaigns have agents taking inbound calls when available and making outbound calls during gaps. This is efficient but makes staffing calculations harder because:

1. Outbound calls have a different AHT than inbound
2. The dialer's predictive algorithm adjusts based on available agents
3. Switching between inbound and outbound modes has a cognitive cost

For blended campaigns, calculate your inbound staffing needs first (that's the hard constraint), then figure out how much outbound capacity you have in the gaps. Don't try to optimize for both simultaneously in one formula -- calculate them separately and overlay.

---

## Step 4: The Real-World Staffing Formula

Here's the formula that actually works:

```
Required staff = (Erlang_C_agents * (1 + multi_skill_factor)) /
                 (1 - shrinkage_rate) /
                 min(target_occupancy, 1.0)
```

With adjustments:

```python
def real_world_staffing(calls_per_interval: float,
                        aht_sec: float,
                        interval_sec: float = 1800,
                        target_sl: float = 0.80,
                        target_wait_sec: float = 20,
                        shrinkage: float = 0.32,
                        max_occupancy: float = 0.82,
                        multi_skill_overhead: float = 0.10,
                        absenteeism_buffer: float = 0.05) -> dict:
    """
    Real-world staffing calculation layered on Erlang C.
    """
    # Step 1: Get Erlang C baseline
    base = find_agents(calls_per_interval, aht_sec,
                       interval_sec, target_sl, target_wait_sec)
    erlang_agents = base['agents']

    # Step 2: Adjust for multi-skill overhead
    skill_adjusted = erlang_agents * (1 + multi_skill_overhead)

    # Step 3: Adjust for shrinkage
    shrinkage_adjusted = skill_adjusted / (1 - shrinkage)

    # Step 4: Adjust for occupancy cap
    occupancy_adjusted = shrinkage_adjusted / max_occupancy

    # Step 5: Absenteeism buffer (unplanned absences beyond shrinkage)
    final = occupancy_adjusted * (1 + absenteeism_buffer)

    return {
        'erlang_c_raw': erlang_agents,
        'after_skill_adj': round(skill_adjusted, 1),
        'after_shrinkage': round(shrinkage_adjusted, 1),
        'after_occupancy': round(occupancy_adjusted, 1),
        'final_required': math.ceil(final),
        'traffic_erlangs': base['traffic_erlangs'],
        'base_service_level': base['service_level'],
    }

# Same scenario: 120 calls/half-hour, 240 sec AHT
result = real_world_staffing(120, 240)
print(f"Erlang C raw:       {result['erlang_c_raw']} agents")
print(f"+ Multi-skill:      {result['after_skill_adj']} agents")
print(f"+ Shrinkage (32%):  {result['after_shrinkage']} agents")
print(f"+ Occupancy (82%):  {result['after_occupancy']} agents")
print(f"+ Absenteeism (5%): {result['final_required']} agents")
```

Output:

```
Erlang C raw:       19 agents
+ Multi-skill:      20.9 agents
+ Shrinkage (32%):  30.7 agents
+ Occupancy (82%):  37.5 agents
+ Absenteeism (5%): 40 agents
```

Forty agents to do what Erlang C says you can do with nineteen. That's a 111% gap. And forty is the right number. Anyone who's run a 120-call-per-half-hour inbound operation knows that 19 agents would be a disaster.

---

## Step 5: Building the Half-Hour Staffing Grid

Your call volume isn't flat across the day. It spikes in the morning, dips at lunch, picks up in the afternoon, and drops off after 5 PM. You need a staffing grid that matches.

Here's how to build one:

```python
import csv

# Half-hour call volumes (example — use your actual data)
half_hours = [
    ('08:00', 45), ('08:30', 72), ('09:00', 110), ('09:30', 120),
    ('10:00', 115), ('10:30', 105), ('11:00', 95),  ('11:30', 80),
    ('12:00', 65), ('12:30', 60), ('13:00', 75),  ('13:30', 90),
    ('14:00', 100), ('14:30', 105), ('15:00', 95),  ('15:30', 85),
    ('16:00', 70), ('16:30', 55), ('17:00', 35),  ('17:30', 20),
]

print(f"{'Time':<8} {'Calls':<7} {'Erlang':<8} {'Real':<6} {'Occ%':<6}")
print('-' * 38)

for time_slot, calls in half_hours:
    result = real_world_staffing(
        calls_per_interval=calls,
        aht_sec=240,
        shrinkage=0.32,
        max_occupancy=0.82
    )
    print(f"{time_slot:<8} {calls:<7} {result['erlang_c_raw']:<8} "
          f"{result['final_required']:<6} "
          f"{round(calls * 240 / 1800 / result['final_required'] * 100, 1):<6}")
```

This produces a grid showing exactly how many bodies you need on the floor at each half-hour interval. The key insight: your staffing needs might double between 8:00 and 9:30 AM, which means you need staggered start times, not everyone clocking in at 8.

### Staggered Start Times

If you need 25 agents at 8 AM and 40 at 9:30 AM, stagger like this:

| Start Time | Agents | End Time |
|-----------|--------|----------|
| 7:00 AM | 5 | 3:30 PM |
| 7:30 AM | 5 | 4:00 PM |
| 8:00 AM | 10 | 4:30 PM |
| 8:30 AM | 5 | 5:00 PM |
| 9:00 AM | 10 | 5:30 PM |
| 9:30 AM | 5 | 6:00 PM |

That gets you 5 agents at 7 AM (pre-peak), ramping to 40 by 9:30, and naturally tapering as early shifts end. Match the ramp to your call volume curve.

### Lunch Staggering

Never let everyone go to lunch at the same time. If you have 40 agents, split lunch into 4 groups of 10 at 11:00, 11:30, 12:00, and 12:30. You lose 10 agents during each lunch slot instead of losing 40. This alone can drop your midday abandon rate by 60%.

---

## Step 6: The Monday Problem and Day-of-Week Adjustment

Monday call volumes are typically 15-25% higher than Friday. Every week. Here's a query to see your day-of-week distribution:

```sql
SELECT
    DAYNAME(call_date) AS weekday,
    DAYOFWEEK(call_date) AS day_num,
    ROUND(AVG(daily_count), 0) AS avg_calls,
    ROUND(AVG(daily_count) /
          (SELECT AVG(dc) FROM (
              SELECT COUNT(*) AS dc FROM vicidial_closer_log
              WHERE call_date >= DATE_SUB(NOW(), INTERVAL 8 WEEK)
                AND campaign_id = 'INBOUND_SALES'
              GROUP BY DATE(call_date)
          ) t) * 100, 1) AS pct_of_avg
FROM (
    SELECT DATE(call_date) AS call_day,
           DAYNAME(call_date) AS weekday_name,
           DAYOFWEEK(call_date) AS day_num,
           COUNT(*) AS daily_count
    FROM vicidial_closer_log
    WHERE call_date >= DATE_SUB(NOW(), INTERVAL 8 WEEK)
        AND campaign_id = 'INBOUND_SALES'
    GROUP BY call_day
) daily
GROUP BY weekday, day_num
ORDER BY day_num;
```

Typical output:

```
Monday:    122% of average
Tuesday:   108% of average
Wednesday: 100% of average
Thursday:   95% of average
Friday:     82% of average
Saturday:   52% of average (if open)
```

Build a separate staffing grid for each day of the week, or at minimum for Monday/Tuesday, Wednesday/Thursday, and Friday. Using a single staffing plan for every day means you're overstaffed on Friday and drowning on Monday.

---

## Step 7: Outbound Staffing — Different Math

Outbound staffing follows different rules because the dialer controls the pace, not the callers. The key variables are:

- **Dial ratio:** How many lines the dialer fires per available agent
- **Connect rate:** Percentage of dials that result in a human answer
- **Talk time:** Average conversation length
- **Wrap-up time:** Time between calls for dispositioning

For a VICIdial predictive campaign, the relevant numbers:

```sql
SELECT
    campaign_id,
    ROUND(AVG(auto_dial_level), 2) AS avg_dial_ratio,
    ROUND(
        SUM(CASE WHEN length_in_sec > 0 THEN 1 ELSE 0 END) /
        COUNT(*) * 100, 1
    ) AS connect_rate_pct,
    ROUND(AVG(CASE WHEN length_in_sec > 0
              THEN length_in_sec END), 0) AS avg_talk_sec,
    COUNT(*) / COUNT(DISTINCT DATE(call_date)) AS avg_dials_per_day
FROM vicidial_log
WHERE call_date >= DATE_SUB(NOW(), INTERVAL 4 WEEK)
GROUP BY campaign_id;
```

For outbound, you're optimizing for throughput, not service level. The formula:

```
Required outbound agents = target_contacts_per_hour /
                           (connect_rate * calls_per_agent_per_hour)

calls_per_agent_per_hour = 3600 / (avg_talk_sec + avg_wrapup_sec +
                           avg_ring_sec / connect_rate)
```

For a typical B2B outbound campaign (10% connect rate, 180-second talk time, 30-second wrap-up, 25 seconds average ring time):

```
calls_per_agent_per_hour = 3600 / (180 + 30 + 25/0.10) = 3600 / 460 = 7.8 contacts/hour
```

So if you need 60 contacts per hour, you need 60 / 7.8 = 8 agents (before shrinkage). After 32% shrinkage: 12 agents scheduled. If you want 300 contacts per day (8-hour shift), you need about 5 productive agents, or 8 scheduled.

---

## Step 8: Validating Your Model Against Reality

Your staffing model is a prediction. Validate it by comparing what you predicted to what actually happened.

Set up a weekly validation query:

```sql
SELECT
    DATE(call_date) AS day,
    CONCAT(LPAD(HOUR(call_date), 2, '0'), ':',
           IF(MINUTE(call_date) < 30, '00', '30')) AS half_hour,
    COUNT(*) AS calls_offered,
    SUM(CASE WHEN status NOT IN ('DROP','XDROP') THEN 1 ELSE 0 END) AS calls_answered,
    ROUND(AVG(CASE WHEN queue_seconds > 0
              THEN queue_seconds END), 0) AS avg_wait_sec,
    ROUND(SUM(CASE WHEN queue_seconds <= 20
              AND status NOT IN ('DROP','XDROP') THEN 1 ELSE 0 END) /
          NULLIF(SUM(CASE WHEN status NOT IN ('DROP','XDROP')
                     THEN 1 ELSE 0 END), 0) * 100, 1) AS sl_pct,
    (SELECT COUNT(DISTINCT user) FROM vicidial_agent_log val
     WHERE val.event_time >= vcl.call_date
       AND val.event_time < DATE_ADD(vcl.call_date, INTERVAL 30 MINUTE)
       AND val.status IN ('INCALL','READY','PAUSED')
    ) AS agents_logged_in
FROM vicidial_closer_log vcl
WHERE call_date >= DATE_SUB(NOW(), INTERVAL 1 WEEK)
    AND campaign_id = 'INBOUND_SALES'
GROUP BY day, half_hour
ORDER BY day, half_hour;
```

Compare your predicted service level for each interval against actual. If your model predicted 80% and you got 65%, either your volume was higher than expected or your shrinkage was higher. Track the variance and adjust your assumptions quarterly.

The operations that get staffing right aren't the ones with the best formula. They're the ones that measure, compare, adjust, and repeat. Every week, forever.

---

## Common Mistakes That Blow Up Staffing Models

**Using daily averages instead of half-hour intervals.** If your average daily call volume is 800, you might calculate that you need 25 agents. But if 200 of those 800 calls come between 9:00 and 10:00 AM, you need 40 agents during that hour and 15 the rest of the day. Daily averages hide the peaks that kill your service level.

**Ignoring AHT variance by agent.** Your average AHT is 240 seconds, but Agent A averages 180 and Agent B averages 380. If Agent B is scheduled during peak hours and Agent A is on the quiet shift, your service level during peak will be worse than the model predicts. Match your best agents (lowest AHT, highest conversion) to your highest-volume intervals.

**Not accounting for new agent ramp-up.** New agents typically have AHTs 40-80% higher than tenured agents during their first 30 days. If you're onboarding 5 new agents and counting them as full-capacity resources, you're overstating your capacity. In VICIdial, track AHT by tenure:

```sql
SELECT
    val.user,
    vu.full_name,
    DATEDIFF(NOW(), vu.modify_date) AS days_active,
    AVG(val.talk_sec + val.dispo_sec) AS avg_aht,
    COUNT(*) AS calls
FROM vicidial_agent_log val
JOIN vicidial_users vu ON val.user = vu.user
WHERE val.event_time >= DATE_SUB(NOW(), INTERVAL 2 WEEK)
    AND val.talk_sec > 0
GROUP BY val.user, vu.full_name, vu.modify_date
HAVING calls >= 100
ORDER BY days_active ASC;
```

**Treating a blended campaign like a pure inbound or pure outbound operation.** In a blended environment, the moment inbound volume spikes, agents get pulled from outbound, the outbound dial level drops, and your outbound throughput craters. Model the inbound constraint first, then see what outbound capacity remains.

**Forgetting that breaks cluster.** You schedule 15-minute breaks at 10:00 and 10:15 in two groups. But five agents take their break 8 minutes late, three come back 4 minutes late, and one disappears for 22 minutes. Your "planned" 15-minute break actually removes agents from the floor for 25-30 minutes. Build in a 50% buffer on scheduled break durations.

---

## Building This Into a Recurring Process

Don't calculate staffing once and forget about it. Set up a weekly cadence:

1. **Monday morning:** Pull last week's actual volume, AHT, and service level by half-hour. Compare to your model.
2. **Adjust assumptions:** If your model was off by more than 10%, update your shrinkage rate, AHT, or volume projection.
3. **Two-week forecast:** Use the adjusted model to predict staffing needs for the next two weeks.
4. **Monthly recalibration:** Every month, recalculate your baseline shrinkage percentage from actual data.

If you're running VICIdial, build a Grafana dashboard that shows predicted vs. actual service level by half-hour. Our guide on [VICIdial Grafana dashboards](https://vicistack.com/blog/vicidial-grafana-dashboards/) covers the setup. When the predicted and actual lines diverge, your model needs recalibration.

For operations with 50+ agents, consider a proper workforce management tool. But for operations under 50 agents, a Python script running the adjusted Erlang C formula against a half-hour volume grid will get you 90% of the way there at zero software cost. The formulas in this guide are everything you need.

The math isn't hard. The discipline to keep updating your inputs is the hard part. But a call center that gets staffing right -- not just once, but every week -- runs at lower cost with better service levels than one that throws bodies at problems. That's the whole game.

If you're setting up your VICIdial operation from scratch and need help sizing your infrastructure alongside your staffing, take a look at our [VICIdial setup guide](https://vicistack.com/blog/vicidial-setup-guide/) for the server-side capacity planning. And if your real problem isn't how many agents you have but how well they perform, our [agent efficiency metrics guide](https://vicistack.com/blog/vicidial-agent-efficiency-metrics/) covers the KPIs that actually predict revenue.

---

*Got a staffing question that doesn't fit neatly into a formula? [Contact ViciStack](https://vicistack.com/contact/) — we've built staffing models for call centers from 5 agents to 500.*

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/call-center-staffing-formula).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
