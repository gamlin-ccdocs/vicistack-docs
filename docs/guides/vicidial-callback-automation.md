# VICIdial Callback Automation: Setup & Best Practices

**Last updated: March 2026 | Reading time: ~16 minutes**

You just had a 3-minute conversation with a hot prospect. They are interested, but they are in a meeting. "Call me back at 2 PM." Your agent dispositions the call, types a note, and moves on to the next dial. At 2 PM, nothing happens. The [callback](/glossary/callback/) was never scheduled, or it was scheduled but the system did not fire it, or it fired but routed to a different agent who had no context. The prospect picks up, hears a cold open from a stranger, and hangs up.

This happens thousands of times a day across VICIdial operations worldwide. Not because VICIdial lacks callback functionality --- it has one of the most capable callback systems of any dialer --- but because most operators never configure it properly and most agents never learn the difference between a CALLBK and a CBHOLD.

This guide covers everything about VICIdial's callback system: how it works at the database and cron job level, the three callback types and when to use each, configuration steps with exact admin paths, scheduling best practices based on lead type and vertical, callback tracking and reporting, and the operational mistakes that cause callbacks to fail silently.

If your callback completion rate is below 80%, you are leaving revenue on the table. Let us fix that.

**ViciStack configures callback automation as part of every deployment**, including optimal scheduling windows, agent routing, and CRM integration. [Get callback right from day one.](/pricing/)

---

## How VICIdial Callbacks Work Internally

Before you can optimize callbacks, you need to understand the machinery. VICIdial's callback system involves three components: the `vicidial_callbacks` table, the `callback_check` cron process, and the [hopper](/glossary/hopper/) injection mechanism.

### The vicidial_callbacks Table

Every scheduled callback in VICIdial creates a row in the `vicidial_callbacks` table:

| Column | Purpose |
|--------|---------|
| `callback_id` | Auto-increment primary key |
| `lead_id` | The lead to call back (links to `vicidial_list`) |
| `list_id` | The list the lead belongs to |
| `campaign_id` | The campaign to dial the callback from |
| `status` | LIVE (pending), ACTIVE (being dialed), INACTIVE (completed/expired) |
| `entry_time` | When the callback was created |
| `callback_time` | When the callback should fire |
| `modify_date` | Last modification timestamp |
| `user` | The agent who scheduled the callback |
| `recipient` | USERONLY, ANYONE, or a specific user ID |
| `comments` | Agent notes about the callback |
| `customer_timezone` | GMT offset of the lead |
| `lead_status` | The lead's disposition when the callback was scheduled |

When an agent schedules a callback through the agent interface, VICIdial inserts a row here with `status = LIVE` and the requested `callback_time`. The system then waits.

### The callback_check Process

VICIdial runs a cron job (actually, a screen process) called `AST_VDcallback_check.pl` that executes every minute. This script:

1. Queries `vicidial_callbacks` for all rows where `status = LIVE` and `callback_time <= NOW()`
2. For each matching callback, checks whether the lead is eligible for dialing (not on [DNC](/glossary/dnc/), not in a restricted [timezone](/settings/timezone-filtering/), not already in an active call)
3. Injects the lead into the [hopper](/glossary/hopper/) with elevated priority
4. Updates the callback status to `ACTIVE`

The critical detail here is **hopper injection priority**. Callbacks are injected into the hopper with a higher priority than regular leads. This means the [predictive dialer](/glossary/predictive-dialing/) will dial callback leads before pulling new leads from the list. The exact priority is configurable, but by default callbacks jump to the front of the line.

This is why callbacks work even when you have thousands of leads in your list --- the callback lead gets dialed first, regardless of how many other leads are waiting.

### What Happens When a Callback Fires

When the callback lead reaches the front of the hopper and the dialer places the call:

- **For USERONLY/CALLBK callbacks:** The system checks if the specific agent is available. If the agent is logged in, on a pause, or in [wrap-up](/glossary/wrap-time/), the call waits in queue for that agent. If the agent is not logged in, the callback remains in LIVE status and rechecks every minute.
- **For ANYONE/CBHOLD callbacks:** The call routes to any available agent in the campaign, just like a regular outbound call.

