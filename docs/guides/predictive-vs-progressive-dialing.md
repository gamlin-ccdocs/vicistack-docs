# Predictive vs Progressive vs Preview Dialing: When to Use Each

The dialing mode you choose for a campaign is the single most impactful configuration decision in outbound calling. It determines how many contacts your agents make per hour, how much idle time they endure between calls, how many leads you burn through per shift, and whether your operation stays on the right side of FCC and [TCPA](/glossary/tcpa/) regulations. Get it wrong and you either leave money on the table (too conservative) or catch a compliance violation that costs more than a month of revenue (too aggressive).

Despite this, most call center operators either stick with whatever their dialer defaulted to, or switch to "predictive" because someone told them it was faster — without understanding the trade-offs, the agent count thresholds, or the compliance implications.

This guide covers the three primary dialing modes in depth: [predictive](/glossary/predictive-dialing/), [progressive](/glossary/progressive-dialing/), and [preview](/glossary/preview-dialing/). For each mode, we explain how it works technically, how VICIdial implements it, when to use it, and what performance to expect. We also cover hybrid approaches, VICIdial-specific configurations, compliance considerations, and a decision framework you can apply to any campaign.

---

## How Each Dialing Mode Works

### Predictive Dialing: The Algorithm

Predictive dialing is exactly what the name implies — the dialer **predicts** when agents will become available and places calls in advance so a live call is waiting when the agent finishes their current one.

The algorithm continuously calculates the optimal number of lines to dial per available agent (the "dial ratio") based on several real-time inputs:

- **Agent availability:** How many agents are currently available or about to become available
- **Average talk time:** How long agents typically spend on connected calls
- **[Average wrap time](/glossary/wrap-time/):** How long agents spend in after-call work
- **[Answer rate](/glossary/answer-rate/):** What percentage of dialed calls actually get answered
- **Current [abandon rate](/glossary/abandon-rate/):** What percentage of answered calls are dropped because no agent was available

Here's the simplified logic: If the algorithm knows 5 agents will be available in 30 seconds (based on their average talk time), and the historical answer rate is 25%, and dial-to-connect takes an average of 15 seconds, it needs to place approximately 20 calls right now so that 5 of them answer just as the agents become free.

In VICIdial, this algorithm runs as a Perl script called `AST_VDadapt.pl`. It recalculates every 15 seconds — examining the current state of all agents, all active calls, recent abandon rates, and campaign performance metrics — and adjusts the dial level up or down accordingly.

**The fundamental trade-off:** Predictive dialing maximizes agent talk time but creates a non-zero probability that a call will be answered with no agent available. This "abandoned" call is a compliance risk under FCC regulations, which mandate a maximum 3% [abandon rate](/settings/abandon-rate-target/) measured across a rolling 30-day period.

### Progressive Dialing: One Call, One Agent

Progressive dialing places **exactly one call per available agent.** When an agent becomes available, the dialer places one call for that agent. If the call is answered, the agent handles it. If it's not answered (busy, no answer, voicemail), the dialer immediately places another call for that agent.

There's no prediction involved. No algorithm calculating future agent availability. No risk of answered calls with nobody to handle them.

**The mechanical simplicity is the point.** Progressive dialing guarantees:
- Zero abandoned calls (every answered call has an agent waiting)
- Every agent gets connected to every answered call (no algorithmic drops)
- Compliance with all outbound calling regulations by design

The trade-off is lower throughput. Agents spend more time waiting between calls because the dialer isn't proactively filling the pipeline. Instead of hearing a connected call within 1-2 seconds of finishing the previous one (predictive), agents wait 10-20 seconds while the dialer places a call, waits for it to ring, determines it won't be answered, and tries the next number.

### Preview Dialing: Agent-Initiated Calls

Preview dialing shows the agent a lead's information **before** the call is placed. The agent reviews the data — name, company, notes from previous calls, account status — and then manually initiates the call when ready. Some implementations include a countdown timer that auto-dials if the agent doesn't act within a set period.

**This isn't dialing at all in the traditional sense.** The agent is in control. They decide when to call, can skip leads they're not prepared for, and can research the contact before initiating conversation.

The productivity hit is significant: agents spend 30-120 seconds reviewing each lead before calling, plus the ring/answer time, plus the actual conversation. But for complex sales cycles, high-value accounts, or situations requiring pre-call research, that preparation time pays for itself in conversion rate.

