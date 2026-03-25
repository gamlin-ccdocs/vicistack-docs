# VICIdial List Management Best Practices: Stop Burning Your Best Leads

## Why List Management Is the Hidden Performance Lever

Most VICIdial operators obsess over [dial levels](/settings/auto-dial-level/) and [trunk capacity](/blog/vicidial-predictive-dialer-settings/) while ignoring the single factor that determines whether any of that infrastructure actually produces revenue: the quality and organization of their lead data.

Here is the uncomfortable truth. Two call centers running the exact same VICIdial build, with the same number of agents, the same carrier, and the same product offer, will produce wildly different results based entirely on how they manage their lists. One center carefully segments leads by age, source, and prior contact history. The other dumps everything into a single list and lets the dialer chew through it. The first center consistently hits 12-15% contact rates and 3-4% conversion. The second hovers around 6% contact and wonders why their cost per acquisition keeps climbing.

The gap is not about technology. It is about list discipline.

List management in VICIdial is the practice of controlling which leads get dialed, in what order, how many times, and at what intervals. It encompasses everything from the moment a lead record enters your `vicidial_list` table to the moment it is retired from active dialing. Done well, it maximizes the revenue extracted from every lead you purchase or generate. Done poorly, it burns through your best prospects in a single afternoon and leaves your agents dialing dead numbers for the rest of the week.

The stakes are real. Lead costs have increased across nearly every vertical over the past three years. If you are paying $8-25 per lead in insurance, solar, or home services, burning through a 50,000-record list in two days without proper recycling and segmentation is not just inefficient --- it is financially reckless. You are essentially paying full price for leads that never get a fair chance at contact because they were buried under a pile of disconnected numbers and wrong-timezone dials.

This guide covers the complete lead management lifecycle inside VICIdial: upload formatting, deduplication, segmentation strategy, recycling logic, disposition codes, time zone compliance, hopper configuration, lead aging, and the KPIs that tell you whether your list strategy is actually working. Every recommendation is specific to VICIdial's admin interface, database schema, and operational model. If you run a different dialer, some concepts will translate, but the implementation details are VICIdial-specific.

Let us fix your lists.

## VICIdial List Architecture: Lists, Campaigns, and Lead Tables

Before you can optimize list management, you need to understand how VICIdial actually organizes lead data at the database level. The architecture is straightforward, but the relationships between lists, campaigns, and the lead table create both flexibility and potential confusion.

### The vicidial_list Table

Every lead in VICIdial lives in a single MySQL table: `vicidial_list`. This is the master lead repository. Whether you have 10,000 leads or 10 million, they all sit in this one table, differentiated by the `list_id` column. The key columns you need to understand are:

| Column | Purpose |
|---|---|
| `lead_id` | Auto-increment primary key, unique per lead |
| `list_id` | Associates the lead with a specific list |
| `status` | Current disposition (NEW, NA, B, A, SALE, DNC, etc.) |
| `phone_number` | Primary dial number (minimum 8 digits) |
| `phone_code` | Country code (1 for US/Canada) |
| `vendor_lead_code` | Your internal lead ID from upstream sources |
| `called_count` | Total number of dial attempts on this lead |
| `called_since_last_reset` | Y/N flag --- controls whether the hopper picks it up |
| `gmt_offset_now` | Time zone offset, used for time zone filtering |
| `rank` | Numeric priority value for lead ordering |
| `owner` | Can be used for agent-specific or team-specific routing |
| `entry_date` | When the lead was loaded into the system |
| `modify_date` | Last time the lead record was updated |

Understanding these columns is critical because every list management operation --- filtering, recycling, segmentation, reporting --- ultimately operates on these fields.

### Lists and Campaigns

A **list** in VICIdial is simply a logical grouping defined by a `list_id` value. Lists are assigned to campaigns, and a single campaign can dial from multiple lists simultaneously. This is the foundation of segmentation: you create separate lists for different lead sources, age cohorts, or quality tiers, then activate or deactivate them within a campaign as needed.

To view and manage lists, navigate to **Admin > Lists** in the VICIdial admin interface. From here you can:

- Create new lists and assign them to campaigns
- Set lists to Active or Inactive
- Reset the `called_since_last_reset` flag for all leads in a list
- Access the lead loader to upload new data
- Override campaign-level settings at the list level (caller ID, scripts, web forms)

A critical architectural point: lists belong to campaigns, but the `vicidial_list` table is global. This means a single phone number can exist in multiple lists. VICIdial does not enforce cross-list deduplication by default --- that is your responsibility, and we will cover how to handle it in the next section.

### Campaign-Level List Controls

Within **Admin > Campaigns > [Campaign Name]**, the **List Settings** section controls how the campaign interacts with its assigned lists:

- **Dial Statuses**: Which lead statuses the campaign will attempt to dial. If a status is not checked here, leads with that status will not enter the hopper regardless of which list they are in.
- **Lead Order**: Controls the sequence in which leads are pulled from the `vicidial_list` table into the hopper. Options include UP (oldest first), DOWN (newest first), UP PHONE (by phone number ascending), DOWN COUNT (lowest call count first), and several rank-based orderings.
- **Lead Filter**: A SQL WHERE clause that further restricts which leads are eligible for the hopper beyond the dial status check.
- **List Order Randomize**: When enabled, adds randomization to the lead ordering to prevent predictable dial patterns.

The interplay between dial statuses, lead filters, and lead ordering is where most list management strategy lives. Getting these three settings right for each campaign is worth more than any amount of trunk optimization.

