# VICIdial for Outbound Sales: The Complete Playbook

**The complete playbook for running outbound sales operations on VICIdial — from your first 10-agent campaign to a 500-seat operation. Campaign architecture, agent management, KPI frameworks, list strategy, AMD tuning, DID management, and the scaling decisions that separate operations that grow from operations that stall.**

---

Outbound sales is VICIdial's bread and butter. The platform was built for it. But there's a massive gap between "we installed VICIdial and started dialing" and "we're running a profitable outbound operation at scale." That gap is filled with campaign configuration decisions, agent management practices, list strategies, and scaling patterns that most operations learn the hard way — through wasted money and burned leads.

This guide is the playbook we wish existed when we started managing outbound VICIdial deployments. It covers everything from initial campaign setup through scaling to hundreds of agents, with the actual numbers, settings, and operational frameworks that drive real results. Whether you're running [solar appointments](/industries/solar/), [insurance lead gen](/industries/insurance/), [B2B prospecting](/industries/b2b-lead-generation/), or any other outbound vertical, the fundamentals covered here apply.

We're not going to waste your time with theory. This is configuration values, KPI targets, management frameworks, and scaling math — the stuff that moves the needle.

---

## The Outbound Sales Operation: Architecture Overview

Before diving into individual settings, you need to understand how all the pieces of an outbound VICIdial operation fit together. Every successful outbound operation has these layers:

**1. Campaign architecture.** How your campaigns are structured, what dial methods they use, how they relate to each other, and how leads flow between them.

**2. List management.** How leads enter the system, how they're prioritized, how they're recycled, and when they're retired. This is where most operations leave the most money on the table.

**3. Dialing optimization.** [Predictive dialer settings](/blog/vicidial-predictive-dialer-settings/), AMD configuration, dial timeouts, caller ID management — the technical layer that determines how efficiently your agents spend their time.

**4. Agent management.** Scripting, [disposition](/glossary/disposition/) workflows, coaching, quality monitoring, and the human side of the operation.

**5. KPI framework.** What you measure, how you measure it, and what you do when numbers are off.

**6. Scaling infrastructure.** The server architecture, telephony capacity, and operational processes that let you grow from 10 agents to 50 to 200 to 500 without the wheels falling off.

Each of these layers interacts with the others. Changing your dial method affects agent utilization, which affects KPIs, which changes your scaling math. This is why cookie-cutter VICIdial configurations underperform — the settings have to be tuned as a system, not as individual parameters.

Let's break down each layer.

---

## Campaign Architecture: Building the Foundation

### Single Campaign vs. Multi-Campaign Strategy

The most common mistake new outbound operations make is running everything through a single campaign. One campaign, one list, one set of settings, all agents in the same pool. This works at 5-10 agents. It falls apart at 25+.

Here's why: different lead types perform differently. Fresh web leads convert at 3-5x the rate of aged purchased lists. Callback leads that requested a specific time need different handling than cold prospects. Hot transfers from an IVR pre-qualification need a closer, not a qualifier. Running all of these through a single campaign with a single dial method and a single set of agent scripts means you're optimizing for nothing because you're trying to optimize for everything.

**The multi-campaign architecture that works:**

| Campaign | Purpose | Dial Method | Agents | Priority |
|---|---|---|---|---|
| SPEED-TO-LEAD | Fresh inbound web leads (< 30 min old) | RATIO at 1.0 | Dedicated top closers | Highest |
| PRIMARY-OUTBOUND | Main prospecting campaign | ADAPT_TAPERED | Core agent pool | High |
| CALLBACK | Scheduled callbacks from all campaigns | RATIO at 1.0 | Original agent or senior agents | High |
| RECYCLE | No-answer/busy recycling (2nd-6th attempts) | ADAPT_AVERAGE_BALANCE | Core agent pool | Medium |
| AGED-LEADS | Leads 30-90 days old, re-approach | RATIO at 1.5-2.0 | Junior agents or training pool | Lower |
| WINBACK | Previously not-interested, 60+ days cold | RATIO at 2.0 | Experienced agents, different pitch | Lowest |

**VICIdial implementation:** Create each as a separate campaign in Admin → Campaigns. Use [list mix](/settings/list-order/) or manual agent assignment to control which agents work which campaigns. For operations with enough agents, dedicate specific agents to high-priority campaigns (SPEED-TO-LEAD, CALLBACK) and let the core pool handle PRIMARY-OUTBOUND and RECYCLE.

### Dial Method Selection

Your dial method is the single most impactful technical decision in your campaign configuration. Get this wrong and you'll either starve your agents (too conservative) or burn your lists and rack up abandoned calls (too aggressive).

Here's when to use each method:

**RATIO mode** — You set a fixed [auto dial level](/glossary/auto-dial-level/) (e.g., 2.0 means 2 lines dialed per available agent). Use this when:
- You need precise control (speed-to-lead campaigns, callback campaigns)
- You're dialing a small or high-value list where abandons are unacceptable
- You're just starting a new campaign and don't have baseline data yet
- Compliance requires zero dropped calls (1:1 dialing)

**ADAPT_TAPERED** — Starts conservative and gradually increases dial level as the system learns the list's answer rate. Use this when:
- You're launching a new list and don't know the contact rate
- Your list quality is variable (mix of cell phones, landlines, disconnected numbers)
- You want automatic optimization but with a conservative bias

**ADAPT_AVERAGE_BALANCE** — Full [predictive dialing](/glossary/predictive-dialing/) that dynamically adjusts based on rolling averages of answer rate, agent availability, and talk time. Use this when:
- You have 15+ agents on the campaign (predictive algorithms need statistical mass)
- Your list quality is known and stable
- You've already run one pass and have baseline data
- Maximum agent utilization is the priority

**ADAPT_HARD_LIMIT** — Aggressive predictive mode that pushes dial levels higher. Use this when:
- You have 30+ agents and very high list volume
- Your lists have low contact rates (< 5%) and you need aggressive dialing to keep agents busy
- You can tolerate closer to the 3% abandon ceiling

