# VICIdial Queue & Inbound Group Configuration Guide

**Last updated: March 2026 | Reading time: ~22 minutes**

A potential customer calls your sales line. They navigated your [IVR](/glossary/ivr/), pressed 2 for Sales, and landed in a [queue](/glossary/queue/). They hear silence for 8 seconds, then a ring that nobody answers, then silence again. After 45 seconds of this they hang up. Your real-time report shows a new abandoned call. The lead is gone.

This happens because VICIdial's inbound system ships with sensible defaults but zero awareness of your actual operation. The default [inbound group](/glossary/inbound-group/) has no hold music, no queue position announcement, no overflow route, and no after-hours handling. If you stood up your inbound DIDs and pointed them at an inbound group without touching the configuration, you are running a skeleton setup that actively loses calls.

VICIdial's inbound engine is genuinely powerful. It supports [skill-based routing](/glossary/skill-based-routing/), weighted ring strategies, queue priority by [DID](/glossary/did/) or caller ID, [blended agent](/glossary/blended-agent/) configurations that mix inbound and outbound on the same campaign, overflow cascades, after-hours voicemail, and real-time queue monitoring. But none of it works unless you configure it, and VICIdial's admin interface gives you approximately 80 settings per inbound group with minimal documentation about what most of them do.

This guide covers every aspect of VICIdial inbound group configuration: creation, queue management, ring strategies, hold music, overflow handling, after-hours routing, blended agent setup, priority routing, and the operational patterns that separate a professional inbound operation from a ringing phone that nobody answers.

**ViciStack deploys inbound configurations tuned to your call volume, agent count, and routing requirements from day one.** [Start with a properly configured inbound system.](/pricing/)

---

## How VICIdial Inbound Routing Works

Before configuring anything, you need to understand the path a call takes from the PSTN to an agent's headset. VICIdial's inbound architecture has four layers: the DID, the inbound group, the queue, and the agent selection.

### The Call Flow

1. **DID receives the call.** An inbound call hits your SIP trunk and reaches VICIdial's Asterisk server. The dialplan matches the dialed number to a [DID](/glossary/did/) entry in VICIdial's `vicidial_inbound_dids` table.

2. **DID routes to an inbound group (or IVR).** The DID configuration specifies where to send the call. For direct-to-queue routing, this is an inbound group. For menu-driven routing, this is an [IVR](/glossary/ivr/) that eventually routes to one or more inbound groups. See our [IVR setup guide](/blog/vicidial-ivr-setup/) for the menu layer.

3. **Inbound group places the call in queue.** The call enters the inbound group's queue. Hold music plays. Queue position announcements fire at configured intervals. The queue engine evaluates which agents are eligible to take this call.

4. **Agent is selected and connected.** Based on the ring strategy (longest wait time, ring all, random, etc.), VICIdial selects an agent, rings their phone, and connects the call. The agent's screen pops with the caller's information.

### Key Database Tables

| Table | Purpose |
|-------|---------|
| `vicidial_inbound_dids` | Maps phone numbers to routing destinations (inbound groups, IVRs, phones) |
| `vicidial_inbound_groups` | Defines inbound group configuration --- ring strategy, hold music, overflow, etc. |
| `vicidial_closer_log` | Logs every inbound call with timestamps, queue time, handle time, agent, disposition |
| `vicidial_auto_calls` | Active calls currently in queue or being handled |
| `vicidial_live_inbound_agents` | Agents currently logged into inbound groups with their status and weight |

Understanding these tables helps when you need to debug routing issues, build custom reports, or troubleshoot calls that are not reaching agents.

---

## Creating an Inbound Group

Navigate to **Admin > Inbound > Add a New In-Group**. This is where you define the fundamental identity and behavior of your queue.

### Basic Setup Fields

| Field | Example Value | Purpose |
|-------|---------------|---------|
| Group ID | SALES_QUEUE | Unique identifier, no spaces, max 20 characters |
| Group Name | Sales Inbound Queue | Human-readable label for reports and agent interface |
| Group Color | #0066CC | Color coding in real-time monitoring screens |
| Active | Y | Whether agents can log into this group |
| Web Form | https://crm.example.com/lead?phone=--A--phone_number--B-- | URL that pops on the agent screen when the call connects |
| Voicemail | vm-salesbox | Voicemail box for after-hours or overflow |
| Next Agent Call | longest_wait_time | Ring strategy (covered in detail below) |
| Fronter Display | Y | Whether agents see the inbound group name on their screen |

**The Group ID matters.** You will reference it in DID routing, IVR configuration, [closer campaigns](/settings/closer-campaigns/) settings, and SQL queries. Use a naming convention that scales: `SALES_QUEUE`, `SUPPORT_T1`, `SUPPORT_T2`, `BILLING_MAIN`. Do not use generic names like `INBOUND1` that tell you nothing when you are troubleshooting at 3 AM.

### Critical Initial Settings

After creating the inbound group, click into its detail page. Here are the settings that must be configured before you take a single call:

**Queue handling:**

| Setting | Recommended Value | Why |
|---------|------------------|-----|
| Hold Time Option | NONE initially, then customize | What happens to callers in queue (announcements, overflow) |
| Hold Time Option Seconds | 60-120 | How long before the hold time option triggers |
| Hold Recall Transfer | CLOSERSELECT | Where to send calls that hit the overflow threshold |
| No Agent No Queue | [N](/settings/no-agent-no-queue/) | If Y, calls are rejected when no agents are logged in --- use voicemail instead |
| After Hours Action | MESSAGE | Play a message and/or route to voicemail after hours |
| After Hours Message Filename | after-hours-sales | Audio file to play outside business hours |
| After Hours Voicemail | vm-salesbox | Voicemail box for after-hours calls |
| Call Time ID | 9am-9pm | Defines business hours for this inbound group |

**Agent connection:**

| Setting | Recommended Value | Why |
|---------|------------------|-----|
| Next Agent Call | longest_wait_time | Routes to the agent who has been waiting longest (most common) |
| Agent Alert Delay | 0 | Seconds to wait before alerting the agent (0 for immediate) |
| Agent Alert Hold | On | Whether the agent hears a brief tone before the call connects |
| Drop Call Seconds | 0 | For inbound, set to 0 --- you do not want to drop inbound calls |
| Max Queue Ingroup Calls | 0 (unlimited) | Maximum calls allowed in queue, 0 for no limit |