> **Your list architecture determines your contact rate.** If your campaigns are dialing from unstructured, unsegmented lists, you are leaving money on the table. [Request a free ViciStack audit](/free-audit/) and we will map your current list structure against best practices.

## Lead Upload Best Practices: Format, Deduplication, and Validation

The quality of your lead data at the point of upload determines the ceiling for every downstream metric. Garbage in, garbage out is not a cliche in outbound dialing --- it is a measurable financial reality. A list with 15% invalid phone numbers will drag down your contact rate, inflate your cost per contact, and waste agent time on dead dials.

### File Format and Field Mapping

VICIdial's lead loader (`/vicidial/admin_listloader_fourth_gen.php`) accepts data files in several formats:

- **CSV** (comma-separated)
- **Tab-delimited TXT**
- **Pipe-delimited TXT**
- **XLS** (Excel)
- **ODS** (OpenDocument)

The standard field layout for VICIdial lead uploads follows this column order:

| Position | Field | Notes |
|---|---|---|
| 1 | Vendor Lead Code | Your upstream lead ID |
| 2 | Source Code | Lead source identifier |
| 3 | List ID | Target list (overridden if set in loader) |
| 4 | Phone Code | Country code (1 for US/Canada) |
| 5 | Phone Number | Must be at least 8 digits |
| 6 | Title | Mr., Mrs., etc. |
| 7 | First Name | |
| 8 | Middle Initial | |
| 9 | Last Name | |
| 10 | Address 1 | |
| 11 | Address 2 | |
| 12 | Address 3 | |
| 13 | City | |
| 14 | State | 2-character abbreviation |
| 15 | Province | |
| 16 | Postal Code | |
| 17 | Country | |
| 18 | Gender | |
| 19 | Date of Birth | |
| 20 | Alt Phone | |
| 21 | Email | |
| 22 | Security Phrase | |
| 23 | Comments | |
| 24 | Rank | Numeric priority value |
| 25 | Owner | Agent or team assignment |

**Pro tip**: Always use tab-delimited format with the "standard" column layout for the most reliable uploads. CSV files with commas embedded in address or comment fields are a common source of parsing errors that silently corrupt your data.

### Pre-Upload Validation

Before any file touches VICIdial's lead loader, run it through these validation checks:

1. **Phone number format**: Strip all non-numeric characters. Remove leading 1 for US numbers (VICIdial adds the phone_code separately). Verify minimum 10-digit length for US numbers. Remove any numbers shorter than 8 digits --- they will fail to load.

2. **State field**: Must be exactly 2 characters. Longer values will be truncated silently, which can break time zone lookups and compliance filtering.

3. **Empty required fields**: Every row must have a phone number. Rows missing phone numbers will either fail to load or create dead records that waste hopper space.

4. **Character encoding**: Use UTF-8. Non-ASCII characters in name or address fields can cause parsing failures in certain VICIdial versions.

5. **DNC scrubbing**: Run your list against the national DNC registry and any internal DNC lists before upload, not after. Loading DNC numbers and relying on VICIdial's internal DNC check adds unnecessary records to your database and slows hopper processing.

### Deduplication Strategy

VICIdial's built-in duplicate checking during upload offers several modes:

- **CHECK FOR DUPLICATES BY PHONE IN LIST**: Checks only within the target list ID
- **CHECK FOR DUPLICATES BY PHONE IN CAMPAIGN LISTS**: Checks across all lists assigned to the same campaign
- **CHECK FOR DUPLICATES BY PHONE IN SYSTEM**: Checks the entire vicidial_list table
- **NO DUPLICATE CHECK**: Loads everything

For most operations, **CHECK FOR DUPLICATES BY PHONE IN CAMPAIGN LISTS** is the right choice. It prevents the same number from being dialed multiple times within a single campaign while still allowing the same lead to exist in lists assigned to different campaigns (which may be desirable for different product offers or follow-up sequences).

For deeper deduplication, VICIdial includes a command-line script:

```
/usr/share/astguiclient/VICIDIAL_DEDUPE_leads.pl
```

This script scans for duplicate phone numbers across campaigns or the entire system and moves the newer duplicates to a designated quarantine list. Run it as a scheduled cron job after batch uploads:

```bash
# Run deduplication nightly at 1 AM, moving dupes to list 99999
0 1 * * * /usr/share/astguiclient/VICIDIAL_DEDUPE_leads.pl --campaign=SALES --duplicate-list=99999
```

You can also find duplicates manually via MySQL:

```sql
SELECT phone_number, COUNT(*) AS dupe_count
FROM vicidial_list
WHERE list_id IN (101, 102, 103)
GROUP BY phone_number
HAVING dupe_count > 1;
```

### Post-Upload Verification

After every upload, verify the load was successful:

```sql
SELECT list_id, COUNT(*) AS lead_count,
       COUNT(CASE WHEN status = 'NEW' THEN 1 END) AS new_leads
FROM vicidial_list
WHERE list_id = 1001
GROUP BY list_id;
```

Check that the lead count matches your source file row count. If the numbers do not match, check the lead loader's error log for parsing failures. A 1-2% variance is common due to duplicate removal, but anything above 5% indicates a formatting problem in your source file.

## Lead Segmentation: How to Build Lists That Perform

Segmentation is the practice of splitting your lead pool into distinct lists based on attributes that predict contact likelihood and conversion potential. It is the single most impactful list management technique, and most VICIdial operators either skip it entirely or implement it too simply.

### Why Segmentation Matters