For most outbound sales operations starting out, **ADAPT_TAPERED on the primary campaign and RATIO on specialty campaigns** is the right answer. Graduate to ADAPT_AVERAGE_BALANCE once you have stable baseline data and enough agents to make predictive algorithms work.

### Core Campaign Settings

Here's the starting configuration for a general outbound sales campaign. These are your baseline — adjust based on your specific vertical, list quality, and results.

```
Campaign Type: Outbound
Dial Method: ADAPT_TAPERED
Auto Dial Level: 1.5 (starting)
Adaptive Maximum Dial Level: 3.5
Adaptive Dropped Percentage: 2.5
Dial Timeout: 26 seconds
Drop Percentage Maximum: 3.0
Safe Harbor Message: Y
Safe Harbor Audio File: [your compliance message]
Hopper Level: 300
DNC Checking: CAMPAIGN AND SYSTEM
Local Call Time: [your custom call time]
Time Zone Checking: ENABLED
```

**Why these values:**

- **Adaptive Maximum Dial Level at 3.5** gives the predictive algorithm room to work while preventing runaway over-dialing. For operations under 20 agents, consider capping at 3.0. Over 50 agents with stable lists, you can push to 4.0.
- **Adaptive Dropped Percentage at 2.5%** keeps you under the TCPA 3% abandon rate ceiling with a safety margin. This is measured on a 30-day rolling basis per campaign. Running at exactly 3.0% is playing with fire — one bad hour pushes you over.
- **Dial Timeout at 26 seconds** is the sweet spot for mixed cell/landline lists. It catches nearly all live answers while avoiding most voicemail greetings. If your list skews heavily cell phone, you can drop to 24 seconds.
- **Hopper Level at 300** keeps the dialer fed without loading thousands of leads into memory that might become time-zone-restricted before they're dialed.

---

## List Management: Where Operations Win or Die

Your lists are your fuel. Bad list management is the number one reason outbound operations underperform, and it's the area most managers spend the least time on. They obsess over dial settings and scripts while their lists are a mess — duplicates, disconnected numbers, recycling too aggressively, no segmentation, no prioritization.

For the complete technical breakdown of VICIdial's list management system, see our [list management guide](/blog/vicidial-list-management/). Here we'll cover the strategic layer.

### Lead Loading and Deduplication

Every lead that enters VICIdial should go through these gates:

**1. Format validation.** Phone numbers cleaned to 10-digit format. Required fields populated. State/time zone mapped correctly. This happens before VICIdial — your lead intake process (API, file upload, CRM sync) should handle formatting.

**2. DNC scrubbing.** Check against the national DNC registry, your internal DNC list, and any state-specific DNC lists. VICIdial has built-in DNC checking, but pre-scrubbing your leads before loading reduces wasted hopper space and avoids even attempting to dial a DNC number.

**3. Deduplication.** VICIdial's `duplicate_check` parameter on the lead loading API supports several modes: `DUPLIST` (unique within a list), `DUPCAMP` (unique within a campaign), `DUPTITLEALTPHONELIST`, and others. At minimum, use `DUPCAMP` to prevent the same number from appearing in multiple lists assigned to the same campaign. For multi-campaign operations, implement system-wide dedup at the lead loading layer.

**4. Segmentation.** Tag leads with metadata that lets you segment and prioritize: lead source, age, geographic market, quality tier, previous disposition history. Load this data into VICIdial custom fields.

### List Prioritization

Not every lead deserves the same urgency. Configure your operation to burn through high-value leads first:

| Priority Tier | Lead Type | Target Speed | Strategy |
|---|---|---|---|
| 1 (Immediate) | Fresh web/inbound leads | < 5 minutes | Dedicated speed-to-lead campaign |
| 2 (Same day) | Scheduled callbacks | At scheduled time | Callback campaign, agent-specific |
| 3 (High) | Fresh purchased leads (0-48 hours) | Same day as received | Primary campaign, high hopper priority |
| 4 (Standard) | Active leads, 1st-3rd attempt | Within recycling schedule | Primary campaign, standard priority |
| 5 (Standard) | Active leads, 4th-6th attempt | Within recycling schedule | Recycle campaign |
| 6 (Low) | Aged leads 30-90 days | As capacity allows | Aged leads campaign |
| 7 (Lowest) | Winback / re-approach | Fill time only | Winback campaign |

In VICIdial, you control priority through hopper_priority on the API (1-99, where 99 is highest), list ordering within campaigns, and dedicated campaigns for priority tiers. The hopper pulls leads in priority order, so a Priority 99 lead will be dialed before a Priority 50 lead regardless of when it was loaded.

### Lead Recycling Strategy

Lead recycling is where you extract the second, third, and fourth pass of value from your lists. Most leads don't answer on the first attempt — industry-wide, first-pass contact rates run 5-15% depending on vertical and list quality. That means 85-95% of your leads need another attempt.

Your recycling rules should be disposition-based:

| Disposition | Recycle After | Max Total Attempts | Time-of-Day Strategy |
|---|---|---|---|
| NA (No Answer) | 2 hours minimum | 6 | Vary by 4+ hours each attempt |
| B (Busy) | 30 minutes | 4 | Same time window is fine |
| AM (Answering Machine) | 4 hours | 4 | Shift to evening if daytime AM |
| DC (Disconnected) | Never | 1 | Remove from all lists |
| NI (Not Interested) | 30-60 days | 2 | Different agent, different angle |
| CB (Callback) | At scheduled time | 3 | Honor the scheduled time |
| DNC | Never | 0 | Add to internal DNC immediately |

Configure these under Campaign → Disposition Recycle in the VICIdial admin panel. The critical principle is **time-of-day variance**: if a lead didn't answer at 10 AM, try them at 5 PM. If they didn't answer on Tuesday afternoon, try Thursday morning. People have patterns, and hitting the same pattern repeatedly is wasted dials.

For the full technical implementation of recycling rules and [callback automation](/blog/vicidial-callback-automation/), see our dedicated guides.