---

## Ring Strategies Explained

The `Next Agent Call` setting determines how VICIdial selects which agent receives the next inbound call. This single setting has more impact on your inbound operation than almost any other configuration.

### longest_wait_time

The agent who has been idle the longest gets the next call. This is the default and the most commonly used strategy for general inbound operations.

**How it works:** VICIdial tracks the timestamp of each agent's last call end time. When a new inbound call needs routing, it selects the agent with the oldest `last_call_time`.

**Best for:** General sales queues, support lines, any scenario where all agents are equally qualified and you want even call distribution.

**Watch out for:** In blended environments, agents who just finished an outbound call have a recent `last_call_time`, so they will receive fewer inbound calls than agents who have been sitting idle. This is usually correct behavior --- but if your outbound agents complain about getting fewer inbound calls, this is why.

### ring_all

Every available agent's phone rings simultaneously. The first agent to pick up gets the call.

**How it works:** VICIdial sends a SIP INVITE to all available agents in the inbound group at the same time. The first agent to answer is connected, and all other agent phones stop ringing.

**Best for:** Small teams (under 10 agents), scenarios where speed-to-answer is critical, operations where you want agents competing for calls.

**Watch out for:** Ring-all does not scale well. With 50 agents, you are generating 50 simultaneous SIP INVITEs per inbound call. This puts load on your Asterisk server and SIP infrastructure. Also, aggressive agents will always answer first, leading to uneven call distribution. For teams over 10-15 agents, use `longest_wait_time` instead.

### random

A random available agent is selected.

**How it works:** VICIdial picks a random agent from the pool of available agents in the inbound group.

**Best for:** Almost nothing, honestly. Random distribution means some agents get 3 calls in a row while others sit idle for 20 minutes. The only valid use case is when you explicitly want unpredictable distribution for compliance or fairness auditing purposes.

### fewest_calls

The agent with the fewest calls in the current session gets the next call.

**How it works:** VICIdial counts each agent's calls since login and routes to the agent with the lowest count.

**Best for:** Operations where you need exactly equal call counts per agent per shift (some union environments, regulated operations).

**Watch out for:** A slow agent who takes 15-minute calls will get the same number of calls as a fast agent who handles calls in 3 minutes. This means your efficient agents are penalized with more idle time.

### fewest_calls_campaign

Same as `fewest_calls`, but counts across the campaign rather than just the inbound group. Relevant for [blended campaigns](/glossary/blended-agent/) where agents handle both inbound and outbound.

### oldest_call_start

Routes to the agent whose last call started earliest (not ended earliest). Subtly different from `longest_wait_time` --- this factors in agents who are on long calls.

### oldest_call_finish

Routes to the agent whose last call finished earliest. This is similar to `longest_wait_time` but explicitly uses the call finish timestamp rather than the last state change timestamp.

### Choosing the Right Strategy

| Operation Type | Recommended Strategy | Reason |
|---------------|---------------------|--------|
| Sales inbound (general) | longest_wait_time | Even distribution, rewards idle agents |
| Small team (<10 agents) | ring_all | Fastest answer time |
| Support tier 1 | longest_wait_time | Even workload distribution |
| Support tier 2 (specialists) | longest_wait_time or fewest_calls | Even distribution among limited specialists |
| Blended inbound + outbound | longest_wait_time | Correctly prioritizes idle agents over agents on outbound |
| Compliance-sensitive | fewest_calls | Provably equal distribution |

---

## Hold Music and Queue Announcements

Nothing tells a caller "we don't have our act together" faster than dead silence in a queue. VICIdial's hold music and announcement system is flexible, but you have to configure it.

### Setting Up Hold Music

VICIdial uses Music on Hold (MOH) classes from Asterisk. The default class is `default`, which plays whatever files are in `/var/lib/asterisk/mohmp3/`.

**To configure hold music for an inbound group:**

1. Upload your hold music files to `/var/lib/asterisk/mohmp3/` (or a custom directory for a custom MOH class). Files should be MP3 or WAV format. For best quality with minimal CPU usage, use 8kHz 16-bit mono WAV files.

2. If using a custom MOH class, define it in `/etc/asterisk/musiconhold.conf`:

```ini
[sales_hold]
mode=files
directory=/var/lib/asterisk/moh/sales
sort=random
```

3. Reload Asterisk MOH: `asterisk -rx "moh reload"`

4. In the inbound group settings, set **On Hold Music** to your MOH class name (e.g., `sales_hold`).

For more on Asterisk audio configuration, see our [Asterisk configuration guide](/blog/vicidial-asterisk-configuration/).

**Hold music best practices:**

- **Keep it short.** 60-90 seconds of music on a loop is sufficient. Callers who are on hold for 5 minutes will hear it repeat, but that is acceptable.
- **Avoid music with vocals.** Instrumental is clearer over phone audio codecs. Vocals compete with announcements and sound garbled on low-bandwidth codecs.
- **Match your brand.** A law firm should not have upbeat pop music. A youth-oriented brand should not have classical.
- **License your music.** Using copyrighted music on hold is technically a public performance. Use royalty-free hold music or license it properly.

### Queue Position Announcements

Callers want to know two things: where they are in the queue and how long they will wait. VICIdial can announce both.

**Configuration in the inbound group settings:**

| Setting | Recommended Value | Purpose |
|---------|------------------|---------|
| Hold Time Option | ANNOUNCE_POSITION | Announces the caller's queue position |
| Hold Time Option Seconds | 30-60 | Interval between announcements |
| Hold Time Option Exten | announce_position | Asterisk extension that plays the announcement |
| Hold Time Option Voicemail | (leave blank for position announcements) | Voicemail box for alternate hold action |
| Onhold CID | INBOUND | Caller ID displayed to agents during hold |

The `ANNOUNCE_POSITION` option tells callers "You are caller number X in the queue." VICIdial dynamically counts the calls ahead of the current caller and plays the number.

**Other Hold Time Options:**

| Option | Behavior |
|--------|----------|
| NONE | No announcements --- caller hears hold music only |
| ANNOUNCE_POSITION | "You are caller number X" |
| ANNOUNCE_WAIT_TIME | "Your estimated wait time is X minutes" |
| ANNOUNCE_POSITION_AND_WAIT | Combines position and estimated wait time |
| CALLERID_CALLBACK | Offers the caller a callback option instead of waiting |