A blended, unsegmented list forces VICIdial to treat every lead identically. A fresh, high-intent web form submission from 10 minutes ago gets the same dial priority as a 45-day-old purchased list lead that has already been called 6 times. The predictive algorithm does not know the difference --- it only sees dial statuses and called counts. Your segmentation strategy is what gives the dialer the intelligence it lacks on its own.

### Segmentation Dimensions

Build your lists along these dimensions, in order of impact:

**1. Lead Age (Speed to Lead)**

This is the highest-impact segmentation variable. Industry data consistently shows that leads contacted within 5 minutes of form submission convert at 8-10x the rate of leads contacted after 30 minutes. Create separate lists for:

- **Hot leads (0-1 hour old)**: Highest priority, highest [auto-dial level](/settings/auto-dial-level/)
- **Warm leads (1-24 hours old)**: Standard priority
- **Aging leads (1-7 days old)**: Lower priority, recycling candidates
- **Cold leads (7+ days old)**: Lowest priority, different script and approach

**2. Lead Source**

Not all lead sources produce equal quality. A lead from a targeted landing page with a multi-step form converts differently than a lead from a shared list broker. Create separate lists per source so you can measure performance at the source level and allocate dial time accordingly.

**3. Geographic Region**

Geographic segmentation serves two purposes: time zone compliance (covered in detail later) and regional performance optimization. Some products sell better in certain states. Some carriers have better completion rates in certain area codes. Separate lists let you see these patterns.

**4. Prior Contact History**

Leads that have been previously contacted but not converted should be in separate lists from fresh leads. A lead that answered once and said "call me back" is fundamentally different from a lead that has never been reached. These require different scripts, different dial cadences, and often different agents.

**5. Product or Offer**

If your operation sells multiple products, segment by which product the lead expressed interest in. Cross-selling from a blended list produces lower conversion than matching leads to the product they originally inquired about.

### Implementation in VICIdial

To create a segmented list structure:

1. Navigate to **Admin > Lists > Add a New List**
2. Assign a meaningful List ID (use a naming convention: e.g., 1001 for Source A / Hot, 1002 for Source A / Warm, etc.)
3. Assign the list to the appropriate campaign
4. Set the list to Active

Then, configure your campaign to leverage segmentation:

- Use **Lead Order: DOWN RANK** in campaign settings, and set the `rank` field during upload to assign priority scores to different lead tiers
- Use **Lead Filters** to add SQL-based restrictions. For example, to only dial leads uploaded in the last 24 hours:

```sql
entry_date >= DATE_SUB(NOW(), INTERVAL 24 HOUR)
```

- Override campaign settings at the list level where needed. In **Admin > Lists > [List ID] > List Modification**, you can set per-list overrides for Caller ID, Agent Script, and Transfer settings. This lets you show a local caller ID for geographic lists or load a different script for callback leads.

> **Segmentation is where amateur operations become professional ones.** If you are running more than 20 agents on unsegmented lists, you are almost certainly underperforming your potential. [Let ViciStack audit your list structure](/free-audit/) --- we have seen segmentation alone lift conversion by 20-40%.

## Recycling Logic: How to Set Up Intelligent Re-dial Sequences

Lead recycling is the mechanism by which VICIdial automatically re-dials leads that received certain dispositions (Busy, No Answer, Answering Machine) after a specified delay. Without recycling, these leads sit untouched until you manually reset the list --- and by then, the optimal contact window may have passed.

### How VICIdial Lead Recycling Works

Lead recycling is configured per-campaign under **Admin > Campaigns > [Campaign Name] > Lead Recycling**. For each recyclable status, you define:

- **Status**: The disposition code to recycle (e.g., B, NA, A)
- **Attempt Delay**: The number of seconds before the lead becomes eligible again (minimum 120 seconds, maximum 43,199 seconds / ~12 hours)
- **Attempt Maximum**: How many recycling attempts are allowed (1 to 10)

When a lead receives a recyclable disposition, VICIdial marks it and waits the specified delay before making it eligible for the hopper again. The lead does not need a list reset --- it automatically re-enters the dialing queue.

**Critical limitation**: VICIdial's lead recycling was designed to operate within a single dialing day. The maximum delay of 43,199 seconds (just under 12 hours) means recycling does not reliably span across midnight boundaries. For multi-day re-dial sequences, you need to use list resets combined with dial statuses, not recycling alone.

### Recommended Recycling Schedule

Here is a recycling configuration that balances contact rate optimization with compliance and lead preservation:

| Status | Description | Attempt Delay | Max Attempts | Rationale |
|---|---|---|---|---|
| B | Busy | 300 seconds (5 min) | 5 | Busy signals clear quickly; rapid retry captures callbacks |
| NA | No Answer | 3600 seconds (1 hour) | 3 | Stagger re-attempts across different times of day |
| A | Answering Machine | 7200 seconds (2 hours) | 2 | If hitting VM twice, they are screening; back off |
| ADC | Auto-Dial Carrier (no answer from carrier) | 1800 seconds (30 min) | 3 | Network congestion clears; retry at moderate intervals |
| AA | Auto Answer (Answering Machine auto-detected) | 10800 seconds (3 hours) | 2 | Similar to A; longer delay to avoid VM fatigue |
| AL | Alternate Number | 600 seconds (10 min) | 3 | Retry alt numbers with moderate spacing |

### Multi-Day Re-dial Strategy

Since recycling maxes out within a single day, your multi-day re-dial strategy must use a combination of:

1. **Dial Statuses**: Configure your campaign to include NA, B, and A in the dial statuses list. After a list reset, these leads become eligible again.