### When to Kill a List

Every list has a point of diminishing returns. Here's how to identify it:

- **Contact rate drops below 2%** on the current pass
- **Cost per contact exceeds 3x** your benchmark for that lead type
- **You're on the 5th+ attempt** for the majority of remaining leads
- **Disposition mix is 80%+ NA/AM** with minimal new contacts

When a list hits these thresholds, stop dialing it. Move remaining viable leads (those with fewer than max attempts) to your aged leads campaign and archive the rest. Continuing to pound a dead list wastes server resources, burns DIDs, and keeps agents in unproductive dial cycles.

---

## AMD Optimization: Stop Wasting Agent Time on Machines

Answering Machine Detection is the technology that determines whether a live person or a voicemail system answered the call. When it works, it saves agents from spending 20-30 seconds per call listening to "Hi, you've reached the Johnson family..." When it doesn't work, it either routes voicemails to agents (wasting time) or misclassifies live humans as machines (losing leads).

For the complete technical deep-dive, see our [AMD configuration guide](/blog/vicidial-amd-guide/). Here's what matters for outbound sales operations.

### Why AMD Matters for Your Bottom Line

On a typical outbound sales campaign calling mixed cell/landline lists:

- **40-60% of connected calls** reach voicemail or an answering machine
- **Without AMD**, agents spend 35-45% of their connected call time dealing with machines
- **With properly tuned AMD**, you recover 25-40% of that wasted time
- **On a 50-agent operation**, that's the equivalent of adding 12-20 agents of productive capacity without hiring anyone

The math is straightforward. If your agents cost $15/hour loaded and you have 50 of them, recovering 30% of machine time saves roughly $45,000/month in effective agent cost. AMD tuning isn't an optimization — it's a major P&L lever.

### Recommended AMD Settings for Outbound Sales

```
AMD Type: AMD
AMD Send to Agent: Y (for HUMAN results)

initialSilence: 2500
greeting: 2000
afterGreetingSilence: 1100
totalAnalysisTime: 5000
minimumWordLength: 120
betweenWordsSilence: 50
maximumNumberOfWords: 4
silenceThreshold: 256
```

**Key tuning considerations:**

- **afterGreetingSilence at 1100ms** is the most sensitive parameter. This is how long the system waits after the greeting speech ends before classifying as MACHINE. Too low (< 900ms) and you'll misclassify slow-to-respond humans as machines. Too high (> 1400ms) and you'll misclassify short voicemail greetings as humans. Start at 1100ms and adjust based on your HUMAN→disconnect rate.
- **maximumNumberOfWords at 4** works for most campaigns. A typical human answer is "Hello?" (1 word) or "Hello, who's this?" (3 words). A typical voicemail greeting is "Hi, you've reached John, please leave a message after the beep" (12+ words). Setting this to 4 means anything longer than 4 words is classified MACHINE. If your vertical calls demographics that tend to answer with longer greetings (e.g., business contacts who say "Good afternoon, this is John Smith with Acme Corp"), increase to 5-6.
- **Cell phone considerations:** Cell phone audio has more latency variance and background noise than landlines, which degrades AMD accuracy. If your lists are 80%+ cell phones (most consumer lists in 2026), expect AMD accuracy in the 75-85% range rather than the 85-95% you'd see on landline-heavy lists. Monitor and adjust weekly.

### Monitoring AMD Performance

Check these metrics weekly:

1. **AMD HUMAN → Agent Disconnect rate.** Pull the disposition report for calls classified AMD=HUMAN. What percentage did the agent immediately disposition as AM, MACH, or HANGUP? This is your false HUMAN rate. Target: under 10%.
2. **AMD MACHINE → Manual review.** Listen to a random sample of calls classified AMD=MACHINE. What percentage were actually live humans? This is your false MACHINE rate — and it represents lost leads. Target: under 5%.
3. **Overall AMD classification distribution.** What percentage of connected calls is AMD classifying as HUMAN vs. MACHINE? If you're seeing 70%+ MACHINE on a list you know has reasonable contact rates, AMD is over-classifying.

---

## DID Management and Caller ID Strategy

Your caller ID strategy directly impacts contact rates, which impacts every downstream metric — [cost per lead](/glossary/cost-per-lead/), [conversion rate](/glossary/conversion-rate/), revenue per agent hour, everything.

For the full technical guide, see our [DID management post](/blog/vicidial-did-management/). Here's the strategic framework for outbound sales.

### Local Presence: The Numbers

The data is consistent across verticals:

- **Local area code caller ID:** 8-14% answer rate
- **Toll-free (800/888) caller ID:** 4-7% answer rate
- **Out-of-area or unknown caller ID:** 2-5% answer rate

Local presence doubles your effective contact rate compared to toll-free and triples it compared to out-of-area. On a 100,000-lead list, that's the difference between 10,000 contacts and 4,000 contacts from the same list with the same agents and the same script.

### DID Pool Sizing

Plan your DID inventory based on:

- **5-10 DIDs per target area code** for moderate volume (under 500 calls/day to that area code)
- **10-20 DIDs per target area code** for high volume (500-2,000 calls/day)
- **Per-DID daily cap of 75-100 outbound calls** to avoid carrier analytics flags
- **20-30% quarterly replacement rate** — DIDs get flagged and need to be rotated

For a 50-agent operation calling nationally across 30 area codes at moderate volume, you need approximately 150-300 DIDs. That sounds like a lot. It is. DID management at scale is a real operational function, not a set-it-and-forget-it configuration.

### STIR/SHAKEN and Attestation

Every outbound DID needs A-level STIR/SHAKEN attestation from your SIP provider. A-level attestation means the carrier has verified your identity and your right to use that number. B or C attestation will result in "Spam Likely" labels on major carrier networks within days of high-volume dialing.

Confirm with your SIP provider that all your DIDs pass A-level attestation. Test by calling yourself and checking the caller ID display on AT&T, Verizon, and T-Mobile handsets. If you see "Spam Likely" or "Scam Likely," something is wrong.