**Estimated wait time** is calculated from the average handle time of the inbound group over the last N calls divided by available agents. It is not perfectly accurate during volume spikes, but it is better than telling callers nothing.

**The CALLERID_CALLBACK option** is particularly powerful for high-volume queues. Instead of waiting on hold, the caller can press a key to receive a callback when an agent becomes available. This reduces abandonment rates dramatically. The system records the caller's number and injects it into the queue as a callback, maintaining the caller's original queue position. When an agent becomes available, VICIdial dials the caller back. Configure this in conjunction with VICIdial's [callback automation system](/blog/vicidial-callback-automation/).

---

## Overflow Handling

What happens when your queue is full, wait times are too long, or all agents are busy? Without overflow configuration, the answer is: the caller sits in queue forever or hangs up. Both outcomes are bad.

### Configuring Overflow Routes

VICIdial's overflow mechanism is controlled by the `Hold Recall Transfer` setting in the inbound group. When a caller has been in queue for longer than `Hold Time Option Seconds`, the system takes action based on the hold time option configuration.

**Overflow to a different inbound group:**

Set `Hold Recall Transfer` to the Group ID of your overflow queue. This is useful for tiered routing:

- Primary queue: `SALES_QUEUE` (specialized sales agents)
- Overflow after 120 seconds: `SALES_OVERFLOW` (general agents who can handle sales)
- Final overflow after 240 seconds: `VOICEMAIL_SALES` (voicemail group)

**Overflow to voicemail:**

Set `Hold Recall Transfer` to `VOICEMAIL` and configure the voicemail box in the `Voicemail` field. After the configured wait time, the caller hears "All agents are currently busy. Please leave a message after the tone."

**Overflow to an external number:**

If you need to overflow to an external call center, BPO, or answering service, set up a DID route that forwards to an external number and use that as your overflow destination.

### Multi-Stage Overflow

For production inbound operations, a single overflow route is not sufficient. Here is a pattern that works:

| Stage | Trigger | Action |
|-------|---------|--------|
| Queue entry | Caller enters queue | Play welcome message, start hold music |
| 30 seconds | First announcement | "Your call is important. You are caller number X." |
| 60 seconds | Second announcement | "Estimated wait time is X minutes. Press 1 for a callback." |
| 120 seconds | Overflow stage 1 | Route to overflow inbound group (wider agent pool) |
| 180 seconds | Overflow stage 2 | "We apologize for the wait. Press 1 for a callback, press 2 to leave a voicemail." |
| 240 seconds | Final overflow | Route to voicemail group |

This requires chaining multiple inbound groups with different `Hold Time Option Seconds` values. Each group in the chain has its own announcements and its own `Hold Recall Transfer` pointing to the next stage.

### The No Agent No Queue Setting

The [No Agent No Queue](/settings/no-agent-no-queue/) setting deserves special attention. When set to `Y`, if no agents are logged into the inbound group when a call arrives, the call is immediately rejected --- the caller hears fast busy or gets disconnected.

**This is almost never what you want.** Set it to `N` and configure an after-hours route or voicemail instead. The only valid use case for `No Agent No Queue = Y` is when you are using VICIdial purely as a call router and the inbound group is a passthrough to a live agent pool that must always be staffed.

Better alternatives to `No Agent No Queue = Y`:

- **Route to voicemail:** Capture the caller's information even when nobody is available
- **Route to IVR:** Let the caller navigate to self-service options or leave a message
- **Route to external number:** Forward to an answering service or overflow center
- **Play a message and disconnect:** "Our office is currently closed. Please call back during business hours: Monday through Friday, 9 AM to 6 PM Eastern."

---

## After-Hours Routing

Every inbound group needs an after-hours plan. Calls do not stop coming in at 5 PM, and a caller who gets dead silence or fast busy at 5:01 PM is a lost opportunity.

### Configuring Business Hours

1. Navigate to **Admin > Call Times** and create (or select) a call time definition for your inbound group's business hours.

Example:

| Setting | Value |
|---------|-------|
| Call Time ID | 9am-6pm-EST |
| Call Time Name | Business Hours Eastern |
| CT Default Start | 900 |
| CT Default Stop | 1800 |
| Sunday Start/Stop | (leave blank for closed) |
| Saturday Start/Stop | (leave blank for closed, or 1000-1400 for half days) |

2. In the inbound group settings, set **Call Time ID** to your business hours definition.

3. Configure the after-hours behavior:

| Setting | Value | Purpose |
|---------|-------|---------|
| After Hours Action | MESSAGE | Play an audio message |
| After Hours Message Filename | after-hours-sales | The audio file to play |
| After Hours Exten | 8300 | Asterisk extension to route to after the message |
| After Hours Voicemail | vm-sales-afterhours | Voicemail box for after-hours messages |

### After-Hours Voicemail Groups

For operations that need after-hours voicemail with notification, set up a voicemail group:

1. In **Admin > Phones**, create a voicemail box or use an existing one
2. Configure email notification on the voicemail box so your team gets notified of after-hours messages
3. Set the inbound group's `After Hours Voicemail` to this box

**The often-missed step:** Configure someone to actually check and return voicemails. VICIdial will store the voicemail, but without a process to review and call back, voicemails are just a slightly polite way to lose leads. Set up a morning task for your first shift --- pull all after-hours voicemails and schedule [callbacks](/blog/vicidial-callback-automation/) for each one.

### Holiday Routing

VICIdial supports holiday-specific routing through the Call Time system. In **Admin > Call Times**, you can add holiday exceptions to any call time:

1. Edit your call time definition
2. In the "CT Holiday" section, add dates with custom start/stop times (or 0000/0000 for closed all day)
3. The after-hours routing automatically applies on holidays

Plan your holiday schedule at the beginning of each year. Nothing is more embarrassing than having your phones ring into a dead queue on Christmas Day because nobody updated the holiday list.

---

## Skill-Based Routing

Not all agents are equal. Some speak Spanish. Some are licensed in certain states. Some are product specialists. [Skill-based routing](/glossary/skill-based-routing/) ensures callers reach agents who can actually help them.