When the call connects and the agent takes it, VICIdial's agent screen displays a callback indicator with the original agent's notes. The callback status changes to `INACTIVE` and the normal disposition workflow takes over.

---

## The Three Callback Types

VICIdial supports three callback mechanisms, each suited to different operational scenarios. Understanding the differences is the key to getting callbacks right.

### CALLBK (Agent-Specific Callback)

When an agent dispositions a call as `CALLBK`, VICIdial prompts them to select a date and time for the callback and optionally add notes. The callback is tied to that specific agent --- only they will receive the call when it fires.

**How to configure:**

1. Navigate to **Admin > Campaigns > [Campaign] > Dispositions**
2. Ensure `CALLBK` is in the campaign's disposition list (it is included by default)
3. In **Admin > Campaigns > [Campaign] > Detail**, verify that [Scheduled Callbacks](/settings/scheduled-callbacks/) is set to `Y`

**When an agent uses CALLBK:**

1. Agent clicks `CALLBK` in the disposition panel
2. A date/time picker appears --- agent selects the callback date and time
3. Agent enters callback notes (e.g., "Interested in premium plan, call back after 2 PM, ask for Sarah")
4. Agent submits --- VICIdial creates the `vicidial_callbacks` entry with `recipient = USERONLY`

**Best used for:**
- Relationship-based sales (insurance, financial services, real estate)
- Complex sales with multi-call close cycles
- Situations where the prospect asked for a specific person
- Warm transfers that need follow-up by the original agent

**Limitations:**
- If the agent is not logged in when the callback fires, it does not route to anyone else --- it just retries every minute
- If the agent quits or is terminated, their callbacks become orphaned (you need a cleanup process)
- Creates agent dependency --- your best leads are locked to specific agents

### CBHOLD (Campaign Callback)

CBHOLD works like CALLBK except the callback routes to **any available agent** in the campaign, not a specific person. This is the "anyone available" callback.

**How to configure:**

1. Navigate to **Admin > Campaigns > [Campaign] > Dispositions**
2. Add `CBHOLD` to the campaign's active dispositions if not present
3. The `Scheduled Callbacks` setting must be `Y`

**When an agent uses CBHOLD:**

The same date/time picker and notes interface appears, but the callback is created with `recipient = ANYONE`. When it fires, the lead enters the hopper and routes to whoever is available.

**Best used for:**
- High-volume operations where individual agent relationships do not matter
- Operations with high agent turnover
- Scenarios where the callback is about timing, not about the specific agent (e.g., "call back after 5 PM" for timezone reasons)
- Lead reactivation campaigns

**Advantages over CALLBK:**
- Never orphaned --- any agent can take the call
- Higher completion rates because availability is not agent-dependent
- Better for operations with shift-based scheduling

### USERONLY (Direct Agent Assignment)

USERONLY is not a disposition status --- it is a callback `recipient` type that can be configured programmatically or through the API. It functions like CALLBK but with more control:

```sql
-- Insert a USERONLY callback via direct database manipulation
INSERT INTO vicidial_callbacks (
    lead_id, list_id, campaign_id, status, entry_time,
    callback_time, user, recipient, comments
) VALUES (
    12345, 100, 'SALES', 'LIVE', NOW(),
    '2026-03-19 14:00:00', 'agent001', 'USERONLY',
    'API-scheduled: follow up on pricing discussion'
);
```

**Best used for:**
- CRM-driven callback scheduling where an external system determines who should call
- Manager-assigned callbacks
- API integrations that schedule callbacks based on business logic outside VICIdial

---

## Configuring Callback Dispositions

Getting callbacks to work correctly requires configuration at the campaign level, the disposition level, and the agent training level.

### Campaign-Level Callback Settings

Navigate to **Admin > Campaigns > [Campaign Name] > Detail**. The callback-relevant settings are:

| Setting | Recommended Value | Purpose |
|---------|------------------|---------|
| [Scheduled Callbacks](/settings/scheduled-callbacks/) | Y | Enables the callback system for this campaign |
| Callback Days Limit | 30-90 | Maximum number of days in the future an agent can schedule a callback |
| Callback Active Limit | 50-100 per agent | Maximum pending callbacks per agent (prevents agents from hoarding callbacks) |
| Callback Hours Block | N | If Y, prevents callbacks from firing outside campaign calling hours |
| Callback List Calltime Check | Y | Ensures callbacks respect list-level [call time restrictions](/settings/timezone-filtering/) |

**The `Callback Active Limit` is important.** Without it, an agent who dispositions every difficult call as CALLBK will accumulate hundreds of pending callbacks, effectively hoarding leads. Set this to a reasonable number based on your sales cycle length. For high-velocity sales (solar, insurance quotes), 20-30 is appropriate. For longer-cycle B2B sales, 50-100 may be needed.

### Disposition Configuration

The default VICIdial dispositions include `CALLBK` and `CBHOLD`. If you have removed them or are using custom dispositions, you need to ensure callback-type dispositions are properly configured.

Navigate to **Admin > Campaigns > [Campaign] > Dispositions**:

1. Click "Add New Status" if CALLBK or CBHOLD are not present
2. Set the disposition properties:
   - **Status**: CALLBK (or your custom callback status name)
   - **Description**: Agent Callback
   - **Selectable**: Y
   - **Human Answered**: Y
   - **Category**: CALLBACK
   - **Min Seconds**: 0
   - **Max Seconds**: 0

The `Category = CALLBACK` designation is what triggers VICIdial's callback scheduling interface when the agent selects this disposition. Without this category, the agent will not get the date/time picker.

### Custom Callback Statuses

You can create multiple callback-type dispositions for different scenarios:

| Status | Description | Recipient | Use Case |
|--------|-------------|-----------|----------|
| CALLBK | Agent Callback | USERONLY | Standard agent-specific callback |
| CBHOLD | Campaign Callback | ANYONE | Standard campaign-wide callback |
| CB15M | Quick Callback (15 min) | ANYONE | Lead is busy, try again shortly |
| CB24H | Next Day Callback | USERONLY | Lead requested next-day follow-up |
| CBPM | PM Callback | ANYONE | Lead only available after 5 PM |

Each custom status needs the `CALLBACK` category to trigger the scheduling interface. For statuses like CB15M where the timing is standardized, you can use VICIdial's auto-callback feature (covered below) to skip the manual scheduling step.

---

## Scheduling Best Practices

When a callback fires matters as much as whether it fires at all. Here are scheduling recommendations based on lead type, vertical, and data from production VICIdial operations.

### Optimal Callback Windows by Lead Type

| Lead Type | First Callback Window | Second Attempt | Third Attempt |
|-----------|----------------------|----------------|---------------|
| Hot inbound lead (just filled out a form) | 5-15 minutes | 1 hour | 4 hours |
| Warm outbound (interested but busy) | 15-30 minutes | 2-4 hours | Next business day |
| Appointment setting | Exact time requested | 15 min after requested time | Next business day |
| Re-engagement (old lead) | 24-48 hours | 3-5 business days | 7-10 business days |
| Post-sale follow-up | 24-72 hours | 7 business days | 30 business days |

**The 5-minute rule for hot leads:** Research consistently shows that the probability of contacting a lead drops by 10x if you wait more than 5 minutes after they submit a form. If you are running inbound web leads, your callback system should fire within 5 minutes. VICIdial can do this --- but only if the lead is loaded into the system quickly (via API, not manual upload) and the [hopper](/glossary/hopper/) is configured to process callbacks with high priority.

### Time-of-Day Optimization

Not all callback times are equal. Based on aggregated [contact rate](/glossary/contact-rate/) data across outbound dialing operations:

| Time Window | Contact Rate Index | Notes |
|-------------|-------------------|-------|
| 8:00-9:00 AM | 85 | People are commuting, lower answer rate |
| 9:00-11:00 AM | 110 | Peak morning window |
| 11:00 AM-1:00 PM | 90 | Lunch hours vary by timezone |
| 1:00-3:00 PM | 100 | Solid afternoon window |
| 3:00-5:00 PM | 105 | Strong close-of-business window |
| 5:00-7:00 PM | 115 | Best overall window for residential leads |
| 7:00-8:30 PM | 95 | Declining but still viable |

