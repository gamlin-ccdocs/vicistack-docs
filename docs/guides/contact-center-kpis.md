# Contact Center KPIs: The Complete Guide to Metrics That Matter

## Why Most Contact Centers Track the Wrong Metrics

Every contact center has a dashboard. Most of them are useless.

Not because the data is wrong -- it's usually accurate enough. The problem is that most operations track metrics without understanding which ones actually drive outcomes and which ones are just noise. A manager staring at 40 real-time stats is a manager who can't tell what's going wrong until it's already cost them money.

What we've seen across hundreds of VICIdial deployments: the centers that outperform track fewer metrics, but they track the right ones, at the right level, at the right frequency. They understand that some KPIs are leading indicators you can act on today and others are lagging indicators that tell you what already happened. They know the difference between an agent-level problem and a campaign-level problem. And they don't waste time watching numbers that don't connect to revenue.

Below we cover every KPI that matters for outbound, inbound, and blended contact center operations. For each metric, you'll get the formula, industry benchmarks, where to find it in VICIdial's reporting, and -- more to the point -- what to actually do when the number moves in the wrong direction.

If you already know your CPL inside and out, good. If you don't, start with our [cost per lead benchmarks guide](/blog/call-center-cost-per-lead-benchmarks/) and come back here for the full picture.

## The KPI Hierarchy: Leading vs. Lagging Indicators

Before diving into individual metrics, you need a framework for thinking about them. Not all KPIs are created equal, and treating them as a flat list is how you end up micromanaging talk time while your conversion rate craters.

### Lagging Indicators (Outcome Metrics)

These tell you what already happened. They're the scorecard. You can't directly change them -- you change them by moving the leading indicators that feed into them.

- **Cost Per Lead (CPL)** -- the fully loaded cost to produce one qualified lead
- **Revenue Per Lead** -- the average revenue generated per qualified lead
- **ROI / ROAS** -- return on investment or return on ad spend for the operation
- **Customer Satisfaction (CSAT)** -- post-interaction satisfaction score
- **Net Promoter Score (NPS)** -- likelihood to recommend

### Leading Indicators (Operational Metrics)

These are the metrics you can act on right now. When they move, the lagging indicators follow -- usually within days or weeks.

- **Contact Rate** -- percentage of dials reaching a live human
- **Conversion Rate** -- percentage of contacts that produce a qualified outcome
- **Calls Per Hour / Dials Per Hour** -- throughput velocity
- **Average Handle Time (AHT)** -- total time per interaction
- **First Call Resolution (FCR)** -- percentage of issues resolved on the first contact
- **Agent Occupancy / Utilization** -- percentage of paid time spent on productive work
- **Service Level / SLA** -- percentage of calls answered within a target threshold
- **Abandon Rate** -- percentage of callers who hang up before reaching an agent

### The Causal Chain

This is how leading indicators flow into lagging ones for a typical outbound operation:

**Dials Per Hour** x **Contact Rate** = Contacts Per Hour

**Contacts Per Hour** x **[Conversion Rate](/glossary/conversion-rate/)** = Leads Per Hour

**Total Cost** / **Total Leads** = **[Cost Per Lead](/glossary/cost-per-lead/)**

**Revenue Per Lead** - **Cost Per Lead** = **Profit Per Lead**

Every metric in this article connects to this chain somewhere. When a lagging indicator moves in the wrong direction, you trace back through the chain to find the leading indicator that broke. That's the metric you fix.

## Outbound KPIs: Every Metric That Matters

Outbound operations live and die on throughput and conversion. The goal is simple: put agents in front of as many qualified prospects as possible and convert those conversations into outcomes. Every outbound KPI measures some aspect of that equation.

### Dials Per Hour (DPH)

**Formula:** Total Dials / Total Agent Hours

**What it measures:** Raw dialing throughput -- how many calls your dialer is generating per agent per hour.

**Benchmarks:**

| Dialer Mode | DPH Range | Typical DPH |
|---|---|---|
| Manual / Click-to-Call | 20 - 40 | 30 |
| Preview | 30 - 60 | 45 |
| Progressive | 50 - 90 | 70 |
| Predictive (10+ agents) | 100 - 250 | 180 |
| Predictive (25+ agents, optimized) | 150 - 350 | 220 |

**Where to find it in VICIdial:** Admin > Reports > Outbound Calling Report. The "Dialable Leads" and "Calls" columns divided by logged-in agent hours give you DPH. Also visible on the Real-Time Report as live dialing velocity. If you want a quick per-agent breakdown outside the GUI, you can query it directly:

```sql
SELECT
  vl.user,
  COUNT(*) AS total_dials,
  ROUND(COUNT(*) / (TIMESTAMPDIFF(MINUTE, MIN(vl.call_date), MAX(vl.call_date)) / 60.0), 1) AS dph
FROM vicidial_log vl
WHERE vl.call_date >= CURDATE()
  AND vl.campaign_id = 'SALESCMP'
GROUP BY vl.user
ORDER BY dph DESC;
```

**What to do when it's off:** Low DPH on a predictive dialer almost always means one of three things: hopper starvation (not enough leads loaded), trunk bottleneck (not enough SIP channels), or overly conservative dialing ratios. Check the hopper level first -- if it drops below 2x your agent count, the dialer can't keep pace.

### [Calls Per Hour](/glossary/calls-per-hour/) (CPH)

**Formula:** Total Calls Handled / Total Agent Hours

**What it measures:** The number of calls an agent actually handles per hour, including conversations, voicemails delivered to agents, and quick dispositions. This is distinct from DPH because not every dial results in a call the agent touches.

**Benchmarks:** 15-40 handled calls per hour for outbound agents on predictive dialers, depending on AMD configuration and contact rate. If AMD is filtering voicemails effectively, most handled calls should be live contacts.