### How VICIdial Implements Skills

VICIdial does not have a dedicated "skills" database. Instead, skill-based routing is implemented through multiple inbound groups, agent group assignments, and [closer campaigns](/settings/closer-campaigns/) selection.

The model works like this:

1. **Create inbound groups per skill.** `SALES_ENGLISH`, `SALES_SPANISH`, `SUPPORT_BILLING`, `SUPPORT_TECHNICAL`.
2. **Assign agents to inbound groups.** Each agent selects (or is assigned via the campaign) which inbound groups they handle.
3. **Route calls to the correct inbound group.** Your IVR routes callers to the appropriate inbound group based on their selection, DID, or caller ID.

This is not as elegant as a tag-based skill system, but it is effective and scales to complex routing scenarios.

### Configuration Steps

**Step 1: Create skill-specific inbound groups.**

Navigate to **Admin > Inbound > Add a New In-Group** and create a group for each skill:

| Group ID | Group Name | Ring Strategy |
|----------|------------|---------------|
| SALES_EN | Sales - English | longest_wait_time |
| SALES_ES | Sales - Spanish | longest_wait_time |
| SUP_BILLING | Support - Billing | longest_wait_time |
| SUP_TECH | Support - Technical | fewest_calls |

**Step 2: Configure campaigns to allow inbound group selection.**

Navigate to **Admin > Campaigns > [Campaign] > Detail** and set:

- [Campaign Allow Inbound](/settings/campaign-allow-inbound/) = `Y`
- [Closer Campaigns](/settings/closer-campaigns/) = list the inbound groups this campaign's agents can receive calls from

The `Closer Campaigns` setting is a space-separated list of inbound group IDs:

```
SALES_EN SALES_ES
```

Only agents on campaigns that include an inbound group in their `Closer Campaigns` list will receive calls from that group. This is the core mechanism for skill-based routing.

**Step 3: Configure agent-level inbound group selection.**

There are two approaches:

**Manager-controlled:** Set `Closer Default Blended` to `1` in the campaign settings. This forces all agents on the campaign to take calls from all inbound groups listed in `Closer Campaigns`. Agents cannot opt out.

**Agent-controlled:** Set `Closer Default Blended` to `0` and allow agents to select their inbound groups when they log in. The agent interface presents a list of available inbound groups, and agents check the ones they want to handle. This works for operations where agents self-report their skills, but it means agents can deselect busy queues to avoid work.

**Recommended approach:** Manager-controlled for core skills, with supervisor overrides for temporary skill adjustments (e.g., pulling a Spanish-speaking agent off the Spanish queue during low-volume hours to help with English overflow).

### Skill Priority and Weighting

VICIdial supports agent rank/weight within inbound groups. Navigate to **Admin > Users > [Agent] > Inbound Group Rank**:

| Inbound Group | Rank | Grade | Calls Today |
|---------------|------|-------|-------------|
| SALES_EN | 5 | A | 23 |
| SALES_ES | 3 | B | 8 |
| SUP_BILLING | 0 | - | 0 |

- **Rank** (0-9): Higher rank means the agent is preferred for this inbound group. A rank-9 agent will receive calls before a rank-1 agent in the same group (assuming `Next Agent Call` supports ranking).
- **Grade**: Optional categorization for reporting.

To use ranking with ring strategies, set `Inbound Group Rank` to `Y` in the inbound group settings. When enabled, VICIdial considers rank as a primary sort before applying the ring strategy. All rank-9 agents are evaluated first; if all are busy, rank-8 agents are evaluated, and so on.

This enables tiered skill routing without separate inbound groups:

- Your best closers are rank 9 in `SALES_EN` --- they get calls first
- Newer agents are rank 3 --- they get calls only when all senior agents are busy
- Overflow agents from other departments are rank 1 --- last resort only

---

## Blended Agent Setup

A [blended agent](/glossary/blended-agent/) handles both inbound and outbound calls on the same campaign. When no inbound calls are queued, the agent makes outbound calls. When an inbound call arrives, the agent is pulled off outbound to handle it.

This is the most efficient way to staff a call center with mixed inbound and outbound volume. Without blending, you either overstaff inbound (agents sit idle between calls) or understaff inbound (callers wait in long queues while outbound agents dial away).

### Configuring a Blended Campaign

**Step 1: Enable inbound on the outbound campaign.**

Navigate to **Admin > Campaigns > [Campaign] > Detail**:

| Setting | Value | Purpose |
|---------|-------|---------|
| [Campaign Allow Inbound](/settings/campaign-allow-inbound/) | Y | Allows agents on this campaign to receive inbound calls |
| [Closer Campaigns](/settings/closer-campaigns/) | SALES_EN SALES_ES | Space-separated list of inbound groups this campaign can receive from |
| Closer Default Blended | 1 | Automatically blend all agents (1) or let agents choose (0) |
| Dial Method | RATIO or ADAPT_HARD_LIMIT | Predictive dialing method for outbound |

**Step 2: Configure inbound priority.**

When an agent is available and both an outbound lead and an inbound call are waiting, which takes priority? This is controlled by [Inbound Queue Priority](/settings/inbound-queue-priority/):

| Setting | Value | Behavior |
|---------|-------|----------|
| Inbound Queue Priority | 99 | Inbound always takes priority over outbound (recommended) |
| Inbound Queue Priority | 50 | Inbound and outbound compete based on timing |
| Inbound Queue Priority | 0 | Outbound takes priority (rarely appropriate) |

**Set this to 99 for most operations.** A caller waiting in queue has already invested time and attention. An outbound lead does not know you are about to call them. Prioritizing inbound is almost always correct.

**Step 3: Configure outbound throttling during inbound peaks.**

VICIdial can automatically reduce outbound dial level when inbound volume spikes. This prevents the scenario where your dialer places 50 outbound calls, all agents get pulled to inbound, and the outbound calls go to voicemail or get dropped.

In **Admin > Campaigns > [Campaign] > Detail**:

| Setting | Value | Purpose |
|---------|-------|---------|
| Auto Dial Level | 3.0 (example) | Base outbound dial level |
| Available Only Ratio Tally | Y | Only count available agents when calculating dial ratio |
| Drop Rate Group | INBOUND_OVERFLOW | Inbound group to route dropped outbound calls to |