Train agents to schedule callbacks during high-contact windows, not just at the time the lead requested. If a lead says "call me tomorrow," an agent should schedule for 10:00 AM or 5:00 PM tomorrow, not 2:30 PM which may be right when the lead is in their afternoon meetings.

### Multi-Timezone Callback Handling

If you are dialing nationally, callbacks need to respect the lead's timezone, not your call center's timezone. VICIdial handles this through the `customer_timezone` field in `vicidial_callbacks` and the campaign's [timezone filtering](/settings/timezone-filtering/) settings.

**Configuration:**

1. In **Admin > Campaigns > [Campaign] > Detail**, set `Local Call Time` to your desired calling window (e.g., 9:00 AM - 9:00 PM)
2. Set `Callback List Calltime Check` to `Y`
3. The callback_check process will hold callbacks that fall outside the lead's local calling window until the window opens

**Example:** An agent on the East Coast schedules a callback for 9:00 AM for a lead in California. The callback is stored as 9:00 AM. When `callback_check` runs at 9:00 AM Eastern (6:00 AM Pacific), it sees that the lead is in the Pacific timezone and 6:00 AM is outside the 9:00 AM - 9:00 PM calling window. The callback remains in LIVE status. At 12:00 PM Eastern (9:00 AM Pacific), the callback fires.

This is one of VICIdial's underrated strengths. Many cheaper dialers do not handle multi-timezone callbacks correctly, leading to [TCPA](/glossary/tcpa/) violations from calling too early or too late.

---

## Automated Callback Systems

Beyond manual agent-scheduled callbacks, VICIdial supports several automated callback mechanisms that can dramatically increase your callback volume and completion rate.

### Auto-Callbacks via Lead Recycling

VICIdial's [lead recycling](/settings/lead-recycling/) system is essentially an automated callback engine. When a lead receives a specific disposition (NA for No Answer, B for Busy, AM for Answering Machine), the recycling system automatically reschedules the lead for another dial attempt after a configurable delay.

Navigate to **Admin > Campaigns > [Campaign] > Lead Recycling**:

| Disposition | Recycle Attempt Max | Recycle Delay (minutes) | Active |
|-------------|-------------------|------------------------|--------|
| NA (No Answer) | 5 | 120 (2 hours) | Y |
| B (Busy) | 3 | 30 | Y |
| AM (Answering Machine) | 3 | 240 (4 hours) | Y |
| DROP (Abandoned) | 2 | 15 | Y |

This is not technically the "callback" system (it uses the hopper and dial status mechanism, not the `vicidial_callbacks` table), but it serves the same purpose: ensuring leads that were not reached get dialed again at optimal intervals.

**Key distinction:** Lead recycling operates at the campaign level and applies to all leads with matching dispositions. Agent-scheduled callbacks operate at the individual lead level with specific timing. Use both.

### API-Driven Callback Scheduling

For operations with external CRM systems, webhooks, or marketing automation platforms, VICIdial's API can create callbacks programmatically:

```bash
# Schedule a callback via VICIdial's non-agent API
curl -s "https://your-vicidial-server/vicidial/non_agent_api.php" \
  --data-urlencode "source=callback_api" \
  --data-urlencode "user=API_USER" \
  --data-urlencode "pass=API_PASS" \
  --data-urlencode "function=add_callback" \
  --data-urlencode "lead_id=12345" \
  --data-urlencode "campaign_id=SALES" \
  --data-urlencode "callback_time=2026-03-19 14:00:00" \
  --data-urlencode "recipient=ANYONE" \
  --data-urlencode "callback_comments=API callback: lead visited pricing page"
```

This enables scenarios like:
- A lead visits your pricing page --- your CRM fires a webhook that schedules a VICIdial callback in 5 minutes
- A lead opens a follow-up email --- your marketing platform schedules a callback for the next business day
- A support ticket escalation automatically schedules a callback to the assigned account manager

### Start Call URL Callback Integration