**Where to find it in VICIdial:** Agent Performance Detail report shows calls handled per agent per hour. The Agent Time Detail report breaks this down further by pause time, wait time, talk time, and disposition time.

**What to do when it's off:** If DPH is high but CPH is low, your AMD is catching most voicemails (good) but contact rate is low (needs investigation). If both are low, it's a dialer configuration or list exhaustion problem.

### Contact Rate

**Formula:** Live Contacts / Total Dials x 100

**What it measures:** The percentage of dials that result in a live human answering the phone. This is the single most important throughput metric in outbound dialing. Everything downstream multiplies after contact rate.

**Benchmarks:**

| Data Type | Contact Rate Range | Median |
|---|---|---|
| Fresh inbound leads (0-7 days) | 18 - 35% | 25% |
| Warm data (7-30 days) | 12 - 22% | 16% |
| Aged data (30-90 days) | 8 - 15% | 11% |
| Cold purchased lists | 4 - 10% | 7% |
| Recycled / multi-pass | 3 - 8% | 5% |

**Where to find it in VICIdial:** Outbound Calling Report shows total dials and human-answered calls per campaign. For a more granular view, the Campaign Calls Detail report breaks out disposition codes -- filter to "human answered" dispositions and divide by total dials.

**What to do when it's off:** Contact rate is driven by data quality, DID reputation, time-of-day dialing strategy, and AMD configuration. If contact rate drops suddenly, check your DID reputation first -- flagged numbers can cut contact rate in half overnight. For a systematic approach, see our [VICIdial reporting guide](/blog/vicidial-reporting-monitoring/).

### [Conversion Rate](/glossary/conversion-rate/)

**Formula:** Qualified Outcomes / Live Contacts x 100

**What it measures:** The percentage of live conversations that produce a qualified lead, sale, appointment, or other desired outcome.

**Benchmarks:**

| Vertical | Conversion Rate Range | Median |
|---|---|---|
| Home Services | 6 - 12% | 8% |
| Solar (Residential) | 4 - 8% | 6% |
| Insurance (Medicare/ACA) | 3 - 7% | 5% |
| Debt Settlement | 2 - 5% | 3.5% |
| B2B Appointment Setting | 1 - 4% | 2.5% |

**Where to find it in VICIdial:** Campaign Status report shows disposition counts by status category. Define your "qualified" dispositions (e.g., SALE, APPT, XFER) and divide by total human-answered calls. The Agent Performance Detail report shows conversion rate by agent.

**What to do when it's off:** Conversion is a human problem -- scripts, training, objection handling, and offer-market fit. But before blaming agents, check if the data changed. A shift from warm leads to cold lists will crater conversion even with the same agents running the same script.

### [Average Handle Time](/glossary/average-handle-time/) (AHT)

**Formula:** (Total Talk Time + Total Hold Time + Total After-Call Work Time) / Total Calls Handled

**What it measures:** The average total time spent on each interaction, from the moment the call connects to the moment the agent is ready for the next call.

**Benchmarks:**

| Call Type | AHT Range | Typical AHT |
|---|---|---|
| Outbound lead qualification | 90 - 180 sec | 120 sec |
| Outbound appointment setting | 180 - 360 sec | 240 sec |
| Outbound sales (close on call) | 300 - 900 sec | 480 sec |
| Inbound customer service | 240 - 480 sec | 360 sec |
| Inbound technical support | 420 - 900 sec | 600 sec |

**Where to find it in VICIdial:** Agent Time Detail report shows average talk time, hold time, and disposition time per agent. The Campaign Calls Detail report gives you aggregate AHT at the campaign level. Note that VICIdial tracks "Talk," "Dispo," and "Pause" as separate time categories -- you need to add Talk + Dispo to get true AHT.

**What to do when it's off:** AHT is tricky because both too high and too low are problems. High AHT means agents are spending too long on non-converting calls (tighten the script, add qualification gates earlier). Low AHT might mean agents are rushing through calls and not properly qualifying (check conversion rate -- if both AHT and conversion dropped, agents are cutting corners).

### Agent Talk Time Ratio

**Formula:** Total Talk Time / Total Logged-In Time x 100

**What it measures:** The percentage of an agent's shift spent in actual live conversation. This is the purest measure of dialer efficiency and the metric that separates a well-tuned operation from a wasteful one.

**Benchmarks:**

| Configuration | Talk Time Ratio |
|---|---|
| Manual dialing | 15 - 22% |
| Preview dialer | 22 - 32% |
| Progressive dialer | 30 - 40% |
| Predictive dialer (default config) | 35 - 45% |
| Predictive dialer (optimized) | 45 - 55% |
| Predictive dialer (aggressive, 25+ agents) | 50 - 65% |

**Where to find it in VICIdial:** Real-Time Report shows current talk/wait/pause status per agent. For historical data, the Agent Time Detail report breaks down total time by category. Calculate: Talk Time / (Talk Time + Wait Time + Pause Time + Dispo Time).

**What to do when it's off:** If talk time ratio is below 35% on a predictive dialer, you're leaving money on the table. Check dial ratios, hopper size, trunk availability, and list penetration. A well-tuned VICIdial predictive dialer should deliver 42-50 minutes of talk time per hour with 15+ agents. For specifics on tuning, see our [VICIdial reporting and monitoring guide](/blog/vicidial-reporting-monitoring/).

### [Occupancy Rate](/glossary/occupancy-rate/)

**Formula:** (Talk Time + Hold Time + After-Call Work) / (Talk Time + Hold Time + After-Call Work + Available/Idle Time) x 100

**What it measures:** The percentage of an agent's available time that is actually spent handling interactions. Unlike talk time ratio, occupancy excludes scheduled breaks and pause codes -- it only counts time when the agent was available and ready for calls.

**Benchmarks:** 80-90% for outbound predictive dialing. 75-85% for inbound operations. Sustained occupancy above 90% leads to agent burnout and increased turnover. Below 75% on outbound indicates dialer inefficiency or list exhaustion.