The `Available Only Ratio Tally` setting is critical for blended campaigns. Without it, VICIdial counts all logged-in agents when calculating how many outbound calls to place --- including agents currently on inbound calls. This leads to over-dialing and dropped calls. With this set to `Y`, only truly available agents are counted.

### How Blending Actually Works in Practice

Here is the real-time behavior of a blended campaign:

1. Agent finishes a call and enters [wrap-up](/glossary/wrap-time/)
2. Wrap-up ends, agent becomes available
3. VICIdial checks: are there inbound calls waiting in any of the campaign's `Closer Campaigns` groups?
4. **If yes:** Route the highest-priority inbound call to the agent
5. **If no:** The outbound dialer includes this agent in its next dial calculation

The transition is seamless from the agent's perspective. Their screen shows whether the incoming call is inbound or outbound, the inbound group name (if inbound), and the caller information.

**Common blended issues and fixes:**

| Issue | Cause | Fix |
|-------|-------|-----|
| Agents never get outbound calls | Inbound volume is too high | Add dedicated inbound agents, or increase inbound agent pool |
| Outbound calls get dropped when inbound spikes | Available Only Ratio Tally is N | Set to Y |
| Agents complain about constant switching | Blending inherently requires context switching | Train agents on both workflows, or separate agents by shift rather than blending |
| Queue times spike during outbound dial bursts | Agents locked into outbound calls cannot take inbound | Reduce dial level, or set Inbound Queue Priority to 99 |

> **Blended campaigns are the highest-ROI configuration in VICIdial --- and the hardest to get right.**
> ViciStack configures and tunes blended campaigns for your specific inbound/outbound ratio. [Get a free audit of your current setup.](/free-audit/)

---

## Priority Routing

Not all inbound calls are equal. A call from a returning customer on your premium support line should not wait behind 15 calls from a general inquiry DID. VICIdial supports priority routing through several mechanisms.

### Priority by DID

Different DIDs can route to the same inbound group with different priority levels. This lets you maintain a single agent pool while giving priority to certain phone numbers.

**Configuration:**

1. Navigate to **Admin > Inbound > DIDs > [DID Number]**
2. Set the DID to route to your inbound group
3. Set the **DID Inbound Queue Priority** field

```
DID: 800-555-1000 (General Sales)     → SALES_EN, Priority 0
DID: 800-555-2000 (Premium Sales)     → SALES_EN, Priority 50
DID: 800-555-3000 (VIP Line)          → SALES_EN, Priority 99
```

All three DIDs route to the same `SALES_EN` inbound group and the same pool of agents. But a caller on the VIP line (priority 99) jumps ahead of callers on the general line (priority 0) in the queue.

This is configured in the `vicidial_inbound_groups` table via the `queue_priority` column, or through the inbound group admin interface.

### Priority by Caller ID

You can route calls based on the caller's phone number. This is useful for:

- Giving priority to known customers whose numbers are in your database
- Routing calls from specific area codes to region-specific agents
- Blocking known spam callers

**CID-based routing** is configured in the DID settings:

1. Navigate to **Admin > Inbound > DIDs > [DID Number] > CID Group**
2. Create a CID group with rules:

| CID Pattern | Action | Destination | Priority |
|-------------|--------|-------------|----------|
| 212XXXXXXX | ROUTE | SALES_NY | 10 |
| 310XXXXXXX | ROUTE | SALES_LA | 10 |
| 8005551234 | ROUTE | VIP_QUEUE | 99 |
| DEFAULT | ROUTE | SALES_EN | 0 |

This sends New York callers to a New York sales team, LA callers to an LA team, a specific known VIP number to the VIP queue, and everyone else to the general queue.

### Priority by IVR Selection

The most common priority routing pattern uses [IVR](/glossary/ivr/) selections. Your IVR menu gives callers options, and each option routes to a different inbound group with different priority:

```
"Press 1 for Sales" → SALES_QUEUE (priority 10)
"Press 2 for Support" → SUPPORT_QUEUE (priority 10)
"Press 3 for Billing" → BILLING_QUEUE (priority 10)
"Press 0 for Operator" → OPERATOR_QUEUE (priority 50)
```

Or, for a single queue with different priorities:

```
"Press 1 if you are a current customer" → SUPPORT_QUEUE (priority 50)
"Press 2 for all other inquiries" → SUPPORT_QUEUE (priority 0)
```

Current customers get priority in the same queue. This is configured in the IVR call menu settings, where each menu option specifies both the destination inbound group and the queue priority. See our [IVR setup guide](/blog/vicidial-ivr-setup/) for detailed IVR configuration.

### Priority Stacking

Priorities from different sources stack. If a call comes in on a priority-50 DID and the caller navigates an IVR option that adds priority 20, the final queue priority is 70. This allows for sophisticated priority schemes:

- Base priority from DID (which product line the caller dialed)
- Added priority from IVR selection (urgency of their need)
- Added priority from CID matching (known customer vs. unknown)

A known customer calling your premium line who selects "urgent" in the IVR could have a combined priority of 99 + 50 + 50, putting them well ahead of any other caller in the queue.

---

## Real-Time Queue Monitoring

You cannot manage a queue you cannot see. VICIdial provides several real-time monitoring tools for inbound operations.

### The Real-Time Report

Navigate to **Reports > Real-Time Report**. This is your inbound command center:

| Column | What It Shows |
|--------|---------------|
| In-Group | Inbound group name |
| Calls Waiting | Number of calls currently in queue |
| Agents Avail | Number of agents available to take calls |
| Agents on Calls | Agents currently on inbound calls |
| Avg Wait | Average wait time for calls currently in queue |
| Longest Wait | The caller who has been waiting the longest |
| Calls Today | Total inbound calls received today |
| Answered | Calls answered by agents today |
| Abandoned | Calls that hung up before reaching an agent |
| SLA % | Percentage of calls answered within the SLA threshold |

**Key metrics to watch:**

- **Calls Waiting > Agents Available:** You are understaffed. Either pull agents from outbound or activate overflow routing.
- **Longest Wait > 120 seconds:** Your queue is backing up. Take immediate action.
- **Abandoned % > 5%:** Callers are giving up. Your hold music, announcements, and callback options may need improvement.

### Queue Alerts

VICIdial can alert managers when queue conditions exceed thresholds:

