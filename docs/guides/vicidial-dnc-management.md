# VICIdial DNC List Management: Federal, State & Internal

**Everything you need to know about DNC list management in VICIdial — federal DNC scrubbing, state-level lists, internal DNC tables, real-time hopper filtering, audit trails, and the SQL queries that prove you're compliant when the lawyers come knocking.**

---

Do Not Call compliance is not optional. It is not a "best practice." It is the single most expensive thing you can get wrong in an outbound VICIdial operation.

A single DNC violation under the [TCPA](/glossary/tcpa/) carries $500 in damages. Willful violations jump to $1,500 per call. And plaintiff attorneys don't file lawsuits over one call — they file class actions covering thousands. We've seen operations hit with seven-figure settlements because their DNC scrubbing process had a gap they didn't know about. A list that wasn't re-scrubbed within the 31-day window. A state DNC list they didn't know existed. An internal DNC table that wasn't being checked during [hopper](/glossary/hopper/) loading because someone toggled the wrong setting during a campaign clone.

The DNC landscape in 2026 involves three overlapping layers of compliance: the federal National Do Not Call Registry managed by the FTC, state-level DNC registries maintained by individual states, and your own internal DNC lists built from consumer opt-out requests. VICIdial handles the internal layer natively. The federal and state layers require external scrubbing processes that feed into VICIdial's filtering system.

This guide covers all three layers — the import processes, the VICIdial configuration, the SQL queries for verification, and the audit trail documentation that constitutes your legal defense if you're ever challenged.

For the DNC glossary definition, see [/glossary/dnc/](/glossary/dnc/). For broader TCPA compliance configuration, see our [VICIdial TCPA Compliance Checklist](/blog/vicidial-tcpa-compliance/).

---

## Federal DNC: The National Do Not Call Registry

The FTC's National Do Not Call Registry is the baseline. Every outbound operation calling U.S. consumers must scrub against it. No exceptions (with narrow carve-outs for existing business relationships and certain nonprofit calls that almost certainly don't apply to your VICIdial campaigns).

### Registering for Access

Before you can download the registry data, you must register your organization with the FTC:

1. **Go to** [telemarketing.donotcall.gov](https://telemarketing.donotcall.gov)
2. **Create an organization account.** You'll need your company's legal name, physical address, EIN, and the name of a responsible individual.
3. **Pay the access fee.** As of 2026, fees are based on the number of area codes you download. A single area code costs $75/year. All area codes (the full national file) costs approximately $21,525/year. Most operations download only the area codes they actively dial, which brings the cost down significantly.
4. **Select your area codes.** Choose the area codes that correspond to the states and regions you're calling into. If you're unsure, pull your active area codes from VICIdial:

```sql
SELECT DISTINCT SUBSTRING(phone_number, 1, 3) as area_code,
       COUNT(*) as lead_count
FROM vicidial_list
WHERE list_id IN (SELECT list_id FROM vicidial_lists WHERE active = 'Y')
GROUP BY area_code
ORDER BY lead_count DESC;
```

5. **Download the data.** The FTC provides the DNC data as flat text files — one phone number per line, organized by area code. Files are updated daily, but you're required to download and scrub at minimum every 31 days.

### Import Process

VICIdial does not have a native "import federal DNC" button. The standard workflow is:

**Option A: Pre-scrub before loading leads into VICIdial**

This is the most common approach. Before importing any new lead list into VICIdial, you scrub it against the federal DNC file externally. The scrubbed list (with DNC numbers removed) is then loaded into VICIdial as usual.

```bash
#!/bin/bash
# Federal DNC scrub script
# Removes numbers found in federal DNC files before VICIdial import

DNC_DIR="/opt/dnc/federal"
LEAD_FILE="$1"
OUTPUT_FILE="${LEAD_FILE%.csv}_scrubbed.csv"
REJECT_FILE="${LEAD_FILE%.csv}_dnc_rejected.csv"

# Combine all area code DNC files into one sorted lookup file
cat ${DNC_DIR}/*.txt | sort -u > /tmp/federal_dnc_combined.txt

# Extract phone numbers from lead file (assumes phone is column 1)
# Compare against DNC list, output clean leads and rejected leads
awk -F',' 'NR==FNR{dnc[$1]; next}
{
    phone=$1;
    gsub(/[^0-9]/, "", phone);
    if (phone in dnc) {
        print $0 >> "'${REJECT_FILE}'"
    } else {
        print $0
    }
}' /tmp/federal_dnc_combined.txt "$LEAD_FILE" > "$OUTPUT_FILE"

TOTAL=$(wc -l < "$LEAD_FILE")
CLEAN=$(wc -l < "$OUTPUT_FILE")
REJECTED=$((TOTAL - CLEAN))

echo "Scrub complete: $TOTAL total, $CLEAN clean, $REJECTED DNC matches"
echo "$(date) | $LEAD_FILE | Total: $TOTAL | Clean: $CLEAN | DNC: $REJECTED" \
    >> /var/log/vicidial/dnc_scrub_audit.log
```

**Option B: Load the federal DNC data into VICIdial's internal DNC table**

This approach imports the entire federal DNC registry into VICIdial's `vicidial_dnc` table, allowing VICIdial's native DNC checking to handle federal DNC filtering at hopper-loading time.

```sql
-- Load federal DNC numbers into VICIdial's internal DNC table
-- WARNING: The federal DNC registry contains 240+ million numbers.
-- This will significantly increase the size of the vicidial_dnc table.
-- Test on a staging environment first and monitor MySQL performance.

LOAD DATA LOCAL INFILE '/opt/dnc/federal/combined_dnc.txt'
INTO TABLE vicidial_dnc
FIELDS TERMINATED BY '\t'
LINES TERMINATED BY '\n'
(phone_number);
```

Before taking this approach, check your current DNC table size and MySQL memory allocation:

```sql
SELECT COUNT(*) as current_dnc_count,
       ROUND(data_length / 1024 / 1024, 2) as data_mb,
       ROUND(index_length / 1024 / 1024, 2) as index_mb
FROM information_schema.tables
WHERE table_name = 'vicidial_dnc';
```

If you're loading 240+ million numbers into this table, your `innodb_buffer_pool_size` needs to accommodate the additional index size. For MySQL performance tuning specific to VICIdial, check our [MySQL optimization guide](/blog/vicidial-mysql-optimization/).

**Option C: Use a third-party DNC scrubbing service**

Services like DNC.com, Gryphon Networks, Contact Center Compliance (DCC), and Tele-Data Solutions offer API-based DNC scrubbing. You upload your list (or send numbers via API), they scrub against federal, state, and litigator lists, and return the clean file. This is the easiest approach but adds per-number cost. Most charge $0.002-$0.005 per number per scrub.

### Scrub Frequency: The 31-Day Rule

The TSR (Telemarketing Sales Rule, enforced by the FTC) requires that you scrub your calling lists against the National DNC Registry **no less frequently than every 31 days**. Not 32 days. Not "monthly." Every 31 days, counted from the date of your last scrub.

This means every active list in your VICIdial system needs a scrub date tracked somewhere. Here's a practical approach:

```sql
-- Create a DNC scrub tracking table
CREATE TABLE IF NOT EXISTS vicistack_dnc_scrub_log (
    scrub_id INT AUTO_INCREMENT PRIMARY KEY,
    list_id BIGINT NOT NULL,
    scrub_type ENUM('federal', 'state', 'litigator') NOT NULL,
    scrub_date DATETIME NOT NULL,
    total_numbers INT NOT NULL,
    numbers_removed INT NOT NULL,
    scrub_source VARCHAR(100),
    performed_by VARCHAR(50),
    notes TEXT,
    INDEX idx_list_date (list_id, scrub_date)
);

-- Check which lists are overdue for federal DNC scrub
SELECT
    vl.list_id,
    vl.list_name,
    vl.list_lastcalldate,
    ds.last_scrub,
    DATEDIFF(NOW(), ds.last_scrub) as days_since_scrub,
    CASE WHEN DATEDIFF(NOW(), ds.last_scrub) > 31 THEN 'OVERDUE'
         WHEN DATEDIFF(NOW(), ds.last_scrub) > 25 THEN 'SCRUB SOON'
         ELSE 'CURRENT'
    END as scrub_status
FROM vicidial_lists vl
LEFT JOIN (
    SELECT list_id, MAX(scrub_date) as last_scrub
    FROM vicistack_dnc_scrub_log
    WHERE scrub_type = 'federal'
    GROUP BY list_id
) ds ON vl.list_id = ds.list_id
WHERE vl.active = 'Y'
ORDER BY days_since_scrub DESC;
```

Run this query weekly. Any list showing "OVERDUE" must be either re-scrubbed or deactivated immediately. Dialing from a list whose federal DNC scrub is older than 31 days is a per-call violation.

---

## State DNC Lists: The Layer Most Operations Miss

The federal DNC registry gets all the attention. State DNC lists are what actually trip people up. Over a dozen states maintain their own Do Not Call registries — **separate from and in addition to the federal list.** A number can be on a state DNC list but not on the federal list. If you're calling into that state and you haven't scrubbed against the state list, you're in violation of state law.

### States With Their Own DNC Registries

Here's the current landscape as of 2026. Each state's list must be obtained separately, and each has its own registration process, fee structure, and update schedule:

| State | Registry | Annual Fee (approx.) | Update Frequency | Notes |
|---|---|---|---|---|
| Colorado | Colorado No Call List | $200-$400 | Quarterly | Managed by Attorney General |
| Florida | Florida Do Not Call List | $100 | Quarterly | Must scrub for FTSA compliance |
| Indiana | Indiana Do Not Call List | $75-$300 | Quarterly | Managed by Attorney General |
| Louisiana | Louisiana No Call List | Free-$200 | Quarterly | Managed by PSC |
| Massachusetts | Massachusetts Do Not Call Registry | $100-$300 | Quarterly | Managed by Attorney General |
| Mississippi | Mississippi No Call List | Free | Quarterly | Managed by PSC |
| Missouri | Missouri No Call List | $200 | Quarterly | Managed by Attorney General |
| New York | New York Do Not Call Registry | $200-$500 | Monthly | Managed by DPS |
| Pennsylvania | Pennsylvania Do Not Call List | $100-$400 | Quarterly | Managed by Attorney General |
| Tennessee | Tennessee Do Not Call List | Free-$100 | Quarterly | Managed by TRA |
| Texas | Texas No Call List | $100-$300 | Quarterly | Managed by PUC |
| Wyoming | Wyoming No Call List | Free | Quarterly | Managed by PSC |

**Important:** This list changes. States add and remove DNC registry programs. Check each state's Attorney General or Public Service Commission website annually to verify current requirements. Some states that previously maintained lists have sunset them in favor of relying solely on the federal registry.

### State DNC Import Process

The process for each state follows the same general pattern:

1. **Register with the state agency.** Each state has its own portal. Some allow online registration; others require paper forms.
2. **Pay the annual fee** (if applicable).
3. **Download the data.** Format varies by state — some provide CSV, some provide fixed-width text files, some provide pipe-delimited files.
4. **Normalize the data.** Strip formatting, ensure 10-digit phone numbers, remove duplicates.
5. **Scrub your lists** against the state data before loading into VICIdial (or load the state DNC numbers into a VICIdial DNC list).

Here's a script that handles normalization and import for state DNC files:

```bash
#!/bin/bash
# State DNC import into VICIdial campaign-level DNC list
# Usage: ./import_state_dnc.sh <state_file> <dnc_campaign_id>

STATE_FILE="$1"
DNC_CAMPAIGN="$2"
MYSQL_USER="cron"
MYSQL_PASS="your_password"
MYSQL_DB="asterisk"

# Normalize phone numbers: strip everything except digits, take last 10
awk '{
    gsub(/[^0-9]/, "");
    if (length($0) == 11 && substr($0,1,1) == "1") $0 = substr($0, 2);
    if (length($0) == 10) print $0;
}' "$STATE_FILE" | sort -u > /tmp/state_dnc_normalized.txt

COUNT=$(wc -l < /tmp/state_dnc_normalized.txt)
echo "Normalized $COUNT unique phone numbers from $STATE_FILE"

# Import into VICIdial campaign DNC table
while IFS= read -r phone; do
    mysql -u"$MYSQL_USER" -p"$MYSQL_PASS" "$MYSQL_DB" -e \
        "INSERT IGNORE INTO vicidial_campaign_dnc (phone_number, campaign_id)
         VALUES ('$phone', '$DNC_CAMPAIGN');"
done < /tmp/state_dnc_normalized.txt

echo "State DNC import complete: $COUNT numbers into campaign DNC $DNC_CAMPAIGN"
echo "$(date) | State DNC | File: $STATE_FILE | Campaign: $DNC_CAMPAIGN | Count: $COUNT" \
    >> /var/log/vicidial/dnc_scrub_audit.log
```

For large state files (Florida and Texas can have millions of entries), batch the MySQL inserts instead of one-at-a-time:

```sql
-- Bulk load state DNC into campaign DNC table
LOAD DATA LOCAL INFILE '/tmp/state_dnc_normalized.txt'
IGNORE INTO TABLE vicidial_campaign_dnc
FIELDS TERMINATED BY '\n'
(phone_number)
SET campaign_id = 'STATEDNCFL';
```

Then assign that DNC campaign ID to any campaigns calling into that state.

### Managing Multiple State DNC Lists

If you're calling into multiple states with their own DNC registries, you need a system. Here's the approach that works:

**Option 1: Merge all state DNC numbers into the system-wide internal DNC table.** Simple, effective, but it blocks those numbers across all campaigns — even campaigns where the state restriction doesn't apply (e.g., a number on Indiana's DNC list being blocked from a campaign that only calls Florida).