### DID Health Monitoring

Build a weekly DID health check into your operations routine:

1. Run a spam reputation check on every active DID (use services like CallerID Reputation, Free Caller Registry, or Numeracle)
2. Pull per-DID answer rates from VICIdial reporting — a sudden drop in answer rate for a specific DID suggests it's been flagged
3. Immediately remove flagged DIDs from rotation
4. Rest flagged DIDs for 30-60 days, then re-test before returning to service
5. Replace permanently burned DIDs with fresh numbers

---

## Agent Management: The Human Layer

All the technical optimization in the world doesn't matter if your agents can't convert a live conversation into a sale or appointment. Agent management is the operational discipline that turns dialer efficiency into revenue.

### Agent Onboarding for VICIdial

New agents need to understand the VICIdial agent screen before they can focus on selling. Here's the onboarding sequence that works:

**Day 1: System training (4 hours)**
- VICIdial agent screen layout and controls
- How to log in, pause, resume, and log out
- Disposition codes and what each one means
- Transfer procedures (warm transfer, cold transfer, 3-way)
- Callback scheduling
- Custom fields and data entry requirements

**Day 1-2: Script training (4-8 hours)**
- Script delivery and branching logic
- Objection handling frameworks
- Qualification criteria (what makes a lead worth pursuing)
- Compliance requirements (disclosures, DNC requests, recording notifications)
- Role-playing with experienced agents or managers

**Day 3-5: Supervised live dialing**
- Agent takes live calls with a supervisor listening
- Real-time coaching on script delivery, tone, and pacing
- Supervisor reviews every disposition for accuracy
- Gradual reduction of supervision as competency builds

**Week 2: Independent dialing with monitoring**
- Agent works independently but with daily call review
- QA pulls 5-10 calls per day for scoring
- Daily debrief on performance metrics and improvement areas

The investment here is 5-7 working days before an agent is fully independent. Operations that shortcut this to 1-2 days pay for it in poor conversion rates, bad dispositions, and compliance violations for months.

### Disposition Discipline

Dispositions are the data layer of your operation. Every downstream process — recycling, reporting, KPI calculation, list management — depends on agents dispositioning calls correctly. Bad disposition data cascades into bad decisions.

**Standard outbound sales disposition set:**

| Code | Meaning | Action |
|---|---|---|
| SALE | Closed sale or set appointment | Push to CRM, commission trigger |
| CB | Callback requested | Schedule in callback campaign |
| NI | Not interested | Recycle in 30-60 days |
| NQ | Not qualified | Remove from campaign |
| DNC | Do not call | Add to internal DNC immediately |
| NA | No answer | Auto-recycle per rules |
| AM | Answering machine | Auto-recycle per rules |
| B | Busy | Auto-recycle per rules |
| HANGUP | Prospect hung up | Log, recycle in 7+ days |
| WRONG | Wrong number/person | Remove from campaign |
| DC | Disconnected number | Remove from all lists |
| XFER | Transferred to closer/specialist | Track transfer outcome |
| NAS | Not available - try later | Short-term callback |

**Common disposition mistakes that hurt operations:**

1. **Agents dispositioning as NI when the prospect didn't actually answer.** This poisons your NI recycle pool with non-contacts and inflates your "not interested" rate.
2. **Using NA for everything that isn't a sale.** If your NA rate is above 60% of total dispositions, agents are being lazy with disposition codes.
3. **Not dispositioning DNC requests correctly.** This is a compliance failure. Every "put me on the do not call list" request must be dispositioned DNC immediately. Train agents that DNC is not optional — it's a legal requirement.

Audit disposition accuracy weekly. Pull a random sample of 20-30 calls per agent, listen to the recordings, and compare the actual call outcome to the agent's disposition. Agents with disposition accuracy below 90% need retraining.

### Script Framework

The best outbound sales scripts follow a predictable structure:

**1. Opening (5-10 seconds).** Identify yourself, your company, and why you're calling. Be direct. "Hi [Name], this is [Agent] calling from [Company]. I'm reaching out because [reason]." Don't ask "How are you today?" — it screams telemarketer and costs you 2-3 seconds of tolerance from every prospect.

**2. Hook (10-15 seconds).** Give them a reason to keep listening. This is your value proposition compressed into one sentence. "We're helping homeowners in [area] cut their [cost/problem] by [benefit], and I wanted to see if that's something you'd be open to learning more about."

**3. Qualification (30-60 seconds).** Ask 2-4 qualifying questions to determine if this prospect is worth your agent's time. For B2B: decision-maker confirmation, company size, current solution. For consumer: eligibility criteria, need confirmation, timeline.

**4. Pitch (60-120 seconds).** For qualified prospects, deliver the value proposition. Focus on benefits, not features. Use specifics, not generalities. "Our clients typically save $200-400/month" is better than "we can save you money."

**5. Close (15-30 seconds).** Ask for the sale or appointment. Be specific: "I have availability this Thursday at 2 PM or Friday at 10 AM — which works better for you?" Don't ask "Would you like to schedule something?" — that's a yes/no question and the default answer is no.

**6. Objection handling.** Build 5-8 objection responses into the script for the most common pushbacks in your vertical. "I need to think about it" / "Send me information" / "I'm not interested" / "I already have a provider" / "What's the cost?" — your agents should have rehearsed responses for each.

Configure VICIdial scripts under Admin → Scripts with dynamic field substitution (--A--first_name--B--, --A--city--B--) so agents see personalized scripts for each call.

---

## KPI Framework: Measuring What Matters

You can't manage what you don't measure, but measuring too much creates noise that obscures signal. Here's the KPI framework that gives you actionable intelligence without drowning in data.

### Tier 1: Daily Operations KPIs (Check Every Day)