---

## VICIdial's Implementation of Each Mode

VICIdial doesn't use the terms "predictive," "progressive," and "preview" in quite the way the industry does. Understanding the mapping between industry terminology and VICIdial's actual settings is crucial for configuring the right behavior.

### Dial Method Settings in VICIdial

The primary control is **Campaign Detail → Dial Method.** Here's how VICIdial's options map to industry-standard dialing modes:

| VICIdial Dial Method | Industry Equivalent | How It Works |
|---------------------|---------------------|-------------|
| **RATIO** | Fixed-ratio dialing | Dials at a fixed ratio you set (e.g., 3 lines per agent). Not truly predictive — doesn't adapt to conditions. |
| **ADAPT_AVERAGE** | Progressive / Adaptive | Adjusts dial ratio based on campaign stats, targets average performance. Conservative. |
| **ADAPT_TAPERED** | Adaptive progressive | Similar to ADAPT_AVERAGE but tapers off more aggressively when approaching drop limits. Most conservative adaptive mode. |
| **ADAPT_HARD_LIMIT** | Predictive with safety net | Adapts aggressively but enforces a hard drop rate ceiling. |
| **INBOUND_MAN** | Preview / Manual | Agents see lead data and initiate calls manually. |
| **MANUAL** | Manual dialing | No auto-dialing at all. Agent manually enters numbers. |

### True Predictive Dialing in VICIdial: RATIO + Tuning

**Here's the industry secret that vendor marketing departments don't tell you:** VICIdial's RATIO mode with a skilled administrator is functionally predictive dialing. Set the [auto dial level](/settings/auto-dial-level/) to `3.0-5.0`, enable [available only ratio tally](/settings/available-only-ratio-tally/) = `Y`, and configure the [dial timeout](/settings/dial-timeout/) appropriately — and you have a predictive dialer that's manually tuned rather than algorithmically adaptive.

The difference between RATIO at 4.0 and a commercial "predictive dialer" like Convoso's is that RATIO holds steady at 4:1 regardless of conditions, while a predictive algorithm adjusts that ratio second by second. In practice, an experienced administrator monitoring the campaign can adjust RATIO periodically and achieve similar results — it's just not hands-free.

### Adaptive (Progressive-Like) Dialing: ADAPT Modes

The three ADAPT modes are where VICIdial implements genuine algorithmic dialing — and they're closer to progressive than predictive in most configurations.

**ADAPT_AVERAGE:**
- Calculates dial ratio based on average campaign metrics
- Adjusts every 15 seconds via `AST_VDadapt.pl`
- Default behavior keeps the dial ratio relatively conservative
- Won't exceed the [Adaptive Maximum Dial Level](/settings/adaptive-dl-level/) you set

**ADAPT_TAPERED:**
- Same as ADAPT_AVERAGE but reduces dial ratio more aggressively as the campaign's drop rate approaches the ceiling
- This is the safest auto-dial mode for compliance-sensitive campaigns
- Typically produces 30-40 minutes of talk time per agent per hour

**ADAPT_HARD_LIMIT:**
- Adapts aggressively toward the maximum dial level
- Enforces a hard ceiling on drop rate — if drops exceed the target, the algorithm immediately reduces the dial level
- Most aggressive adaptive mode — closest to true predictive behavior

### Critical VICIdial Settings for Each Mode

**For Predictive (RATIO) Campaigns:**

| Setting | Value | Purpose |
|---------|-------|---------|
| [Auto Dial Level](/settings/auto-dial-level/) | 3.0-5.0 | Lines per available agent |
| [Available Only Ratio Tally](/settings/available-only-ratio-tally/) | Y | Only count truly available agents in ratio calculation |
| [Dial Timeout](/settings/dial-timeout/) | 18-22 seconds | How long to ring before giving up |
| [Drop Rate Group](/settings/drop-rate-group/) | CAMPAIGN_ONLY | Monitor drops per-campaign |
| [Hopper Level](/settings/hopper-level/) | 200-500 | Leads pre-loaded for immediate dialing |
| [AMD Type](/settings/amd-type/) | CPD or AMD | Answering machine detection mode |

**For Progressive (ADAPT) Campaigns:**

