# VICIdial Inbound Queue Routing

**Last updated: March 2026 | Reading time: ~24 minutes**

Outbound dialing gets all the attention in VICIdial. The predictive dialer, the AMD configuration, the dial [level tuning](/blog/vicidial-auto-dial-level-tuning/) — that's the flashy stuff. But for blended operations (and a growing number of pure inbound shops running VICIdial), getting inbound routing right is the difference between happy customers and abandoned calls.

VICIdial's inbound routing is built on a concept called "in-groups" — which are essentially named queues that calls flow into. The system routes calls from in-groups to agents based on availability, priority, and rank (VICIdial's version of skills-based routing). It works well once you understand the model, but the documentation is scattered across forum threads and source code comments.

Here's the complete picture.

---

## The Inbound Routing Architecture

VICIdial's inbound call flow:

```
DID (phone number) → Call Menu (IVR) → In-Group (queue) → Agent
```

1. An inbound call arrives at a DID (Direct Inward Dialing number) configured in your carrier
2. VICIdial matches the DID to a routing rule in Admin > Inbound > DIDs
3. The call optionally goes through a [Call Menu](/blog/vicidial-ivr-setup/) (IVR) for self-service routing
4. The call lands in an In-Group (queue) where it waits for an available agent
5. The system selects an agent based on in-group assignments, ranks, and availability

### DIDs — Where Calls Enter

Configure DIDs at Admin > Inbound > DIDs > Add New DID:

- **DID Extension**: The DID number as delivered by your carrier (usually the last 10 digits)
- **DID Route**: Where the call goes. Options:
 - `IN_GROUP` — Send directly to an in-group (queue)
 - `CALLMENU` — Send to an IVR first
 - `AGENT` — Send directly to a specific agent (no queue)
 - `EXTEN` — Send to a custom dialplan extension
 - `VOICEMAIL` — Send to voicemail

For most operations, you'll use either `CALLMENU` (if you have an IVR) or `IN_GROUP` (if calls should go straight to a queue).

### Call Menus — The IVR Layer

If DID Route is set to `CALLMENU`, the call hits your IVR first. Configure at Admin > Inbound > Call Menus.

A basic IVR structure:

```
Welcome to Acme Corp.
Press 1 for Sales.
Press 2 for Support.
Press 3 for Billing.
Press 0 to speak with an operator.
```

Each keypress routes to an in-group:
- 1 → SALES in-group
- 2 → SUPPORT in-group
- 3 → BILLING in-group
- 0 → OPERATOR in-group
- Timeout/Invalid → Default in-group

Set these mappings in the Call Menu configuration by specifying the destination for each DTMF option.

---

## In-Groups: The Queue System

In-groups are VICIdial's queues. Each in-group represents a skill, department, or call type. Create them at Admin > Inbound > In-Groups > Add New In-Group.

### Key In-Group Settings

**Group ID**: A unique identifier (8 chars max). Use descriptive names: `SALES`, `SUPPORT`, `BILLING`, `SPANISH`.

**Group Name**: Display name shown in reports and the admin panel.

**Group Color**: The color shown on the real-time report for calls in this queue. Helps supervisors visually identify queue types at a glance.

**Queue Priority**: A number that determines this in-group's priority relative to other in-groups and outbound campaigns. Default is 0.

**How queue_priority works**:
- Higher number = higher priority
- Outbound campaign calls have priority 0 by default
- If an agent handles both inbound (SALES, priority 5) and outbound (COLD_CALL, priority 0), an inbound call will always be routed first when the agent becomes available
- If two inbound calls are waiting in different in-groups, the one with higher priority gets routed first

**Ring Time**: How many seconds each agent's phone rings before trying the next agent. Default is 15. For fast-paced operations, 10-12 is better. For complex calls where agents need prep time, 20-25.

**Retry Delay**: Seconds between ring attempts across agents. If Agent A doesn't answer within Ring Time, the system waits Retry Delay seconds before trying Agent B.

**Max Hold Time**: Maximum seconds a call can wait in queue before triggering an overflow action. Set this to match your [service level](/blog/call-center-staffing-formula/) target (e.g., 120 seconds for an 80/120 SLA).

**Hold Time Action**: What happens when Max Hold Time is reached:
- `CALLMENU` — Send to another IVR
- `IN_GROUP` — Transfer to a different in-group (overflow queue)
- `EXTEN` — Send to a custom extension
- `VOICEMAIL` — Send to voicemail
- `MESSAGE` — Play a message and disconnect