2. **Nightly List Reset**: Schedule a list reset via cron to reset the `called_since_last_reset` flag:

```sql
UPDATE vicidial_list
SET called_since_last_reset = 'N'
WHERE list_id = 1001
  AND status IN ('NA', 'B', 'A')
  AND called_count < 8;
```

3. **Lead Filters for Attempt Caps**: Use a campaign lead filter to enforce a maximum attempt count across days:

```sql
called_count < 8
```

This combination gives you recycling within each dialing day (handled by the recycling feature) and re-dial across days (handled by nightly resets with attempt caps).

### The 7-Day Re-dial Cadence

A recommended re-dial cadence for a standard outbound campaign:

| Day | Attempts | Strategy |
|---|---|---|
| Day 1 | 3-4 attempts | Intra-day recycling at 1-2 hour intervals |
| Day 2 | 2-3 attempts | Shifted time window (if you called morning on Day 1, try afternoon on Day 2) |
| Day 3 | 2 attempts | Different time of day again |
| Day 4 | Rest day | No dials --- breaks up the pattern and reduces spam risk |
| Day 5 | 2 attempts | Resume with fresh timing |
| Day 6 | 1 attempt | Final standard attempt |
| Day 7 | 1 attempt | Last attempt before moving to aged/recycled list |

After Day 7, leads that have not been contacted should be moved to an aging list with a reduced dial frequency (covered in the Lead Aging section).

> **Most centers either over-dial or under-dial their recycled leads.** Over-dialing burns caller ID reputation and triggers spam flags. Under-dialing leaves contactable leads sitting in limbo. [Get a ViciStack recycling audit](/free-audit/) to find the right balance for your vertical.

## Disposition Management: Which Statuses to Create and Why

Disposition codes are the taxonomy of your operation. Every call outcome maps to a status, and that status determines what happens to the lead next: does it get re-dialed, retired, moved to a callback queue, or flagged for DNC? Getting your disposition structure right is foundational to every other list management practice.

### VICIdial System Default Statuses

VICIdial ships with a set of system-wide default statuses. These cannot be deleted, and they have built-in behaviors:

| Status | Name | Type | Behavior |
|---|---|---|---|
| NEW | New Lead | System | Lead has not been dialed |
| NA | No Answer | System | Outbound call received no answer signal |
| B | Busy | System | Agent or system detected busy signal |
| A | Answering Machine | System | Agent flagged as voicemail/VM |
| AA | Auto Answering Machine | System | AMD (answering machine detection) auto-flagged |
| DC | Disconnected | System | Number is not in service |
| DNC | Do Not Call | System | Adds lead to VICIdial internal DNC list |
| N | Not Interested | System | Prospect declined |
| NI | Not Interested | System | Alternate not-interested code |
| SALE | Sale Made | System | Successful conversion |
| CALLBK | Call Back | System | Scheduled callback |
| DROP | Dropped Call | System | Predictive dialer dropped the call (no agent available) |
| XFER | Transfer | System | Call transferred to another queue |

For a full reference of all default statuses and their behaviors, see our [VICIdial statuses reference](/statuses/).

### Custom Statuses You Should Create

Beyond the defaults, create campaign-specific statuses that map to your actual business outcomes. Add custom statuses via **Admin > Campaign > Campaign Statuses** or system-wide via **Admin > Statuses**.

Here is a recommended custom status framework for a typical outbound sales operation:

| Status | Name | Category | Dialable | Human Answer | Purpose |
|---|---|---|---|---|---|
| CBHOLD | Callback Hold | Followup | N | Y | Prospect interested but needs time; do not auto-dial |
| DNQUAL | Did Not Qualify | Terminal | N | Y | Contacted but does not meet criteria |
| LMVM | Left Message on VM | Recyclable | Y | N | Agent left voicemail; recycle after delay |
| WRNUMB | Wrong Number | Terminal | N | Y | Number does not belong to intended prospect |
| DECMKR | Decision Maker Unavail | Recyclable | Y | Y | Reached household but not the decision maker |
| APPT | Appointment Set | Terminal | N | Y | Appointment booked; remove from outbound |
| NOFUND | No Funding/Budget | Terminal | N | Y | Qualified contact but cannot afford product |
| CMPTN | Went with Competitor | Terminal | N | Y | Prospect chose another provider |
| HUAGNT | Hung Up on Agent | Recyclable | Y | Y | Immediate hangup; may recycle once |
| INVNUM | Invalid Number | Terminal | N | N | Number format valid but reaches wrong party/fax/modem |

### Status Categories

VICIdial supports **Status Categories** that group individual statuses into logical buckets. Create categories like:

- **CONTACTED**: All statuses where a human answered (SALE, N, DNQUAL, CBHOLD, DECMKR, WRNUMB, APPT, NOFUND, CMPTN)
- **UNCONTACTED**: All statuses where no human answered (NA, B, A, AA, DC, DROP)
- **TERMINAL**: All statuses where the lead should never be re-dialed (SALE, DNC, DC, DNQUAL, WRNUMB, APPT, INVNUM, CMPTN)
- **RECYCLABLE**: All statuses that should be re-attempted (NA, B, A, LMVM, DECMKR, HUAGNT)

Status categories simplify reporting and allow you to build campaign logic around categories rather than individual codes. If you later add a new terminal status, you just add it to the TERMINAL category --- no campaign reconfiguration required.

### Disposition Code Rules

Follow these rules when setting up your dispositions:

1. **Every human-answered call must get a specific disposition.** Do not let agents use generic codes like NA for calls they actually answered. If an agent spoke to a person, the disposition should reflect the outcome of that conversation.

2. **Mark non-dialable statuses correctly.** Any terminal status (SALE, DNC, DC, WRNUMB, INVNUM) should have Dialable set to N so it is never included in future dial attempts.

3. **Separate "left voicemail" from "answering machine."** The system status A means the agent heard a voicemail greeting. LMVM means they actually left a message. The re-dial strategy for each should differ: A gets recycled aggressively, LMVM gets a longer delay because the prospect may call back.

4. **Create a clear path from disposition to next action.** Every status should have an obvious implication for what happens next. If you cannot explain what the dialer should do with a lead in status X, you do not need status X.

## Time Zone Filtering: The Legal and Performance Implications

Time zone filtering is not optional. Under the TCPA and FTC Telemarketing Sales Rule, marketing calls to consumers must occur between 8:00 AM and 9:00 PM in the **recipient's local time zone**. Violating these windows exposes your operation to per-call statutory damages that can reach $500-$1,500 per violation. A single day of out-of-timezone dialing across a large list can generate six-figure liability.

### How VICIdial Handles Time Zones

VICIdial determines each lead's time zone based on the phone number's area code. The system uses the `gmt_offset_now` column in the `vicidial_list` table, which is populated by mapping the lead's area code to a timezone offset using the `vicidial_phone_codes` table.

For this to work correctly, you must ensure:

1. **The area code table is populated.** Run the NANPA area code population script if you have not already:

```bash
/usr/share/astguiclient/ADMIN_area_code_populate.pl
```

2. **Campaign time zone settings are configured.** In **Admin > Campaigns > [Campaign Name]**, set the **Local Call Time** to define the allowable dialing window. The standard setting is 8:00 AM - 9:00 PM, but some states have stricter windows (e.g., some states restrict calling after 8:00 PM).

3. **GMT offset is calculated correctly.** VICIdial recalculates `gmt_offset_now` based on the area code lookup. For leads with ported numbers (where the area code does not match the actual location), this can produce incorrect timezone assignments. There is no perfect solution for ported numbers, but using NANPA prefix-level timezone encoding (available in newer VICIdial versions) provides higher accuracy than area-code-level lookups alone.

### Performance Implications

Beyond compliance, time zone filtering has direct performance implications:

- **East Coast leads expire first.** If your operation is on the West Coast and starts dialing at 8:00 AM Pacific, your East Coast leads (11:00 AM Eastern) are already past the optimal morning contact window. By 6:00 PM Pacific, your East Coast leads are locked out (9:00 PM Eastern), but you still have three hours of West Coast dialing ahead. Plan your list activation sequence accordingly.

- **Stagger lists by time zone.** Create separate lists for Eastern, Central, Mountain, and Pacific time zones. Activate the Eastern list first each morning, add Central an hour later, and so on. This maximizes your productive dialing window across all zones.

- **Lunchtime and evening patterns.** Contact rates peak at different times in different regions. Generally, 10:00 AM - 12:00 PM and 4:00 PM - 6:00 PM local time produce the highest contact rates for residential calls. Use lead filters or list activation timing to concentrate your dial attempts during these windows.

You can create a lead filter to restrict dialing to specific GMT offsets:

```sql
gmt_offset_now >= -5.00 AND gmt_offset_now <= -5.00
```

This filter would restrict dialing to only Eastern time zone leads. Adjust the offsets as your dialing day progresses.

### State-Level Restrictions

Some states impose calling restrictions beyond the federal 8 AM - 9 PM window. VICIdial supports per-state and per-day-of-week time restrictions through the **Call Times** and **State Call Times** settings. Configure these under **Admin > Call Times** and reference the applicable state regulations for your campaign. States like California, Florida, and New York have specific telemarketing time restrictions that may differ from the federal standard.

> **Time zone violations are the most expensive mistake in outbound dialing.** A single week of misconfigured timezone settings can generate more liability than a year of lead costs. [Request a ViciStack compliance audit](/free-audit/) to verify your time zone filtering is bulletproof.

## The Daily Lead Flow: Hopper Configuration and Refill Strategy

The hopper is VICIdial's staging area for leads about to be dialed. Understanding how it works --- and how to configure it --- is essential for maintaining consistent dial velocity without wasting database resources.

### What the Hopper Does

The VICIdial hopper (`vicidial_hopper` table) is a small, fast-access table that the dialer pulls from when it needs the next number to call. Instead of querying the massive `vicidial_list` table every time an agent becomes available (which would be extremely slow on large databases), the hopper pre-stages a subset of eligible leads.

A background process (`AST_VDhopper.pl`) runs continuously, scanning `vicidial_list` for leads that match the campaign's dial statuses, pass the lead filter, fall within the allowed time zone window, and have `called_since_last_reset` set to N. It loads these leads into the hopper in the order specified by the campaign's lead ordering settings.

### Key Hopper Settings

Configure these settings under **Admin > Campaigns > [Campaign Name]**:

**[Hopper Level](/settings/hopper-level/)**: The target number of leads to maintain in the hopper at all times. Set this based on your agent count and dial rate. The formula for automatic hopper level is:

```
Hopper Level = Agents x Auto Dial Level x (60 / Dial Timeout) x Hopper Multiplier
```

For example, with 20 agents, a [dial level](/settings/auto-dial-level/) of 3, a [dial timeout](/settings/dial-timeout/) of 26 seconds, and a multiplier of 1:

```
20 x 3 x (60 / 26) x 1 = 138 leads in hopper
```

**Automatic Hopper Level**: Set to **Y** to let VICIdial calculate the hopper level dynamically based on the formula above. This is recommended for most operations because it scales with agent count and dial level changes throughout the day.

**Auto Hopper Multiplier**: A modifier for the automatic hopper calculation. Setting this to 1.5 would increase the calculated hopper level by 50%. Increase this if you notice the hopper draining to zero during high-activity periods. Decrease it if you have a small list and want to avoid the hopper loading leads faster than you can dial them.

**Auto Trim Hopper**: Set to **Y** to automatically remove excess leads from the hopper when the lead count exceeds the target. This prevents the hopper from growing unbounded and ensures that leads removed from eligibility (by time zone lockout or filter changes) are purged promptly.

**Lead Order**: This setting controls how leads are sorted when loaded into the hopper:

| Setting | Behavior |
|---|---|
| UP | Oldest leads first (by lead_id) |
| DOWN | Newest leads first |
| UP PHONE | Ascending by phone number |
| DOWN PHONE | Descending by phone number |
| UP COUNT | Lowest called_count first |
| DOWN COUNT | Highest called_count first |
| UP RANK | Ascending by rank field |
| DOWN RANK | Descending by rank field (use positive numbers for highest priority) |
| RANDOM | Random ordering |

**Recommended setting**: **DOWN RANK** for operations that use lead scoring, or **DOWN** (newest first) for operations focused on speed-to-lead. Avoid **UP** unless you have a specific reason to dial your oldest leads first, as this typically produces the worst contact rates.

### Hopper Priority

Within the hopper, certain leads take priority over standard leads:

1. **Scheduled Callbacks** (highest priority) --- Always dialed first
2. **Recycled Leads** --- Leads re-entering the hopper via recycling logic
3. **Auto-Alt-Dial Leads** --- Alternate numbers triggered by prior disposition
4. **Standard Leads** --- Normal hopper entries loaded by AST_VDhopper.pl

You can also assign explicit priority values when inserting leads via the VICIdial Non-Agent API, which is particularly useful for web-form leads that should jump the queue:

```
/vicidial/non_agent_api.php?function=add_lead&phone_number=5551234567&list_id=1001&phone_code=1&rank=99
```

Leads inserted with a high rank value and a campaign using DOWN RANK ordering will be dialed before lower-ranked leads.

### Daily Hopper Management Routine

A healthy daily hopper routine looks like this:

1. **Before shift start**: Verify that active lists have sufficient undailed leads. Check via:

```sql
SELECT list_id, COUNT(*) AS available
FROM vicidial_list
WHERE list_id IN (1001, 1002, 1003)
  AND status IN ('NEW', 'NA', 'B', 'A')
  AND called_since_last_reset = 'N'
GROUP BY list_id;
```

2. **Shift start**: Monitor the campaign's real-time report to confirm the hopper is filling. If the hopper shows 0 leads, check your dial statuses, lead filter, and time zone settings.

3. **Mid-shift**: Watch for hopper depletion patterns. If the hopper is draining faster than it refills, either increase the hopper multiplier or add more lists to the campaign.

4. **End of shift**: Review hopper counts and plan for the next day's list resets.

> **A starved hopper kills your dial rate. An overstuffed hopper wastes database queries.** Getting the balance right requires understanding your specific agent count, dial level, and list size. [Talk to ViciStack about hopper optimization](/free-audit/) for your operation.

## Lead Aging: When to Retire, Suppress, or Reactivate

Not every lead deserves unlimited dial attempts. At some point, continued dialing produces negative returns: wasted agent time, degraded [caller ID reputation](/blog/vicidial-did-management/), and increased spam complaints. Lead aging is the discipline of knowing when a lead has been worked enough and should be moved out of active rotation.

### The Aging Framework

Organize your leads into aging tiers based on a combination of lead age (time since upload) and call attempts:

| Tier | Age | Called Count | Action |
|---|---|---|---|
| Fresh | 0-7 days | 0-8 attempts | Active dialing, full recycling |
| Warm | 8-21 days | 9-15 attempts | Reduced dial frequency, longer recycle delays |
| Aged | 22-45 days | 16-20 attempts | Minimal dialing, 1-2 attempts per week |
| Retired | 46-90 days | 21+ attempts | Removed from active dialing, held for reactivation |
| Expired | 90+ days | Any | Archived or purged, depending on data retention policy |

### Moving Leads Between Tiers

Use MySQL queries to batch-move leads between lists as they age:

```sql
-- Move leads older than 21 days with 15+ attempts to the Aged list
UPDATE vicidial_list
SET list_id = 2003
WHERE list_id IN (2001, 2002)
  AND entry_date < DATE_SUB(NOW(), INTERVAL 21 DAY)
  AND called_count >= 15
  AND status NOT IN ('SALE', 'DNC', 'CALLBK', 'APPT');
```

Schedule these moves as nightly cron jobs so your active lists always contain leads in the appropriate aging tier.

```bash
# Nightly lead aging script - runs at midnight
0 0 * * * mysql -u vicidialuser -p'password' asterisk -e "UPDATE vicidial_list SET list_id = 2003 WHERE list_id IN (2001, 2002) AND entry_date < DATE_SUB(NOW(), INTERVAL 21 DAY) AND called_count >= 15 AND status NOT IN ('SALE', 'DNC', 'CALLBK', 'APPT');"
```

### When to Reactivate