| Setting | Value | Purpose |
|---------|-------|---------|
| Dial Method | ADAPT_TAPERED | Progressive-like behavior |
| [Adaptive DL Level](/settings/adaptive-dl-level/) | 1.5-2.5 | Maximum ratio the algorithm can reach |
| [Abandon Rate Target](/settings/abandon-rate-target/) | 1-2% | Conservative drop target |
| [Drop Lockout Time](/settings/drop-lockout-time/) | 1 day | How long a dropped lead is locked from redialing |
| Auto Dial Level | 1.0 | Starting dial ratio (algorithm adapts up) |

**For Preview (MANUAL/INBOUND_MAN) Campaigns:**

| Setting | Value | Purpose |
|---------|-------|---------|
| Dial Method | INBOUND_MAN | Shows lead data, agent initiates call |
| Manual Dial | Y | Agent has dial button |
| [Lead Order](/settings/lead-order/) | UP_LAST_NAME_ASC or MANUAL | How leads are presented to agents |

> **Not Sure Which Mode Your Campaign Needs?**
> ViciStack configures every campaign for the optimal dial method based on your agent count, vertical, and compliance requirements. [Get Your Free Campaign Assessment →](/free-audit/)

---

## When to Use Each Mode: The Decision Framework

### Predictive Dialing: Maximum Throughput

**Use predictive when:**

- You have **15+ agents** on a single campaign. Below 15, the algorithm doesn't have enough statistical data to predict accurately, and the dial ratio becomes erratic.
- Your campaign is **high-volume, low-complexity** — consumer outbound, appointment setting, surveys, debt collection reminders.
- Your operation can tolerate a **1-3% abandon rate** (within FCC limits).
- [Lead lists](/blog/vicidial-list-management/) are large enough that burning through contacts quickly is acceptable.
- Revenue per contact is **moderate** — you make money on volume, not individual deal size.
- Your compliance team and [TCPA](/glossary/tcpa/) exposure are managed.

**Don't use predictive when:**
- You have fewer than 10 agents per campaign
- You're calling cell phones in a TCPA-sensitive context (prior express consent not clearly documented)
- Your leads are high-value and limited — predictive's speed burns through lists fast
- You're in a heavily regulated vertical where any abandoned call creates liability

**Performance benchmarks (predictive):**
- Agent talk time: **45-55 minutes per hour** at 25+ agents
- Agent talk time: **38-45 minutes per hour** at 15-25 agents
- Contacts per hour: **8-15** (depending on answer rate and talk time)
- List burn rate: **200-400 dials per agent per 8-hour shift**

### Progressive Dialing: The Compliance Default

**Use progressive when:**

- You have **fewer than 15 agents** per campaign. Progressive works well at any scale — even 3 agents.
- **B2B campaigns** where contact rates are lower and each contact is more valuable.
- **Compliance is a primary concern** — TCPA-sensitive campaigns, cell phone lists without clear consent documentation, financial services under CFPB oversight.
- **High-value leads** where you want agents to handle every answered call — no algorithmic drops.
- You need to **preserve lead quality** by not over-dialing your list.
- The campaign is in a **state with strict calling regulations** (California, Florida, etc.)

**Performance benchmarks (progressive):**
- Agent talk time: **30-40 minutes per hour**
- Contacts per hour: **5-10**
- List burn rate: **100-200 dials per agent per 8-hour shift**
- Abandon rate: **0%** (inherently compliant)

### Preview Dialing: Quality Over Quantity

**Use preview when:**

- Each call requires **pre-call research** — reviewing account history, looking up previous notes, checking external data sources.
- **Complex B2B sales** where the agent needs to understand the prospect's business before calling.
- **High-value accounts** where a poorly prepared call damages the relationship (enterprise sales, wealth management, executive-level outreach).
- **Callback campaigns** where the agent needs context from the previous interaction.
- **Collections** where the agent needs to review the account balance, payment history, and applicable regulations before initiating contact.
- Your agents are **highly skilled and expensive** — paying them to research is worth it because their conversion rate on prepared calls justifies the lower throughput.

**Performance benchmarks (preview):**
- Agent talk time: **15-25 minutes per hour**
- Contacts per hour: **3-6**
- List burn rate: **30-80 dials per agent per 8-hour shift**
- Conversion rate: **1.5-3x higher than predictive** on the same leads (because agents are prepared)

### The Decision Flowchart