In the inbound group settings, configure:

| Setting | Value | Purpose |
|---------|-------|---------|
| Queue Calls Count Alert | 10 | Alert when more than 10 calls are waiting |
| Queue Seconds Alert | 120 | Alert when any call has waited more than 120 seconds |
| Alert Email | manager@example.com | Email address for queue alerts |

These alerts are not instantaneous --- they depend on VICIdial's reporting refresh cycle. For operations where queue conditions change rapidly, designate a supervisor to monitor the real-time report continuously.

### Custom Real-Time Monitoring via SQL

For real-time dashboards or integration with external monitoring tools:

```sql
-- Current queue status across all inbound groups
SELECT
    campaign_id as inbound_group,
    COUNT(*) as calls_waiting,
    MIN(TIMESTAMPDIFF(SECOND, call_time, NOW())) as newest_call_seconds,
    MAX(TIMESTAMPDIFF(SECOND, call_time, NOW())) as longest_wait_seconds,
    AVG(TIMESTAMPDIFF(SECOND, call_time, NOW())) as avg_wait_seconds
FROM vicidial_auto_calls
WHERE call_type = 'IN'
  AND status = 'LIVE'
GROUP BY campaign_id;

-- Available agents per inbound group
SELECT
    vlia.group_id,
    COUNT(*) as agents_available
FROM vicidial_live_inbound_agents vlia
JOIN vicidial_live_agents vla ON vlia.user = vla.user
WHERE vla.status = 'READY'
GROUP BY vlia.group_id;

-- Inbound SLA report (calls answered within 30 seconds)
SELECT
    campaign_id as inbound_group,
    COUNT(*) as total_calls,
    SUM(CASE WHEN queue_seconds <= 30 THEN 1 ELSE 0 END) as within_sla,
    ROUND(SUM(CASE WHEN queue_seconds <= 30 THEN 1 ELSE 0 END) / COUNT(*) * 100, 1) as sla_pct
FROM vicidial_closer_log
WHERE call_date >= CURDATE()
GROUP BY campaign_id;
```

---

## Voicemail Groups

Voicemail is the safety net for your inbound operation. When queues overflow, after-hours calls come in, or no agents are available, voicemail ensures the caller can still leave a message.

### Setting Up Voicemail

**Step 1: Create a voicemail box in Asterisk.**

```bash
# Add to /etc/asterisk/voicemail.conf under [default] context
# Format: mailbox => password,name,email,pager,options
8300 => 1234,Sales Voicemail,sales@company.com,,tz=eastern|attach=yes|saycid=yes
8301 => 1234,Support Voicemail,support@company.com,,tz=eastern|attach=yes|saycid=yes
```

Reload voicemail: `asterisk -rx "voicemail reload"`

**Step 2: Record a voicemail greeting.**

Record a professional greeting via the Asterisk voicemail system (dial *98 from an extension, enter the mailbox number and PIN, then follow prompts to record a greeting) or upload a pre-recorded WAV file to `/var/spool/asterisk/voicemail/default/8300/greet.wav`.

**Step 3: Link voicemail to the inbound group.**

In the inbound group settings:

| Setting | Value |
|---------|-------|
| Voicemail | 8300 |
| After Hours Voicemail | 8300 |

The `Voicemail` field is used for overflow routing. `After Hours Voicemail` is used when calls arrive outside business hours.

### Voicemail Notification and Follow-Up

Getting voicemails into a mailbox is only half the job. They need to trigger follow-up:

1. **Email notification:** Configure `attach=yes` in voicemail.conf to email the recording as a WAV attachment. Ensure your Asterisk server has a working MTA (sendmail/postfix).

2. **CRM integration:** Use a script that monitors the voicemail spool directory (`/var/spool/asterisk/voicemail/default/XXXX/INBOX/`) and creates leads or tasks in your CRM when new voicemails arrive.

3. **Morning callback queue:** Create a process where your first-shift supervisor reviews all overnight voicemails and schedules [callbacks](/blog/vicidial-callback-automation/) for each one. This turns voicemails from dead leads into the first callbacks of the day.

---

## Advanced Configuration Patterns

### Multi-Language Routing

For operations serving multilingual markets:

1. Create an IVR that offers language selection: "For English, press 1. Para Espanol, oprima 2."
2. Each option routes to a language-specific inbound group: `SALES_EN`, `SALES_ES`
3. Staff each group with appropriately skilled agents
4. Configure overflow from `SALES_ES` to `SALES_EN` with a longer wait time (bilingual customers may accept English support if the Spanish wait is too long)

### Geographic Routing

Route callers to regional teams based on DID or area code:

```
DIDs: 212-555-XXXX → SALES_NORTHEAST
DIDs: 310-555-XXXX → SALES_WEST
DIDs: 312-555-XXXX → SALES_MIDWEST
DIDs: 800-555-XXXX → SALES_NATIONAL (overflow from all regions)
```

Each regional group has agents with local market knowledge. The national group handles overflow and after-hours from all regions.

### Tiered Support Routing

For support operations with escalation tiers:

| Tier | Inbound Group | Agents | Overflow |
|------|---------------|--------|----------|
| Tier 1 | SUP_T1 | General support agents | After 180s → SUP_T2 |
| Tier 2 | SUP_T2 | Senior support agents | After 300s → SUP_T3 |
| Tier 3 | SUP_T3 | Engineering/specialists | After 600s → Voicemail with urgent page |

Tier 1 handles most calls. Overflow goes to Tier 2, which handles complex issues and overflow. Tier 3 is for critical issues only. Each tier has its own hold music, announcements, and SLA thresholds.

### Time-of-Day Routing

Route calls differently based on time of day without changing the DID configuration:

1. Create multiple call time definitions: `morning-shift` (8AM-12PM), `afternoon-shift` (12PM-6PM), `evening-shift` (6PM-10PM)
2. Use VICIdial's filter routing (or a custom AGI script) to evaluate the current time and select the appropriate inbound group
3. Staff each shift's inbound group with the agents working that shift

This is particularly useful for operations that split shifts between in-house and remote agents, or between domestic and offshore teams.

---

## DID to Inbound Group Mapping

Every inbound operation starts with mapping DIDs to inbound groups. This is where the call enters VICIdial's routing engine.

### Creating DID Entries