| KPI | Definition | Target Range | Why It Matters |
|---|---|---|---|
| Contact Rate | Contacts / Total Dials | 8-15% | Measures list + DID quality |
| Conversion Rate | Sales or Appointments / Contacts | 8-25% (vertical-dependent) | Measures agent + script effectiveness |
| Agent Utilization | Talk Time / Logged-In Time | 35-55% | Measures dialer efficiency |
| [Cost Per Lead](/glossary/cost-per-lead/) | Total Cost / Qualified Leads | Vertical-dependent | Bottom-line efficiency metric |
| Abandon Rate | Dropped Calls / Connected Calls | < 2.5% | Compliance + prospect experience |
| Dials Per Agent Hour | Total Dials / Agent Hours | 30-80 (method-dependent) | Dialer performance indicator |

### Tier 2: Weekly Strategic KPIs (Review Weekly)

| KPI | Definition | Target Range | Why It Matters |
|---|---|---|---|
| [Conversion Rate](/glossary/conversion-rate/) by Agent | Per-agent sales / contacts | Identify top/bottom performers | Coaching and staffing decisions |
| List Penetration | Leads Attempted / Total Leads | Track over time | List lifecycle management |
| AMD Accuracy | Correct classifications / Total | > 85% | Agent time optimization |
| DID Answer Rate | Per-DID answer rates | > 6% average | DID health monitoring |
| Callback Conversion | Callback contacts / Callback sets | > 60% contact, > 15% convert | Callback campaign effectiveness |
| Revenue Per Agent Hour | Total Revenue / Agent Hours | Vertical-dependent | Ultimate efficiency metric |

### Tier 3: Monthly Strategic KPIs (Monthly Review)

| KPI | Definition | Why It Matters |
|---|---|---|
| Customer Acquisition Cost | All-in cost per closed customer | Profitability |
| Lead Source ROI | Revenue by lead source / Cost by lead source | Budget allocation |
| Agent Ramp Time | Days to target productivity for new hires | Training effectiveness |
| Agent Retention | Monthly turnover rate | Operational stability |
| Compliance Score | QA pass rate on compliance elements | Risk management |

### VICIdial Reporting for KPIs

VICIdial's built-in reporting covers most of these metrics:

- **Agent Performance Report:** Calls handled, talk time, pause time, dispositions per agent. This is your daily management tool.
- **Campaign Summary Report:** Dials, contacts, abandon rate, dial level. This is your daily dialer performance tool.
- **Outbound Calling Report:** Detailed per-call data including AMD outcomes, dial time, talk time, dispositions. Use this for deep-dive analysis.
- **DID Report:** Per-DID call volume and outcomes. Use this for DID health monitoring.

For KPIs that VICIdial doesn't natively calculate (cost per lead, revenue per agent hour, lead source ROI), export VICIdial data to a spreadsheet or BI tool and combine with your financial data. The API makes this straightforward — pull call and disposition data programmatically and join it with cost and revenue data from your CRM and accounting system.

For a comprehensive guide to VICIdial's reporting capabilities, see our [reporting and monitoring guide](/blog/vicidial-reporting-monitoring/).

---

## Scaling: From 10 Agents to 500

Scaling an outbound VICIdial operation isn't linear. The challenges at 10 agents are completely different from the challenges at 50, which are different from 200, which are different from 500. Here's what changes at each stage and how to prepare.

### Stage 1: 10-25 Agents (Foundation)

**What works at this stage:**
- Single VICIdial server (properly specced — 32GB RAM, 8+ cores, SSD storage)
- 1-2 campaigns
- Manual agent management (the manager knows every agent personally)
- Simple list loading (CSV uploads, maybe basic API)
- Basic reporting (VICIdial built-in reports)

**What you should be building:**
- Disposition discipline from day one — this is 10x harder to fix later
- KPI tracking, even if it's a spreadsheet
- Documented processes for list loading, DNC management, agent onboarding
- A proper DID rotation strategy (don't wait until you're flagged)

**Common mistakes at this stage:**
- Skipping AMD configuration because "it's only 10 agents"
- No QA process because "I can hear everyone from my desk"
- Loading all leads into one list with no segmentation
- Running a single caller ID for all outbound calls

**Server sizing:** A single ViciBox server with 32GB RAM, 8-core CPU, and SSD storage handles 25 concurrent agents comfortably. Monitor CPU usage during peak dialing — if Asterisk is consistently above 60% CPU, you're approaching the ceiling.

### Stage 2: 25-75 Agents (Professionalization)

This is where amateur operations stall and professional operations separate. The transition from "small operation where everyone knows everything" to "structured operation with defined roles and processes" happens here.

