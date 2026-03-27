# VICIdial Dial Hopper: How It Works and Why Yours Is Empty

**Last updated: March 2026 | Reading time: ~18 minutes**

Your agents are sitting idle. The predictive dialer paused itself. You pull up the Real-Time Report and there it is — **Hopper: 0**. Campaign is active, lists are loaded, agents are logged in, and absolutely nothing is happening.

This is the single most common crisis in VICIdial operations, and it's almost never about running out of leads. It's about a chain of settings, filters, and cron jobs that have to align perfectly for leads to flow from your lists into the hopper and then into the dialer.

If you've been running VICIdial for any length of time, you've hit this wall. Let's pull it apart.

---

## What the Dial Hopper Actually Is

The dial hopper is a staging table — `vicidial_hopper` — that sits between your lead lists and the dialer engine. Think of it as a buffer. The dialer doesn't reach into `vicidial_list` directly to find the next number to call. Instead, a background process (the VDHopper cron job) continuously scans your lists, applies every filter and rule you've configured, and loads qualifying leads into the hopper. The dialer then pulls from this pre-filtered, pre-sorted buffer.

Why the indirection? Performance. `vicidial_list` on a busy system can have tens of millions of rows. Doing a full table scan with all your filters applied every time the dialer needs a lead would hammer the database. The hopper keeps a small, ready-to-dial set in memory so the dialer can grab leads in microseconds.

Here's the flow:

```
vicidial_list (millions of leads)
        │
        ▼
  VDHopper cron (runs every minute)
  Applies: list status, lead status, timezone, filters,
           DNC check, lead recycling rules, dial count limits
        │
        ▼
  vicidial_hopper (small buffer, typically 50-1000 leads)
        │
        ▼
  VDauto_dial (predictive engine)
  Grabs leads → dials → routes to agents
```

The hopper_level setting in your campaign controls how many leads the VDHopper process tries to keep in the hopper at any given time. Default is usually 50. For high-volume campaigns with 20+ agents, you need 200-500.

---

## The VDHopper Cron Job: Your Hopper's Heartbeat

Everything starts with the cron job. If this isn't running, your hopper never fills. Period.

Check if it's running:

```bash
# See if VDHopper is in the cron
crontab -l | grep VDHopper

# Expected output (every minute):
# * * * * * /usr/share/astguiclient/AST_VDhopper.pl

# Check if it's actually executing
ps aux | grep VDHopper
```

On a ViciBox install, this is set up automatically. On manual installs or after system migrations, it's the first thing that breaks. The script lives at `/usr/share/astguiclient/AST_VDhopper.pl` and it needs to run every minute.

If the cron is running but the hopper is still empty, the problem is downstream — in your campaign settings, list configuration, or filtering rules.

### VDHopper Execution Time

On systems with large lists (5M+ leads), the VDHopper script can take longer than 60 seconds to complete. When that happens, you get overlapping executions that fight over database locks, or worse, the hopper falls behind your dial velocity.

Check execution time:

```bash
# Find how long VDHopper takes to run
grep "VDHopper" /var/log/astguiclient/process_log.txt | tail -20
```

If execution consistently exceeds 45 seconds, you need to either reduce the number of active lists, add a MySQL index, or increase your database server's resources. We've seen shops running 15M-lead lists on a single-core VM wondering why their hopper is empty — the cron literally can't finish before it's supposed to start again.

---

## Why Your Hopper Is Empty: The 12 Most Common Causes

I'm going to go through these in order of how often we see them. In eight years of running VICIdial systems, cause #1 accounts for maybe 40% of empty-hopper tickets by itself.

### 1. No Dialable Leads Left (But You Think There Are)

You loaded 100,000 leads. You've been dialing for three days. You check vicidial_list and see 87,000 records. Plenty of leads, right?

Wrong. What matters isn't total leads — it's leads with a **dialable status** that haven't exceeded your **dial count limit**.

In the VICIdial admin, go to **Campaign Settings** and look at **Dial Status**. This is the list of lead statuses that the hopper considers dialable. By default it's usually `NEW` — meaning only leads that have never been called. After one dial attempt, a lead gets a disposition (NA, B, A, etc.) and it's no longer `NEW`. Unless you've added those statuses to your dialable list, those leads are invisible to the hopper.