**Where to find it in VICIdial:** Not directly reported as a single metric. Calculate from Agent Time Detail: (Talk + Dispo + Hold) / (Talk + Dispo + Hold + Wait). Exclude time in pause codes (breaks, lunch, meetings).

**What to do when it's off:** High wait time between calls is the primary driver of low occupancy. On outbound, this means the dialer isn't generating calls fast enough. On inbound, it means call volume is below forecast. For blended operations, routing idle outbound agents to take inbound calls (or vice versa) is the primary occupancy optimization lever.

### [Agent Utilization](/glossary/agent-utilization/)

**Formula:** (Total Productive Time) / (Total Paid Time) x 100

**What it measures:** The broadest productivity metric -- what percentage of all paid hours are spent on revenue-generating activity. This includes talk time, after-call work, and other productive tasks, but excludes breaks, training, meetings, system downtime, and administrative time.

**Benchmarks:** 75-85% for well-managed operations. 85%+ is rare and usually unsustainable (agents need breaks). Below 70% means too much non-productive time is eating your payroll.

**Where to find it in VICIdial:** Requires combining data from VICIdial's Agent Time Detail with your HR/scheduling system. VICIdial tracks logged-in time and its subcategories, but doesn't know about total paid hours including offline activities.

**What to do when it's off:** Low utilization usually comes from excessive meetings, training that could be done during slow periods, long breaks between sessions, or system downtime. Track pause codes religiously -- they tell you exactly where non-productive time goes.

> **Are Your Agents Spending Half Their Shift Waiting?**
> Most VICIdial deployments run 15-20% below optimal talk time ratios. Our free audit measures your agent productivity metrics and shows you exactly where the lost time goes. [Get Your Free Audit -->](/free-audit/)

## Inbound KPIs: Every Metric That Matters

Inbound operations have a completely different optimization problem. You don't control how many calls come in -- you control how efficiently you handle them and how effectively you resolve customer needs. The KPIs shift from throughput and conversion to speed, quality, and resolution.

### [Service Level / SLA](/glossary/service-level-sla/)

**Formula:** Calls Answered Within Threshold / Total Calls Offered x 100

**What it measures:** The percentage of inbound calls answered within a target time -- usually expressed as "X% of calls answered in Y seconds." The most common target is 80/20 (80% of calls answered within 20 seconds), though this varies by industry and call type.

**Benchmarks:**

| Industry | Common SLA Target | Typical Achievement |
|---|---|---|
| General Customer Service | 80/20 | 75 - 85% |
| Technical Support | 80/30 | 70 - 80% |
| Sales / Order Taking | 90/15 | 80 - 90% |
| Healthcare | 80/30 | 70 - 80% |
| Financial Services | 80/20 | 75 - 85% |
| Emergency / Critical Support | 95/10 | 90 - 95% |

**Where to find it in VICIdial:** Inbound Report shows calls offered, calls answered, and answer speed distribution. The Service Level report (if enabled) directly calculates SLA against your configured threshold. You can also pull this from the Real-Time Report's queue statistics.

**What to do when it's off:** SLA failures almost always come from understaffing, unexpected call volume spikes, or long handle times. Short-term fixes: reduce after-call work time, pull agents from outbound campaigns, enable overflow routing. Long-term fix: improve your workforce management forecasting.

### [Abandon Rate](/glossary/abandon-rate/)

**Formula:** Abandoned Calls / Total Calls Offered x 100

**What it measures:** The percentage of inbound callers who hang up before reaching an agent. This is the most visible indicator of customer frustration with wait times.

**Benchmarks:**

| Scenario | Abandon Rate |
|---|---|
| Excellent | < 3% |
| Good | 3 - 5% |
| Average | 5 - 8% |
| Below Average | 8 - 12% |
| Poor | > 12% |

For outbound predictive dialing, abandon rate has a regulatory dimension: the FTC's Telemarketing Sales Rule caps outbound abandonment at 3% measured over a 30-day campaign period. VICIdial's adaptive algorithm is designed to stay within this limit, but aggressive dial ratios can push you over.

**Where to find it in VICIdial:** Inbound Report shows abandoned call count and percentage. For outbound abandon rate (dropped calls), the Campaign Detail report tracks drops as a percentage of human-answered calls. The Real-Time Report shows current queue depth and wait times -- if queue depth is climbing, abandons will follow.

**What to do when it's off:** Every second of wait time increases abandon probability. The relationship is nonlinear -- abandons are low for the first 20-30 seconds, then spike sharply. If your average speed of answer is over 30 seconds, abandon rate will be elevated regardless of what else you do. Fix the staffing or fix the handle time.

### Average Speed of Answer (ASA)

**Formula:** Total Wait Time for Answered Calls / Total Calls Answered

**What it measures:** The average time a caller waits in queue before reaching an agent. This is distinct from service level -- SLA is a threshold measurement (what percentage hit the target), while ASA is an average.

**Benchmarks:** 20-30 seconds for customer service, 10-20 seconds for sales queues, 30-60 seconds for technical support. ASA above 60 seconds is a red flag for any queue type.

**Where to find it in VICIdial:** Inbound Report shows average hold time (which in VICIdial terminology refers to queue wait time for inbound calls). Real-Time Report shows current wait time per queue.

**What to do when it's off:** ASA is a direct function of staffing versus volume. The Erlang C model -- the standard workforce management formula -- takes your call volume, AHT, and target ASA to calculate required staffing. If you're consistently missing ASA targets, you need either more agents, shorter handle times, or better call deflection (IVR self-service, callback options).

### [First Call Resolution](/glossary/first-call-resolution/) (FCR)

**Formula:** Issues Resolved on First Contact / Total Issues x 100

**What it measures:** The percentage of customer issues resolved during the first interaction, without requiring a callback, transfer, or follow-up. FCR is the strongest single predictor of customer satisfaction in inbound operations.