**Option 2: Create state-specific campaign DNC lists.** More granular. Create DNC list entries tagged by state, and assign the relevant state DNC lists to campaigns based on their target geography. This is more work to maintain but prevents over-blocking.

**Option 3: Pre-scrub externally by state.** Before loading leads into VICIdial, scrub against all applicable state DNC lists for the states represented in the lead file. Only clean leads enter VICIdial. This is the cleanest approach if you have a robust pre-import pipeline.

---

## Internal DNC: VICIdial's Built-In DNC System

VICIdial has two native [DNC](/glossary/dnc/) mechanisms: the system-wide internal DNC list and campaign-level DNC lists. Both are critical. Both need to be enabled.

### System-Wide Internal DNC (vicidial_dnc)

The `vicidial_dnc` table is VICIdial's master DNC list. When enabled on a campaign, any number in this table will be excluded from the [hopper](/glossary/hopper/) — it will never be dialed by any campaign that has internal DNC checking turned on.

**Enable it:** Admin --> Campaigns --> [Campaign] --> Campaign Detail --> **Use Internal DNC List: Y**

There is no reason to ever set this to N on a production campaign. None.

**How numbers get into the internal DNC table:**

1. **Manual entry via admin:** Admin --> DNC Numbers --> Add DNC Number
2. **Disposition processing:** When an agent dispositions a call with a DNC-flagged disposition, the number is automatically added to the internal DNC table (if configured — see DNCL Disposition below)
3. **API insertion:** Using VICIdial's non-agent API:

```
/vicidial/non_agent_api.php?source=dnc&function=add_dnc_phone&user=api_user&pass=api_pass&phone_number=5551234567
```

4. **Direct database insert:**

```sql
INSERT IGNORE INTO vicidial_dnc (phone_number) VALUES ('5551234567');
```

5. **Bulk import via admin:** Admin --> DNC Numbers --> Load DNC Numbers from File

**Verify a number's DNC status:**

```sql
SELECT phone_number
FROM vicidial_dnc
WHERE phone_number = '5551234567';

-- If a row is returned, the number is on the internal DNC.
-- Empty result = not on internal DNC.
```

### Campaign-Level DNC (vicidial_campaign_dnc)

Campaign DNC lists allow you to maintain separate DNC lists for different campaigns. This is useful when a consumer says "don't call me about solar panels" but may still be a valid lead for a different product campaign.

**Enable it:** Admin --> Campaigns --> [Campaign] --> Campaign Detail --> **Use Campaign DNC List: Y**

Campaign DNC entries are stored in the `vicidial_campaign_dnc` table with both the phone number and the campaign ID:

```sql
-- Check if a number is on a specific campaign's DNC list
SELECT phone_number, campaign_id
FROM vicidial_campaign_dnc
WHERE phone_number = '5551234567'
  AND campaign_id = 'SOLARCMP';

-- View all campaign DNC entries for a phone number
SELECT phone_number, campaign_id
FROM vicidial_campaign_dnc
WHERE phone_number = '5551234567';
```

### DNCL Disposition Processing

This is the mechanism that makes real-time DNC processing work. When an agent dispositions a call with a status that has the DNC flag set, VICIdial automatically adds the number to the internal DNC list and/or the campaign DNC list.