**The fix:** In Campaign Settings, add statuses like `NA` (No Answer), `B` (Busy), `A` (Answering Machine), `CALLBK` (Callback) to your Dial Status list. Also check your **Dial Count Limit** — if it's set to 1, leads get exactly one attempt and then they're done.

### 2. Lists Are Not Active

Every list in VICIdial has a status: Active or Inactive. The hopper only pulls from Active lists that are assigned to the campaign.

Go to **Lists** in the admin, find your lists, and verify:
- List status is **Active**
- The list is assigned to the correct **Campaign**
- The list hasn't expired (check the **Expiration Date** field)

We've seen operations run into this after a list import where the import completed successfully but the list defaulted to Inactive. You load leads, start the campaign, and wonder why nothing happens.

### 3. Timezone Restrictions Filtering Out Everything

VICIdial has built-in timezone-aware dialing. If your campaign has **Local Call Time** enforcement enabled (and it should be, for TCPA), the hopper only loads leads whose timezone + local time falls within your allowed calling window.

So if it's 9 AM Eastern and your calling window is 9 AM - 9 PM local time, leads in Pacific timezone (where it's 6 AM) won't load into the hopper. This is working correctly. But if ALL your leads are in a timezone that's currently outside the window, your hopper shows zero.

Check your campaign's **Local Call Time** setting. Then check what timezones your leads are in. VICIdial assigns timezone by area code using the `vicidial_phone_codes` table. If your leads have malformed phone numbers or area codes that don't map to a timezone, they can fall through the cracks.

### 4. Lead Filter Is Too Restrictive

Campaign lead filters are SQL WHERE clauses that get appended to the hopper query. They're powerful, but they can also silently kill your hopper.

In **Campaign Settings**, check the **Lead Filter** field. If there's a filter active, it might be filtering out more leads than you expect. A filter like `state='CA'` when your list has states stored as `California` will match zero leads.

Common filter mistakes:
- Case sensitivity (`state='ca'` vs `state='CA'`)
- Field doesn't exist or is misspelled
- Date comparisons with wrong format
- Filter references a field that's empty for most leads

**The fix:** Test your filter by checking how many leads it actually matches. Look at the **Dialable Leads Count** in the campaign detail page — this number already accounts for filters, statuses, and dial count limits.

### 5. DNC List Is Blocking Everything

If you have an active DNC (Do Not Call) list loaded and your leads overlap heavily with that list, the hopper will skip those leads. This is correct behavior, but it can be surprising when you load a purchased list that turns out to be 80% DNC numbers.

Check your **DNC settings** in Campaign Settings. VICIdial supports both internal DNC lists and campaign-level DNC lists.

### 6. Hopper Level Is Too Low

The default `hopper_level` of 50 means the system tries to keep 50 leads ready in the hopper. With a predictive dialer running at an auto-dial level of 3.0 and 10 agents, you're burning through 30 leads per minute (10 agents × 3 dials per agent). A hopper of 50 gives you less than two minutes of buffer.

If VDHopper takes 60 seconds to run (which it does on large lists), you can drain the hopper faster than it refills.

**The fix:** Set `hopper_level` proportional to your dial velocity. Rule of thumb: **agents × auto_dial_level × 5**. For 20 agents at dial level 3.0, that's 300. For 50 agents, 750 or higher.

In the admin GUI, go to **Campaign Settings** and adjust **Hopper Level**.

### 7. Database Performance Issues

The VDHopper script runs a series of MySQL queries against `vicidial_list`. On large tables with poor indexes, these queries can timeout or take so long the cron can't keep up.

Signs of database bottlenecks:

```bash
# Check for slow queries
mysqladmin processlist | grep -i hopper

# Check vicidial_list table size
mysql -e "SELECT COUNT(*) FROM vicidial_list;" asterisk

# Check if the table needs optimization
mysql -e "SHOW TABLE STATUS LIKE 'vicidial_list';" asterisk
```

If `vicidial_list` has more than 10 million rows and you're running on hardware with less than 16 GB of RAM, the hopper query can become a bottleneck. The solution is usually proper indexing, query caching, or splitting old leads into archive tables.

### 8. Lead Recycling Not Configured

After your first pass through a list, most leads end up in non-dialable statuses. Without lead recycling, those leads are permanently dead to the hopper.