Retired leads are not necessarily dead leads. Circumstances change: a prospect who was not interested in January may have a new need in April. Reactivation campaigns can be surprisingly productive when executed correctly.

Reactivation criteria:

- **Minimum rest period**: At least 30-60 days since last contact attempt
- **Reset status**: Change status back to a dialable status (e.g., a custom REACT status) so you can track reactivated leads separately from fresh leads
- **Different approach**: Use a different caller ID, script, and offer. The prospect already rejected your first approach; repeating it will produce the same result
- **Lower priority**: Reactivated leads should always run behind fresh leads in hopper priority

```sql
-- Reactivate leads that have been resting for 60+ days
UPDATE vicidial_list
SET list_id = 3001, status = 'REACT', called_since_last_reset = 'N'
WHERE list_id = 2004
  AND modify_date < DATE_SUB(NOW(), INTERVAL 60 DAY)
  AND status NOT IN ('SALE', 'DNC', 'DC', 'INVNUM');
```

### Suppression vs. Deletion

Never delete lead records from `vicidial_list` unless required by data retention regulations (GDPR, CCPA, etc.). Instead, suppress leads by:

1. Moving them to an inactive list that is not assigned to any campaign
2. Setting their status to a non-dialable terminal status
3. Keeping the record for historical reporting and deduplication purposes

Deletion creates gaps in your reporting history and removes deduplication protection. A lead you delete today can be uploaded again tomorrow as a "new" lead, wasting money on a prospect you have already exhausted.

### Data Retention Compliance

If your operation falls under GDPR, CCPA, or similar regulations, implement a data retention schedule:

```sql
-- Purge leads older than 2 years (adjust based on your retention policy)
DELETE FROM vicidial_list
WHERE entry_date < DATE_SUB(NOW(), INTERVAL 2 YEAR)
  AND list_id IN (SELECT list_id FROM vicidial_lists WHERE active = 'N');
```

Run this quarterly and log the deletion count for compliance records.

## How ViciStack's List Management Module Works

ViciStack's [List Management module](/features/list-management/) was built specifically to address the gaps in VICIdial's native list management tools. While VICIdial provides the raw infrastructure --- the lead loader, the hopper, the recycling engine --- it leaves the strategic layer entirely to the operator. ViciStack fills that gap with automation, analytics, and guardrails that prevent the most common list management mistakes.

### Automated List Segmentation

ViciStack automatically segments uploaded leads into tiered lists based on configurable rules. When you upload a batch of leads through the ViciStack interface, the system evaluates each lead against your segmentation rules (source, geography, age, prior history) and routes it to the appropriate list. No manual sorting. No spreadsheet gymnastics. Leads land in the right list from the moment they enter the system.

### Intelligent Recycling Engine

ViciStack's recycling engine goes beyond VICIdial's built-in recycling by supporting multi-day cadences, time-of-day variation, and disposition-weighted retry logic. Instead of a flat delay in seconds, ViciStack lets you define recycling rules like "retry NA leads at a different time of day than the original attempt" or "reduce retry frequency after the third failed attempt." The system automatically generates the appropriate list resets and filter adjustments to implement these cadences within VICIdial's architecture.

### Lead Lifecycle Dashboard

The ViciStack dashboard provides real-time visibility into your lead lifecycle: how many leads are in each aging tier, which lists are approaching exhaustion, which sources are producing the highest contact and conversion rates, and where leads are getting stuck in the funnel. This replaces the manual MySQL queries that most operators rely on (or, more commonly, skip entirely).

### DNC and Compliance Automation

ViciStack integrates DNC scrubbing, time zone validation, and attempt-cap enforcement into a single compliance layer. Every lead is validated against federal and state DNC registries before entering the dial queue. Time zone assignments are verified against NANPA prefix data, not just area codes. And attempt caps are enforced automatically, preventing leads from being over-dialed even if an operator forgets to set a lead filter.

### How It Integrates with VICIdial

ViciStack operates alongside your existing VICIdial installation. It does not replace VICIdial's dialer engine or agent interface --- it manages the data layer that feeds into those systems. Lead uploads go through ViciStack's validation and segmentation pipeline, then land in your `vicidial_list` table correctly formatted, deduplicated, and assigned to the right lists. Campaign settings, recycling rules, and lead filters are configured through ViciStack's interface and pushed to VICIdial's database automatically.

> **You do not need to be a MySQL expert to run a well-organized list operation.** ViciStack's List Management module automates the segmentation, recycling, and aging workflows described in this guide. [See it in action with a free audit](/free-audit/).

## Measuring List Performance: The KPIs That Actually Matter

You cannot improve what you do not measure. But most VICIdial operators either track the wrong metrics or track the right metrics at the wrong level of granularity. List performance must be measured per-list, per-source, and per-campaign --- not just as a blended average across your entire operation.

### The Core List KPIs

**1. Contact Rate (per list)**

Contact rate is the percentage of dial attempts that result in a human answer. This is your primary list quality indicator.

```
Contact Rate = (Human Answered Calls / Total Dial Attempts) x 100
```

Measure this per list, not per campaign. A campaign with a 12% blended contact rate might have one list at 20% and another at 4%. The blended number hides the problem --- and the opportunity.

**Benchmarks**: Fresh, opted-in web leads should produce 15-25% contact rates. Purchased aged lists typically fall in the 5-12% range. Anything below 5% suggests serious data quality issues or over-dialed leads.