```
START: How many agents on this campaign?
│
├─ Less than 10 → Progressive (ADAPT_TAPERED)
│   └─ Are leads high-value/complex? → Yes → Preview (INBOUND_MAN)
│
├─ 10-15 agents
│   ├─ B2B or compliance-sensitive? → Yes → Progressive (ADAPT_AVERAGE)
│   └─ B2C high-volume? → Progressive (ADAPT_HARD_LIMIT)
│
└─ 15+ agents
    ├─ TCPA-sensitive (cell phones, uncertain consent)?
    │   └─ Yes → Progressive (ADAPT_TAPERED with 1% drop target)
    │   └─ No → Continue below
    ├─ Lead volume unlimited, moderate deal value?
    │   └─ Yes → Predictive (RATIO at 3.0-5.0)
    ├─ Lead volume limited, high deal value?
    │   └─ Yes → Progressive (ADAPT_AVERAGE with 2.0 max)
    └─ Complex sale requiring research?
        └─ Yes → Preview (INBOUND_MAN)
```

---

## Compliance Implications by Dialing Mode

The compliance differences between dialing modes aren't academic — they're measured in six-figure penalties and class-action lawsuits.

### FCC Abandoned Call Rules

The FCC's Telemarketing Sales Rule requires that no more than **3% of answered calls may be abandoned** (no agent available within 2 seconds of the consumer saying "hello"), measured across a **30-day rolling period** per campaign.

| Dialing Mode | Abandon Risk | Compliance Approach |
|--------------|-------------|---------------------|
| Predictive | **Inherent risk** — abandoned calls are a mathematical certainty at high dial ratios | Monitor and enforce 3% ceiling via [drop rate group](/settings/drop-rate-group/) settings. Use [safe harbor message](/settings/safe-harbor-message/) on abandoned calls. |
| Progressive | **Zero risk** — one call per agent means no abandoned calls | Inherently compliant |
| Preview | **Zero risk** — agent initiates every call | Inherently compliant |

### TCPA Implications

The Telephone Consumer Protection Act restricts automated dialing to cell phones without prior express consent. The legal definition of an "automatic telephone dialing system" (ATDS) has been the subject of extensive litigation.

**Predictive dialing** clearly uses an ATDS — the system selects numbers from a list and dials them automatically without human intervention in the dialing decision.

**Progressive dialing** is less clear. The system automatically dials, but at a 1:1 ratio triggered by agent availability. Some courts have ruled this constitutes an ATDS; others disagree.

**Preview dialing** is the safest. The agent reviews the lead and initiates the call. This is generally not considered ATDS-driven dialing because a human is making the dialing decision.

**Practical guidance for VICIdial operators:**
- For cell phone lists with documented prior express consent: predictive is defensible
- For cell phone lists without clear consent: progressive (ADAPT_TAPERED) minimizes ATDS exposure
- For cell phone lists where consent is uncertain: preview is the only defensible mode
- For landline-only lists: predictive is fine (TCPA ATDS restrictions apply primarily to cell phones)

### State-Level Regulations

Some states impose calling restrictions beyond federal law:

- **California (CCPA/CPRA):** Additional consent requirements for data usage
- **Florida:** Strict registration and bonding requirements for telemarketers
- **Indiana/Ohio/etc.:** State-level Do Not Call restrictions with shorter cure periods

Dialing mode doesn't directly affect these, but the volume of calls placed (predictive >> progressive >> preview) determines how quickly a compliance failure scales into a catastrophe.

---

## Hybrid Approaches in VICIdial

VICIdial's dialing engine is flexible enough to implement hybrid strategies that don't fit neatly into the three standard categories.

### Ratio Dialing with ADAPT Modifiers

Set the Dial Method to RATIO but configure ADAPT settings as a safety net:

```
Dial Method: RATIO
Auto Dial Level: 3.5
Adaptive DL Level: 4.0 (maximum the algorithm can reach)
Drop Rate Group: CAMPAIGN_ONLY
Max Abandon Rate: 2.5%
```

This gives you fixed-ratio dialing at 3.5:1 with an ADAPT ceiling that prevents the ratio from exceeding 4.0. If abandon rates spike, the ADAPT logic can override the manual ratio downward. It's predictive dialing with a safety valve.

### Campaign-Level Mode Switching

In VICIdial, you can change the dial method mid-shift without restarting the campaign. A common pattern:

1. **Morning (high staffing):** Switch to RATIO 4.0 — maximum throughput with 30+ agents available
2. **Afternoon (staffing drops):** Switch to ADAPT_HARD_LIMIT — let the algorithm handle the reduced agent pool
3. **Evening (skeleton crew):** Switch to ADAPT_TAPERED — conservative dialing with 5-8 agents