**Setting up DNC disposition:**

1. Go to **Admin --> System Statuses** (or Campaign Statuses for campaign-level dispositions)
2. Find or create a status like `DNCL` (Do Not Call List)
3. Set **DNC** to `Y`
4. Set **Human Answered** to `Y` (because DNC requests come from live conversations)

When an agent selects the DNCL disposition:
- The phone number is immediately added to `vicidial_dnc` (system-wide internal DNC)
- If campaign DNC is enabled, it's also added to `vicidial_campaign_dnc` for that campaign
- The lead status is updated to DNCL
- The number is immediately excluded from future hopper loads

**Verify DNCL disposition is configured correctly:**

```sql
-- Check that your DNC disposition has the dnc flag set
SELECT status, status_name, human_answered, dnc
FROM vicidial_statuses
WHERE status = 'DNCL';

-- If dnc = 'Y', you're good. If dnc = 'N' or the status doesn't exist,
-- DNC requests dispositioned as DNCL are NOT being added to the DNC table.
-- This is a compliance gap.
```

**Also check campaign-level statuses:**

```sql
SELECT status, status_name, human_answered, dnc, campaign_id
FROM vicidial_campaign_statuses
WHERE status IN ('DNCL', 'DNC', 'DNCC', 'OPTOUT')
ORDER BY campaign_id;
```

Any status used to indicate a DNC request must have `dnc = 'Y'`. If your agents are using a custom status like `OPTOUT` or `REMOVE` to flag DNC requests but that status doesn't have the DNC flag set, those numbers are not being added to the DNC table. This is one of the most common DNC compliance failures we see — the disposition exists, agents use it, but the flag isn't set so nothing happens in the database.

---

## Real-Time DNC Checking During Hopper Loading

The [hopper](/glossary/hopper/) is VICIdial's queue of phone numbers ready to be dialed. Every time the hopper is refreshed (typically every few seconds for active campaigns), VICIdial runs through a series of checks to determine which leads are eligible for dialing. DNC checking is one of those filters.

### How Hopper DNC Filtering Works

When VICIdial loads the hopper, it executes queries against the lead tables with JOIN or subquery exclusions against the DNC tables. The flow is:

1. **Lead selection:** VICIdial identifies leads in active lists that match the campaign's filter criteria, call time windows, and dial count limits.
2. **Internal DNC check:** If `Use Internal DNC List = Y`, leads whose phone numbers exist in `vicidial_dnc` are excluded.
3. **Campaign DNC check:** If `Use Campaign DNC List = Y`, leads whose phone numbers exist in `vicidial_campaign_dnc` for the current campaign are excluded.
4. **Filter phone check:** VICIdial can also filter against the `vicidial_filter_phone_numbers` table if filter phone groups are configured.
5. **Hopper insertion:** Only leads that pass all checks are inserted into `vicidial_hopper` for dialing.

This means DNC checking happens **before the call is placed**, not during. A number added to the DNC table will be excluded from the hopper on the next refresh cycle — typically within seconds for active campaigns.

**Verify hopper DNC filtering is active:**

```sql
-- Check campaign DNC settings
SELECT campaign_id, campaign_name,
       use_internal_dnc,
       use_campaign_dnc
FROM vicidial_campaigns
WHERE active = 'Y';
```

Every row should show `Y` for `use_internal_dnc`. If any active campaign shows `N`, that campaign is dialing without DNC checking. Fix it immediately.

### Hopper Load Timing and DNC Latency

There's a window between when a number is added to the DNC table and when the hopper excludes it. If a number is already in the hopper when the DNC entry is created, it will remain in the hopper until either:

- It's dialed (and the call goes through — this is a violation)
- The hopper is flushed and rebuilt
- The lead ages out of the hopper

For compliance-critical operations, consider flushing the hopper after bulk DNC imports:

```sql
-- Force hopper refresh for a specific campaign
-- This removes all entries and forces a complete rebuild on next cycle
DELETE FROM vicidial_hopper WHERE campaign_id = 'YOURCAMPAIGN';
```

Don't do this during peak dialing unless you're comfortable with a brief pause in dialing while the hopper repopulates.

### Real-Time DNC Additions via API

For operations that receive DNC requests from external sources (web forms, CRM systems, email opt-outs), the VICIdial API provides real-time DNC insertion:

```bash
# Add a number to the system-wide internal DNC list via API
curl -s "https://your-vicidial-server/vicidial/non_agent_api.php?\
source=compliance&\
function=add_dnc_phone&\
user=api_user&\
pass=api_pass&\
phone_number=5551234567"
```

This takes effect on the next hopper refresh — typically within seconds. For operations processing high volumes of opt-out requests, batch the API calls and verify with a post-import count:

```sql
-- Verify DNC additions from today
SELECT COUNT(*) as dnc_added_today
FROM vicidial_dnc
WHERE phone_number IN (
    -- Replace with today's additions from your opt-out log
    SELECT phone_number FROM your_optout_staging_table
    WHERE processed_date = CURDATE()
);
```

---

## Campaign-Level DNC Settings: The Complete Configuration

Every campaign in VICIdial has DNC-related settings that must be configured correctly. Here's the complete list and what each one does.

### Campaign Detail Settings

**Path:** Admin --> Campaigns --> [Campaign Name] --> Campaign Detail

| Setting | Location | Recommended Value | What It Does |
|---|---|---|---|
| Use Internal DNC List | DNC/Filtering | Y | Checks system-wide DNC table during hopper load |
| Use Campaign DNC List | DNC/Filtering | Y | Checks campaign-specific DNC table during hopper load |
| DNC List ID | DNC/Filtering | (campaign-specific) | Links to specific campaign DNC list for shared DNC across campaigns |
| Use DNC List from Campaign | DNC/Filtering | (optional) | Inherits DNC list from another campaign |