VICIdial's [Start Call URL](/settings/start-call-url/) feature can notify external systems when a callback fires. This lets your CRM know that a callback is in progress and can update the agent's CRM screen with relevant context:

```
https://your-crm.com/api/callback_fired?lead_id=--A--lead_id--B--&agent=--A--user--B--&callback_id=--A--callback_id--B--
```

The `--A--` and `--B--` delimiters are VICIdial's variable replacement tokens. They get replaced with actual values when the URL is called.

---

## Callback Tracking and Reporting

You cannot manage what you do not measure. VICIdial provides several built-in reporting mechanisms for callbacks, and you can build custom reports from the underlying tables.

### The Callback Report

Navigate to **Reports > Callbacks** in VICIdial admin. This report shows:

- All pending (LIVE) callbacks by campaign and agent
- Callback completion rates (how many fired callbacks resulted in a connected call)
- Overdue callbacks (callbacks where the scheduled time has passed and the callback has not been completed)
- Callback conversion rates (callbacks that resulted in a sale or positive disposition)

### Key Callback Metrics to Track

| Metric | Target | How to Calculate |
|--------|--------|-----------------|
| Callback Completion Rate | >85% | Callbacks completed / Callbacks scheduled |
| Callback Contact Rate | >50% | Callbacks where lead answered / Callbacks attempted |
| Callback Conversion Rate | 2-3x normal conversion | Sales from callbacks / Total callbacks completed |
| Callback Abandonment Rate | <10% | Expired/cancelled callbacks / Total callbacks scheduled |
| Average Time to Callback | <5 min variance | Actual callback time - Scheduled callback time |

**Callback conversion should be significantly higher than cold dial conversion.** If your cold outbound conversion rate is 2%, your callback conversion rate should be 5-8%. If it is not, your agents are either scheduling callbacks for unqualified leads (using callbacks to avoid cold calling) or they are not preparing for callbacks (no notes review, no pre-call research).

### Custom Callback Reports via SQL

```sql
-- Callback completion rate by agent (last 30 days)
SELECT
    vc.user,
    COUNT(*) as total_callbacks,
    SUM(CASE WHEN vc.status = 'INACTIVE' THEN 1 ELSE 0 END) as completed,
    SUM(CASE WHEN vc.status = 'LIVE' AND vc.callback_time < NOW() THEN 1 ELSE 0 END) as overdue,
    ROUND(SUM(CASE WHEN vc.status = 'INACTIVE' THEN 1 ELSE 0 END) / COUNT(*) * 100, 1) as completion_pct
FROM vicidial_callbacks vc
WHERE vc.entry_time >= DATE_SUB(NOW(), INTERVAL 30 DAY)
GROUP BY vc.user
ORDER BY completion_pct DESC;

-- Callbacks that converted to sales
SELECT
    vc.user,
    vc.callback_time,
    vc.comments,
    vl.phone_number,
    vl.first_name,
    vl.last_name,
    vl.status as final_status
FROM vicidial_callbacks vc
JOIN vicidial_list vl ON vc.lead_id = vl.lead_id
WHERE vc.status = 'INACTIVE'
  AND vl.status = 'SALE'
  AND vc.entry_time >= DATE_SUB(NOW(), INTERVAL 30 DAY)
ORDER BY vc.callback_time DESC;

-- Orphaned callbacks (agent no longer active)
SELECT vc.callback_id, vc.lead_id, vc.user, vc.callback_time, vc.comments
FROM vicidial_callbacks vc
LEFT JOIN vicidial_users vu ON vc.user = vu.user
WHERE vc.status = 'LIVE'
  AND vc.recipient = 'USERONLY'
  AND (vu.user IS NULL OR vu.active = 'N')
ORDER BY vc.callback_time;
```

The orphaned callbacks query is critical. Run it weekly and reassign orphaned callbacks to active agents or change their recipient to ANYONE.

> **Stop Losing Callbacks.**
> ViciStack deploys callback automation with monitoring, orphan detection, and CRM integration from day one. [See How.](/free-audit/)

---

## Callback Abandonment: Why Callbacks Fail and How to Fix It