**Benchmarks:**

| Channel | FCR Range | Top Quartile |
|---|---|---|
| Phone (Customer Service) | 65 - 80% | 80%+ |
| Phone (Technical Support) | 55 - 75% | 75%+ |
| Email | 50 - 65% | 65%+ |
| Chat | 60 - 75% | 75%+ |

**Where to find it in VICIdial:** FCR isn't natively calculated by VICIdial. You need to define it through dispositions: mark calls as "Resolved" or "Requires Follow-Up" and track the ratio. Alternatively, measure repeat calls -- if the same customer calls back within 24-48 hours on the same issue, the first call wasn't resolved. The Closer In-Group report can help identify repeat callers by phone number.

**What to do when it's off:** Low FCR is usually caused by one of three things: agents lack the authority to resolve issues (policy problem), agents lack the knowledge to resolve issues (training problem), or the issue genuinely requires multi-step resolution (process problem). Fix in that order -- authority is fastest, process redesign is slowest.

### Customer Satisfaction Score (CSAT)

**Formula:** Positive Responses / Total Survey Responses x 100

**What it measures:** Direct customer feedback on their interaction experience, typically collected via post-call IVR survey or follow-up SMS/email. Usually measured on a 1-5 or 1-10 scale.

**Benchmarks:** 80-85% positive (4-5 on a 5-point scale) is typical for well-run inbound operations. Below 75% indicates systemic issues. Above 90% is exceptional. Note that CSAT suffers from response bias -- unhappy customers are more likely to respond, so the true satisfaction level is usually higher than reported.

**Where to find it in VICIdial:** If you're using VICIdial's built-in survey functionality (post-call IVR), results are stored in the survey log and accessible through custom reports. Most operations use external survey tools (e.g., a post-call SMS with a link) and track CSAT outside VICIdial.

**What to do when it's off:** CSAT is a lagging indicator driven by FCR, ASA, and agent quality. Fix those three and CSAT follows. The most impactful quick win is reducing transfers -- every transfer drops CSAT by 10-15 points on average.

## Blended & Campaign-Level KPIs

These metrics apply across both inbound and outbound operations, or they measure performance at the campaign or center level rather than the individual interaction level.

### [Cost Per Lead](/glossary/cost-per-lead/) (CPL)

**Formula:** Total Campaign Cost / Qualified Leads Generated

**What it measures:** The fully loaded cost to produce one qualified lead. This is the master outbound metric -- the one number that tells you whether your operation is healthy or hemorrhaging cash.

**Benchmarks:** Varies enormously by vertical. Solar residential: $25-$55 median $38. Insurance: $30-$60 median $42. B2B appointment setting: $55-$150 median $95. Home services: $15-$35 median $24. For the complete benchmark table and a deep dive into the formula, see our [full CPL benchmarks guide](/blog/call-center-cost-per-lead-benchmarks/).

**Where to find it in VICIdial:** VICIdial doesn't calculate CPL natively because it doesn't track costs. You need to export lead counts from the Campaign Status report (filter by qualified dispositions) and divide by your total campaign costs from your accounting system. ViciStack's analytics dashboard automates this calculation by integrating cost data with VICIdial disposition data.

**What to do when it's off:** CPL is a composite. When it moves, something upstream changed. Trace back through the chain: did contact rate drop? Did conversion rate drop? Did costs increase? Our [ROI formula guide](/blog/call-center-roi-formula/) walks through the full diagnostic framework.

### Cost Per Acquisition (CPA) / Cost Per Sale

**Formula:** Total Campaign Cost / Total Closed Sales (or Acquisitions)

**What it measures:** The fully loaded cost to produce one closed sale or customer acquisition. CPA goes one step further than CPL by measuring the final revenue-producing event rather than an intermediate qualification step.

**Benchmarks:**

| Vertical | CPA Range | Median CPA |
|---|---|---|
| Solar (Residential) | $250 - $600 | $400 |
| Insurance (Medicare) | $150 - $400 | $250 |
| Home Improvement | $120 - $350 | $200 |
| Debt Settlement | $300 - $700 | $480 |
| B2B SaaS | $400 - $1,200 | $700 |
| Home Services | $80 - $200 | $130 |

**Where to find it in VICIdial:** Same approach as CPL but filter dispositions to closed sales only (SALE, CLOSED, etc.) rather than all qualified leads.

**What to do when it's off:** If CPL is fine but CPA is high, the problem is downstream -- your closers or your transfer process are losing qualified leads. If both CPL and CPA are high, the problem is upstream in your outbound operation.

### Revenue Per Call / Revenue Per Contact

**Formula:** Total Revenue Attributed / Total Calls (or Contacts)

**What it measures:** The average revenue generated per call attempt or per live contact. This connects dialing activity directly to revenue and helps you evaluate whether a campaign is worth running at all.