This is done in Campaign Detail → Dial Method or via VICIdial's API:

```
/vicidial/non_agent_api.php?function=update_campaign&campaign_id=YOUR_CAMP&dial_method=ADAPT_TAPERED&auto_dial_level=1.0
```

### Blended Campaigns: Inbound + Outbound Modes

VICIdial supports blended campaigns where agents handle both inbound and outbound calls. The dialing mode for outbound interacts with inbound queue management:

- When an inbound call arrives for a [closer campaign](/settings/closer-campaigns/), the agent is pulled from the outbound pool
- The predictive/progressive algorithm recalculates with one fewer available agent
- When the inbound call ends, the agent returns to the outbound pool

For blended operations, ADAPT modes work better than fixed RATIO because the constantly changing agent pool requires adaptive dial-level calculations. RATIO mode doesn't account for agents being pulled into inbound — it keeps dialing at the fixed ratio, causing spikes in abandoned outbound calls when several agents simultaneously take inbound calls.

### Power Dialing: The Fourth Mode

[Power dialing](/glossary/power-dialing/) — sometimes called "ratio dialing at 1:1" — places one call per available agent but immediately places the next call when the first attempt fails (busy, no answer, disconnected). Unlike progressive, which has the algorithm controlling the timing, power dialing is a simple loop: dial, fail, dial, fail, dial, answer, connect.

In VICIdial, power dialing is implemented as:
- Dial Method: RATIO
- [Auto Dial Level](/settings/auto-dial-level/): 1.0
- [Power Dial Mode](/settings/power-dial-mode/): Y (if available in your version)

This produces roughly the same throughput as progressive but with slightly different timing behavior — calls are placed more rapidly after failures.

---

## Performance Comparison: Real-World Numbers

These benchmarks are aggregated from ViciStack deployments across multiple verticals. They represent average performance for well-configured campaigns on optimized infrastructure — not theoretical maximums or vendor marketing claims.

### Side-by-Side Comparison Table

| Metric | Predictive (25+ agents) | Progressive (any size) | Preview |
|--------|------------------------|----------------------|---------|
| **Talk time per hour** | 45-55 min | 30-40 min | 15-25 min |
| **Idle time per hour** | 5-15 min | 20-30 min | 35-45 min |
| **Contacts per hour** | 8-15 | 5-10 | 3-6 |
| **Dials per hour** | 50-80 | 25-45 | 8-20 |
| **Leads consumed per hour** | 40-65 | 20-35 | 8-15 |
| **Typical abandon rate** | 1-3% | 0% | 0% |
| **Conversion rate** | Baseline | +5-15% vs predictive | +30-100% vs predictive |
| **Revenue per contact** | Baseline | Similar | Higher (prepared calls) |
| **Revenue per agent hour** | Highest (volume) | Moderate | Varies (deal-size dependent) |
| **[Cost per lead](/glossary/cost-per-lead/)** | Lowest (at scale) | Moderate | Highest (but offset by quality) |
| **Compliance risk** | Moderate | Very low | Minimal |
| **Minimum recommended agents** | 15 | 1 | 1 |
| **Best for** | High volume B2C | Small teams, B2B, compliance | Complex sales, high-value |

### The Volume vs. Value Matrix

The right dialing mode depends on where your campaign falls on two axes: **contact volume needed** and **value per contact**.

```
                    HIGH VOLUME NEEDED
                          │
     Predictive           │           Predictive
     (Standard B2C)       │           + AMD
                          │           (Insurance, Solar)
                          │
   LOW VALUE ─────────────┼───────────── HIGH VALUE
                          │
     Progressive          │           Preview
     (B2B mid-market,     │           (Enterprise sales,
      compliance)         │            wealth management)
                          │
                    LOW VOLUME OK
```

Campaigns in the upper-right quadrant (high volume + high value) benefit most from predictive dialing with [AMD](/glossary/amd/) — because you need the throughput of predictive but the value per contact justifies the AMD investment and [DID management](/blog/vicidial-did-management/) effort to maximize answer rates.

---

## Real-World Campaign Configurations

### Configuration 1: High-Volume Medicare Lead Generation (Predictive)

50 agents, B2C, moderate-value leads ($50-$80 per qualified lead), cell phones with documented consent.