### Music On Hold

Each in-group can have its own music-on-hold setting. Configure the audio file or Asterisk MoH class to play while callers wait. Many centers use a combination of hold music and periodic comfort messages: "Your call is important to us. An agent will be with you shortly."

Set the MOH context in the in-group settings. The audio files live in `/var/lib/asterisk/mohmp3/` (for MP3) or `/var/lib/asterisk/moh/` (for native format).

---

## Skills-Based Routing with Agent Ranks

VICIdial implements skills-based routing through in-group assignments and ranks. Think of each in-group as a "skill" and the rank as the agent's proficiency in that skill.

### Assigning Agents to In-Groups

You can assign agents to in-groups in two places:

**From the user record** (Admin > Users > Modify User):
Under the "Closer In-Groups" section, you'll see a list of all in-groups. Select which ones this agent should handle.

**From the campaign settings** (Campaign > Closer In-Groups):
Select which in-groups the campaign's agents will handle by default. Individual agent overrides take precedence.

### Setting Agent Ranks

Agent rank within an in-group determines routing priority. Ranks range from 0-9, where:
- **Rank 0**: Lowest priority — this agent only gets calls from this in-group if no higher-ranked agents are available
- **Rank 5**: Default/medium priority
- **Rank 9**: Highest priority — this agent gets calls from this in-group first

Configure ranks at Admin > Users > Modify User, in the Closer In-Groups section. Each in-group assignment has a rank field.

### How Routing Decisions Work

When a call is waiting in an in-group and an agent becomes available:

1. VICIdial checks if the agent is assigned to the in-group
2. If multiple agents are available, the one with the **highest rank** in that in-group gets the call
3. If ranks are tied, the agent who has been **waiting longest** (in READY state) gets the call
4. If the call is competing with other in-group calls or outbound calls, **queue_priority** determines which call gets routed first

### Skills-Based Routing Example

A bilingual call center with Sales and Support queues:

| Agent | SALES Rank | SUPPORT Rank | SPANISH Rank |
|-------|-----------|-------------|-------------|
| Agent A | 9 | 5 | 0 |
| Agent B | 5 | 9 | 0 |
| Agent C | 7 | 7 | 9 |
| Agent D | 3 | 3 | 7 |

How this plays out:
- English Sales call → Routes to Agent A first (rank 9 in SALES)
- English Support call → Routes to Agent B first (rank 9 in SUPPORT)
- Spanish call → Routes to Agent C first (rank 9 in SPANISH)
- If Agent C is busy, Spanish call goes to Agent D (rank 7)
- If Agents A and C are both busy and a Sales call comes in, Agent B (rank 5 in SALES) gets it

This model works well for most centers. Its limitation is that you can't express complex rules like "Agent C should only handle Spanish calls when the queue has been waiting more than 30 seconds." For that level of logic, you'd need custom dialplan modifications.

---

## Blended Operations: Inbound + Outbound

Blended campaigns are where queue_priority becomes critical. Agents handle both outbound calls (predictive dialing) and inbound calls (from in-groups).

### Setting Up a Blended Campaign

1. **Create your outbound campaign** (Admin > Campaigns) with normal settings
2. **Enable Closer (Inbound) on the campaign**: Set "Allow Inbound and Blended" to Y
3. **Select in-groups**: In the campaign's Closer In-Groups settings, check the in-groups this campaign's agents should handle
4. **Set queue_priority on the in-groups**: Give your inbound in-groups priority > 0 so they take precedence over outbound

### Queue Priority for Blended Operations

The queue_priority setting determines the order of call delivery. Default outbound priority is 0.

| Call Type | queue_priority | Behavior |
|-----------|---------------|----------|
| Outbound campaign | 0 (default) | Fills in when no inbound calls are waiting |
| General inbound | 5 | Takes priority over outbound |
| VIP inbound | 10 | Takes priority over everything |
| Emergency line | 99 | Always routes first |

**Practical example**: You have 30 agents on a blended campaign. 25 are dialing outbound, 5 are handling inbound. An inbound call arrives.

- If queue_priority on the in-group is > 0 and an agent just became READY, the inbound call gets that agent instead of the next outbound dial
- Agents already on outbound calls are NOT interrupted. Only the next-available agent gets redirected
- When the inbound queue is empty, all READY agents go back to outbound dialing automatically

