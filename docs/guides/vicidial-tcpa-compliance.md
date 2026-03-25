# VICIdial TCPA Compliance Checklist [2026]

**Every VICIdial setting that affects TCPA compliance, every state mini-TCPA you need to know about, and the exact configuration checklist that keeps your operation on the right side of a $1,500-per-violation penalty.**

---

The Telephone Consumer Protection Act was passed in 1991. It's been amended, reinterpreted, and weaponized by plaintiff attorneys for 35 years. And in 2025-2026, the FCC rolled out the most significant TCPA regulatory changes in a decade — changes that specifically target outbound call centers running predictive dialers on platforms exactly like VICIdial.

If you're running a VICIdial-based outbound operation in 2026 and you haven't updated your compliance configuration since the FCC's new rules took effect, you are exposed. Not "might be" exposed. Are exposed.

This guide covers every aspect of TCPA compliance as it applies to VICIdial: the 2026 regulatory landscape, every relevant VICIdial configuration setting with the exact admin path, federal DNC and internal DNC management, the 3% abandon rate rule, time zone enforcement, state mini-TCPA requirements, consent tracking, and a printable checklist you can hand to your compliance officer.

For the TCPA glossary definition, see [/glossary/tcpa/](/glossary/tcpa/).

---

## The 2026 TCPA Regulatory Landscape

Before touching VICIdial settings, you need to understand what changed. Three major FCC rulings in 2024-2025 redefined the compliance landscape for outbound dialers:

### 1. One-to-One Consent Rule (Effective January 27, 2025)

**What changed:** Prior to this rule, a single consumer consent could authorize calls from multiple companies — the classic "lead gen" model where a consumer fills out a form and their information is sold to 5-10 companies, all claiming consent from the same form submission.

The new rule requires that consent to receive autodialed or prerecorded calls must be given to **one specific seller at a time**. Consent obtained through "comparison shopping" forms that authorize multiple sellers from a single checkbox is no longer valid under the TCPA.

**What this means for VICIdial operators:**
- If you buy leads from aggregators, each lead must include proof that the consumer specifically consented to calls from **your company** — not "our partners" or "participating companies"
- The consent language shown to the consumer must clearly identify your company by name
- Consent records must be retrievable — you need to produce the specific form, timestamp, and consent language for any number you dial if challenged

### 2. Revocation Rules (Effective April 11, 2025)

**What changed:** Consumers can now revoke consent to receive calls through **any reasonable means** — including replying "stop" to a text, telling an agent on the phone, emailing, submitting a web form, or any other method the consumer chooses. The FCC explicitly stated that companies cannot limit revocation to a single specific channel.

More critically: once a consumer revokes consent, you have **a reasonable time** to process the revocation — which the FCC has indicated means no more than **10 business days**. Any calls made after a reasonable processing period are violations.