```
Campaign Settings:
  Dial Method: RATIO
  Auto Dial Level: 4.0
  Available Only Ratio Tally: Y
  Dial Timeout: 21 seconds
  AMD Type: CPD (Asterisk native)
  AMD Agent Route: PROG (route to agent if human, message if machine)
  Hopper Level: 500
  Drop Rate Group: CAMPAIGN_ONLY
  Drop Action: MESSAGE (play safe harbor message on abandon)
  Abandon Rate Target: 2.5%
```

Expected performance: 48 min talk time/hour, 12 contacts/hour, ~900 leads qualified per day across the floor.

### Configuration 2: B2B Software Demo Booking (Progressive)

12 agents, B2B, high-value leads ($500+ per booked demo), mix of cell and landline.

```
Campaign Settings:
  Dial Method: ADAPT_AVERAGE
  Adaptive DL Level: 1.8
  Auto Dial Level: 1.0
  Dial Timeout: 24 seconds (B2B phones ring longer)
  AMD Type: DISABLED (agents handle VMs manually — many B2B VMs are worth leaving a message)
  Hopper Level: 100
  Drop Rate Group: CAMPAIGN_ONLY
  Abandon Rate Target: 0.5%
  Scheduled Callbacks: Y
```

Expected performance: 35 min talk time/hour, 7 contacts/hour, ~85 demos booked per day.

### Configuration 3: Wealth Management Client Outreach (Preview)

5 agents, high-net-worth individuals, $2,000+ revenue per appointment set.

```
Campaign Settings:
  Dial Method: INBOUND_MAN
  Manual Dial: Y
  Script: Y (personalized talking points per client tier)
  Web Form: Y (CRM embedded showing full client portfolio)
  Scheduled Callbacks: Y
  Lead Order: UP_LAST_NAME_ASC
  Campaign Recording: ALLFORCE
```

Expected performance: 20 min talk time/hour, 4 contacts/hour, conversion rate 15-20%. Revenue per agent hour: $1,200+ (at $2,000/appointment × 15% conversion × 4 contacts).

### Configuration 4: Blended Insurance Campaign (Hybrid)

35 outbound agents + 10 inbound-capable agents, mixed cell/landline, transfers to closers.

```
Campaign Settings:
  Dial Method: ADAPT_HARD_LIMIT
  Adaptive DL Level: 3.5
  Auto Dial Level: 1.0 (algorithm ramps up from here)
  Campaign Allow Inbound: Y
  Closer Campaigns: INSURANCE_CLOSER
  Dial Timeout: 20 seconds
  AMD Type: CPD
  Hopper Level: 300
  Abandon Rate Target: 2.0%
```

Expected performance: 42 min talk time/hour (outbound), blended agents handle 3-5 inbound calls per hour during lulls. Total floor throughput: ~400 contacts/hour across all modes.

---

## Switching Modes: When and How

Campaign conditions change. Lists get older, agent counts fluctuate, compliance requirements shift. Knowing when to switch dialing modes is as important as choosing the right one initially.

### Signals It's Time to Switch