A callback that never connects is worse than never scheduling it at all --- it wastes the original agent's relationship-building time and creates a dead spot in your pipeline. Here are the common failure modes and fixes.

### Agent Not Logged In

**Problem:** USERONLY callbacks fire but the assigned agent is on break, called in sick, or quit.

**Fix:** Set a reasonable `Callback Active Limit` (see campaign settings above) and implement a daily orphan check script. Also consider using CBHOLD (campaign-wide) callbacks instead of CALLBK for operations with unstable agent schedules.

For permanent agent departures, reassign their callbacks:

```sql
-- Reassign all callbacks from departed agent to campaign-wide
UPDATE vicidial_callbacks
SET recipient = 'ANYONE'
WHERE user = 'departed_agent'
  AND status = 'LIVE';
```

### Callback Fires Outside Calling Hours

**Problem:** An East Coast agent schedules a 9 AM callback for a West Coast lead. The system tries to dial at 6 AM Pacific, which is outside calling hours. The callback sits in LIVE status, retrying every minute until the window opens at 9 AM Pacific, by which time the agent is on other calls.

**Fix:** Set `Callback List Calltime Check = Y` in campaign settings. This prevents callbacks from firing outside the lead's local [calling time window](/settings/timezone-filtering/). Also train agents to schedule callbacks in the lead's local time, not their own.

### Lead Already Contacted

**Problem:** A callback fires, but the lead was already contacted on a different campaign or by a different agent (in a CBHOLD scenario) and is now in a terminal disposition.

**Fix:** The `callback_check` process checks lead status before injecting into the hopper, but the check is not atomic. In high-volume environments, there is a small race condition window. Minimize this by ensuring disposition workflows mark leads as DNC, SALE, or other terminal statuses immediately upon conversation end, not during [wrap time](/settings/wrap-up-seconds/).

### Hopper Contention

**Problem:** Callbacks fire and inject into the hopper, but the hopper is already full of regular leads. The callback lead sits in the hopper competing with thousands of other leads.

**Fix:** Callbacks are injected with higher priority than regular hopper leads by default. Verify this by checking the `priority` column in the `vicidial_hopper` table --- callback leads should have a higher priority value. If callbacks are not getting dialed promptly, the issue is likely agent availability, not hopper contention. Increase your [dial level](/settings/auto-dial-level/) or add agents during peak callback windows.

---

## DNC Interaction with Callbacks

A common question: what happens if a lead is added to the [DNC](/glossary/dnc/) list after a callback is scheduled but before it fires?

VICIdial's `callback_check` process runs a DNC check before injecting the lead into the hopper. If the lead's phone number has been added to the internal DNC list or the campaign's DNC list since the callback was scheduled, the callback will **not** fire. The callback status remains LIVE, effectively dead but not cleaned up.

**Best practice:** Run a monthly cleanup of dead callbacks:

```sql
-- Deactivate callbacks for leads that are now DNC
UPDATE vicidial_callbacks vc
JOIN vicidial_list vl ON vc.lead_id = vl.lead_id
SET vc.status = 'INACTIVE'
WHERE vc.status = 'LIVE'
  AND vl.status IN ('DNC', 'DNCL', 'DNCC');

-- Deactivate callbacks for leads that have been sold
UPDATE vicidial_callbacks vc
JOIN vicidial_list vl ON vc.lead_id = vl.lead_id
SET vc.status = 'INACTIVE'
WHERE vc.status = 'LIVE'
  AND vl.status IN ('SALE', 'XSALE');
```

---

## Integration with External CRM Callback Schedulers

Many VICIdial operations use an external CRM alongside VICIdial. The CRM often has its own callback/task scheduling system. Running dual callback systems (VICIdial callbacks and CRM tasks) creates confusion and missed follow-ups.

**The recommended approach:** Use one system as the source of truth for callbacks. Most operations choose one of two patterns:

### Pattern 1: VICIdial as Callback Master

All callbacks are scheduled through VICIdial. The CRM is notified via Start Call URL when callbacks fire. Agent notes from VICIdial's callback interface are synced to the CRM via the API.