**Benchmarks:** Highly variable by vertical and offer. For outbound sales, typical revenue per contact ranges from $2-$15 for B2C and $10-$50 for B2B. Revenue per dial is typically 10-20% of revenue per contact (because 80-90% of dials don't reach a live person).

**Where to find it in VICIdial:** Requires integrating revenue data from your CRM or sales system with VICIdial call data. Export total dials and contacts from the Outbound Calling Report, then divide attributed revenue.

### Schedule Adherence

**Formula:** Time Spent in Scheduled Activity / Total Scheduled Time x 100

**What it measures:** Whether agents are doing what they're supposed to be doing when they're supposed to be doing it. This includes being logged in during scheduled shifts, taking breaks at scheduled times, and being in the correct campaigns or queues.

**Benchmarks:** 90-95% is the target for most operations. Below 85% indicates a workforce management or discipline problem. Above 97% is essentially perfect and may indicate the schedule isn't being tracked accurately.

**Where to find it in VICIdial:** The Agent Detail report shows login/logout times and pause code usage by time of day. Compare against your schedule in your WFM tool. VICIdial's Timeclock feature can track scheduled vs. actual hours if enabled.

**What to do when it's off:** Low schedule adherence kills inbound service levels because staffing models assume agents will be available when scheduled. If adherence is low, figure out whether it's a few chronic offenders (discipline issue) or a systemic problem (schedules are unrealistic, break rooms are too far from desks, login process is too slow).

### After-Call Work (ACW) Time

**Formula:** Total Disposition + Notes Time / Total Calls Handled

**What it measures:** The time agents spend on administrative tasks after each call -- dispositioning, writing notes, updating CRM records, scheduling callbacks. ACW is part of AHT but worth tracking separately because it's the most compressible component.

**Benchmarks:** 15-45 seconds for simple outbound dispositions. 30-90 seconds for inbound customer service. 60-180 seconds for complex B2B or technical support. If ACW exceeds 60 seconds on outbound calls, your disposition workflow needs simplification.

**Where to find it in VICIdial:** Agent Time Detail report shows "Dispo" time, which is VICIdial's term for ACW. Campaign settings control the maximum allowed dispo time before auto-pause.

**What to do when it's off:** Simplify disposition codes (most campaigns need 8-12, not 40). Implement auto-fill for common fields. Use VICIdial's custom disposition form to capture only what's necessary. Every 10 seconds shaved off ACW across 100 calls per agent per day is 17 minutes of recovered productive time -- per agent.

## Agent-Level vs. Campaign-Level vs. Center-Level Metrics

One of the biggest mistakes in contact center analytics is measuring things at the wrong level. An agent's conversion rate tells you about that agent. A campaign's conversion rate tells you about the script, the data, and the offer. The center's conversion rate tells you almost nothing useful because it blends everything together.

### Agent-Level Metrics

These should be tracked per agent, compared against peer averages, and used for coaching and performance management:

| Metric | What It Tells You | Action Threshold |
|---|---|---|
| Conversion Rate | Sales skill and script adherence | Below campaign average by 2+ points |
| AHT | Efficiency and call control | 20%+ above or below campaign average |
| [Calls Per Hour](/glossary/calls-per-hour/) | Productivity and system usage | Below 80% of campaign average |
| ACW Time | Process efficiency | 50%+ above campaign average |
| Schedule Adherence | Reliability and discipline | Below 90% |
| Quality Score | Compliance and customer experience | Below 80% on monitored calls |
| Hold Time | Knowledge gaps | 2x+ campaign average |
| Transfer Rate | Escalation patterns | 50%+ above campaign average |

**Where to find it in VICIdial:** Agent Performance Detail report is the starting point. For deeper analysis, the Agent Time Detail report breaks down every minute of an agent's day. The Recording Log lets you listen to individual calls for quality scoring.

### Campaign-Level Metrics

These should be tracked per campaign, compared against vertical benchmarks, and used for strategic decisions about scaling, sunsetting, or optimizing campaigns:

| Metric | What It Tells You | Action Threshold |
|---|---|---|
| CPL | Overall campaign economics | 20%+ above vertical median |
| Contact Rate | Data quality and DID health | Below 8% on any data type |
| Conversion Rate | Script-market fit | Below vertical median |
| [Abandon Rate](/glossary/abandon-rate/) | Dialer aggressiveness or staffing gaps | Above 3% (outbound), above 5% (inbound) |
| List Penetration | Data exhaustion | Above 80% penetrated |
| Revenue Per Lead | Downstream value | Declining trend over 4+ weeks |
| Callback Success Rate | Recycling effectiveness | Below 40% callback completion |

**Where to find it in VICIdial:** Campaign Summary report gives you the top-level view. Outbound Calling Report gives you dialing and contact metrics. Campaign Status report gives you disposition distributions.

### Center-Level Metrics

These should be tracked for the entire operation, reported to leadership, and used for budgeting and strategic planning:

| Metric | What It Tells You | Review Frequency |
|---|---|---|
| Total CPL (blended) | Operational efficiency | Weekly |
| Revenue / Seat / Month | Productivity at scale | Monthly |
| [Agent Utilization](/glossary/agent-utilization/) | Workforce efficiency | Weekly |
| Attrition Rate | Retention and culture health | Monthly |
| Cost Per Seat / Hour | Total operational cost | Monthly |
| Compliance Incident Rate | Risk exposure | Weekly |
| System Uptime | Technology reliability | Daily |

**Where to find it in VICIdial:** Most center-level metrics require aggregating data across multiple VICIdial reports. The System Summary report gives you high-level call volume data. For a more comprehensive dashboard, see our guide on [building VICIdial dashboards with Grafana and Metabase](/blog/vicidial-grafana-dashboards/).

## Real-Time vs. Historical Metric Tracking

Not every metric needs to be watched in real time. Staring at a live dashboard that shows 30 numbers updating every 5 seconds creates anxiety, not insight. The key is knowing which metrics require real-time monitoring (because you can and should act immediately when they move) and which only need historical review (because the trends matter more than the moment).

### Metrics That Require Real-Time Monitoring

These metrics need to be visible on a wallboard or live dashboard because they indicate problems that get worse by the minute if left unaddressed:

| Metric | Why Real-Time Matters | Alert Threshold |
|---|---|---|
| Queue Depth / Calls Waiting | Callers are actively waiting and may abandon | > 2x staffed agents |
| Service Level (rolling 30 min) | SLA failures compound within the half-hour | Drops below 70% |
| Abandon Rate (rolling 30 min) | Indicates immediate staffing shortage | Exceeds 8% |
| Agent Status (Ready/Paused/Talk) | Identifies agents in excessive pause or dead air | Agent in pause > 10 min without approved code |
| Outbound Drop Rate | FTC compliance violation risk | Exceeds 2.5% (approaching 3% limit) |
| Hopper Level | Dialer will stall if hopper runs dry | Below 2x agent count |
| Trunk Utilization | Insufficient trunks throttle dialing | Above 85% capacity |

**Where to find it in VICIdial:** The Real-Time Report is your primary tool. It shows agent statuses, queue depths, wait times, and campaign performance in a live-updating view. Configure the Real-Time Report's refresh interval to 5-10 seconds for wallboard use. VICIdial also supports custom real-time monitoring via the non-agent API, which can feed external dashboards.

### Metrics That Require Historical / Trend Analysis

These metrics are meaningless in real time because they need larger sample sizes and trend context to be actionable:

| Metric | Why Historical Matters | Review Frequency |
|---|---|---|
| CPL | Daily fluctuations are noise; weekly trends are signal | Weekly |
| Conversion Rate | Needs 100+ contacts to be statistically meaningful | Weekly by campaign |
| [AHT](/glossary/average-handle-time/) | Daily averages are affected by call mix; weekly trends show real changes | Weekly by agent |
| [FCR](/glossary/first-call-resolution/) | Requires callback tracking over 24-48 hours | Weekly |
| CSAT / NPS | Survey sample sizes need 50+ responses | Monthly |
| Agent Utilization | Needs full shift data, not partial | Weekly |
| Contact Rate by List Source | Needs 1,000+ dials per source to be reliable | Weekly |
| Schedule Adherence | Partial-day data is misleading | Weekly by agent |

**Where to find it in VICIdial:** Historical reports are accessed through Admin > Reports. Key reports: Outbound Calling Report (date range selectable), Agent Performance Detail (supports custom date ranges), Campaign Status (cumulative or periodic), and Export Calls Data (raw call records for custom analysis).

### Building a Monitoring Cadence

The monitoring cadence we recommend for a 25+ agent operation:

**Every 5-10 minutes (wallboard):** Agent statuses, queue depth, service level, abandon rate, outbound drop rate, hopper level

**Every hour (supervisor check-in):** Talk time ratio by agent, conversion rate vs. target (if volume is sufficient), callback queue status

**Daily (end-of-shift review):** DPH and CPH by agent, contact rate by campaign, AHT trends, schedule adherence, disposition distribution

**Weekly (management review):** CPL by campaign, conversion rate by agent and campaign, list source performance, agent utilization, quality scores

**Monthly (strategic review):** CPA, revenue per lead, attrition rate, cost per seat, vendor performance, compliance incidents

> **Want Real-Time Dashboards Without the Dashboard Build?**
> ViciStack's analytics platform connects to your VICIdial data and delivers pre-built dashboards for every KPI listed above -- real-time wallboards, agent scorecards, and campaign performance trending. [See What You're Missing -->](/free-audit/)