**What this means for VICIdial operators:**
- Your agents must be trained to process opt-out requests on every call, regardless of how the consumer phrases it
- "Take me off your list," "don't call me again," "stop calling" — all of these constitute valid revocation
- You need a centralized opt-out processing system that feeds into VICIdial's [DNC lists](/glossary/dnc/) within 10 business days (sooner is better — most plaintiff attorneys argue for 24-48 hours as "reasonable")
- Text-based opt-outs (if you're running SMS campaigns alongside VICIdial) must also be processed into your dialing DNC lists

### 3. STIR/SHAKEN Enforcement Expansion

The FCC continued expanding STIR/SHAKEN requirements through 2025, with additional robocall mitigation program requirements for intermediate and smaller carriers. For VICIdial operators, this means:

- Your SIP provider **must** support STIR/SHAKEN with A-level attestation for your outbound DIDs
- Calls without proper attestation are increasingly being blocked at the carrier level (not just labeled as spam — blocked outright)
- See our [complete STIR/SHAKEN implementation guide](/blog/stir-shaken-vicidial-guide/) for the VICIdial-specific configuration

---

## VICIdial Settings That Affect TCPA Compliance: The Complete List

Here's every VICIdial setting that has TCPA implications, organized by where you find it in the admin interface.

### Campaign Detail Settings

**Path: Admin → Campaigns → [Campaign Name] → Campaign Detail**

#### Drop Percentage / Abandon Rate

- **Setting:** `Adaptive Dropped Percentage`
- **Location:** Campaign Detail → Dialing section
- **Compliant value:** 2.5% or lower
- **Why:** The TCPA (specifically the FCC's 2008 Declaratory Ruling and TSR Section 310.4(b)(4)(i)) requires that no more than 3% of answered calls be abandoned, measured over a 30-day period per campaign. Setting your adaptive dropped percentage at 2.5% gives you a safety margin for statistical variance. Running at exactly 3.0% means any spike pushes you over.

**Important nuance:** The 3% is measured against **answered calls**, not total dials. VICIdial's `Adaptive Dropped Percentage` setting controls the target — the algorithm adjusts dial levels to keep abandoned calls at or below this percentage. But the setting is a target, not a guarantee. During sudden shifts in answer rates (e.g., a batch of numbers from a newly activated list segment), actual abandons can momentarily spike above the target.

To verify compliance, run this query against your VICIdial database:

```sql
SELECT
    campaign_id,
    COUNT(CASE WHEN status = 'DROP' THEN 1 END) as drops,
    COUNT(CASE WHEN status NOT IN ('NA','B','DC','N','AFTHRS') THEN 1 END) as answered,
    ROUND(COUNT(CASE WHEN status = 'DROP' THEN 1 END) /
          NULLIF(COUNT(CASE WHEN status NOT IN ('NA','B','DC','N','AFTHRS') THEN 1 END), 0) * 100, 2)
          as drop_pct
FROM vicidial_log
WHERE call_date >= DATE_SUB(NOW(), INTERVAL 30 DAY)
GROUP BY campaign_id;
```

If any campaign shows above 3.0%, you have a compliance problem that needs immediate attention.

#### Safe Harbor Message

- **Setting:** `Safe Harbor Audio`
- **Location:** Campaign Detail → Dialing section → Safe Harbor settings
- **Compliant value:** ENABLED with a compliant audio file
- **Why:** When a call is abandoned (agent not available after the consumer answers), the TCPA requires that a prerecorded message play within 2 seconds identifying the caller with: company name, phone number the consumer can call to request no further calls, and a statement that the call was made for telemarketing purposes. This is the "safe harbor" provision — if you play the message, the abandoned call may be excused from the 3% calculation under certain interpretations.

Configure in VICIdial:
```
Safe Harbor Audio: your_safe_harbor_message.wav
Safe Harbor Message on Drop: Y
Safe Harbor Exten: 8320 (default)
```

Your safe harbor audio should say something like: *"We apologize for the inconvenience. This call was made by [Company Name] regarding [product/service]. If you wish to be placed on our do not call list, please call [phone number]. Thank you."*

Record this as a WAV file (8kHz, 16-bit, mono) and upload to the VICIdial audio store, or place the file in `/var/lib/asterisk/sounds/`.

#### Call Time Settings

- **Setting:** `Local Call Time`
- **Location:** Campaign Detail → Campaign Detail → Local Call Time
- **Compliant value:** Must conform to TCPA hours (8 AM - 9 PM local time of the called party)
- **Why:** The TCPA prohibits telemarketing calls before 8:00 AM or after 9:00 PM in the **called party's local time zone**. Not your time zone. Their time zone. Many state mini-TCPAs have stricter hours (see State Requirements section below).

Create a compliant Call Time under **Admin → Call Times:**

```
Monday-Friday: 8:00 AM - 9:00 PM (or stricter per state rules)
Saturday: 8:00 AM - 9:00 PM (check state restrictions)
Sunday: Check state restrictions (some prohibit Sunday calling)
```

**Critical:** Assign this Call Time as the `Local Call Time` on the campaign, not just the `Call Time`. The `Local Call Time` setting enforces time zone-aware dialing — VICIdial checks the lead's time zone (derived from area code or the `gmt_offset_now` field) before loading it into the hopper.

### Time Zone Enforcement

- **Setting:** `Hopper GMT Settings`
- **Location:** Campaign Detail → Hopper section
- **Compliant value:** ENABLED with proper configuration

VICIdial's hopper system handles time zone enforcement. When loading leads into the hopper (the queue of numbers ready to be dialed), VICIdial checks each lead's GMT offset against the campaign's Local Call Time to determine if the number is currently within legal calling hours.

For this to work correctly:

1. **Phone timezone data must be populated.** VICIdial populates timezone data based on area code when leads are loaded. Verify the `vicidial_phone_codes` table is current:

```sql
SELECT country_code, areacode, state, GMT_offset, DST
FROM vicidial_phone_codes
WHERE country_code = '1'
LIMIT 10;
```

2. **Enable timezone checking on the campaign:**
   - **Use Campaign Timezone:** Set to the timezone checking method that matches your compliance needs. The safest option is `TZCODE` which uses the lead's postal code for precise timezone determination (more accurate than area code for ported numbers).

3. **Account for ported numbers.** A consumer in Florida (Eastern) may have a 213 (Los Angeles, Pacific) cell phone number. Area code-based timezone lookup would assume Pacific time. For maximum compliance, use postal code-based timezone lookup when you have the lead's ZIP code, or default to the **more restrictive** interpretation (if unsure between two timezones, use the earlier calling start time).

### DNC List Management

- **Setting:** `Use Internal DNC List` and `Use Campaign DNC List`
- **Location:** Campaign Detail → DNC/Filtering section
- **Compliant value:** Both ENABLED

VICIdial supports multiple layers of DNC checking:

**1. Internal DNC List (System-wide)**

The system-wide internal DNC list applies across all campaigns. Numbers added here will not be dialed by any campaign.

- **Location:** Admin → DNC Lists → Internal DNC
- **Adding numbers:** Admin interface, API (`add_dnc_phone`), or direct database insert to `vicidial_dnc` table
- **Checking:** VICIdial checks this list during hopper loading. Numbers matching the internal DNC are excluded from the hopper.

**2. Campaign-specific DNC Lists**

Each campaign can have its own DNC list for campaign-level opt-outs (consumers who asked not to be called about this specific product/campaign, but may have consented to other campaigns).

- **Location:** Campaign Detail → DNC List ID
- **Adding numbers:** Via dispositions (DNC disposition auto-adds to campaign DNC), API, or admin

**3. National Do Not Call Registry**

The FTC's National DNC Registry is **separate** from VICIdial's internal DNC. You must:
- Download the DNC registry data (requires registration at telemarketing.donotcall.gov)
- Scrub your lead lists against the national DNC **before** loading them into VICIdial
- Re-scrub every 31 days (the maximum interval allowed by law)

VICIdial does not natively integrate with the national DNC registry in real-time. The standard practice is to scrub lists externally before import, or to load the national DNC data into VICIdial's internal DNC table (this table can hold millions of entries but monitor MySQL performance).

**4. State Do Not Call Lists**

Several states maintain their own DNC registries separate from the national list. If you're calling into these states, you must obtain and scrub against their state lists:

- Indiana, Colorado, Florida, Louisiana, Mississippi, Missouri, Pennsylvania, Texas, Wyoming, and others
- Each state has its own registration and data access process
- Annual fees range from free to several hundred dollars

**DNC compliance checklist for VICIdial:**

- [ ] Internal DNC list enabled on all campaigns
- [ ] Campaign DNC lists enabled where appropriate
- [ ] National DNC scrub performed within last 31 days
- [ ] State DNC scrubs current for all target states
- [ ] DNC disposition auto-adds to internal DNC list
- [ ] Opt-out processing workflow established (under 10 business days)
- [ ] DNC list import tested and verified
- [ ] Regular DNC audit (monthly) to verify list integrity

---

## The 3% Abandon Rate Rule: Configuration and Monitoring

The TCPA's abandon rate rule is one of the most commonly violated provisions — often unintentionally. Here's exactly how to configure and monitor it in VICIdial.

### How VICIdial Calculates Drop Rate

VICIdial tracks drops (abandoned calls) in the `vicidial_log` table with a status of `DROP`. The drop rate calculation is:

```
Drop Rate = (DROP calls) / (Total answered calls) × 100
```

"Answered calls" are all calls where the consumer picked up — including calls routed to agents, drops, and AMD-classified calls (if AMD is enabled). Calls that rang with no answer (NA), busy (B), or disconnected (DC) are excluded from both numerator and denominator.

### VICIdial Settings for 3% Compliance

| Setting | Location | Value | Purpose |
|---|---|---|---|
| Adaptive Dropped Percentage | Campaign Detail → Dialing | 2.5% | Target drop rate (margin below 3%) |
| Drop Action | Campaign Detail → Dialing | AUDIO (safe harbor message) | What happens to dropped calls |
| Safe Harbor Audio | Campaign Detail → Dialing | [your message file] | Plays compliance message |
| Force Drop Timer | Campaign Detail → Dialing | 5 seconds | Max ring time before drop disposition |
| Drop Percentage Max | Campaign Detail → Dialing | 3.0% | Hard ceiling — dialer pauses if hit |

**The `Drop Percentage Max` setting** is your emergency brake. When the actual drop percentage hits this ceiling, VICIdial's adaptive algorithm reduces dial levels to prevent further drops. Set this at 3.0% (the legal maximum). Your `Adaptive Dropped Percentage` target should be set lower (2.0-2.5%) so the emergency brake rarely triggers.

### Monitoring Drop Rate in Real Time

VICIdial's real-time reports show current drop rates. But for compliance, you need the 30-day rolling calculation. Run this query weekly (or daily if you're paranoid, which you should be):

```sql
SELECT
    campaign_id,
    DATE(call_date) as call_day,
    COUNT(CASE WHEN status = 'DROP' THEN 1 END) as daily_drops,
    COUNT(CASE WHEN status NOT IN ('NA','B','DC','N','AFTHRS','XDROP') THEN 1 END) as daily_answered,
    ROUND(COUNT(CASE WHEN status = 'DROP' THEN 1 END) /
          NULLIF(COUNT(CASE WHEN status NOT IN ('NA','B','DC','N','AFTHRS','XDROP') THEN 1 END), 0) * 100, 2)
          as daily_drop_pct
FROM vicidial_log
WHERE call_date >= DATE_SUB(NOW(), INTERVAL 30 DAY)
GROUP BY campaign_id, DATE(call_date)
ORDER BY campaign_id, call_day;
```

And the 30-day aggregate:

```sql
SELECT
    campaign_id,
    COUNT(CASE WHEN status = 'DROP' THEN 1 END) as total_drops_30d,
    COUNT(CASE WHEN status NOT IN ('NA','B','DC','N','AFTHRS','XDROP') THEN 1 END) as total_answered_30d,
    ROUND(COUNT(CASE WHEN status = 'DROP' THEN 1 END) /
          NULLIF(COUNT(CASE WHEN status NOT IN ('NA','B','DC','N','AFTHRS','XDROP') THEN 1 END), 0) * 100, 2)
          as rolling_30d_drop_pct
FROM vicidial_log
WHERE call_date >= DATE_SUB(NOW(), INTERVAL 30 DAY)
GROUP BY campaign_id;
```

**If your rolling 30-day drop rate exceeds 3.0% on any campaign**, take immediate action:
1. Reduce the campaign's `Adaptive Dropped Percentage` to 1.5%
2. Lower the `Adaptive Maximum Dial Level` to 1.5
3. Add agents to the campaign to absorb more connects
4. Investigate whether a specific date or time period caused the spike
5. Document your remediation steps (this matters if you're ever audited)

---

## State Mini-TCPA Requirements

The federal TCPA sets the floor. State laws often go further. Here's every major state mini-TCPA that affects VICIdial configuration:

### Florida — Florida Telephone Solicitation Act (FTSA)

The FTSA is the most aggressive state telemarketing law in the country. Updated in 2021, it creates a private right of action (consumers can sue you directly) with significant per-violation damages.

**Key requirements:**
- **Calling hours:** 8:00 AM - 8:00 PM local time (one hour shorter than federal)
- **Call frequency:** Maximum **3 calls per 24-hour period** to the same residential telephone number
- **Do not call:** Must scrub against Florida's state DNC list ($100/year access)
- **Written consent:** Required for automated calls to cell phones (higher bar than federal)
- **Damages:** $500 per violation, $1,500 for willful violations — with a private right of action

**VICIdial configuration for Florida:**

1. Create a Florida-specific Call Time: `8:00 AM - 8:00 PM ET`
2. Set per-number daily attempt limits using VICIdial's dial count features — configure [Auto Alt Dial Limit](/settings/auto-alt-dial/) appropriately and use filter settings to cap Florida area codes at 3 attempts per 24 hours
3. Maintain Florida state DNC scrub (updated quarterly)
4. Store written consent records for all Florida cell phone numbers

**Florida area codes to flag:** 239, 305, 321, 352, 386, 407, 561, 727, 754, 772, 786, 813, 850, 863, 904, 941, 954

### Texas — SB 140 and Texas Telemarketing Disclosure Act

**Key requirements:**
- **Registration required:** Must register with the Texas Secretary of State before conducting telemarketing in Texas
- **Fee:** Registration fee applies
- **Penalties:** Up to $10,000 per violation
- **Calling hours:** 8:00 AM - 9:00 PM local time (matches federal)

**VICIdial configuration for Texas:**
- Ensure company registration is current before loading Texas leads
- No special VICIdial settings needed beyond standard federal compliance
- Maintain proof of registration in compliance files

### Virginia — Virginia Telephone Privacy Protection Act

**Key requirements:**
- **10-year opt-out retention:** Once a Virginia consumer requests no further calls, you must maintain that opt-out for **10 years** — one of the longest retention requirements in the country
- **DNC processing:** Must process opt-out within 30 days (federal is technically 31 days, but Virginia courts have been stricter)

**VICIdial configuration for Virginia:**
- Never purge DNC entries for Virginia numbers (434, 540, 571, 703, 757, 804)
- Implement a DNC retention policy that prevents accidental deletion
- Consider a separate DNC list or table with a `date_added` field for Virginia entries

### Oklahoma — Oklahoma Telephone Solicitation Act (OTSA)

**Key requirements:**
- **Registration:** Must register as a telephone solicitor with the Oklahoma Attorney General
- **Calling hours:** 8:00 AM - 9:00 PM local time
- **Written confirmation:** Must send written confirmation of any purchase within 3 business days (if applicable)
- **Bond requirement:** $50,000 surety bond in some cases

**VICIdial configuration:**
- Verify registration before dialing Oklahoma (405, 539, 580, 918)
- Standard call time compliance

### Indiana — Indiana Telephone Solicitation Act

**Key requirements:**
- **Registration:** Must register with Indiana Attorney General
- **Do not call:** Must scrub against Indiana state DNC list
- **Calling hours:** 8:00 AM - 9:00 PM local time (matches federal)
- **Auto-dial:** Specific restrictions on automated dialing to cell phones

**VICIdial configuration:**
- Maintain Indiana state DNC scrub
- Verify registration current

### California — Multiple Overlapping Laws

California combines the CCPA (data privacy), state telemarketing laws, and aggressive enforcement:

**Key requirements:**
- **CCPA:** Consumers have the right to know what personal data you've collected, the right to delete it, and the right to opt out of data sales — this applies to lead data used in VICIdial campaigns
- **Opt-out processing:** 15 business days (CCPA) — implement with VICIdial internal DNC
- **Calling hours:** No state-specific override (federal 8 AM - 9 PM applies)
- **Cell phone consent:** Written consent required for automated calls to cell phones (California Civil Code § 1770)
- **Private right of action:** Yes, for both TCPA and CCPA violations

**VICIdial configuration for California:**
- Maintain CCPA-compliant data handling processes
- Process DNC requests within 15 business days
- Store consent records for all California cell phone leads (213, 310, 323, 408, 415, 424, 442, 510, 530, 559, 562, 619, 626, 628, 650, 657, 661, 669, 707, 714, 747, 760, 805, 818, 831, 858, 909, 916, 925, 949, 951)

### Washington — Washington Automatic Dialing and Announcing Device Act

**Key requirements:**
- **Prior consent:** Required for any automated call or prerecorded message
- **Calling hours:** 8:00 AM - 8:00 PM local time (same as Florida — one hour shorter than federal)
- **Identification:** Must identify the caller and provide a phone number within the first 30 seconds

**VICIdial configuration:**
- Create Washington-specific Call Time: `8:00 AM - 8:00 PM PT`
- Ensure agent scripts include company identification within 30 seconds

---

## Consent Tracking in VICIdial

The one-to-one consent rule makes consent tracking a compliance requirement, not a best practice. Here's how to implement it in VICIdial.

### Custom Fields for Consent Data

Create custom fields on your VICIdial lead records to store consent information:

```sql
-- Add consent tracking fields to vicidial_lists_fields
-- (or configure via Admin → Custom Fields)

consent_timestamp    -- When consent was obtained
consent_source       -- Where consent was obtained (URL, form name)
consent_language     -- The specific consent text shown to consumer
consent_company      -- Your company name as displayed in consent
consent_ip           -- Consumer's IP address at time of consent
consent_method       -- How consent was given (checkbox, signature, verbal)
```

### Lead Loading with Consent Validation

When loading leads via VICIdial's API or list import, validate that consent fields are populated before allowing the lead to enter the hopper:

**API lead loading with consent fields:**
```
/vicidial/non_agent_api.php?
source=lead_import&
function=add_lead&
user=api_user&
pass=api_pass&
phone_number=5551234567&
first_name=John&
last_name=Smith&
list_id=10001&
consent_timestamp=2026-03-15T14:30:00Z&
consent_source=solar-quote-form-v3&
consent_company=ABC+Solar+Company&
custom_fields=Y
```

Implement a pre-hopper validation that checks for the presence of consent data. This can be done via a custom lead filter that excludes leads without consent fields populated.

### Consent Record Retrieval

When a consumer (or their attorney) requests proof of consent, you need to pull the complete record. Build this into your compliance workflow:

```sql
SELECT
    l.phone_number,
    l.first_name,
    l.last_name,
    cf.consent_timestamp,
    cf.consent_source,
    cf.consent_language,
    cf.consent_company,
    cf.consent_ip,
    cf.consent_method
FROM vicidial_list l
JOIN custom_[list_id] cf ON l.lead_id = cf.lead_id
WHERE l.phone_number = '5551234567';
```

Store consent records indefinitely — or at minimum for 5 years after the last contact with the consumer. TCPA's statute of limitations is 4 years, but state laws vary.

---

## The Complete VICIdial TCPA Compliance Checklist

Print this. Tape it to the wall. Review it monthly.

### Federal TCPA Compliance

- [ ] **Abandon rate** below 3% (30-day rolling) — `Adaptive Dropped Percentage` set to 2.5% or lower
- [ ] **Safe harbor message** plays on all abandoned calls — `Safe Harbor Audio` configured with compliant message
- [ ] **Calling hours** enforced: 8 AM - 9 PM called party's local time — `Local Call Time` configured with timezone checking
- [ ] **National DNC** scrub current within last 31 days — external scrub before lead import
- [ ] **Internal DNC** list enabled on all campaigns — both system and campaign DNC checked
- [ ] **DNC processing** within 10 business days of opt-out request
- [ ] **One-to-one consent** documented for every lead — consent fields populated and retrievable
- [ ] **Consent revocation** processed from any channel (phone, text, email, etc.)
- [ ] **Company identification** at start of every call — agent scripts include company name
- [ ] **Opt-out mechanism** offered on every call — agents trained to process DNC requests
- [ ] **STIR/SHAKEN** A-level attestation on all outbound DIDs — verified with SIP provider

### State Mini-TCPA Compliance

- [ ] **Florida (FTSA):** 3 calls/24 hrs max, 8 AM-8 PM, state DNC scrubbed, written consent for cell phones
- [ ] **Texas (SB 140):** Registered with Secretary of State
- [ ] **Virginia:** 10-year DNC retention policy, DNC entries never purged
- [ ] **Oklahoma (OTSA):** Registered with Attorney General, bond in place if required
- [ ] **California (CCPA):** 15-day opt-out processing, data handling documented
- [ ] **Washington:** 8 AM-8 PM calling hours, caller identification within 30 seconds
- [ ] **Indiana:** Registered with Attorney General, state DNC scrubbed
- [ ] **All target states:** Verified calling hours, DNC list requirements, registration requirements

### VICIdial Technical Configuration

- [ ] **Adaptive Dropped Percentage:** 2.0-2.5%
- [ ] **Drop Percentage Max:** 3.0%
- [ ] **Safe Harbor Audio:** Configured with compliant recording
- [ ] **Safe Harbor Message on Drop:** Y
- [ ] **Local Call Time:** Configured with state-compliant hours
- [ ] **Time Zone Checking:** Enabled (TZCODE preferred)
- [ ] **Use Internal DNC List:** Y
- [ ] **Use Campaign DNC List:** Y (where applicable)
- [ ] **DNC auto-add on DNC disposition:** Configured
- [ ] **Hopper GMT Settings:** Enabled
- [ ] **Phone timezone data:** Current in `vicidial_phone_codes`
- [ ] **Consent custom fields:** Created and required for lead loading
- [ ] **Dial timeout:** 26-30 seconds (prevent premature hangups)
- [ ] **AMD safe harbor:** Message plays on AMD-classified machine calls if message is left

### Operational Compliance

- [ ] **Agent training:** TCPA compliance training completed and documented
- [ ] **Script compliance:** Scripts include company identification and opt-out language
- [ ] **Call recording:** All calls recorded and retained per applicable law (check two-party consent states)
- [ ] **Monthly compliance audit:** Drop rates, DNC processing times, consent records reviewed
- [ ] **Incident response plan:** Documented process for handling TCPA complaints
- [ ] **Compliance officer designated:** Named individual responsible for TCPA compliance
- [ ] **Attorney relationship:** Outside counsel identified for TCPA matters

### Recording and Two-Party Consent States

VICIdial records calls by default (configurable per campaign). In two-party consent states, you must inform the called party that the call is being recorded. These states require all-party consent for recording:

California, Connecticut, Delaware, Florida, Illinois, Maryland, Massachusetts, Michigan, Montana, Nevada, New Hampshire, Oregon, Pennsylvania, Vermont, Washington

**VICIdial configuration:** Enable a pre-call recording notification (either an automated message via the `campaign_recording_message` setting or require agents to state the recording disclosure in their opening script).

---

## Consent Management: The One-to-One Rule in Practice

The FCC's one-to-one consent rule deserves additional detail because it fundamentally changes how most outbound operations acquire leads.

### What Valid Consent Looks Like in 2026

A valid consent record under the new rule must show:

1. **Specific seller identification.** The consent form must name your company — not "our partners," not "participating providers," not "companies in our network." Your company name, clearly and conspicuously displayed.

2. **Specific product/service.** The consent must relate to the product or service you're calling about. Consent to receive calls about solar panels does not authorize calls about home security systems, even if both products are sold by the same company (unless the consent language covers both).

3. **Clear and conspicuous disclosure.** The consent language cannot be buried in terms of service or hidden behind a hyperlink. It must be visible and readable at the point where the consumer takes the action that constitutes consent.

4. **Affirmative action.** The consumer must take an affirmative action to consent — checking an unchecked box, signing a form, clicking a button that clearly indicates consent. Pre-checked boxes do not constitute valid consent.

5. **Logically and topically related.** The call must be logically related to the interaction in which consent was obtained. If someone fills out a form requesting a solar quote, calling them about solar is topically related. Calling them about windows or roofing is not (unless the form explicitly covered those products).

### Lead Source Compliance Verification

Before loading any purchased leads into VICIdial, verify the consent chain:

**Tier 1: Exclusive leads (highest compliance confidence)**
- Consumer filled out your company's own web form
- Consent language names your company specifically
- You have the full consent record (form, timestamp, IP, language)
- Risk: Low

**Tier 2: Exclusive leads from a lead gen partner (moderate compliance confidence)**
- Consumer filled out a partner's web form that named your company
- You receive the consent record from the partner
- You've verified the partner's form and consent language comply with one-to-one rule
- Risk: Moderate (depends on partner's compliance practices)

**Tier 3: Shared leads / aggregator leads (high compliance risk)**
- Consumer filled out a form that authorized multiple companies
- Pre-2025 consent model — **likely non-compliant under one-to-one rule**
- Risk: High — these leads should be treated as non-consented unless the aggregator has updated their forms to comply with the new rule

**Tier 4: Purchased lists without consent records (non-compliant)**
- No consent documentation
- Risk: Extreme — do not dial these with predictive/automatic dialing

### Implementing Consent Validation in Lead Loading

Add a validation step to your VICIdial lead loading process:

```bash
#!/bin/bash
# Pre-load consent validation for VICIdial lead import
# Rejects leads without required consent fields

# Required consent fields - all must be non-empty
REQUIRED_FIELDS="consent_timestamp consent_source consent_company"

# Validate each lead before API submission
while IFS='|' read -r phone first last consent_ts consent_src consent_co; do
    VALID=true
    if [ -z "$consent_ts" ] || [ -z "$consent_src" ] || [ -z "$consent_co" ]; then
        echo "REJECTED: $phone - missing consent fields" >> /var/log/vicidial/consent_rejects.log
        VALID=false
    fi

    if [ "$VALID" = true ]; then
        # Submit to VICIdial API
        curl -s "https://vicidial-server/vicidial/non_agent_api.php?\
source=consent_import&function=add_lead&user=api_user&pass=api_pass&\
phone_number=${phone}&first_name=${first}&last_name=${last}&\
list_id=10001&custom_fields=Y&\
consent_timestamp=${consent_ts}&consent_source=${consent_src}&\
consent_company=${consent_co}"
    fi
done < leads_to_import.txt
```

---

## Enforcement Trends: What's Getting Operations Sued in 2026

Understanding current enforcement patterns helps you prioritize compliance efforts:

**1. Consent chain challenges.** Plaintiff attorneys are aggressively challenging lead gen consent chains. They fill out forms themselves, receive calls from multiple companies, and then file class actions alleging the shared consent model violates the one-to-one rule. If you can't produce a one-to-one consent record for the specific phone number, you lose.

**2. Revocation timing.** Consumers who request removal and then receive a call 3 days later are filing complaints. The FCC's "reasonable time" standard is being interpreted by courts as 24-72 hours in most cases, not the 10 business days the FCC suggested as an outer limit. Process opt-outs faster.

**3. State FTSA (Florida) litigation explosion.** The Florida FTSA's private right of action has created a cottage industry of plaintiff attorneys filing FTSA class actions. The 3-call-per-24-hour limit is the most commonly violated provision because many VICIdial operations don't configure per-number daily limits for Florida leads.

**4. ATDS definition still in flux.** After the Supreme Court's 2021 Facebook v. Duguid decision narrowed the ATDS definition, litigation shifted to state laws (which often have broader ATDS definitions) and to consent-based claims. VICIdial's predictive dialer does qualify as an ATDS under some state definitions even if it may not under the narrowed federal definition. Don't rely on ATDS arguments as a defense — focus on consent and DNC compliance.

**5. Recording consent violations.** Two-party consent states are generating a new wave of litigation. An agent in a one-party state (like Texas) calling a consumer in a two-party state (like California) must comply with California's recording consent law. If your VICIdial campaigns record calls without informing consumers in two-party consent states, you're exposed.

---

## Frequently Asked Questions

### What is the maximum abandon rate allowed under the TCPA?

The FCC's safe harbor allows a maximum 3% abandon rate measured over a 30-day campaign period. This means no more than 3% of **answered calls** can be abandoned (no live agent available when the consumer picks up). In VICIdial, set `Adaptive Dropped Percentage` to 2.0-2.5% and `Drop Percentage Max` to 3.0%. Monitor your actual 30-day rolling rate weekly using the SQL queries provided in this guide.

### How quickly do I need to process a do-not-call request?

The FCC's revocation rules (effective April 2025) require processing within a "reasonable time," which the FCC has indicated as no more than 10 business days. However, courts and plaintiff attorneys are increasingly arguing that 24-72 hours is reasonable given modern technology. Best practice: process opt-outs within 24 hours. In VICIdial, configure the DNC disposition to auto-add to the internal DNC list immediately, which handles the dialing side. Ensure your CRM and lead sources also receive the opt-out notification.

### Does VICIdial automatically check the National Do Not Call Registry?

No. VICIdial has internal DNC lists (system and campaign level) but does not natively integrate with the FTC's National DNC Registry. You must scrub your lead lists against the national DNC externally before loading them into VICIdial, and re-scrub every 31 days. Some third-party list scrubbing services (DNC.com, Gryphon, Contact Center Compliance) offer automated scrubbing that can be integrated into your VICIdial lead loading workflow.

### What happens if my VICIdial abandon rate goes over 3%?

You're in violation of the TCPA's abandoned call provisions. Each abandoned call above the 3% threshold is a potential violation carrying $500-$1,500 in damages. For a campaign making 10,000 answered calls per month at 4% drops instead of 3%, that's 100 excess drops — potentially $50,000-$150,000 in exposure per month. Immediately reduce dial levels, add agents, and investigate the cause. Document your remediation actions.

### Do I need separate VICIdial campaigns for different states?

Not necessarily separate campaigns, but you need state-specific configurations. VICIdial's Call Time and lead filter features can handle state-specific calling hour restrictions. For states like Florida with per-number daily call limits, you'll need to configure lead filters or custom dial count limits for Florida area codes. Some operations create state-specific sub-campaigns for heavy-restriction states (Florida, California) to simplify compliance management.

### Is the FCC one-to-one consent rule retroactive?

The FCC stated the rule applies to consents obtained on or after January 27, 2025. Consents obtained before that date under the old rules remain valid — but only for the duration specified in the original consent (or until revoked). In practice, if you're still dialing leads from pre-2025 consent records in mid-2026, verify that the original consent hasn't expired and hasn't been revoked. The safest approach: transition entirely to one-to-one compliant consent by Q2 2026.

### How do I handle TCPA compliance for SMS campaigns run alongside VICIdial?

SMS (text messaging) is covered by the TCPA and has its own consent requirements. Key difference: the TCPA requires express written consent (not just express consent) for telemarketing text messages to cell phones. Your consent form must explicitly mention text messages. In VICIdial, if you're using the built-in SMS capabilities or integrating with an external SMS platform, ensure: (1) separate consent for SMS is obtained, (2) STOP/opt-out processing is automated, (3) message frequency matches what was disclosed in the consent, and (4) every message includes opt-out instructions.

### What records should I keep for TCPA compliance?

Retain the following for at least 5 years (longer for Virginia — 10 years for DNC records): consent records for every lead (timestamp, source, language, company identified), complete call logs (VICIdial's `vicidial_log` table), DNC list history (with dates added), agent training records, campaign configuration snapshots, abandon rate reports, and any consumer complaints received. VICIdial's MySQL database retains call logs indefinitely by default, but ensure your backup and archival processes preserve this data.

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-tcpa-compliance).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