Navigate to **Admin > Inbound > DIDs > Add a New DID**:

| Field | Example | Purpose |
|-------|---------|---------|
| DID Extension | 8005551000 | The phone number (no dashes, no spaces) |
| DID Description | Main Sales Line | Human-readable label |
| DID Route | IN_GROUP | Where to send the call (IN_GROUP, IVR, PHONE, etc.) |
| In-Group ID | SALES_EN | Which inbound group receives the call |
| DID Filter | (optional) | CID-based routing filter |
| DID Active | Y | Whether this DID is routing calls |

**DID Route options:**

| Route Type | Behavior |
|------------|----------|
| IN_GROUP | Route directly to an inbound group (queue) |
| IVR | Route to an IVR call menu |
| PHONE | Route directly to a specific phone/extension |
| VOICEMAIL | Route directly to a voicemail box |
| EXTEN | Route to a custom Asterisk extension |

For most operations, DIDs route to either an IVR (for menu-driven routing) or directly to an inbound group (for dedicated lines).

### Bulk DID Management

If you manage dozens or hundreds of DIDs, the admin interface is painfully slow. Use direct SQL for bulk operations:

```sql
-- Bulk update: Route all DIDs in the 800-555-1XXX range to SALES_EN
UPDATE vicidial_inbound_dids
SET group_id = 'SALES_EN',
    did_route = 'IN_GROUP'
WHERE did_pattern LIKE '8005551%';

-- Audit: Show all active DIDs and their routing
SELECT
    did_pattern,
    did_description,
    did_route,
    group_id,
    menu_id,
    did_active
FROM vicidial_inbound_dids
WHERE did_active = 'Y'
ORDER BY did_pattern;
```

For comprehensive DID management strategies, see our [DID management guide](/blog/vicidial-did-management/).

---

## Troubleshooting Inbound Issues

When inbound calls are not routing correctly, here is the diagnostic process.

### Calls Not Reaching the Queue

**Symptom:** Callers hear ringing or fast busy, but no call appears in the queue.

**Diagnostic steps:**

1. Check the DID entry: Is it active? Is the DID pattern matching the incoming call?
```sql
SELECT * FROM vicidial_inbound_dids WHERE did_pattern = '8005551000';
```

2. Check Asterisk CLI for the inbound call:
```bash
asterisk -rx "core show channels" | grep -i inbound
```

3. Check that the SIP trunk is delivering the call with the correct DID in the DNIS field. Some carriers strip leading digits or add country codes.

4. Verify the inbound group exists and is active:
```sql
SELECT group_id, group_name, active FROM vicidial_inbound_groups WHERE group_id = 'SALES_EN';
```

### Calls in Queue But Not Reaching Agents

**Symptom:** Calls appear in the queue (real-time report shows calls waiting), but agents are available and not receiving calls.

**Diagnostic steps:**

1. Verify agents are logged into the correct inbound groups:
```sql
SELECT vlia.user, vlia.group_id, vla.status FROM vicidial_live_inbound_agents vlia
JOIN vicidial_live_agents vla ON vlia.user = vla.user
WHERE vlia.group_id = 'SALES_EN' AND vla.status = 'READY';
```

2. Check the campaign's `Closer Campaigns` setting --- does it include the inbound group?

3. Check agent phone registration --- the agent may appear logged in but their SIP phone is not registered:
```bash
asterisk -rx "sip show peers" | grep agent_extension
```

4. Check for stuck calls --- a previous call may not have been properly cleared:
```sql
SELECT * FROM vicidial_auto_calls WHERE user = 'agent001';
```

### High Abandonment Rate

**Symptom:** Many callers hang up before reaching an agent.

**Root causes and fixes:**

| Cause | Indicator | Fix |
|-------|-----------|-----|
| No hold music | Callers hear silence | Configure MOH class in inbound group |
| No announcements | Callers do not know their queue position | Enable ANNOUNCE_POSITION |
| Understaffed | Average wait > 60 seconds | Add agents or activate blended campaign |
| No overflow route | Callers wait indefinitely | Configure Hold Recall Transfer |
| No callback option | Callers must wait or abandon | Enable CALLERID_CALLBACK hold time option |

### Audio Quality Issues

**Symptom:** Callers or agents report choppy audio, echo, or one-way audio on inbound calls.

This is typically an Asterisk/SIP configuration issue rather than a VICIdial issue. Check:

1. NAT configuration on the SIP trunk
2. Codec negotiation (prefer G.711 for quality, G.729 for bandwidth)
3. RTP port ranges in `rtp.conf`
4. Network QoS settings

See our [Asterisk configuration guide](/blog/vicidial-asterisk-configuration/) for detailed SIP and codec troubleshooting.

> **Inbound issues cost you money every minute they persist.**
> ViciStack provides 24/7 monitoring and rapid troubleshooting for VICIdial inbound operations. [Find out how we can help.](/free-audit/)

---

## Performance Benchmarks

Here are target benchmarks for a well-configured VICIdial inbound operation:

| Metric | Target | How to Measure |
|--------|--------|----------------|
| Average Speed of Answer (ASA) | < 30 seconds | `vicidial_closer_log` → avg `queue_seconds` |
| Service Level (calls answered within 20s) | > 80% | `queue_seconds <= 20` / total calls |
| Abandonment Rate | < 5% | Calls with `status = DROP` / total inbound calls |
| First Call Resolution | > 70% | Calls not followed by another call from the same number within 24h |
| Agent Occupancy | 75-85% | Time on calls / total logged-in time |
| Average Handle Time | Varies by operation | `length_in_sec` from `vicidial_closer_log` |

**ASA under 30 seconds** is the industry standard for sales lines. Support lines can tolerate up to 60 seconds. If your ASA is consistently above these thresholds, you have a staffing or routing problem.

**Abandonment under 5%** is achievable with proper hold music, queue announcements, and callback options. Operations running ViciStack-configured inbound with CALLERID_CALLBACK enabled typically see abandonment rates below 3%.