## KPI Benchmarks Master Table

Consolidated reference table below. Bookmark this section. Print it. Tape it to the wall next to your wallboard.

### Outbound Metrics Benchmarks

| KPI | Formula | Below Average | Average | Good | Excellent |
|---|---|---|---|---|---|
| Contact Rate | Contacts / Dials | < 7% | 7 - 11% | 11 - 16% | > 16% |
| [Conversion Rate](/glossary/conversion-rate/) | Leads / Contacts | < 3% | 3 - 5% | 5 - 8% | > 8% |
| Dials Per Hour | Dials / Agent Hrs | < 120 | 120 - 170 | 170 - 220 | > 220 |
| Talk Time Ratio | Talk / Logged-In | < 30% | 30 - 40% | 40 - 50% | > 50% |
| [Occupancy](/glossary/occupancy-rate/) | Productive / Available | < 75% | 75 - 82% | 82 - 88% | > 88% |
| AHT (Lead Qual) | Talk+Hold+ACW / Calls | > 210 sec | 150 - 210 sec | 100 - 150 sec | < 100 sec |
| ACW Time | Dispo Time / Calls | > 45 sec | 30 - 45 sec | 15 - 30 sec | < 15 sec |
| [Abandon Rate](/glossary/abandon-rate/) (Outbound) | Drops / Human Answers | > 5% | 3 - 5% | 1 - 3% | < 1% |

### Inbound Metrics Benchmarks

| KPI | Formula | Below Average | Average | Good | Excellent |
|---|---|---|---|---|---|
| [Service Level](/glossary/service-level-sla/) (80/20) | Ans in 20s / Offered | < 70% | 70 - 78% | 78 - 85% | > 85% |
| Abandon Rate | Abandoned / Offered | > 10% | 6 - 10% | 3 - 6% | < 3% |
| ASA | Wait Time / Answered | > 45 sec | 25 - 45 sec | 15 - 25 sec | < 15 sec |
| [FCR](/glossary/first-call-resolution/) | Resolved First / Total | < 60% | 60 - 70% | 70 - 80% | > 80% |
| AHT (Cust Service) | Talk+Hold+ACW / Calls | > 500 sec | 360 - 500 sec | 240 - 360 sec | < 240 sec |
| CSAT | Positive / Responses | < 72% | 72 - 80% | 80 - 88% | > 88% |
| Transfer Rate | Transferred / Handled | > 20% | 12 - 20% | 6 - 12% | < 6% |

### Center-Level Benchmarks

| KPI | Below Average | Average | Good | Excellent |
|---|---|---|---|---|
| [Agent Utilization](/glossary/agent-utilization/) | < 68% | 68 - 76% | 76 - 84% | > 84% |
| Schedule Adherence | < 85% | 85 - 90% | 90 - 95% | > 95% |
| Attrition Rate (Monthly) | > 12% | 7 - 12% | 4 - 7% | < 4% |
| System Uptime | < 98% | 98 - 99.2% | 99.2 - 99.8% | > 99.8% |
| Cost Per Agent Hour | > $38 | $30 - $38 | $24 - $30 | < $24 |

## Setting KPI Targets: A Practical Framework