### The Dial Level Difference Target Trick

VICIdial has a setting called **Dial Level Difference Target** that reserves agents for inbound. If set to 1, the dialer always keeps 1 agent in READY state instead of sending them all outbound calls. This ensures there's always someone available for incoming calls.

Set this in Campaign > Detail Settings > Dial Level Difference Target.

For a 30-agent blended team: set Dial Level Difference Target to 2-3. This keeps 2-3 agents in READY at all times for inbound, while the rest handle outbound.

---

## Advanced Queue Configuration

### Queue Prioritization for Different Customer Types

Route VIP customers to the front of the queue:

1. **Create two in-groups**: `SUPPORT` (priority 5) and `VIP_SUPPORT` (priority 10)
2. **Route VIP DIDs**: If you have a separate phone number for VIP customers, point its DID to the VIP_SUPPORT in-group
3. **Or use IVR identification**: Have your Call Menu ask for an account number, use an AGI script to look up the account, and route VIP accounts to the higher-priority in-group

### Time-Based Routing

Route calls differently based on time of day:

**Option 1: DID Time-Based Route**
In the DID configuration, VICIdial supports setting different routes for different time frames. Configure "Route Option" with time conditions to send calls to different in-groups during business hours vs. [after hours](/blog/call-center-after-hours/).

**Option 2: Call Menu with Time Check**
Create a Call Menu that uses the `GotoIfTime()` dialplan application to branch based on current time:

During business hours → Route to live agent in-groups
After hours → Route to voicemail or after-hours message

### Overflow Between In-Groups

When one queue is overloaded, overflow to another:

1. On the primary in-group, set **Max Hold Time** to your threshold (e.g., 120 seconds)
2. Set **Hold Time Action** to `IN_GROUP`
3. Set the **Hold Time Target** to your overflow in-group

The overflow in-group might have:
- A different set of agents (your overflow team)
- The same agents plus additional ones from another department
- A lower priority to prevent stealing agents from the primary queue during normal load

### Multi-Site Routing

For organizations with multiple VICIdial clusters (different offices or regions):

VICIdial supports multi-server clusters where agents at different physical locations share the same campaign and in-group pools. Calls route to agents regardless of which server they're logged into, as long as the servers are configured as a cluster.

This gives you geographic redundancy — if one site goes down, calls automatically route to agents at other sites.

---

## Monitoring Inbound Queue Performance

### Real-Time Report for Inbound

The [real-time report](/blog/vicidial-realtime-report-guide/) shows inbound queue status:
- **Calls Waiting**: Number of callers in queue right now
- **Average Hold Time**: How long callers are waiting on average
- **Longest Wait**: The oldest call in queue (this is the one about to abandon)
- **Agents on Inbound**: How many agents are handling inbound calls

### Key Inbound KPIs

**Service Level**: Percentage of calls answered within a target time. the baseline for a 50-agent VICIdial shop is "80/20" — 80% of calls answered within 20 seconds. Some centers target 80/30 or 80/60 depending on call type.

Calculate from VICIdial data:
```sql
SELECT
 DATE(closecalldate) AS call_date,
 COUNT(*) AS total_calls,
 SUM(CASE WHEN queue_seconds <= 20 THEN 1 ELSE 0 END) AS answered_in_20s,
 ROUND(
 SUM(CASE WHEN queue_seconds <= 20 THEN 1 ELSE 0 END) * 100.0 / COUNT(*),
 1
 ) AS service_level_pct
FROM vicidial_closer_log
WHERE closecalldate >= DATE_SUB(NOW(), INTERVAL 7 DAY)
GROUP BY DATE(closecalldate)
ORDER BY call_date;
```

**Abandonment Rate**: Percentage of callers who hang up before reaching an agent. Target: under 5%.

```sql
SELECT
 DATE(closecalldate) AS call_date,
 COUNT(*) AS total_calls,
 SUM(CASE WHEN status = 'DROP' OR status = 'HXFER' THEN 1 ELSE 0 END) AS abandoned,
 ROUND(
 SUM(CASE WHEN status = 'DROP' OR status = 'HXFER' THEN 1 ELSE 0 END) * 100.0 / COUNT(*),
 1
 ) AS abandonment_pct
FROM vicidial_closer_log
WHERE closecalldate >= DATE_SUB(NOW(), INTERVAL 7 DAY)
GROUP BY DATE(closecalldate);
```