```sql
-- Daily inbound performance dashboard
SELECT
    DATE(call_date) as date,
    campaign_id as inbound_group,
    COUNT(*) as total_calls,
    SUM(CASE WHEN status != 'DROP' AND status != 'AFTHRS' THEN 1 ELSE 0 END) as answered,
    SUM(CASE WHEN status = 'DROP' THEN 1 ELSE 0 END) as abandoned,
    ROUND(AVG(queue_seconds), 1) as avg_queue_seconds,
    ROUND(AVG(length_in_sec), 1) as avg_handle_seconds,
    ROUND(SUM(CASE WHEN queue_seconds <= 20 THEN 1 ELSE 0 END) / COUNT(*) * 100, 1) as sla_20s_pct,
    ROUND(SUM(CASE WHEN status = 'DROP' THEN 1 ELSE 0 END) / COUNT(*) * 100, 1) as abandon_pct
FROM vicidial_closer_log
WHERE call_date >= DATE_SUB(CURDATE(), INTERVAL 7 DAY)
GROUP BY DATE(call_date), campaign_id
ORDER BY date DESC, inbound_group;
```

---

## Frequently Asked Questions

### What is the difference between an inbound group and a campaign in VICIdial?

A campaign is the operational container for outbound dialing --- it holds the dial settings, lead lists, agents, and dispositions. An [inbound group](/glossary/inbound-group/) is the container for inbound call handling --- it holds the queue settings, ring strategy, hold music, and overflow configuration. Agents can belong to both simultaneously (this is the [blended agent](/glossary/blended-agent/) setup). Think of campaigns as outbound engines and inbound groups as queues. They are separate entities that share agents through the [Closer Campaigns](/settings/closer-campaigns/) configuration.

### How many inbound groups can VICIdial handle?

There is no hard limit on the number of inbound groups. VICIdial operations with 50+ inbound groups running simultaneously are common. The practical limit is administrative --- managing 100 inbound groups with different configurations, overflow routes, and agent assignments becomes complex. Use a naming convention and documentation. Performance is not a concern; VICIdial's inbound routing is lightweight compared to outbound predictive dialing.

### Can I route the same DID to different inbound groups based on time of day?

Yes, but not through a single DID configuration. The cleanest approach is to route the DID to an [IVR](/glossary/ivr/) (call menu) and use VICIdial's time-based routing within the IVR to select different inbound groups based on the current time. Alternatively, you can use a custom AGI script in the Asterisk dialplan that evaluates the time and routes accordingly. The DID's `Call Time` setting handles after-hours routing, but for mid-day routing changes (e.g., morning team vs. afternoon team), IVR-based routing is the standard approach. See our [IVR setup guide](/blog/vicidial-ivr-setup/) for details.

### What happens if all agents log out while calls are still in queue?

The calls remain in queue and continue hearing hold music and announcements. If [No Agent No Queue](/settings/no-agent-no-queue/) is set to `N` (recommended), the calls persist until the caller hangs up or the overflow timer triggers. If you have configured overflow routing (Hold Recall Transfer), calls will overflow to the next destination after the configured wait time. If you have configured after-hours routing, calls that arrive after hours will follow the after-hours path. The worst case --- no overflow, no after-hours, all agents gone --- is a caller sitting in queue hearing hold music until they give up. This is why overflow configuration is not optional.

### How do I set up a blended campaign so agents take inbound calls between outbound dials?

Three settings make blending work: (1) Set [Campaign Allow Inbound](/settings/campaign-allow-inbound/) to `Y` on the outbound campaign, (2) list the inbound groups in [Closer Campaigns](/settings/closer-campaigns/), and (3) set `Closer Default Blended` to `1` to auto-enable blending for all agents. Then set [Inbound Queue Priority](/settings/inbound-queue-priority/) to `99` so inbound calls always take precedence. When an agent finishes any call (inbound or outbound), VICIdial checks for waiting inbound calls first. If none are queued, the agent goes back into the outbound dial pool. Set `Available Only Ratio Tally` to `Y` on the campaign to prevent over-dialing when agents are pulled to inbound.

### Can I give priority to certain callers without creating separate inbound groups?

Yes. Use DID-level queue priority. Each DID entry has a `Queue Priority` field. Calls from higher-priority DIDs jump ahead in the same inbound group's queue. You can also use CID-based routing on the DID to assign priority based on the caller's phone number. If your [IVR](/glossary/ivr/) routes callers, each IVR option can pass a different priority value to the same inbound group. These priorities stack, so a VIP caller (CID priority 50) calling your premium line (DID priority 50) who selects "urgent" in the IVR (IVR priority 30) enters the queue with priority 130.

### How do I handle inbound calls after hours without losing leads?

Configure these settings on your inbound group: set the `Call Time ID` to your business hours, set `After Hours Action` to `MESSAGE`, record a professional after-hours greeting and set it as `After Hours Message Filename`, and set `After Hours Voicemail` to a monitored voicemail box. Enable email notification on the voicemail box so your team gets notified immediately. Then implement a morning process: your first-shift supervisor pulls all overnight voicemails and schedules [callbacks](/blog/vicidial-callback-automation/) for each one. The initial VICIdial setup guide covers base configuration if you are starting from scratch --- see our [complete setup guide](/blog/vicidial-setup-guide/).

### What is the best ring strategy for a small team versus a large call center?

For teams under 10 agents, `ring_all` gives you the fastest answer times --- every available agent's phone rings at once, and the first to answer gets the call. For teams of 10-50 agents, `longest_wait_time` is the standard --- it evenly distributes calls and prevents agent cherry-picking. For large operations (50+ agents), `longest_wait_time` with agent ranking enabled provides both even distribution and [skill-based routing](/glossary/skill-based-routing/). Avoid `random` in almost all cases --- it creates uneven distribution and unpredictable wait times. The `fewest_calls` strategy is useful for compliance-sensitive operations that need provably equal call distribution per agent.

### How can I monitor queue performance in real time?

VICIdial's built-in Real-Time Report (Reports > Real-Time Report) shows calls waiting, agents available, average wait time, longest wait, and abandonment counts per inbound group. For custom dashboards, query the `vicidial_auto_calls` table for current queue state and `vicidial_closer_log` for historical metrics. Configure queue alerts on the inbound group (Queue Calls Count Alert and Queue Seconds Alert) to receive email notifications when thresholds are exceeded. For operations that need sub-minute visibility, build a polling script that queries the live tables every 10-15 seconds and pushes to an external dashboard (Grafana, Datadog, or a custom web dashboard).

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-inbound-setup).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