Benchmarks are useful for context, but your targets should be set based on your operation's current performance, not industry averages. This is how we set targets that are ambitious but achievable.

### Step 1: Establish Your Baseline

Pull 4-6 weeks of data for every KPI. Don't use a single week -- call center performance has natural week-to-week variation driven by data freshness cycles, staffing fluctuations, and seasonal patterns. You need enough data to see through the noise.

### Step 2: Identify Your Quartile

For each metric, determine where you sit relative to the benchmarks:

- **Bottom quartile:** Your target is reaching the "Average" column within 90 days
- **Average range:** Your target is reaching "Good" within 90 days
- **Good range:** Your target is reaching "Excellent" within 6 months
- **Excellent range:** Your target is holding this level while improving other metrics

### Step 3: Set Cascading Targets

Start with the lagging indicator target (e.g., "Reduce CPL from $52 to $42 within 90 days"), then work backward to the leading indicator targets that will get you there:

Example cascade for a CPL reduction from $52 to $42:

| Metric | Current | 30-Day Target | 60-Day Target | 90-Day Target |
|---|---|---|---|---|
| CPL | $52.00 | $48.00 | $44.50 | $42.00 |
| Contact Rate | 9.2% | 11.0% | 12.5% | 13.0% |
| Conversion Rate | 4.8% | 5.0% | 5.5% | 6.0% |
| Talk Time Ratio | 32% | 38% | 42% | 44% |
| ACW Time | 52 sec | 40 sec | 30 sec | 25 sec |

The 30-day targets are infrastructure wins (dialer tuning, AMD optimization). The 60-day targets add data strategy improvements. The 90-day targets layer in agent performance gains.

### Step 4: Assign Accountability

Every KPI target needs an owner:

- **Contact Rate** -- owned by the dialer administrator or operations manager
- **Conversion Rate** -- owned by the sales manager or team leads
- **Talk Time Ratio** -- owned by the dialer administrator
- **AHT / ACW** -- owned by team leads and QA
- **Service Level / Abandon Rate** -- owned by the workforce management team
- **CPL** -- owned by the campaign manager (accountable for the composite)

A metric without an owner is a metric that won't improve.

## Common KPI Mistakes and How to Avoid Them

### Mistake 1: Optimizing One Metric at the Expense of Others

The classic example: pushing agents to reduce AHT causes them to rush calls, which drops conversion rate, which raises CPL. Every KPI exists in tension with others. When you set a target for one metric, check the adjacent metrics for unintended consequences.

**The fix:** Always set KPI targets in groups, not individually. If you're targeting lower AHT, simultaneously set a floor on conversion rate. If you're targeting higher conversion, simultaneously set a ceiling on AHT.

### Mistake 2: Comparing Agents Across Different Campaigns

An agent running a warm solar appointment campaign at 8% conversion is not outperforming an agent running cold B2B appointment setting at 3% conversion. The campaigns have entirely different dynamics. Compare agents against the average for their specific campaign, not against each other across campaigns.

**The fix:** Normalize agent metrics by campaign. Report agent performance as a percentage of campaign average (e.g., "+15% above campaign conversion rate") rather than absolute numbers.

### Mistake 3: Treating Weekly Variance as Signal

Contact rate dropped from 11.2% to 10.4% this week. Is something wrong? Maybe. Or maybe it's normal variance. In a 25-agent operation making 693,000 dials per month, weekly contact rate will naturally fluctuate by 0.5-1.5 percentage points based on list mix, day-of-week effects, and random chance.

**The fix:** Use 4-week rolling averages for trend analysis. Only investigate when a metric moves more than 2 standard deviations from its rolling average, or when the rolling average itself shows a sustained directional change over 3+ weeks.

### Mistake 4: Ignoring Qualitative Metrics

Numbers tell you what happened. They don't tell you why. A conversion rate drop could be a script problem, a data quality shift, a competitor launching a better offer, or regulatory changes making prospects more cautious. You won't find the cause in the quantitative KPIs alone.

**The fix:** Supplement quantitative KPIs with regular call monitoring, agent feedback sessions, and disposition code analysis. VICIdial's recording log lets you listen to calls by disposition -- pull a sample of 20-30 non-converting calls each week and listen for patterns.

### Mistake 5: Reporting Vanity Metrics to Leadership

Leadership doesn't need to know your dials per hour. They need to know CPL, CPA, revenue per lead, and ROI. Reporting operational metrics to people who can't act on them creates confusion and erodes trust. Report outcomes to leadership, reserve operational metrics for operational managers.

**The fix:** Build two dashboards. The leadership dashboard shows 5-7 lagging indicators with trend lines. The operations dashboard shows 15-20 leading and lagging indicators with drill-down capability. For guidance on building these, see our [Grafana and Metabase dashboard guide](/blog/vicidial-grafana-dashboards/).

## Putting It All Together: The KPI Review Meeting

The agenda we recommend for a weekly KPI review with an operations team of 25+ agents:

**Duration:** 30 minutes. Not 60. If your KPI meeting takes an hour, you're reviewing too many metrics or discussing action items that belong in separate follow-ups.

**Attendees:** Campaign managers, team leads, QA lead, dialer administrator. Not agents -- agent coaching happens 1-on-1.

**Agenda:**

1. **Scorecard review (5 min):** CPL, conversion rate, contact rate, service level -- the top-line numbers. Are they on target? Green/yellow/red status for each campaign.

2. **Variance analysis (10 min):** For any metric in yellow or red, what changed? Trace the causal chain from lagging to leading indicators. Did contact rate drop because a list expired? Did conversion drop because we changed the script? Did AHT spike because we onboarded new agents?

3. **Action items from last week (5 min):** Did we implement the changes we committed to? What was the measurable result?

4. **New action items (10 min):** Based on today's data, what are the top 1-2 changes we'll implement this week? Assign owners, set expected impact, define how we'll measure it.