### System-Wide DNC Enforcement

For multi-campaign environments, you need consistent DNC enforcement across the board. A consumer who says "take me off your list" expects to stop receiving calls from your entire organization, not just the one campaign that happened to be dialing them.

**Best practice configuration:**

1. **Internal DNC = Y on every campaign.** No exceptions. This ensures the `vicidial_dnc` table is checked for every hopper load.
2. **DNCL disposition set to add to internal DNC.** This way, any agent-captured DNC request blocks the number across all campaigns.
3. **Campaign DNC = Y where needed.** Use campaign DNC for product-specific opt-outs only — situations where a consumer consents to calls about Product A but opts out of calls about Product B.

**Audit your campaigns for DNC configuration:**

```sql
-- Find campaigns with DNC checking disabled
SELECT campaign_id, campaign_name,
       use_internal_dnc,
       use_campaign_dnc,
       active
FROM vicidial_campaigns
WHERE active = 'Y'
  AND (use_internal_dnc = 'N' OR use_campaign_dnc = 'N')
ORDER BY campaign_id;
```

If this query returns any results, those campaigns have a DNC compliance gap.

### Daily Call Limit Integration

DNC management works hand-in-hand with call frequency limits. A consumer who receives five calls in a single day is far more likely to file a complaint — and in states like Florida, calling the same number more than three times in 24 hours is a per-call violation regardless of DNC status.

Configure [daily call limits](/settings/daily-call-limit/) on all campaigns and use them as a complement to DNC filtering. The daily call limit prevents over-calling numbers that haven't yet opted out. The DNC list prevents any calling of numbers that have opted out.

---

## Wireless and Cell Phone Specific DNC Requirements

Cell phones add another layer of complexity to DNC management. Under the [TCPA](/glossary/tcpa/), calling or texting a cell phone using an automatic telephone dialing system (ATDS) or prerecorded/artificial voice requires **prior express consent** — and for telemarketing calls, **prior express written consent**.

### Cell Phone Identification

VICIdial doesn't natively distinguish between landline and cell phone numbers. You need an external data source:

1. **Phone type append services:** Companies like Neustar, Melissa Data, and TeleSign offer phone line type identification. Append this data to your leads before import.
2. **Create a custom field for phone type:**

```sql
-- Track phone type at the lead level
-- Populate during lead import from your phone type append service
-- L = Landline, C = Cell/Wireless, V = VoIP, U = Unknown
```

3. **Filter based on phone type and consent level:**

If you're running an ATDS (which VICIdial's predictive dialer qualifies as under most state definitions and arguably under the federal TCPA), you need express written [consent](/glossary/consent/) before dialing cell phones for telemarketing purposes. Without that consent, the number should be treated as effectively DNC — even if it's not on any DNC list.

### Reassigned Numbers

The FCC established a Reassigned Numbers Database (RND) to address a chronic problem: phone numbers get reassigned from one consumer to another, but the new consumer never consented to your calls. The previous consumer's consent doesn't transfer to the new number holder.

As of 2026, the RND is operational and provides a safe harbor. If you check the RND and it confirms the number hasn't been reassigned since your consent was obtained, you have an affirmative defense against TCPA claims from the new number holder.

**Integrate RND checking into your lead scrubbing pipeline:**

1. Register for RND access at the FCC's Reassigned Numbers Database portal
2. Query the RND with your phone numbers and the date consent was obtained
3. Flag any numbers that have been reassigned since consent date
4. Remove flagged numbers from your VICIdial lists or add them to the internal DNC

This is an additional scrub step, separate from federal and state DNC scrubbing. It's not technically a "DNC" check, but it serves the same purpose — preventing calls to numbers you don't have valid consent to call.

---

## DNC Audit Trail and Documentation

When a plaintiff attorney sends you a demand letter — or worse, when the FTC opens an investigation — the first thing they ask for is your DNC compliance documentation. Not a verbal explanation. Documentation. Dates. Records. Proof.

### What You Need to Document

**1. Federal DNC scrub records**
- Date of each scrub
- Which lists were scrubbed
- Number of records scrubbed
- Number of DNC matches removed
- Who performed the scrub
- Source of DNC data (FTC download date, third-party service)

**2. State DNC scrub records**
- Same fields as federal, plus state identifier
- Registration/subscription status for each state

**3. Internal DNC additions**
- Every number added to the internal DNC, with timestamp
- Source of addition (agent disposition, API, web opt-out, manual)
- Who or what process added it

**4. DNC removal requests (if any)**
- This is rare, but consumers occasionally ask to be removed from your DNC list so they can receive calls. Document these requests carefully.

### Building the Audit Trail in VICIdial

VICIdial's `vicidial_dnc` table stores phone numbers but **does not store metadata** — no timestamp, no source, no who-added-it. For compliance documentation, you need to augment VICIdial's native DNC tracking.

**Option 1: Create a DNC audit table**

```sql
CREATE TABLE IF NOT EXISTS vicistack_dnc_audit (
    audit_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    phone_number VARCHAR(18) NOT NULL,
    action ENUM('ADD', 'REMOVE') NOT NULL,
    dnc_type ENUM('internal', 'campaign', 'federal_scrub', 'state_scrub') NOT NULL,
    campaign_id VARCHAR(8) DEFAULT NULL,
    source VARCHAR(100) NOT NULL,
    source_detail TEXT,
    performed_by VARCHAR(50) NOT NULL,
    action_timestamp DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_phone (phone_number),
    INDEX idx_timestamp (action_timestamp),
    INDEX idx_source (source)
);
```

**Option 2: Use a MySQL trigger to auto-log DNC additions**