**Advantages:** Callbacks are dialed automatically by the predictive dialer. No manual agent action required to initiate the callback.

**How to implement:**
1. Set [Start Call URL](/settings/start-call-url/) to notify your CRM: `https://crm.example.com/webhook/callback?lead_id=--A--lead_id--B--`
2. Use VICIdial's agent-side web form to display CRM lead data during the callback
3. Sync callback dispositions back to the CRM via VICIdial's `dispo_call_url` setting

### Pattern 2: CRM as Callback Master

The CRM schedules callbacks. A middleware script monitors CRM callback tasks and creates corresponding VICIdial callbacks via the API. VICIdial handles the dialing.

**Advantages:** Callback scheduling benefits from CRM context (deal stage, activity history, priority scoring). Agents manage callbacks in one place (the CRM).

**How to implement:**
1. Build a polling script or webhook listener that watches for new CRM callback tasks
2. Use VICIdial's non-agent API to create callbacks: `/vicidial/non_agent_api.php?function=add_callback&lead_id=X&callback_time=Y`
3. When VICIdial fires the callback, use Start Call URL to update the CRM task status

---

## Agent Training: Getting Callbacks Right

The best callback system in the world fails if agents do not use it correctly. Here is what to train on.

### When to Use CALLBK vs CBHOLD

| Scenario | Correct Disposition | Why |
|----------|-------------------|-----|
| Prospect says "call me back at 3" | CALLBK | They expect the same person |
| Prospect is in a meeting, no specific request | CBHOLD | Timing matters, person does not |
| Prospect wants to discuss with spouse first | CALLBK | Agent has context on the conversation |
| No answer, want to retry later | Use [lead recycling](/settings/lead-recycling/) | Not a callback --- this is a retry |
| Prospect says "not interested right now" | NA or NI | Not a callback --- this is a rejection |

The most common agent mistake is using CALLBK as a dumping ground for leads they do not want to deal with. If a lead said "not interested," that is an NI disposition, not a callback. If the agent wants to try a lead again because they think the lead has potential, that is what lead recycling handles. Callbacks are for **leads that have expressed interest and requested a specific follow-up**.

### Writing Effective Callback Notes

Train agents to include these four elements in every callback note:

1. **What was discussed** --- "Quoted $180/month for premium plan"
2. **Why the callback** --- "In a meeting, can't talk now"
3. **When to call** --- Already captured in the date/time field, but context helps: "After 2 PM, she picks up kids at 3"
4. **How to open** --- "Ask for Jennifer, reference the premium plan quote"

**Bad callback note:** "call back later"
**Good callback note:** "Jennifer, interested in premium @ $180/mo, in a meeting until 2PM ET, ask about spouse add-on"

The difference in callback conversion between agents who write good notes and agents who write "call back" is measurable and significant. In operations we manage at ViciStack, agents with detailed callback notes convert callbacks at 2-3x the rate of agents with minimal notes.

---

## Advanced: Callback Priority and Routing

### Callback Priority in the Hopper

By default, callbacks enter the hopper with a priority of 99 (higher number = higher priority, normal leads default to 0). You can adjust this in the campaign settings if you need callbacks to have even higher priority or if you want to balance them with other priority leads.

Check current callback priority:

```sql
-- See callbacks in the hopper and their priorities
SELECT vh.lead_id, vh.priority, vh.source, vc.callback_time, vc.user
FROM vicidial_hopper vh
JOIN vicidial_callbacks vc ON vh.lead_id = vc.lead_id AND vc.status = 'ACTIVE'
WHERE vh.campaign_id = 'SALES'
ORDER BY vh.priority DESC;
```

### Callback Routing for Blended Campaigns

If you are running a blended campaign (outbound dialing with inbound overflow), callbacks interact with the [closer campaigns](/settings/closer-campaigns/) settings. When a callback fires in a blended campaign:

1. The callback lead enters the outbound hopper
2. If the campaign has `Campaign Allow Inbound` enabled, the callback competes with inbound calls for agent attention
3. Inbound calls typically take priority over outbound dials (including callbacks)