That's it. No deep dives into agent-level data (that's for team lead 1-on-1s). No system architecture discussions (that's for the technical team). No strategic planning (that's monthly). The weekly KPI meeting is purely: where are we, what changed, what are we doing about it.

## Frequently Asked Questions

### What are the most important KPIs for an outbound call center?

The five that matter most for outbound operations are: Cost Per Lead (the master metric), Contact Rate (the biggest operational lever), Conversion Rate (the revenue driver), Talk Time Ratio (the efficiency indicator), and Abandon Rate (the compliance guardrail). If you can only track five numbers, track those. CPL tells you whether you're making or losing money. Contact rate and conversion rate are the two multipliers that determine lead volume. Talk time ratio tells you whether your dialer is actually working. And abandon rate keeps you out of trouble with the FTC's 3% rule. Every other outbound metric is either an input to these five or a secondary diagnostic you only need when one of these five goes sideways.

### How do I calculate cost per lead for my call center?

Fully loaded CPL = Total Campaign Cost / Qualified Leads Generated. The key word is "fully loaded" -- include agent wages, payroll taxes, benefits, telecom, list/data purchases, dialer infrastructure, management and QA salaries, compliance costs, facilities, and overhead. Most centers only count agent wages and list costs, which understates CPL by 30-60%. Calculate it at the campaign level first, never as a blended average across campaigns. And count only leads that survive qualification -- not raw dispositions. Our [CPL benchmarks guide](/blog/call-center-cost-per-lead-benchmarks/) walks through a detailed worked example with a cost breakdown table for a 25-agent operation.

### What's a good conversion rate for outbound calls?

It depends on the vertical, the data type, and what you're converting to. For B2C outbound lead qualification on warm data, 5-8% is good and above 8% is excellent. For cold data, 2-4% is good. For B2B appointment setting, 1-3% is realistic and above 3% is strong. The biggest factor isn't agent skill -- it's data quality and recency. Fresh inbound leads (0-7 days old) convert at 2-4x the rate of aged purchased lists, regardless of who's dialing them. If your conversion rate is below the vertical median in our benchmarks table, check the data source before blaming agents.

### How do I measure agent performance in VICIdial?

Start with the Agent Performance Detail report, which shows calls handled, talk time, pause time, disposition time, and conversion metrics per agent for any date range. The Agent Time Detail report breaks down every minute of the agent's shift. For quality measurement, use the Recording Log to pull and score a sample of each agent's calls weekly. The most actionable agent metrics are: conversion rate vs. campaign average, AHT vs. campaign average, calls per hour, schedule adherence, and quality scores from call monitoring. Compare agents against their campaign peers, not against agents on different campaigns. A 6% converter on a cold data campaign is likely outperforming a 6% converter on warm inbound leads.

### What's the difference between occupancy rate and utilization rate?

[Occupancy rate](/glossary/occupancy-rate/) measures the percentage of an agent's *available* time spent handling calls (talk + hold + after-call work / available time). It excludes breaks, training, and other scheduled off-phone activities. [Agent utilization](/glossary/agent-utilization/) measures the percentage of an agent's total *paid* time spent on productive work, including off-phone productive activities. Occupancy can be 88% while utilization is 76% -- because the agent was only available for calls during 86% of their shift (the rest was breaks, meetings, and training). Both matter. Occupancy tells you whether the dialer is keeping agents busy when they're available. Utilization tells you whether the overall schedule is efficient. Target 82-88% occupancy and 76-84% utilization. Sustained occupancy above 90% leads to burnout and turnover.

### How often should I review contact center KPIs?

Real-time metrics (queue depth, agent status, service level, abandon rate, outbound drop rate) should be on a wallboard or live dashboard and monitored continuously by supervisors. Daily metrics (DPH, CPH, contact rate, schedule adherence) should be reviewed at end-of-shift by team leads. Weekly metrics (CPL, conversion rate, AHT trends, list source performance, agent utilization) should be reviewed in a 30-minute management meeting. Monthly metrics (CPA, revenue per lead, attrition rate, cost per seat, CSAT/NPS) should go to leadership. The biggest mistake is reviewing weekly-cadence metrics in real time -- daily fluctuations in conversion rate or CPL are noise, not signal. Use 4-week rolling averages for trend analysis.

### What is a good service level for an inbound call center?

The industry standard is 80/20 -- 80% of calls answered within 20 seconds. But "good" depends on your context. Sales queues where every missed call is lost revenue should target 90/15 or better. Technical support with complex issues can often get away with 80/30. Cost-sensitive operations sometimes drop to 70/30 and absorb the higher abandon rate. The tradeoff is always service level vs. staffing cost -- achieving 90/15 requires significantly more agents than 80/20 for the same call volume. Use Erlang C calculations to model the staffing impact of different [SLA](/glossary/service-level-sla/) targets before committing. Whatever target you set, measure it in 30-minute intervals, not daily averages -- a daily average of 82% can mask severe intraday dips that drive abandonment and customer frustration during peak hours.

### How do I improve first call resolution in my contact center?

[First call resolution](/glossary/first-call-resolution/) improves through three levers, in order of speed and impact. First, expand agent authority -- if agents have to transfer or escalate issues they're capable of handling because policy won't let them, fix the policy. This is the fastest FCR win and costs nothing. Second, improve agent knowledge -- build a searchable knowledge base, create decision trees for common issues, and ensure agents can find answers without putting customers on hold. Third, redesign processes that inherently require multiple contacts -- if a request "always" needs a callback because a different department handles part of it, integrate that department's capability into the front-line agent's toolkit. Track FCR by measuring repeat calls from the same phone number within 48 hours. In VICIdial, you can identify repeat callers through the Closer In-Group report and call logs filtered by phone number. Target 75%+ FCR for customer service and 70%+ for technical support.

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/contact-center-kpis).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