Lead recycling lets you automatically move leads back to a dialable status after a waiting period. For example: leads with status `NA` (No Answer) get recycled back to dialable after 4 hours.

Configure this in **Campaign Settings → Lead Recycling**. Set the statuses you want to recycle, the delay between attempts, and the maximum number of recycle attempts.

### 9. Duplicates Handling Blocking Leads

If you have **Duplicate Check** enabled at the campaign or system level, the hopper will skip leads where the phone number already exists in the hopper or has been recently dialed. This is usually what you want — nobody likes getting called twice in 10 minutes. But if your list has high duplication rates, this can significantly reduce your dialable pool.

### 10. Call Count Limits Hit

Two settings interact here: **Dial Count Limit** (how many times the system will attempt a lead total) and **Call Count Limit** (per-campaign cap on attempts). If either is reached, that lead is dead to the hopper.

Check both settings in Campaign Settings. A Dial Count Limit of 3 means after three attempts (regardless of disposition), that lead won't load into the hopper again.

### 11. Agent Availability Affecting Hopper

In predictive mode, the dialer scales its dial velocity based on agent availability. If no agents are in READY status (all paused, all on calls, all in wrap-up), the dialer stops pulling from the hopper, which can make it appear that the hopper is the problem when it's actually an agent staffing issue.

Check the Real-Time Report. If you see agents in PAUSED status or all on active calls, the issue isn't the hopper — it's that the dialer has nobody to route calls to.

### 12. Time-Based Campaign Restrictions

Campaigns can have start and stop times configured. Outside these windows, the hopper process skips the campaign entirely.

Check **Campaign Settings → Active Time**. Also check if the campaign itself is set to Active. A campaign in "Paused" state won't get hopper loads.

---

## Hopper Settings Deep Dive

Let's go through every hopper-related setting in the Campaign admin page and what it actually does.

### hopper_level

How many leads the system tries to maintain in the hopper. This is a target, not a hard limit. VDHopper will try to fill up to this number on each pass.

| Agents | Auto-Dial Level | Recommended hopper_level |
|--------|----------------|-------------------------|
| 1-5    | 1.0-2.0        | 50-100                 |
| 5-15   | 2.0-3.0        | 100-300                |
| 15-30  | 3.0-4.0        | 300-500                |
| 30-50  | 3.0-5.0        | 500-750                |
| 50+    | 4.0-8.0        | 750-1500               |

### lead_order

Controls the sort order of leads loaded into the hopper. Options include:

- **DOWN** — Highest lead_id first (newest leads)
- **UP** — Lowest lead_id first (oldest leads)
- **DOWN COUNT** — Leads with fewer dial attempts first
- **UP COUNT** — Leads with most dial attempts first (unusual, but useful for aggressive recycling)
- **DOWN PHONE** — Descending phone number
- **UP PHONE** — Ascending phone number
- **RANDOM** — Random order each hopper load
- **UP LAST CALL TIME** — Leads called longest ago first (good for recycling)

For most outbound campaigns, **DOWN COUNT** or **UP LAST CALL TIME** are the best choices. DOWN COUNT ensures fresh leads get dialed first while recycled leads are backfilled. UP LAST CALL TIME spreads out call attempts over time, which is better for contact rates and less likely to annoy people.

### list_order_mix

When a campaign has multiple active lists, this controls how leads from different lists are mixed in the hopper. Options:

- **NONE** — No mixing, lists are processed in order of list ID
- **EVEN** — Equal distribution across lists
- **MULTI** — Custom ratio per list (you set percentages)

For campaigns running multiple lead sources (e.g., web leads + purchased list + callbacks), use MULTI and weight toward your highest-value source.

### adaptive_dropped_percentage

This affects how aggressively the dialer pulls from the hopper. If your drop rate exceeds this target percentage, the dialer slows down and pulls fewer leads. This indirectly affects hopper drain rate.

Default is 3% (the FTC safe harbor limit). Don't change this unless you understand the TCPA implications.

---

## Monitoring Hopper Health

### Real-Time Report

The fastest way to check hopper status is the Real-Time Report in the VICIdial admin. The hopper count shows next to each campaign. Green is good. Red (or zero) means you're about to have idle agents.