For operations where callback completion is critical, consider dedicating specific agents to callbacks during peak callback hours rather than running fully blended.

---

## Frequently Asked Questions

### What happens to callbacks when I pause or deactivate a campaign?

When a campaign is paused or deactivated, the `callback_check` process still runs and evaluates callbacks for that campaign. However, since no agents are logged into the campaign, the callbacks remain in LIVE status. They will fire once the campaign is reactivated and agents log in. Callbacks do not expire unless you manually deactivate them or they exceed the `Callback Days Limit` setting.

### Can an agent see their pending callbacks before they fire?

Yes. In the VICIdial agent interface, agents can view their scheduled callbacks by clicking the "Callbacks" button (if enabled in the campaign settings). This shows a list of all LIVE callbacks assigned to them with dates, times, and notes. Agents cannot edit the callback time from this view --- they need to contact a manager to reschedule.

### How many callbacks can VICIdial handle simultaneously?

There is no hard limit in VICIdial's callback system. The `callback_check` script processes all pending callbacks every 60 seconds. Practically, the bottleneck is agent availability, not the callback engine. Operations with 1,000+ pending callbacks across multiple campaigns run without issues. However, if you have more pending callbacks than available agent hours, your completion rate will suffer.

### Can I schedule callbacks for a different campaign than the one the lead was dialed from?

Not through the standard agent interface. The CALLBK and CBHOLD dispositions create callbacks for the current campaign. To schedule a callback for a different campaign, you need to use the non-agent API or direct database insert. This is a valid use case for cross-sell campaigns where an agent on Campaign A identifies a lead that should be called by Campaign B.

### How do I prevent agents from abusing callbacks to avoid cold calling?

Set the `Callback Active Limit` in campaign settings to a reasonable number (20-50 depending on your sales cycle). Monitor the callback-to-dial ratio per agent --- an agent scheduling callbacks on 30%+ of their calls is likely gaming the system. Run the callback conversion report weekly: agents whose callbacks convert at the same rate as cold dials are not adding value with their callbacks.

### Do callbacks respect the daily call limit for a lead?

Yes. If you have set a [daily call limit](/settings/daily-call-limit/) on the campaign (e.g., max 3 calls per lead per day), callbacks count toward that limit. If a lead has already been called 3 times today and a callback fires, it will be held until the next day. This is correct behavior for [TCPA compliance](/glossary/tcpa/) but can surprise agents who schedule same-day callbacks for leads that have already been dialed multiple times.

### Can I bulk-schedule callbacks via file upload?

Not through the standard VICIdial admin interface. However, you can create a script that reads a CSV file and inserts callbacks via the non-agent API or direct `vicidial_callbacks` table inserts. This is common for operations migrating from another dialer or re-engaging a batch of aged leads with scheduled follow-up times.

```bash
#!/bin/bash
# bulk_callbacks.sh - Schedule callbacks from a CSV file
# CSV format: lead_id,callback_datetime,agent,notes
while IFS=',' read -r lead_id callback_time agent notes; do
    curl -s "https://your-server/vicidial/non_agent_api.php" \
        --data-urlencode "source=bulk_cb" \
        --data-urlencode "user=API_USER" \
        --data-urlencode "pass=API_PASS" \
        --data-urlencode "function=add_callback" \
        --data-urlencode "lead_id=${lead_id}" \
        --data-urlencode "campaign_id=SALES" \
        --data-urlencode "callback_time=${callback_time}" \
        --data-urlencode "recipient=USERONLY" \
        --data-urlencode "callback_user=${agent}" \
        --data-urlencode "callback_comments=${notes}"
done < callbacks.csv
```

### What is the difference between callbacks and lead recycling?

[Lead recycling](/settings/lead-recycling/) is an automatic retry mechanism that reschedules leads based on disposition rules (e.g., retry all "No Answer" leads after 2 hours). It operates on bulk populations of leads. Callbacks are individual, agent-initiated (or API-initiated) scheduled follow-ups for specific leads at specific times. Recycling handles the leads nobody answered. Callbacks handle the leads that answered and asked for a follow-up. Use both --- they serve completely different functions.

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-callback-automation).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