```sql
DELIMITER //

CREATE TRIGGER trg_dnc_audit_insert
AFTER INSERT ON vicidial_dnc
FOR EACH ROW
BEGIN
    INSERT INTO vicistack_dnc_audit
        (phone_number, action, dnc_type, source, performed_by, action_timestamp)
    VALUES
        (NEW.phone_number, 'ADD', 'internal', 'vicidial_auto', 'system', NOW());
END//

CREATE TRIGGER trg_dnc_audit_delete
AFTER DELETE ON vicidial_dnc
FOR EACH ROW
BEGIN
    INSERT INTO vicistack_dnc_audit
        (phone_number, action, dnc_type, source, performed_by, action_timestamp)
    VALUES
        (OLD.phone_number, 'REMOVE', 'internal', 'vicidial_auto', 'system', NOW());
END//

DELIMITER ;
```

Now every DNC addition and removal is logged with a timestamp. When the attorneys ask "when was this number added to your DNC list?" you can answer with a specific date and time.

### Compliance Reports

Generate these reports monthly (or weekly if you're dialing aggressively) and store them in your compliance files:

**DNC Processing Time Report** — How quickly are opt-out requests being processed?

```sql
-- Compare agent DNC dispositions to DNC table additions
-- This identifies delays in DNC processing
SELECT
    vl.call_date as dnc_request_time,
    vl.phone_number,
    vl.campaign_id,
    da.action_timestamp as dnc_added_time,
    TIMESTAMPDIFF(MINUTE, vl.call_date, da.action_timestamp) as processing_minutes
FROM vicidial_log vl
JOIN vicidial_list vlist ON vl.lead_id = vlist.lead_id
LEFT JOIN vicistack_dnc_audit da ON vl.phone_number = da.phone_number
    AND da.action = 'ADD'
    AND da.action_timestamp >= vl.call_date
WHERE vl.status = 'DNCL'
  AND vl.call_date >= DATE_SUB(NOW(), INTERVAL 30 DAY)
ORDER BY processing_minutes DESC;
```

If the DNCL disposition is properly configured with the DNC flag, processing time should be effectively zero (the number is added to the DNC table at disposition time). If you see delays, the DNC flag isn't set on the disposition or there's a database replication lag in a cluster environment.

**DNC Volume Report** — How many DNC entries are you accumulating?

```sql
SELECT
    DATE(action_timestamp) as date,
    dnc_type,
    COUNT(CASE WHEN action = 'ADD' THEN 1 END) as added,
    COUNT(CASE WHEN action = 'REMOVE' THEN 1 END) as removed
FROM vicistack_dnc_audit
WHERE action_timestamp >= DATE_SUB(NOW(), INTERVAL 90 DAY)
GROUP BY DATE(action_timestamp), dnc_type
ORDER BY date DESC;
```

**Calls-to-DNC-Numbers Report** — The nightmare scenario: did you call any number that was already on your DNC list?

```sql
-- Find calls placed to numbers on the internal DNC list
-- These are potential violations
SELECT
    vl.call_date,
    vl.phone_number,
    vl.campaign_id,
    vl.status,
    vl.length_in_sec,
    da.action_timestamp as dnc_added_date,
    da.source as dnc_source
FROM vicidial_log vl
JOIN vicistack_dnc_audit da ON vl.phone_number = da.phone_number
    AND da.action = 'ADD'
    AND da.action_timestamp < vl.call_date
WHERE vl.call_date >= DATE_SUB(NOW(), INTERVAL 30 DAY)
ORDER BY vl.call_date DESC;
```

If this query returns results, you have a compliance breach. Investigate why: Was the number in the hopper before the DNC entry was created? Was internal DNC checking disabled on the campaign? Was the DNC entry added to a campaign DNC but the call came from a different campaign?

> **Not sure if your DNC management is airtight?** We audit VICIdial operations for DNC compliance gaps — the configuration settings, the scrub processes, the audit trails. Most operations we audit have at least one gap they didn't know about. [Request a free compliance audit](/free-audit/) before someone else finds it first.

---

## Building a Complete DNC Compliance Workflow

Individual pieces of DNC management aren't enough. You need a workflow — a repeatable process that covers every DNC layer on a defined schedule.

### Weekly DNC Compliance Workflow

**Monday: Scrub Status Check**

Run the list scrub status query (from the Federal DNC section above). Identify any lists approaching the 31-day scrub deadline. Schedule scrubs for lists due this week.

**Tuesday-Wednesday: Execute Scrubs**

Download fresh federal DNC data (if due). Run scrubs against federal and applicable state DNC lists. Log results in the scrub tracking table. Import any new DNC entries into VICIdial.

**Thursday: Audit DNC Processing**

Run the DNC processing time report. Verify all agent-captured DNC requests were processed within your target window (24 hours recommended). Run the calls-to-DNC-numbers report to identify any violations.

**Friday: Compliance Documentation**

Archive the week's DNC reports. Update the scrub tracking log. Document any issues identified and remediation steps taken. File compliance reports where your compliance officer or attorney can access them.

### Monthly DNC Review

Once per month, run a comprehensive review:

1. **Full campaign audit:** Verify every active campaign has `use_internal_dnc = Y` and `use_campaign_dnc = Y`
2. **Disposition audit:** Verify all DNC-related dispositions have the `dnc = Y` flag set
3. **DNC table integrity:** Compare `vicidial_dnc` count against your expected total (federal + state + internal additions)
4. **State DNC currency:** Verify state DNC subscriptions are active and data is current
5. **FTC registration:** Verify your DNC registry subscription is active and not expired
6. **Process testing:** Have someone submit a test DNC request through each channel (agent call, web form, API) and verify end-to-end processing

```sql
-- Monthly DNC integrity check
SELECT
    'Internal DNC' as dnc_type,
    COUNT(*) as total_entries
FROM vicidial_dnc
UNION ALL
SELECT
    CONCAT('Campaign DNC: ', campaign_id),
    COUNT(*)
FROM vicidial_campaign_dnc
GROUP BY campaign_id
UNION ALL
SELECT
    'Filter Phone Numbers',
    COUNT(*)
FROM vicidial_filter_phone_numbers;
```

---

## FCC Regulations and DNC Enforcement Trends in 2026

The [FCC](/glossary/fcc-regulations/) has been steadily increasing DNC enforcement over the past several years. Understanding the current enforcement climate helps you prioritize where to invest your compliance resources.

### Recent Enforcement Actions

The FCC and FTC have issued record fines in 2024-2026 for DNC violations. Common enforcement patterns include:

**1. Failure to scrub within 31 days.** The FTC treats the 31-day scrub requirement as a bright-line rule. If you can't produce documentation showing a scrub within 31 days of any call, the call is presumed to be a violation.

**2. State DNC list neglect.** State attorneys general have been increasingly active in enforcing their state DNC registries. Florida's Attorney General, in particular, has pursued enforcement actions against out-of-state callers who failed to scrub against the Florida DNC list.

**3. Internal DNC processing failures.** Operations that have an opt-out process on paper but fail to actually implement it in their dialer configuration. The agent takes the DNC request, dispositions the call, but the disposition doesn't have the DNC flag set — so the number stays in rotation.

**4. Post-revocation calling.** Under the 2025 revocation rules, calling a consumer who has revoked consent is both a TCPA violation and a DNC violation. Plaintiff attorneys are filing dual-theory lawsuits that stack both federal and state DNC penalties with TCPA consent penalties.

### Litigation Trends

The DNC litigation landscape in 2026 is dominated by:

- **Serial plaintiffs** who deliberately place their numbers on DNC lists and then wait for calls, filing individual lawsuits or joining class actions
- **Class action attorneys** who use public records (FTC complaints, BBB complaints) to identify operations with DNC compliance problems and recruit class members
- **State AG enforcement** driven by consumer complaint volumes — states track complaints per company and prioritize enforcement against repeat offenders

Your defense against all of these is documentation. The DNC audit trail described in this guide is your evidence that you maintained a compliant DNC process. Without it, you're fighting with empty hands.

> **DNC compliance is not a one-time setup — it's an ongoing operational discipline.** If you're running VICIdial and want an expert review of your DNC management process, from federal scrub schedules to disposition configuration to audit trail integrity, [schedule a free audit](/free-audit/). We'll tell you exactly where your gaps are.

---

## Advanced DNC Configurations

### Filter Phone Groups

VICIdial's Filter Phone Groups feature provides an additional layer of phone number filtering beyond the standard DNC tables. You can create filter lists and apply them to campaigns for specialized blocking:

**Use cases:**
- Litigator phone lists (numbers associated with known TCPA plaintiff attorneys)
- Previously disconnected numbers (to avoid wasted dials)
- Competitor employee numbers
- Company employee numbers (to prevent calling your own staff)

**Path:** Admin --> Filter Phone Groups --> Add New Filter Phone Group

Filter phone groups are checked during hopper loading alongside internal and campaign DNC. They function identically to DNC lists but are managed separately, keeping your DNC tables clean for compliance documentation purposes.

### DNC Across VICIdial Clusters

If you're running a multi-server VICIdial cluster, DNC data must replicate across all servers in real time. VICIdial cluster configurations typically use MySQL replication — but if your DNC tables aren't included in the replication scope, a DNC entry added on Server A won't be checked when Server B loads its hopper.

**Verify DNC replication:**

```sql
-- Run on each slave server
-- Compare counts to the master
SELECT
    (SELECT COUNT(*) FROM vicidial_dnc) as internal_dnc_count,
    (SELECT COUNT(*) FROM vicidial_campaign_dnc) as campaign_dnc_count;
```

If counts differ between master and slave, your replication is broken or the DNC tables are excluded from the replication filter. Fix this before your next dial session.

For VICIdial cluster architecture details, see our [cluster guide](/blog/vicidial-cluster-guide/).

### Handling DNC for Inbound Callbacks

Here's a scenario operations frequently overlook: a consumer is on your DNC list but calls your inbound line and requests a callback. Can you call them back?

The answer depends on the source of the DNC entry:

- **Internal DNC (consumer opted out of your calls):** The consumer calling you back may constitute revocation of their opt-out — but document it. Record the inbound call. Note that the consumer initiated contact and requested a callback. Make the callback within a reasonable time (24-48 hours) and only regarding the topic the consumer called about.
- **Federal/state DNC registry:** A consumer being on the federal or state DNC list doesn't prevent you from returning their call. The DNC registry restricts unsolicited telemarketing calls. If the consumer initiates contact and requests a callback, you have an established business relationship exception. Document the inbound call and callback thoroughly.
- **Best practice:** Remove the number from your internal DNC for the specific callback, make the call, and re-add if the consumer doesn't explicitly consent to future calls. Log every step.

---

## Frequently Asked Questions

### How often do I need to scrub my VICIdial lists against the federal DNC registry?

Every 31 days, measured from the date of your last scrub for each active list. This is a hard rule under the Telemarketing Sales Rule (TSR). There's no grace period and no "close enough." If you scrubbed on March 1st and make a call on April 2nd without re-scrubbing, every call to a number that was added to the federal DNC between March 1st and April 2nd is a potential violation. Set calendar reminders. Build automated tracking (use the scrub tracking table and query from this guide). Some operations scrub every 14 days to build in a safety margin — if something delays a scrub by a week, you're still within the 31-day window.

### Can I load the entire federal DNC registry into VICIdial's internal DNC table?

You can, but think carefully before doing it. The federal DNC registry contains 240+ million phone numbers. Loading that into VICIdial's `vicidial_dnc` table will work — MySQL can handle it — but it will significantly increase your database size and may impact hopper loading performance if your `innodb_buffer_pool_size` isn't sized accordingly. You'll need at least 4-6 GB of additional buffer pool allocation. The bigger issue is maintenance: you'll need to diff each new DNC download against your existing table to add new numbers and (theoretically) remove expired ones. Most operations find it cleaner to pre-scrub externally and only load internal opt-outs and state DNC numbers into VICIdial's DNC tables.

### What happens if an agent forgets to disposition a call as DNCL when the customer requests it?

The number stays in rotation and gets dialed again — which is a DNC violation. This is a training issue, not a technical one. Train agents that DNC requests are non-negotiable: if a consumer says "take me off your list," "stop calling me," "don't call again," or any variation, the call gets a DNCL disposition. Period. No trying to save the lead. No transferring to a manager first. Disposition it DNCL, and move on. Additionally, QA your call recordings. Run a weekly sample of calls to verify agents are correctly identifying and dispositioning DNC requests. Any missed DNC request is a compliance exposure that compounds every time the number is redialed.

### How do I handle DNC requests that come in through email or web forms (not during a call)?

These must be processed into your VICIdial DNC tables within a reasonable time — the FCC's 2025 revocation rules make this explicit. The consumer can revoke consent through **any reasonable means**, including email, web forms, social media messages, or carrier complaints. Build an intake process: set up an email alias (like optout@yourcompany.com) and a web form that feeds DNC requests into a queue. Process that queue daily (or more frequently) using VICIdial's API to add numbers to the internal DNC table. Log every request with its source, timestamp, and processing timestamp in your DNC audit table. The entire pipeline from request to DNC table entry should take no more than 24 hours.

### Do I need to maintain DNC records for numbers that I've never called?

If a number is on the federal or state DNC registry, you don't need to maintain your own record of it — that's the registry's job. But for your internal DNC list (numbers that opted out of your calls specifically), yes — maintain those records indefinitely, or at minimum for 5 years after the last contact attempt. Virginia requires 10-year retention for DNC opt-outs. The DNC audit table described in this guide provides this record. Never purge your internal DNC table unless you have a documented, legally reviewed reason to do so. The cost of storing a few million phone numbers in a MySQL table is trivial compared to the cost of re-calling a number that was previously opted out.

### What's the difference between VICIdial's internal DNC and campaign DNC?

The internal DNC (`vicidial_dnc` table) is system-wide — a number on this list is blocked across **all** campaigns that have `Use Internal DNC List = Y`. The campaign DNC (`vicidial_campaign_dnc` table) is campaign-specific — a number on this list is only blocked for the specific campaign it's associated with. Use internal DNC for consumers who say "never call me again" (total opt-out). Use campaign DNC for consumers who say "don't call me about this" but may be valid leads for other products/campaigns. When in doubt, add to the internal (system-wide) DNC. It's always safer to over-block than to under-block.

### How can I verify that VICIdial is actually checking the DNC list before dialing?

Three verification methods. First, add a known test number to your internal DNC table, load it into an active list, and confirm it never appears in the hopper: `SELECT vh.* FROM vicidial_hopper vh JOIN vicidial_list vl ON vh.lead_id = vl.lead_id WHERE vl.phone_number = '5551234567';` — this should return empty. Second, check the campaign settings: `SELECT use_internal_dnc, use_campaign_dnc FROM vicidial_campaigns WHERE campaign_id = 'YOURCAMPAIGN';` — both should be `Y`. Third, check the hopper loading logs. VICIdial logs hopper activity in `vicidial_admin_log` and you can watch the process in real-time by tailing `/var/log/astguiclient/AST_VDhopper.log` or the equivalent log file on your system. If DNC checking is active, you'll see DNC-matched numbers being skipped during hopper loads.

### What should I do if I discover we've been calling DNC numbers?

Stop the affected campaigns immediately. Run the calls-to-DNC-numbers query from this guide to identify the scope — how many calls, over what period, to how many unique numbers. Identify the root cause: Was DNC checking disabled? Was the scrub overdue? Was a disposition misconfigured? Document everything. Fix the root cause. Then consult with a TCPA attorney before resuming dialing. If the violation count is small and you can demonstrate a good-faith compliance program with a specific identifiable failure point, your legal exposure is lower than if you have no compliance process at all. The audit trail and documentation practices described in this guide exist specifically for this scenario — they're your evidence that you were trying to comply and experienced a specific, correctable failure.

---

DNC management isn't glamorous. It doesn't make your agents faster or your conversion rates higher. But it's the compliance discipline that keeps your operation running. A six-figure DNC settlement doesn't just hurt financially — it can trigger carrier blocks, reputation damage, and regulatory scrutiny that compounds for years.

The configuration is straightforward. The scrubbing processes are mechanical. The audit trail is just logging. None of this is technically difficult. The operations that get burned are the ones that treated DNC as a checkbox instead of a process — set it once, never verified it was working, and found out the hard way when the demand letter arrived.

Set up the scrub schedule. Configure the dispositions. Build the audit trail. Verify it monthly. That's the entire playbook.

For the complete TCPA compliance picture beyond DNC management, see our [VICIdial TCPA Compliance Checklist](/blog/vicidial-tcpa-compliance/). For protecting your outbound caller IDs from spam flags, see our [STIR/SHAKEN implementation guide](/blog/stir-shaken-vicidial-guide/). And for the lead list hygiene practices that keep your DNC ratios manageable, see our [list management best practices](/blog/vicidial-list-management/) guide.

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-dnc-management).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