**Average Speed of Answer (ASA)**: Mean time callers wait before connecting. Target: under 30 seconds.

---

## Common Inbound Routing Problems

### "Calls sit in queue even though agents show READY"

**Cause 1**: Agent isn't assigned to the in-group. Check the agent's Closer In-Groups in their user record.

**Cause 2**: Campaign's Closer In-Groups don't include this in-group. Check the campaign settings.

**Cause 3**: The `AST_VDauto_dial_FILL.pl` or `AST_VDremote_agents.pl` processes aren't running. These scripts manage inbound call routing:

```bash
ps aux | grep AST_VD
```

If the FILL script isn't running, inbound calls won't route to agents.

### "Inbound calls always go to outbound agents instead of dedicated inbound agents"

Check your queue_priority settings. If the in-group's queue_priority is 0 (same as outbound), the system may route inbound calls to whichever agent is available first, regardless of whether they're "supposed to be" inbound-only.

Fix: Set a higher queue_priority on your inbound in-groups (5 or higher). This ensures inbound calls take priority over outbound auto-dials.

### "After-hours calls still ring agents"

Check your DID time routing or Call Menu time conditions. Also verify that:
- Your server's timezone is set correctly (`timedatectl`)
- The Campaign's "Local Call Time" settings match your business hours
- Agents aren't still logged in after business hours (auto-logout settings)

### "VIP routing doesn't seem to prioritize"

Confirm the VIP in-group has a **higher** queue_priority than the standard in-group. Confirm the agents assigned to the VIP in-group have appropriate ranks. And confirm calls are landing in the correct in-group — add logging to your IVR or DID routing to verify.

---

## Making Inbound Work at Scale

Inbound routing in VICIdial is flexible enough for most call center scenarios. The key is getting the priority stack right — which in-groups matter most, which agents are skilled for which queues, and how overflow handles the inevitable surges.

---

---

**Related reading:**
- [VICIdial Inbound Queue Routing: Priority, Skills, and the Stuff That Actually Works](https://vicistack.com/blog/vicidial-inbound-queue-routing/)
- [VICIdial CNAM Lookup Integration for Inbound Routing](https://vicistack.com/blog/vicidial-cnam-lookup-inbound/)
- [VICIdial Queue & Inbound Group Configuration Guide](https://vicistack.com/blog/vicidial-inbound-setup/)
- [Migrating from GoAutoDial to VICIdial: Step-by-Step](https://vicistack.com/blog/goautodial-to-vicidial-migration/)
## Queue Performance Tuning Tips

### Optimize Ring Time for Your Operation

The default 15-second ring time works for most centers, but tuning it can improve [answer rates](/blog/vicidial-caller-id-reputation/):

- **Reduce to 10-12 seconds** for high-volume centers where agents are always available. Shorter ring times mean faster rotation to the next available agent if the first one doesn't pick up.
- **Increase to 20-25 seconds** for complex calls where agents need a moment to prepare. Some agents review the screen pop before answering.

### Comfort Messages Reduce Perceived Wait Time

Configure periodic announcements in your in-group:

```
# In-Group Settings:
# Hold Time Message: "Thank you for holding. An agent will be with you shortly."
# Message Frequency: Every 30 seconds
# Position Announcement: "You are caller number [X] in the queue."
```

Studies show that callers who hear position announcements wait 40% longer before abandoning than callers who hear only music. A 30-second announcement interval is the sweet spot — frequent enough to reassure, infrequent enough to not annoy.

### Use Queue Callbacks for Long Wait Times

When hold times exceed 2 minutes, offer callers a callback option. VICIdial can handle this through a Call Menu that detects long queue times and offers:

"Your estimated wait time is [X] minutes. Press 1 to remain on hold, or press 2 to receive a callback when an agent is available."

Option 2 captures the caller's number and injects it as a callback lead in the campaign, and then disconnects the call. The agent calls back when available. The caller's experience is dramatically better, and your abandonment rate drops.

If your inbound operation is struggling with long hold times, misrouted calls, or blended agent confusion, the [ViciStack engagement](https://vicistack.com/) includes a complete inbound routing audit and configuration. We've set up blended operations for centers handling 500+ inbound calls per day alongside 20,000+ outbound dials. Getting the queue_priority and agent ranking right for that volume takes experience — and it's the kind of thing that pays for itself in the first week through reduced abandonment and faster answer times.

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-inbound-queue-routing).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