**Switch FROM predictive TO progressive when:**
- Agent count drops below 12 for the campaign
- Abandon rate consistently exceeds 2.5% (algorithm can't keep up)
- You're entering a compliance-sensitive period (audit, regulatory inquiry)
- Lead volume is running low and you need to preserve remaining leads

**Switch FROM progressive TO predictive when:**
- Agent count exceeds 15 for the campaign
- Progressive idle time exceeds 25 min/hour (agents waiting too much)
- You have a surplus of leads and need to maximize throughput
- The campaign's compliance posture is clean and well-documented

**Switch FROM either TO preview when:**
- You're entering a high-value lead segment that requires research
- Conversion rate drops significantly and you suspect agent preparation is the issue
- You're recycling already-called leads where context from previous calls matters
- Regulatory changes restrict automated dialing for your contact method

### How to Switch in VICIdial Without Disrupting Active Calls

VICIdial allows real-time dial method changes:

1. Go to Campaign Detail → Dial Method
2. Change the method
3. Click Update

Active calls are not affected. The new dial method takes effect on the next dial cycle (within 15 seconds). Agents don't need to log out or do anything different — the change is transparent to them.

For API-driven switching (useful for scheduled mode changes):

```bash
# Switch to predictive at 8 AM when full staff arrives
0 8 * * 1-5 curl -s "https://your-vicidial.com/vicidial/non_agent_api.php?source=api&user=apiuser&pass=apipass&function=update_campaign&campaign_id=INSURANCE&dial_method=RATIO&auto_dial_level=4.0"

# Switch to progressive at 5 PM when staff thins out
0 17 * * 1-5 curl -s "https://your-vicidial.com/vicidial/non_agent_api.php?source=api&user=apiuser&pass=apipass&function=update_campaign&campaign_id=INSURANCE&dial_method=ADAPT_TAPERED&auto_dial_level=1.0"
```

---

## The Economics of Each Mode

Dialing mode choice isn't just a technical or compliance decision — it's a financial one. Let's quantify the economic impact.

### Revenue Per Agent Hour by Mode

Using our standard model (50-agent floor, $65/qualified lead, 10% qualification rate):

| Mode | Contacts/Hour | Qualification Rate | Revenue/Agent/Hour |
|------|---------------|-------------------|-------------------|
| Predictive | 12 | 10% | $78.00 |
| Progressive | 7 | 12% | $54.60 |
| Preview | 4 | 18% | $46.80 |

**Predictive wins on revenue per agent hour** for this campaign type — and it's not close. The volume advantage overwhelms the conversion rate improvement of progressive and preview.

### But Revenue Per Agent Hour Isn't the Whole Story

Factor in the cost of compliance violations:

| Mode | Annual Compliance Risk Cost (estimated) |
|------|-----------------------------------------|
| Predictive | $15,000-$50,000 (potential TCPA/FCC fines, legal fees) |
| Progressive | $2,000-$5,000 (minimal — mostly procedural) |
| Preview | $500-$1,000 (nearly zero automated-dialing risk) |

And the cost of lead consumption:

| Mode | Leads Consumed Per Agent Per Day | Effective Lead Cost |
|------|----------------------------------|-------------------|
| Predictive | 350-500 | $0.02-$0.10/lead (assuming $50-100/1000 leads) |
| Progressive | 150-250 | $0.02-$0.10/lead |
| Preview | 50-100 | $0.02-$0.10/lead |

When leads cost $0.05 each, the difference is modest. When leads cost $1.00+ each (B2B data), predictive dialing burns through $350-$500 of leads per agent per day vs. preview's $50-$100. For lead-constrained campaigns, the per-lead economics can flip the ROI math entirely.

### Break-Even Analysis: When Does Predictive Beat Progressive?

Predictive dialing produces roughly 70% more contacts per hour than progressive. But it also:
- Consumes leads ~100% faster
- Carries compliance risk that progressive doesn't
- Requires 15+ agents to function properly

**Predictive breaks even with progressive when:**
- You have 15+ agents per campaign
- Lead cost is under $0.25 per lead
- Compliance risk is managed (proper consent documentation, FCC-compliant [safe harbor message](/settings/safe-harbor-message/), [drop rate monitoring](/settings/drop-rate-group/))
- Revenue per contact is moderate ($25-$200)

**Progressive beats predictive when:**
- Fewer than 15 agents per campaign (predictive algorithm is unreliable)
- Lead cost exceeds $0.50 per lead (predictive burns leads too fast)
- Compliance exposure is uncertain (cell phones without clear consent)
- Revenue per contact exceeds $500 (quality matters more than quantity)

---

## Where ViciStack Fits In

Every ViciStack deployment starts with a campaign analysis. We look at your agent count, your leads, your vertical, your compliance requirements, and your revenue model — and we configure the dial method that maximizes your [ROI](/blog/call-center-roi-formula/), not just your talk time.

We've seen operations running predictive at 5 agents (the algorithm is basically random at that scale) and operations running preview at 50 agents (leaving 60% of their agent time on the table). Both are leaving money on the table. The right dial method for your campaign probably isn't the one you're using right now.

ViciStack configures every campaign with the optimal dial method, the correct ADAPT settings, the right [hopper level](/settings/hopper-level/), and the AMD configuration that matches your specific workflow. When your agent count changes or your campaign shifts, we adjust.

[Get a free dialing mode assessment →](/free-audit/)

---

*This guide is maintained by the ViciStack team and updated as VICIdial's dialing algorithms and compliance requirements evolve. Last updated: March 2026.*

---

## Frequently Asked Questions

### What's the minimum number of agents for predictive dialing to work effectively?

15 is the practical minimum. Below 15, the predictive algorithm doesn't have enough statistical data (simultaneous call outcomes, talk time variance) to make accurate predictions. The dial ratio becomes erratic — over-dialing when two agents happen to finish calls simultaneously, then under-dialing when the algorithm over-corrects. At 10 agents, you'll see frequent spikes in abandon rate alternating with periods of agent idle time. At 5 agents, the algorithm is essentially guessing. Use progressive (ADAPT modes) for campaigns under 15 agents.

### Can I use predictive dialing for cell phone lists?

Legally, it depends on your consent documentation. The TCPA restricts the use of automatic telephone dialing systems (ATDS) to call cell phones without prior express consent. Predictive dialing clearly qualifies as an ATDS. If your leads provided prior express consent to be called (and you can document it), predictive is legally defensible. If consent is ambiguous, progressive is safer. If consent is undocumented, preview is the only defensible option. Consult your compliance team or legal counsel — the TCPA landscape changes frequently and the cost of getting it wrong is severe ($500-$1,500 per violation).

### How does AMD interact with dialing mode?

[AMD (Answering Machine Detection)](/glossary/amd/) works with all dialing modes but has the biggest impact on predictive campaigns. In predictive mode, AMD filters out answering machines before they reach agents, increasing productive talk time by 15-25%. In progressive mode, AMD has less impact because agents already handle fewer calls per hour — filtering machines saves time but the agents weren't at maximum throughput anyway. In preview mode, AMD is typically disabled because agents expect to handle every call themselves, including leaving voicemails when appropriate.

### What does VICIdial's "ADAPT_TAPERED" actually do differently from "ADAPT_AVERAGE"?

Both modes use VICIdial's adaptive algorithm (`AST_VDadapt.pl`) to adjust the dial ratio dynamically. ADAPT_AVERAGE targets the campaign's average performance metrics and adjusts smoothly. ADAPT_TAPERED adds a tapering function that reduces the dial ratio more aggressively as the current drop rate approaches the campaign's [abandon rate target](/settings/abandon-rate-target/). In practice, ADAPT_TAPERED produces about 5-10% lower throughput than ADAPT_AVERAGE but virtually eliminates drop rate spikes. Use TAPERED for compliance-sensitive campaigns and AVERAGE for campaigns where you want more aggressive dialing while still having adaptive control.

### How do I know if my current dial method is costing me money?

Two metrics tell you: **agent idle time** and **abandon rate**. If agents are idle more than 30% of the hour on a predictive campaign, your dial level is too low or you need more leads in the [hopper](/glossary/hopper/). If your abandon rate is above 2% consistently, you're probably losing leads you could have converted. If agents are idle more than 40% on progressive, you should consider predictive (if you have 15+ agents). Pull these metrics from VICIdial's real-time reports and agent performance reports — they're available out of the box.

### Can I run different dialing modes on different campaigns simultaneously?

Absolutely. Each VICIdial [campaign](/glossary/campaign/) has its own independent dial method, dial level, and ADAPT settings. You can run predictive on your high-volume B2C campaign, progressive on your B2B campaign, and preview on your VIP client outreach — all simultaneously with agents logged into different campaigns. You can even move agents between campaigns mid-shift, and they'll automatically operate under whatever dialing mode that campaign uses.

### How does list quality affect the choice between predictive and progressive?

Dramatically. Predictive dialing magnifies list quality — both good and bad. A high-quality list (fresh, verified numbers, high contact rate) produces excellent results on predictive because the algorithm maintains a high dial ratio with low abandonment. A low-quality list (old data, many disconnects, low answer rate) forces the predictive algorithm to dial more aggressively to compensate, which burns through leads faster and increases abandon rates. For low-quality or recycled lists, progressive dialing is often more economical because it doesn't burn leads as quickly and gives agents time to handle each contact carefully.

### What's the impact of switching dialing modes on agent morale?

Real and often underestimated. Agents who are accustomed to predictive dialing (constant call flow, minimal idle time) often resist switching to progressive because the increased idle time between calls feels unproductive and boring. Conversely, agents accustomed to preview or progressive mode can feel overwhelmed by predictive's relentless pace — calls connecting before they've finished their notes on the previous call. When switching modes, communicate the reason to your team, adjust expectations for talk time metrics, and give agents 2-3 days to adapt before evaluating performance under the new mode.

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/predictive-vs-progressive-dialing).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