Watch for these patterns:
- **Hopper steadily at zero** — One of the 12 causes above. Start debugging.
- **Hopper oscillates between 0 and hopper_level** — VDHopper is running but the dialer drains it faster than it fills. Increase hopper_level or check cron frequency.
- **Hopper full but no calls going out** — The problem isn't the hopper. Check agent availability, trunk registration, and Asterisk channel availability.

### Asterisk CLI

```bash
# Check active channels (are calls actually going out?)
asterisk -rx "core show channels"

# Check SIP trunk status (can we even make calls?)
asterisk -rx "sip show peers"

# Check current dial attempts
asterisk -rx "core show channels verbose" | head -30
```

### Database Queries

Sometimes you need to look directly at the hopper table. These are read-only queries — we're just looking, not changing anything:

```bash
# How many leads in hopper right now, by campaign
mysql -e "SELECT campaign_id, COUNT(*) FROM vicidial_hopper GROUP BY campaign_id;" asterisk

# Check hopper lead details
mysql -e "SELECT * FROM vicidial_hopper WHERE campaign_id='YOURCAMPAIGN' LIMIT 10;" asterisk

# Count dialable leads in your list (matching campaign dial statuses)
mysql -e "SELECT status, COUNT(*) FROM vicidial_list WHERE list_id='YOUR_LIST_ID' GROUP BY status;" asterisk
```

That last query is the one that tells you the truth. If 95% of your leads are in status `SALE`, `DNC`, or `XFER`, you don't have a hopper problem — you have a "you've already dialed the whole list" problem.

---

## Performance Tuning for High-Volume Operations

If you're running 50+ agents and processing 500K+ dials per day, the standard hopper configuration won't cut it. Here's what production systems at that scale look like.

### Database Optimization

The hopper query joins `vicidial_list` with `vicidial_hopper`, `vicidial_dnc`, and potentially your lead filter conditions. On large tables, this join can be brutal.

Key indexes that help:

```bash
# Check if critical indexes exist
mysql -e "SHOW INDEX FROM vicidial_list WHERE Key_name IN ('index_phone', 'status');" asterisk
```

If you're running MariaDB 10.11+ (ViciBox 12.0.2 default), you get better query optimization out of the box. But if you migrated from an older system, your indexes might be suboptimal.

### Splitting the Database Load

On large clusters, the standard approach is to run the hopper cron on a dedicated database server or read replica. The VDHopper script hammers `vicidial_list` with SELECT queries, and on the same box that's handling agent phone queries and CDR writes, it competes for I/O.

VICIdial supports multi-server setups where the database runs on its own hardware. If your hopper is slow because of database contention, this is the real fix.

### Hopper Pre-warming

Before a shift starts, you want the hopper full and ready. Some operations run the VDHopper script manually before agents log in:

```bash
# Manually trigger hopper fill
/usr/share/astguiclient/AST_VDhopper.pl --force-fill
```

This preloads the hopper so there's no initial lag when agents start taking calls.

---

## Common Mistakes and Misconceptions

**"My hopper should always be at hopper_level"** — Not necessarily. The hopper drains between cron runs. If your dial velocity exceeds refill rate, the hopper will oscillate. This is normal as long as it doesn't stay at zero.

**"I'll just set hopper_level to 10,000"** — Don't. A huge hopper_level means the VDHopper script takes longer to run because it's loading more leads. It also means your lead_order is less meaningful because you're pre-loading leads that won't be dialed for hours. Keep it proportional to your actual dial velocity.

**"The hopper is empty so I need more leads"** — Maybe. But nine times out of ten, you have plenty of leads. They're just in non-dialable statuses, filtered out, or in the wrong timezone. Check your Dialable Leads Count first.

**"I'll just restart Asterisk"** — Restarting Asterisk drops all active calls. It doesn't fix the hopper. The hopper is a database table populated by a Perl script. Asterisk has nothing to do with it.

---

## Hopper Troubleshooting Flowchart

When the hopper hits zero and agents are going idle, follow this path:

**Step 1:** Is VDHopper cron running?
- Check `crontab -l | grep VDHopper`
- If not running → Add the cron entry

**Step 2:** Is the campaign active with agents in READY?
- Check Real-Time Report
- If campaign paused or no agents ready → Fix staffing

**Step 3:** Are lists assigned, active, and non-expired?
- Admin → Lists → Check status for each list in campaign
- If inactive or expired → Activate or extend