**What changes:**
- You need a dedicated QA function (can't rely on the manager listening ad hoc)
- Multi-campaign architecture becomes necessary (see campaign architecture section above)
- List management requires a defined process and probably a dedicated person
- Agent management requires team leads / supervisors (1 supervisor per 10-15 agents)
- DID pool grows to 50-150 numbers and needs active management
- Reporting shifts from "how did we do today" to "what's trending this week/month"

**Infrastructure scaling:**
- Single server can handle up to ~50 concurrent agents if properly tuned
- Beyond 50 agents, consider a two-server setup: one for web/database, one for telephony (Asterisk)
- SIP trunk capacity: plan for 3-4x your agent count in concurrent SIP channels (75 agents = 225-300 concurrent channels) to support predictive dialing overhead
- For cloud deployments, see our [VICIdial cloud deployment guide](/blog/vicidial-cloud-deployment/)

**Process formalization:**
- Written agent scripts (not verbal "just follow my lead")
- Daily team huddles (15 minutes: yesterday's numbers, today's focus)
- Weekly QA reviews with each agent
- Monthly list performance analysis
- Documented escalation procedures for compliance issues (DNC requests, threats, etc.)

### Stage 3: 75-200 Agents (Operations Management)

At this stage, you're running a real call center operation. The challenges shift from "how do I configure VICIdial" to "how do I manage a complex operation."

**What changes:**
- Multiple supervisors, possibly a call center manager separate from the business owner
- Workforce management becomes critical — scheduling, shift optimization, break management
- Training becomes a continuous function, not an onboarding event
- Technology stack expands: QA tools, workforce management, analytics/BI, CRM integration
- Compliance becomes a formal program with regular audits
- DID management at this scale is a part-time or full-time job

**Infrastructure scaling:**
- VICIdial cluster is mandatory beyond ~100 agents. See our [cluster configuration guide](/blog/vicidial-cluster-guide/) for architecture details.
- Typical cluster: 1 web/database server + 2-4 telephony servers
- Database optimization becomes important — see our [MySQL optimization guide](/blog/vicidial-mysql-optimization/)
- Redundancy: you need failover for your web/database server. A 200-agent operation losing its dialer for 2 hours costs thousands of dollars.
- SIP trunk diversity: don't run all traffic through a single SIP provider. Split across 2-3 providers for redundancy and rate optimization.

**Management infrastructure:**
- Real-time wallboard for supervisor monitoring (VICIdial's real-time report, supplemented with custom dashboards)
- Automated alerts: abandon rate exceeding threshold, agent idle time spikes, system performance degradation
- Agent scorecards: weekly performance summary for each agent combining QA scores, KPIs, and compliance metrics
- Incentive programs tied to quality metrics (not just volume)

### Stage 4: 200-500 Agents (Enterprise Scale)

At this scale, you're operating an enterprise contact center. The operational complexity increases exponentially.

**What changes:**
- Dedicated IT/systems team for VICIdial infrastructure
- Formal workforce management with forecasting and scheduling software
- Multi-site or remote agent deployment (VICIdial with [WebRTC/ViciPhone](/blog/vicidial-webrtc-setup/) supports remote agents natively)
- Enterprise CRM integration with real-time data sync
- Dedicated compliance officer or team
- Advanced analytics: predictive modeling for list performance, agent attrition risk, optimal staffing models

**Infrastructure scaling:**
- VICIdial cluster with 4-8+ telephony servers
- Database replication for reporting (don't run heavy reports against your production database)
- Load-balanced web servers for the agent interface
- Dedicated recording storage (500 agents generate 50-100GB of call recordings per day)
- Network engineering: QoS configuration, dedicated VLANs for voice traffic, redundant internet connections
- SIP trunk capacity: 1,500-3,000+ concurrent channels across multiple providers

**Operational considerations at 500 agents:**
- Agent attrition management is a strategic function. At 500 agents with typical outbound turnover (60-100% annually), you're hiring and training 25-40 new agents per month.
- Campaign segmentation is granular: different campaigns for different products, markets, lead types, times of day, and agent skill levels.
- You need a data team: someone analyzing list performance, lead source ROI, predictive dialing optimization, and agent effectiveness at a level that goes beyond VICIdial's built-in reports.

> **Running an outbound operation on VICIdial and not sure if your configuration is leaving money on the table?** We audit VICIdial deployments from 10 to 500+ seats and identify the specific settings changes that improve contact rates, agent utilization, and cost per lead. [Request a free audit](/free-audit/) — we'll tell you exactly what to fix.

---

## Compliance Framework for Outbound Sales

Compliance isn't optional and it isn't a one-time configuration. It's an ongoing operational discipline. The regulatory landscape for outbound calling tightens every year, and the penalties for violations are severe enough to shut down an operation.

### Federal Requirements

**TCPA (Telephone Consumer Protection Act):**
- Maximum 3% abandon rate per campaign over a 30-day rolling window
- Prior express consent required for autodialed calls to cell phones
- One-to-one consent rule (January 2025): consent must be specific to your company
- Safe harbor message required for abandoned calls
- Calling hours: 8:00 AM - 9:00 PM local time of the called party
- Internal DNC list maintenance: honor opt-out requests within 30 days (immediately is best practice)

**TSR (Telemarketing Sales Rule - FTC):**
- Caller must disclose identity and purpose of the call promptly
- Must honor DNC requests
- No misleading or deceptive representations
- Specific disclosure requirements for certain industries

**STIR/SHAKEN:**
- All outbound calls should carry A-level attestation
- Calls without proper attestation are increasingly blocked or labeled by carriers

### VICIdial Compliance Configuration Checklist

```
Drop Percentage Maximum: 3.0 (hard ceiling)
Adaptive Dropped Percentage: 2.5 (operational target)
Safe Harbor Message: Y
Safe Harbor Audio File: [compliance message identifying your company]
DNC Checking: CAMPAIGN AND SYSTEM
Time Zone Checking: ENABLED
Local Call Time: [configured per legal calling hours]
Recording: ALL (required for dispute resolution)
Agent DNC button: ENABLED (agents must be able to DNC a number immediately)
```

### State-Specific Considerations

Multiple states have telemarketing laws that are stricter than federal requirements. The big ones:

- **Florida (FTSA):** 3 calls per 24 hours max, 8 AM - 8 PM, state DNC registration required
- **California:** CCPA data requirements, written consent for automated cell calls, 15-day opt-out processing
- **Texas (SB 140):** State registration required, $10,000/violation
- **Virginia:** 10-year opt-out retention requirement
- **Oklahoma (OTSA):** Registration and strict calling hour restrictions

For the complete compliance configuration walkthrough, see our [TCPA compliance guide](/blog/vicidial-tcpa-compliance/).

---

## Advanced Optimization: Extracting More From What You Have

Once your foundation is solid — campaigns structured, lists managed, AMD tuned, DIDs rotating, agents trained — these are the optimizations that push you from average to top-quartile performance.

### Time-of-Day Optimization

Pull your contact rate and conversion rate data broken down by hour of day and day of week. You'll find patterns:

- Certain hours have 2-3x the contact rate of others
- Conversion rates often peak at different hours than contact rates (evening contacts may be higher but conversion may peak mid-morning when people are more focused)
- Weekend patterns differ dramatically from weekday patterns

Use this data to:
1. **Staff your highest-performing hours with your best agents.** If 4-7 PM has 2x the contact rate, that's where your closers should be.
2. **Adjust dial levels by time of day.** VICIdial doesn't natively support time-based dial level changes, but you can create separate campaigns with different settings and shift agents between them.
3. **Schedule list types by time of day.** Dial B2B leads during business hours, consumer leads during evening hours.

### Agent Skill-Based Routing

Not all agents are equal. Your top 20% of agents likely produce 50%+ of your conversions. Route your highest-value leads to your best agents:

**VICIdial implementation options:**
- **Separate campaigns:** Put your best leads in a campaign staffed only by top performers.
- **Agent rank/grade:** VICIdial supports agent grade settings that influence call routing priority. Higher-ranked agents receive calls before lower-ranked agents within the same campaign.
- **Closer campaigns:** For two-stage sales (qualifier → closer), use VICIdial's CLOSER campaigns to route qualified transfers to your best closers.

### Voicemail Drop

For answering machine dispositions, leaving a targeted voicemail message can generate inbound callbacks at 2-8% rates. VICIdial supports automatic voicemail message drop when AMD detects a machine:

```
Answering Machine Message: Y
Answering Machine Audio File: [your VM drop recording]
```

The voicemail should be 15-25 seconds, include your company name, a brief value proposition, and a callback number that routes to a VICIdial inbound campaign. Track the callback rate by tagging the voicemail-drop DID as a specific inbound group.

This is essentially free lead generation from calls that would otherwise be completely wasted.

### Multi-Channel Follow-Up

VICIdial's Dispo Call URL feature lets you trigger external actions on specific dispositions. Use this to build multi-channel follow-up:

- **NI disposition → SMS follow-up** via your SMS platform (30 days later, "We spoke last month about [product], wanted to check if anything's changed...")
- **AM disposition → Email** with your value proposition and a call-to-action
- **SALE disposition → CRM push + confirmation SMS + calendar invite**
- **CB disposition → SMS confirmation** of the callback time

This requires middleware between VICIdial and your SMS/email platform, but the infrastructure is straightforward and the ROI is significant.

---

## Vertical-Specific Notes

While the fundamentals in this guide apply across all outbound sales verticals, each industry has specific nuances:

### Solar Lead Generation

Solar is one of the highest-volume VICIdial verticals. The key differences: homeowner qualification is complex (property ownership, roof type, utility provider, credit), seasonality is dramatic, and regulatory scrutiny is intense. For the complete solar playbook, see our [VICIdial solar lead generation guide](/blog/vicidial-solar-lead-generation/) and [solar industry page](/industries/solar/).

### Insurance

Insurance outbound (health, auto, life, home) has its own compliance layer — state-specific licensing requirements mean your agents may only be authorized to sell in specific states. VICIdial's lead filter and list segmentation should enforce state-level routing. Medicare calling has additional CMS regulations on calling hours and script requirements. See our [insurance industry page](/industries/insurance/).

### B2B Lead Generation

B2B outbound has fundamentally different contact patterns — you're calling business numbers during business hours, dealing with gatekeepers, and typically running a two-stage process (qualify → schedule meeting for sales rep). Contact rates are generally higher (15-25%) but decision-maker reach rates are lower. TCPA rules differ for business numbers (more permissive for autodialing). See our [B2B lead generation industry page](/industries/b2b-lead-generation/).

---

## Putting It All Together: The 90-Day Launch Plan

If you're launching a new outbound sales operation on VICIdial or rebuilding an existing one, here's the phased approach:

### Days 1-14: Foundation

- [ ] VICIdial installed and configured on properly specced hardware
- [ ] Primary outbound campaign created with ADAPT_TAPERED, settings per this guide
- [ ] Callback campaign created with RATIO at 1.0
- [ ] DID pool acquired (5-10 per target area code), AREACODE CID group configured
- [ ] AMD configured and tested with recommended parameters
- [ ] Disposition set defined and loaded
- [ ] Scripts written and loaded into VICIdial
- [ ] Call times configured with time zone enforcement
- [ ] DNC lists loaded (national + internal)
- [ ] First lead list loaded, scrubbed, and segmented

### Days 15-30: Launch and Calibrate

- [ ] First agents trained and on live calls (start with 5-10)
- [ ] Daily KPI tracking in place
- [ ] AMD accuracy monitored and first tuning pass completed
- [ ] DID answer rates monitored
- [ ] Disposition accuracy audited (listen to calls, compare to dispositions)
- [ ] Lead recycling rules configured based on first two weeks of disposition data
- [ ] CRM integration tested and live
- [ ] First list performance analysis completed

### Days 31-60: Optimize

- [ ] Dial method graduated from ADAPT_TAPERED to ADAPT_AVERAGE_BALANCE if data supports it
- [ ] Agent performance rankings established — top/bottom performers identified
- [ ] Time-of-day optimization data collected and shift schedules adjusted
- [ ] Lead source performance analyzed — double down on what works, cut what doesn't
- [ ] QA program formalized (scoring rubric, weekly reviews)
- [ ] Second wave of agents added if KPIs support expansion
- [ ] DID health first monthly review completed

### Days 61-90: Scale

- [ ] Agent count ramped to target staffing level
- [ ] Multi-campaign architecture fully implemented (speed-to-lead, primary, recycle, aged)
- [ ] Automated reporting/dashboards in place
- [ ] Supervisor structure formalized (1:10-15 supervisor to agent ratio)
- [ ] Lead pipeline formalized (sources, loading frequency, quality criteria)
- [ ] First monthly business review completed (P&L, KPIs, compliance audit)
- [ ] Infrastructure scaled to support current and projected agent count

> **Want a faster path to a fully optimized outbound operation?** ViciStack manages VICIdial infrastructure and configuration for outbound operations from 10 to 500+ seats. We handle the technical complexity so you can focus on sales. [See our pricing](/pricing/) or [request a free infrastructure audit](/free-audit/).

---

## Frequently Asked Questions

### How many agents do I need to make predictive dialing effective in VICIdial?

Predictive dialing algorithms need statistical mass to work correctly. The system uses rolling averages of answer rate, agent availability, and talk time to predict when to launch the next batch of calls. With fewer than 10-12 agents, the sample sizes are too small for accurate prediction, and you'll see erratic dial levels — aggressive one minute, starved the next. At 10-15 agents, ADAPT_TAPERED works reasonably well. At 20+ agents, ADAPT_AVERAGE_BALANCE becomes reliable. At 50+ agents, the predictive algorithm is highly efficient and you'll see agent utilization rates in the 45-55% range. If you're running under 10 agents, stick with RATIO mode and manually set your dial level based on observed contact rates.

### What's a realistic cost per lead on VICIdial for outbound sales?

It depends entirely on your vertical, list quality, and operation maturity. For B2B appointment setting, expect $25-$75 per qualified appointment on purchased lists. For consumer verticals (solar, insurance, home services), expect $15-$50 for exclusive web leads and $40-$120 for cold list prospecting. The all-in agent cost (salary, benefits, seat cost, telephony, leads) typically runs $18-$30 per agent hour for US-based operations and $8-$15 for offshore. Your [cost per lead](/glossary/cost-per-lead/) is that hourly cost divided by leads per agent hour. A well-optimized operation generating 1.5 qualified leads per agent hour at $22/hour all-in runs a $14.67 CPL. Check our [cost per lead benchmarks](/blog/call-center-cost-per-lead-benchmarks/) for detailed vertical breakdowns.

### Should I use AMD or let agents handle answering machines manually?

Use AMD. The math is overwhelming in favor of it. Without AMD, agents on a typical consumer list spend 35-45% of their connected call time on answering machines. With properly tuned AMD, you recover 25-40% of that wasted time, which is equivalent to adding 25-40% more productive agent capacity at zero additional cost. The only scenario where AMD is counterproductive is if your lists are very small and very high-value (e.g., a 200-lead list of C-suite executives where a false MACHINE classification on a live answer is devastating). For standard outbound sales volume, always run AMD. Tune it using the parameters in this guide and monitor accuracy weekly.

### How do I prevent my caller IDs from getting flagged as spam?

Four things matter: per-DID daily volume limits (75-100 calls max per DID per day), A-level STIR/SHAKEN attestation from your SIP provider, inbound routing on every outbound DID (so callbacks don't hit a dead number), and active reputation monitoring. Use VICIdial's AREACODE-based CID groups to distribute calls across your DID pool automatically. Monitor spam reputation weekly using a service like Numeracle or Free Caller Registry. The moment a DID shows "Spam Likely" on any major carrier, pull it from rotation immediately — continuing to use a flagged DID damages your entire operation's reputation, not just that one number. Budget for 20-30% quarterly DID replacement in your telephony costs.

### What's the difference between VICIdial's campaign types for outbound sales?

VICIdial has three main dial methods relevant to outbound sales: RATIO, ADAPT (with variants), and MANUAL. RATIO lets you set a fixed dial level (e.g., 2.0 = two lines dialed per available agent) — use this for small campaigns, high-value leads, or when you need precise control. ADAPT_TAPERED starts conservative and ramps up as the system learns the list — best for new campaigns or variable list quality. ADAPT_AVERAGE_BALANCE is full [predictive dialing](/glossary/predictive-dialing/) that dynamically adjusts based on rolling statistics — best for 20+ agent campaigns with known list quality. ADAPT_HARD_LIMIT is aggressive predictive mode for high-volume operations comfortable running closer to the abandon rate ceiling. MANUAL disables the auto-dialer entirely, requiring agents to click-to-dial — used for preview dialing where agents need to review lead information before calling.

### Can VICIdial handle remote agents for outbound sales?

Yes, and it's increasingly common. VICIdial supports remote agents through two methods: SIP softphone (agents install a SIP client on their computer and connect to the VICIdial Asterisk server over the internet) and WebRTC via ViciPhone (agents use a browser-based phone built into the VICIdial agent interface). WebRTC is the preferred method for remote agents — no software installation, works through the browser, and handles NAT traversal better than traditional SIP. The key requirements are stable internet (minimum 100kbps up/down per agent, but 1Mbps+ recommended), low latency (under 150ms round-trip to the VICIdial server), and QoS configuration on the agent's network to prioritize voice traffic. At scale, remote agents perform within 5-10% of in-office agents on most KPIs — the performance gap is almost entirely in management and monitoring, not technology.

### How do I handle callback management at scale in VICIdial?

Callbacks are one of the highest-converting lead types in outbound sales — the prospect explicitly asked you to call back, which is an intent signal that cold leads don't have. VICIdial supports two types: ANYONE callbacks (any available agent handles the callback at the scheduled time) and USERONLY callbacks (only the original agent handles it). For sales operations, USERONLY callbacks convert 20-30% better because the prospect already has rapport with that agent. Configure a dedicated callback campaign with RATIO at 1.0 (no predictive over-dialing — every callback should reach an agent). Set the callback alert to notify agents 2-5 minutes before their scheduled callback. At scale (100+ agents), callback volume can be significant — expect 10-20% of daily dials to be callbacks. Monitor callback contact rate (should be 40-60%) and callback-to-conversion rate (should be 15-30%, significantly above cold contact rates). For a complete callback setup, see our [callback automation guide](/blog/vicidial-callback-automation/).

### What VICIdial server specs do I need for my agent count?

General guidelines: **10-25 agents** — single server, 32GB RAM, 8-core CPU, SSD storage, handles VICIdial web, database, and Asterisk on one box. **25-50 agents** — single server with 64GB RAM, 16-core CPU, dedicated SSD for database, or begin splitting to two servers. **50-100 agents** — two-server minimum: one for web/database (32-64GB RAM), one for Asterisk telephony (32GB RAM, CPU-optimized). **100-200 agents** — web/database server + 2-3 telephony servers, database on dedicated SSD or NVMe. **200-500 agents** — full cluster: load-balanced web servers, replicated database, 4-8 telephony servers, dedicated recording storage. Always monitor CPU utilization on your Asterisk servers — if you're consistently above 70% during peak dialing, add another telephony node. For detailed architecture guidance, see our [VICIdial cluster guide](/blog/vicidial-cluster-guide/).

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-outbound-sales).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