```sql
SELECT vl.list_id,
       COUNT(vcl.uniqueid) AS total_dials,
       COUNT(CASE WHEN vcl.status IN ('SALE','N','NI','CALLBK','CBHOLD','DNQUAL','WRNUMB','DECMKR','APPT','NOFUND','CMPTN','HUAGNT') THEN 1 END) AS human_answers,
       ROUND(COUNT(CASE WHEN vcl.status IN ('SALE','N','NI','CALLBK','CBHOLD','DNQUAL','WRNUMB','DECMKR','APPT','NOFUND','CMPTN','HUAGNT') THEN 1 END) / COUNT(vcl.uniqueid) * 100, 2) AS contact_rate
FROM vicidial_log vcl
JOIN vicidial_list vl ON vcl.lead_id = vl.lead_id
WHERE vcl.call_date >= DATE_SUB(NOW(), INTERVAL 7 DAY)
GROUP BY vl.list_id
ORDER BY contact_rate DESC;
```

**2. Conversion Rate (per list)**

Conversion rate measures the percentage of contacts (human answers) that result in your desired outcome (sale, appointment, qualified lead).

```
Conversion Rate = (Conversions / Human Answered Calls) x 100
```

This tells you about the quality of the leads themselves, independent of reachability. A list with a high contact rate but low conversion rate contains reachable but unqualified leads. A list with a low contact rate but high conversion rate contains great leads that are hard to reach --- focus on optimizing dial timing and recycling for that list rather than abandoning it.

**Benchmarks**: Outbound B2B campaigns typically see 10-20% conversion on contacted leads. B2C campaigns vary widely by vertical, but 3-8% on contacted leads is common.

**3. Cost Per Contact**

```
Cost Per Contact = Total Lead Cost / Human Answered Calls
```

This is the metric that ties list performance to financial outcomes. If you paid $15 per lead and your contact rate is 10%, your cost per contact is $150. If a different source at $25 per lead delivers a 25% contact rate, the cost per contact is $100. The cheaper lead is actually more expensive.

Track this alongside the [cost per lead benchmarks](/blog/call-center-cost-per-lead-benchmarks/) for your vertical to understand where you stand.

**4. Cost Per Acquisition (CPA)**

```
CPA = Total Lead Cost / Conversions
```

The ultimate financial metric. This rolls up contact rate, conversion rate, and lead cost into a single number that tells you whether a lead source is profitable. Track CPA per list, per source, and per campaign.

**5. Leads Per Agent Hour**

```
Leads Per Agent Hour = Total Leads Dialed / Total Agent Hours
```

This measures operational efficiency at the list level. A list with many disconnected numbers or wrong numbers will produce a low leads-per-agent-hour despite high dial attempts, because agents are spending time on post-call disposition and wrap-up for unproductive calls.

**6. List Exhaustion Rate**

```
Exhaustion Rate = (Leads at Terminal Status / Total Leads in List) x 100
```

Track how quickly each list reaches exhaustion. A list that burns through in 2 days may need to be paced more slowly (lower dial level) or supplemented with fresh leads. A list that is only 30% exhausted after a week may be under-dialed.

```sql
SELECT list_id,
       COUNT(*) AS total_leads,
       COUNT(CASE WHEN status IN ('SALE','DNC','DC','DNQUAL','WRNUMB','INVNUM','CMPTN','APPT') THEN 1 END) AS terminal_leads,
       ROUND(COUNT(CASE WHEN status IN ('SALE','DNC','DC','DNQUAL','WRNUMB','INVNUM','CMPTN','APPT') THEN 1 END) / COUNT(*) * 100, 2) AS exhaustion_pct
FROM vicidial_list
WHERE list_id IN (1001, 1002, 1003)
GROUP BY list_id;
```

### Building a List Performance Dashboard

Pull these KPIs into a daily report that your operations team reviews every morning. The report should answer three questions:

1. **Which lists are performing?** (Highest contact rate and conversion rate)
2. **Which lists are exhausting?** (Approaching terminal status saturation)
3. **Which lists need intervention?** (Low contact rate, high cost per contact, or stalled exhaustion suggesting a filter or timezone issue)

If you are running VICIdial's built-in reporting, the **Outbound Calling Report** and **List Status** reports provide some of this data, but they do not calculate cost metrics or trend over time. Most serious operations export the data to a BI tool or use ViciStack's analytics dashboard for this level of analysis.

> **If you are not measuring list performance at the per-list and per-source level, you are flying blind.** Half of the call centers we audit discover that 20-30% of their lead spend goes to sources that produce below-breakeven results. [Find out where your money is going](/free-audit/) with a free ViciStack audit.

### Putting It All Together

List management is not a one-time setup task. It is a daily operational discipline that separates high-performing call centers from the ones that wonder why their numbers keep declining. The framework laid out in this guide --- structured uploads, intentional segmentation, calibrated recycling, precise dispositions, compliant time zone filtering, optimized hopper settings, disciplined aging, and rigorous KPI tracking --- gives you a complete system for extracting maximum value from every lead that enters your VICIdial instance.

Start with the highest-impact changes: segment your lists by lead age and source, set up proper recycling logic for your top three dispositions, and build a daily per-list KPI report. These three changes alone can lift contact rates by 15-30% and reduce cost per acquisition by 20% or more.

Then work your way through the rest of the framework. Fix your upload validation. Tighten your time zone filtering. Implement lead aging tiers. Each improvement compounds on the ones before it.

Your leads are expensive. Your agents' time is expensive. Your [dialer infrastructure](/blog/vicidial-predictive-dialer-settings/) is expensive. The list management layer is what makes all of those investments pay off --- or not.

Stop burning your best leads.

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-list-management).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