**Step 4:** Check Dialable Leads Count in Campaign Settings
- If zero → Your leads are exhausted or all filtered out
- If > 0 → The filter/timezone/DNC is blocking the hopper load

**Step 5:** Check timezone restrictions
- Is it currently within your Local Call Time for any timezone in your lead set?
- If leads are all Pacific and it's 6 AM PT → Wait for the window to open

**Step 6:** Check lead filter
- Remove the filter temporarily
- Does the hopper fill? → Your filter is the problem, fix the SQL

**Step 7:** Check database performance
- Look at VDHopper execution time
- If > 45 seconds → Database bottleneck, optimize or scale

---

## The Hopper in Multi-Server Clusters

In a VICIdial cluster (separate web, telephony, and database servers), the VDHopper cron should run on the **web/admin server** — the one that has direct, low-latency access to the database. Running it on a telephony server that connects to the database over the network adds latency to every hopper query.

ViciBox clusters configure this automatically. If you built your cluster manually, verify which server is running the hopper cron:

```bash
# On each server in the cluster
crontab -l | grep VDHopper
```

Only one server should be running VDHopper for each campaign. Running it on multiple servers doesn't load the hopper faster — it creates race conditions where both instances try to load the same leads.

---

## Lead Recycling and Hopper Interaction

Lead recycling is the feature that keeps your hopper alive after the first pass through a list. Without it, once every lead has been dialed once (assuming Dial Count Limit of 1), you're done.

Here's how recycling works with the hopper:

1. Lead gets dialed, agent dispositions it as NA (No Answer)
2. Lead's status in `vicidial_list` is now NA
3. If NA is in your **Dial Status** list AND lead recycling is configured for NA, the hopper will pick it up again after the recycling delay
4. The delay is configurable per status — e.g., NA waits 2 hours, B (Busy) waits 30 minutes, A (Answering Machine) waits 4 hours

Configure recycling in **Campaign Settings → Lead Recycling**.

The common mistake: setting Dial Status to include `NA` but NOT configuring a recycling delay. This means NA leads get immediately re-loaded into the hopper and re-dialed. Your leads get called every few minutes, which destroys your reputation and probably violates calling regulations.

Set realistic delays:
- **NA (No Answer):** 2-4 hours
- **B (Busy):** 30-60 minutes
- **A (Answering Machine):** 4-8 hours
- **CALLBK (Callback):** Handled separately by the callback system

---

## When the Hopper Isn't the Problem

Sometimes you're staring at the hopper count and it looks fine — 200, 300, 400 leads — but calls still aren't going out. The hopper is full and the dialer just isn't pulling from it.

Possible causes:
- **No trunks available:** Your SIP carrier is down or at capacity. Check `asterisk -rx "sip show peers"`.
- **All agents paused:** The predictive dialer scales to zero if nobody is in READY status.
- **Auto-dial level at 0:** Someone set the dial level to zero. Check Campaign Settings.
- **Asterisk maxchannels hit:** If your Asterisk instance is at its channel limit, no new calls can originate regardless of hopper state.
- **Max trunk lines exceeded:** Check your carrier's concurrent call limit.

---

## Wrapping Up

The dial hopper is the most misunderstood component in VICIdial. It's simple in concept (a buffer between your lists and the dialer) but complex in practice because it's affected by every campaign setting, every filter, every timezone rule, and the database performance underneath it all.

When your hopper goes empty, resist the urge to blame the software. 95% of the time, it's a configuration issue that's fixable in the admin GUI within five minutes once you know where to look.

**Need someone to look for you?** ViciStack provides fully managed VICIdial infrastructure with proactive hopper monitoring, auto-scaling dial levels, and 24/7 coverage. We've fixed this exact problem on hundreds of production systems. Our standard engagement is $5K ($1K down, $4K on completion) and the hopper issue is usually resolved in the first 48 hours along with the other optimization work. [Get in touch](/contact/) if staring at a zero hopper is costing you revenue.

---

*Related: [VICIdial Predictive Dialer Settings](/blog/vicidial-predictive-dialer-settings/) | [VICIdial Lead Recycling](/blog/vicidial-lead-recycling/) | [VICIdial List Management](/blog/vicidial-list-management/) | [VICIdial Auto Dial Level Tuning](/blog/vicidial-auto-dial-level-tuning/)*

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-dial-hopper-guide).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
