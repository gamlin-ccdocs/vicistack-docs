# Call Center Compliance Checklist: FTC, TCPA, TSR, and State Laws in 2026

**TCPA class actions jumped 112% in Q1 2025. The FTC can now fine you $53,088 per call. One auto warranty scheme drew a $300 million penalty. If you're running outbound campaigns in 2026, this is the checklist that keeps you out of the crosshairs.**

---

The compliance landscape for outbound call centers changed more between 2024 and 2026 than it did in the previous decade. The FCC rewrote consent revocation rules. The FTC expanded TSR coverage to B2B calls. AI voice calls got classified as robocalls overnight. And several states passed mini-TCPA laws stricter than the federal rules.

TCPA litigation is at an all-time high -- 2,788 cases filed in 2024, up 67% from 2023. Q1 2025 class action filings ran 112% above the prior year. Average settlement: $6.6 million.

This post covers every federal and state regulation that applies to your outbound operation, the dialer settings you need to configure (with VICIdial admin GUI paths), and the 10 misconfigurations that get call centers sued.

## What Changed in 2025-2026

Four major regulatory shifts redefined compliance for outbound call centers. If you haven't updated your setup since early 2024, you're exposed.

### The One-to-One Consent Rule: Born and Killed

In December 2023, the FCC adopted a rule (CG Docket No. 02-278) that would have required lead generators to obtain consent for one seller at a time -- killing the model where a single checkbox on a comparison site authorizes calls from a dozen companies.

The rule was supposed to take effect January 27, 2025. It never did. On January 24, 2025, the Eleventh Circuit vacated it in *Insurance Marketing Coalition Ltd. v. FCC* (No. 24-10277), ruling that the FCC exceeded its authority. The court held that the one-to-one restriction impermissibly altered the ordinary statutory meaning of "prior express consent" under the TCPA.

By April 2025, the FCC announced it wouldn't challenge the ruling. In August 2025, they formally reinstated the pre-2023 consent standard.

**What this means now:** Lead generators can still collect consent for multiple sellers in a single interaction. You still need prior express written consent (in writing, with a signature, with clear disclosures per 47 C.F.R. § 64.1200), but the one-to-one requirement is dead. For now.

### Consent Revocation: 10 Business Days, Any Method

This one is in effect and actively enforced. As of April 11, 2025 (47 C.F.R. § 64.1200(a)(9)(i)(F), (a)(10), (a)(11), and (d)(3)):

- Consumers can revoke consent through **any reasonable method** -- text, email, voicemail, verbal request during a call, carrier-level blocking, anything
- You must honor revocation within **10 business days** (down from the previous 30-day standard)
- The FCC designated seven mandatory opt-out keywords: **stop, quit, revoke, opt out, cancel, unsubscribe, end**

Important carve-out: the "revocation-all" provision (where opting out of one message type revokes ALL consent from that caller) was stayed on April 7, 2025 (DA 25-312) and delayed until **January 31, 2027** (DA 26-12). You don't have to treat a text opt-out as revoking phone call consent yet. Build the systems now anyway.

### AI-Generated Voices = Robocalls

The FCC's February 2024 declaratory ruling (FCC 24-17, CG Docket No. 23-362) classified AI-generated voices -- including voice cloning -- as "artificial or prerecorded voice" under TCPA § 227(b)(1). This applies retroactively.

If you use AI voices for voicemail drops, IVR prompts, or outbound messages, every existing TCPA restriction applies. No AI-specific exemption exists. You need the same level of prior express consent as any prerecorded human voice call.

### TSR Amendments: B2B Coverage and 5-Year Records

Effective January 9, 2025 (89 FR 98,220):

- **B2B calls are no longer partially exempt.** The TSR's prohibitions against misrepresentations and misleading statements now apply to business-to-business telemarketing calls. If your B2B agents are making claims about ROI or product capabilities, those claims are now TSR-regulated.
- **Recordkeeping extended to 5 years.** All records related to telemarketing transactions must be retained for at least 5 years. Not 3. Not "until we run out of disk space." Five years.

## Federal Requirements: The Full List

Here's every federal requirement that applies to outbound call centers, with the specific citation and the VICIdial configuration that addresses it.

### DNC (Do Not Call) Compliance

**Law:** TCPA § 227 + TSR 16 CFR § 310.4(b)(1)(iii)

| Requirement | Detail | VICIdial Setting |
|---|---|---|
| Federal DNC scrub | Scrub all calling lists against the National DNC Registry every **31 days** | Import FTC data via DNC.com integration or manual upload. Admin > DNC Numbers for bulk import |
| Internal DNC list | Maintain a company-specific DNC list. When someone says "don't call me," add them immediately | Campaign Detail > Use Internal DNC List = **Y** |
| Per-campaign DNC | Separate DNC lists scoped to individual campaigns | Campaign Detail > Use Campaign DNC List = **Y** |
| Opt-out processing | Honor opt-out requests within **10 business days** | Train agents to disposition as DNC immediately. Verify DNC lists propagate system-wide |
| State DNC registries | 11 states maintain separate DNC lists: CO, FL, IN, LA, MA, MO, OK, PA, TN, TX, WY | Scrub via DNC.com before importing lists. VICIdial does not auto-sync state DNC data |
| EBR time limits | Existing Business Relationship exemption expires **18 months** after last transaction, **3 months** after last inquiry | Track in CRM. Automate EBR expiration checks before building call lists |
| Reassigned Number Database | Query the FCC RND before calling numbers with consent older than 30 days. One query = one protected call | Use DNC.com or similar service. 47 CFR § 64.1200(n) provides safe harbor |

**VICIdial note:** Do NOT load the entire federal DNC registry (240+ million numbers) into VICIdial's internal DNC table. The hopper checks every lead against it on every pass -- 240M rows will crush database performance. Pre-scrub through a third-party service before importing.

**Registry access fees for FY 2026:** $82 per area code, $22,626 max for nationwide access. First 5 area codes are free.

### Calling Hours

**Law:** TSR 16 CFR § 310.4(c)

Outbound telemarketing calls are permitted only between **8:00 AM and 9:00 PM in the called party's local time zone**. Not your time zone. Theirs.

**VICIdial configuration:**

1. **Admin > Call Times** -- create a Call Time Definition (e.g., "TCPA_8am_9pm") with start 0800, stop 2100 for each weekday
2. **Campaign Detail > Local Call Time** -- assign this Call Time to your campaign
3. The hopper script (`AST_VDhopper.pl`) checks each lead's `gmt_offset_now` against the campaign's Local Call Time before loading it into the hopper. Leads outside the window are skipped.
4. **Critical:** Use "Postal Code First" timezone lookup when loading lists (List Loader screen). Area code lookup is unreliable because of number portability -- a 312 (Chicago) number might belong to someone who moved to Denver. ZIP code mapping is more accurate.

### Abandoned Call Rate

**Law:** TSR 16 CFR § 310.4(b)(4)(i) / FCC 2008 Declaratory Ruling

No more than **3% of calls answered by a live person** may be abandoned, measured per campaign over each 30-day period. An "abandoned" call is one where a live person answers but no agent is available to take the call.

| VICIdial Setting | Location | Compliant Value |
|---|---|---|
| Drop Percentage Limit | Campaign Detail | **2% or lower** (gives buffer below the 3% legal max) |
| Drop Action | Campaign Detail | **CALLMENU** (not HANGUP) |
| Safe Harbor Call Menu | Campaign Detail | Custom call menu that identifies your company, provides callback number, offers DNC opt-out |
| Drop Call Seconds | Campaign Detail | Match to your safe harbor message length |
| Dial Method | Campaign Detail | **ADAPT_HARD_LIMIT** or **ADAPT_TAPERED** (never RATIO for compliance campaigns) |
| Adaptive Intensity | Campaign Detail | **0.2-0.5** (above 0.5 causes high abandon rates during traffic spikes) |

**Setting up the safe harbor call menu:**

1. Admin > Call Menus > Create new (e.g., "FTC_safe_harbor")
2. Add an audio file that: identifies your company by name, states the call was made on behalf of [company], provides a callback number
3. Add a keypress option (e.g., press 1) that triggers `cm_dnc.agi` to add the caller to your DNC list
4. In Campaign Detail, set Drop Action = CALLMENU and Safe Harbor Call Menu = FTC_safe_harbor

### Caller ID Requirements

**Law:** TSR 16 CFR § 310.4(a)(8) + Truth in Caller ID Act, 47 U.S.C. § 227(e)

You must transmit a valid caller ID on every outbound call -- your company's phone number and, when technically possible, your company's name. Displaying a number that doesn't ring back to your business or using a number you're not authorized to use is a federal crime under the Truth in Caller ID Act. Penalties: up to **$10,000 per violation**.

**VICIdial configuration:** Campaign Detail > Campaign CallerID -- set to a valid, registered 10-digit number you own or are authorized to use. For multi-number setups, use Admin > CID Groups with type AREACODE or STATE to match caller ID to the lead's location.

### STIR/SHAKEN

**Law:** TRACED Act (Pub. L. 116-105)

STIR/SHAKEN is a caller ID authentication framework. Your carrier assigns an attestation level to each call:

| Level | Meaning | What Happens |
|---|---|---|
| **A** (Full) | Carrier verified you're authorized to use this number | Calls go through normally |
| **B** (Partial) | Carrier authenticated you but can't verify the specific number | Calls may be flagged |
| **C** (Gateway) | Carrier has no relationship with the originator | Calls likely blocked or voicemail-only |

VICIdial doesn't implement STIR/SHAKEN directly -- your SIP carrier does. But your configuration determines your attestation level. Use numbers registered to your account, verify your business identity with your carrier, avoid unregistered or improperly ported numbers.

For carriers that don't handle signing, VICIdial has native TILTX Call Shaper integration -- TILTX verifies your CID, creates a signed PASSporT token, and VICIdial passes it in the SIP INVITE header.

Key deadlines: September 18, 2025, providers can only use third parties for signing if attestation decisions use their own certificate. March 1, 2026 was the first annual RMD (Robocall Mitigation Database) recertification deadline.

### Call Recording

**Law:** 18 U.S.C. § 2511 (Wiretap Act / ECPA)

Federal law requires **one-party consent** -- the agent's knowledge that recording is on is sufficient at the federal level. But 12 states require **all-party consent**:

California (Cal. Penal Code § 632), Connecticut (§ 52-570d), Delaware (11 Del. C. § 2402), Florida (§ 934.03), Illinois (720 ILCS 5/14-2), Maryland (§ 10-402), Massachusetts (ch. 272, § 99), Michigan (MCL § 750.539c), Montana (§ 45-8-213), New Hampshire (§ 570-A:2), Oregon (ORS § 165.540), Pennsylvania (18 Pa. C.S. § 5703), and Washington (RCW § 9.73.030).

**Interstate rule:** When a call crosses state lines, the stricter state's law applies. If your call center is in Texas but you're calling someone in California, you must comply with California's all-party consent.

**VICIdial configuration:**
- Campaign Detail > Campaign Recording = **ALLFORCE** (records every call; agents cannot stop recording)
- Play a "this call may be recorded" announcement at the start of every call, regardless of destination state. Configure this in your IVR or Asterisk dialplan. VICIdial does NOT automatically play this announcement -- you must set it up.
- For Massachusetts calls specifically, implied consent (continuing the call after hearing the announcement) may not be sufficient. Train agents to get verbal confirmation for MA numbers.

### Caller Identification Disclosure

**Law:** TSR 16 CFR § 310.4(d)

Within the first few seconds of every outbound call, your agent must identify: themselves (name), their company, and the purpose of the call. For prerecorded messages, the same information must be in the recording.

This is agent training, not a VICIdial setting. Enforce it with VICIdial's **Agent Script** feature (Campaign Detail > Script) -- create a script with mandatory disclosure language as the first thing agents see when a call connects.

## State Laws: Where It Gets Ugly

Federal law is the floor. Several states have built higher walls, and violating a state mini-TCPA means state-level penalties on top of federal exposure.

### Florida -- Florida Telephone Solicitation Act (Fla. Stat. § 501.059)

The most aggressive state-level telemarketing law in the country.

- **Calling hours:** 8 AM to **8 PM** (one hour shorter than federal)
- **Max call attempts:** 3 per person per 24 hours on the same subject
- **Autodialer definition:** Device must have capacity to both "select AND dial" numbers (narrower than federal -- click-to-dial systems are excluded)
- **Consent:** Prior express written consent required for autodialed cell phone calls
- **Text message safe harbor:** Consumer must send a stop request and give you **15 days** to cease before filing suit
- **Penalties:** $500/violation, $1,500/willful. Private right of action.

**VICIdial config:** Create a State Call Time for FL with Stop = 2000. Admin > Call Times > State Call Times > Add New > State: FL, Start: 0800, Stop: 2000. Attach to your Call Time Definition.

### Oklahoma -- Oklahoma Telephone Solicitation Act (15 O.S. § 775C)

- **Calling hours:** 8 AM to **8 PM**
- **Max call attempts:** 3 per 24 hours on the same subject
- **Autodialer definition:** Broader than federal -- covers "automated system for the selection OR dialing of telephone numbers"
- **Voice alteration:** Illegal to alter your voice to disguise identity for fraud
- **Penalties:** $500/violation, $1,500/willful (uncapped). Private right of action.
- **Exemptions:** Existing business relationships, B2B, charitable/religious/political

**VICIdial config:** Same State Call Time approach as Florida. For the 3-call limit, manage through lead recycling rules and max daily dial attempts in campaign settings.

### Texas -- SB 140 (Tex. Bus. & Com. Code Ch. 302)

Effective September 1, 2025. This one adds a registration requirement on top of calling restrictions.

- **Registration required:** Businesses making telephone solicitations to/from Texas must file with the Secretary of State, pay a **$200 annual fee**, and post a **$10,000 bond** (surety bond runs about $250 for 3 years)
- **Scope:** Explicitly includes text messages and electronic communications
- **Quarterly reporting** to the Secretary of State
- **Consent exemption:** Businesses sending texts with prior consent are NOT required to register
- **Violations = UDAP:** Any violation is an "unfair and deceptive practice" under Texas law
- **Penalties:** $500-$1,500 per call/text; AG can seek $5,000/violation. Private right of action (new under SB 140).

If you're calling into Texas, check whether your operation needs to register. Most outbound centers do unless every call is consent-based.

### Washington -- Robocall Scam Protection Act (RCW 80.36 / RCW 19.158)

- Broad prohibition on assisting in transmission of unsolicited commercial solicitations via autodialers
- Prohibits caller ID spoofing with intent to defraud -- and unlike federal TCPA, Washington provides a **private right of action** for caller ID violations
- Penalties: up to $1,000/violation with increased statutory awards for repeat violations

### Other States to Watch

| State | Key Provision | Citation |
|---|---|---|
| Georgia | Express consent required for prerecorded calls | O.C.G.A. § 46-5-27 |
| Illinois | Automatic Telephone Dialers Act -- one of the oldest mini-TCPAs | 815 ILCS 305 |
| Indiana | Calling hours restricted to **9 AM - 8 PM** (narrower than federal on both ends) | IC 24-4.7 |
| Louisiana | No calls on Sundays before noon or holidays | La. R.S. 45:844.11-844.18 |
| Maine | Requires checking FCC Reassigned Numbers Database before calling (2024) | LD 2234 |
| Michigan | Michigan Telephone Consumer Protection Act -- expanded 2024 | MCL 445.111a |
| New York | State registration required for telemarketers | N.Y. Gen. Bus. Law Art. 22-A |

## The 10 Misconfigurations That Get Call Centers Sued

Every one of these has produced real lawsuits, real fines, or both. Every one of them is fixable through admin GUI settings and process changes.

### 1. Drop Percentage Limit Set Too High (or Not Set)

**The mistake:** Leaving the Drop Percentage Limit at default or setting it above 3%.

**The damage:** FTC fines up to $53,088 per abandoned call. In a 200-seat center making 50,000 calls/day, even a 4% abandon rate generates 2,000 violations per day. That's $106 million in potential fines. Per day.

**The fix:** Campaign Detail > Drop Percentage Limit = **2%**. Use ADAPT_HARD_LIMIT or ADAPT_TAPERED dial methods. Keep Adaptive Intensity between 0.2-0.5.

### 2. DNC and DNCL in Dial Statuses

**The mistake:** Including DNC or DNCL dispositions in the campaign's Dial Statuses list, causing the dialer to re-call numbers that agents already marked as Do Not Call.

**The damage:** TCPA violations at $500-$1,500 per call. Class action exposure. VICIdial's config validator flags this as a hard error for a reason.

**The fix:** Audit every campaign's Dial Statuses. Remove DNC and DNCL. No exceptions.

### 3. No Safe Harbor Message on Dropped Calls

**The mistake:** Setting Drop Action to HANGUP. Customer answers, hears silence, gets hung up on.

**The damage:** Every dropped call without a safe harbor message is a TSR violation. Plus it destroys your brand. Nobody calls back a number that called them and hung up.

**The fix:** Campaign Detail > Drop Action = **CALLMENU**. Build a Safe Harbor Call Menu (Admin > Call Menus) that identifies your company, provides a callback number, and offers a press-1 DNC opt-out.

### 4. Wrong Timezone on Leads

**The mistake:** Loading leads without proper timezone coding. Relying on area code lookup for cell phones (number portability makes area codes unreliable -- a 212 area code might be sitting in Montana).

**The damage:** Calling at 7 AM or 10 PM local time. Per-call violation, both federal and state.

**The fix:** Use "Postal Code First" timezone lookup in the List Loader. Verify a sample of leads after loading to confirm GMT offsets are correct.

### 5. No State Call Times

**The mistake:** Running a single "8am to 9pm" call time for all states. Florida stops at 8 PM. Oklahoma stops at 8 PM. Indiana starts at 9 AM.

**The damage:** 500 calls at 8:15 PM to Oklahoma = 500 violations x $500 = $250,000 minimum exposure. And that's just one evening.

**The fix:** Create State Call Time definitions for every state with non-standard hours. Admin > Call Times > State Call Times. Add FL (0800-2000), OK (0800-2000), IN (0900-2000). Attach them to your Call Time Definition. Build in a 15-minute buffer.

### 6. Fake or Rotating Caller IDs

**The mistake:** Using 0000000000, a number you don't own, or "snowshoeing" across hundreds of numbers to avoid spam flags.

**The damage:** Truth in Caller ID Act violation -- federal crime, up to $10,000 per call. STIR/SHAKEN failures (C-level attestation) mean your calls get blocked anyway. The FCC runs traceback investigations and they will find you.

**The fix:** Only use valid, registered numbers you control. Campaign Detail > Campaign CallerID or CID Groups with numbers your carrier has authorized. Monitor number reputation and pull flagged numbers out of rotation.

### 7. Recording Set to ONDEMAND or NEVER

**The mistake:** Letting agents decide when recording starts and stops. Or not recording at all.

**The damage:** No evidence to defend against customer complaints, lawsuits, or regulatory inquiries. In two-party consent states, no proof you disclosed recording. When the plaintiff's attorney asks for the recording and you don't have one, you lose.

**The fix:** Campaign Detail > Campaign Recording = **ALLFORCE**. Every call recorded, no agent override.

### 8. Not Scrubbing Against DNC Lists

**The mistake:** Downloading the federal DNC list once and never updating it. Or skipping state DNC lists entirely.

**The damage:** The 31-day scrub requirement is strictly enforced. Numbers are added daily. The 11 state DNC lists carry independent penalties.

**The fix:** Pre-scrub every list through DNC.com or equivalent before importing into VICIdial. DNC.com has a native VICIdial integration (`AST_DNCcom_filter.pl`). Set up in Admin > Settings Containers with a "DNCDOTCOM" container holding your API credentials. Schedule scrubs in crontab or run manually before each list load.

### 9. Ignoring Opt-Out Processing

**The mistake:** Only honoring "STOP" texts while ignoring verbal requests, emails, or voicemails. Or taking 30+ days to process.

**The damage:** Since April 2025, consent revocation through any reasonable method must be honored within 10 business days. The FCC lists seven mandatory keywords. Plaintiff attorneys argue "reasonable" means 24-48 hours.

**The fix:** Build keyword detection for all seven FCC keywords (stop, quit, revoke, opt out, cancel, unsubscribe, end). Create a multi-channel opt-out intake workflow. Disposition as DNC the moment a customer requests it. Verify "Use Internal DNC List" = Y so the disposition actually blocks future calls.

### 10. Callbacks Scheduled in Wrong Timezone

**The mistake:** Agent schedules a callback at "2 PM" using the server's timezone instead of the customer's timezone. Customer gets called at 5 PM or 11 AM their time.

**The damage:** Calling outside legal hours. A Pacific-time server scheduling an "8 PM" callback to an Eastern customer means 11 PM their time.

**The fix:** Verify that Campaign Detail > Callback Timezone is configured to use the lead's timezone. Train agents to confirm the customer's timezone verbally.

## The Penalty Landscape: What Real Violations Cost

If the compliance checklist feels like overkill, look at what happens when you skip it.

### Largest TCPA/TSR Penalties (All Time)

| Company | Penalty | Year | What They Did |
|---|---|---|---|
| Roy Cox Jr. / Sumco Panama | **$299.997M** | 2023 | 5 billion illegal robocalls in 3 months. Auto warranty scam. |
| Dish Network | **$210M** | 2020 | 66M+ telemarketing violations through dealers. DOJ + 4 states. |
| Caribbean Cruise Line | **$76M** | 2017 | Millions of robocalls offering "free" cruises via fake political surveys |
| Capital One | **$75.5M** | 2014 | Autodialers and prerecorded calls to cell phones without consent |
| Keller Williams Realty | **$40M** | 2023 | Prerecorded calls to DNC numbers without consent |
| National Grid | **$38.5M** | 2022 | Automated bill/disconnect calls for over a decade without consent |

$500 per violation, $1,500 for willful, no cap on aggregate damages. A 20-seat center with a systemic DNC failure can rack up $5 million in exposure in a single week.

The FTC's "Operation AI Comply" (September 2024) swept companies using AI voices for illegal robocalls -- combined fines exceeding $5M plus permanent bans. Citizens Disability took a $1M penalty (September 2025). After the Cox enforcement, auto warranty spam dropped 99%.

**Class action stats:** 2,788 TCPA cases in 2024 (67% increase). 507 class actions in Q1 2025 (112% increase). Average settlement: $6.6 million. Litigator scrubbing ($0.002-$0.005/number via DNC.com) is cheap insurance against serial TCPA plaintiffs.

## The Complete Checklist

Print this. Hand it to your compliance officer. Audit against it quarterly.

### Consent and DNC

- [ ] All calling lists scrubbed against the federal DNC registry within the last 31 days
- [ ] All calling lists scrubbed against relevant state DNC registries (11 states maintain separate lists)
- [ ] Internal company DNC list active (Campaign Detail > Use Internal DNC List = Y)
- [ ] Per-campaign DNC list active (Campaign Detail > Use Campaign DNC List = Y)
- [ ] Opt-out requests processed within 10 business days
- [ ] Written consent documentation stored for every lead (minimum 5 years per TSR)
- [ ] Lead vendors audited for consent collection practices (quarterly minimum)
- [ ] EBR exemption time limits tracked (18 months transaction / 3 months inquiry)
- [ ] Reassigned Number Database queried for consent records older than 30 days

### Dialing Configuration

- [ ] Drop Percentage Limit set to 2% or lower (Campaign Detail)
- [ ] Drop Action set to CALLMENU with compliant safe harbor message
- [ ] Calling hours configured per state (8 AM-9 PM federal; 8 PM for FL/OK; 9 AM start for IN)
- [ ] State Call Times created and attached to Call Time Definitions
- [ ] Dial method set to ADAPT_HARD_LIMIT or ADAPT_TAPERED
- [ ] Adaptive Intensity at 0.5 or below
- [ ] Max call attempts per number per day: 3 or fewer (safest across all jurisdictions)
- [ ] DNC and DNCL statuses removed from all campaigns' Dial Statuses

### Caller ID and Authentication

- [ ] Valid, registered caller ID on every campaign (Campaign Detail > Campaign CallerID)
- [ ] Carrier provides STIR/SHAKEN Level A attestation for your numbers
- [ ] Number reputation monitored -- flagged numbers pulled from rotation
- [ ] Numbers registered with major analytics databases (TNS, Hiya, First Orion)

### Recording and Disclosure

- [ ] Campaign Recording set to ALLFORCE on all outbound campaigns
- [ ] "This call may be recorded" announcement plays at start of every call
- [ ] Agents identify themselves, their company, and call purpose within first few seconds
- [ ] Agent scripts include mandatory disclosure language (Campaign Detail > Script)
- [ ] Recordings retained for 5+ years (use `ADMIN_archive_log_tables.pl` for archival)

### AI Voice and Technology

- [ ] AI-generated voices NOT used without prior express consent
- [ ] All AI voice usage documented with same consent standard as prerecorded calls
- [ ] No caller ID spoofing or number snowshoeing

### State-Specific

- [ ] Texas Secretary of State registration filed if calling TX without prior consent (SB 140, $200/year + $10,000 bond)
- [ ] Florida/Oklahoma 3-call-per-24-hour limit enforced
- [ ] Louisiana no-calls-on-Sunday-before-noon rule implemented
- [ ] Two-party recording consent handled for all 12 all-party states

### Monitoring and Documentation

- [ ] Outbound Calling Report reviewed weekly for drop rate compliance (Admin > Reports)
- [ ] DNC Log Report audited monthly
- [ ] Compliance settings changes logged in vicidial_admin_log (automatic if using admin GUI)
- [ ] Quarterly compliance audit scheduled and documented
- [ ] All logs archived for 5+ years (TSR requirement)

## What Comes Next

The "revocation-all" rule (DA 26-12) is delayed to January 31, 2027 -- but it's not dead. If it takes effect, opting out of text messages would also revoke consent for phone calls from the same caller. Build the cross-channel opt-out infrastructure now so you're not scrambling in December 2026.

The FCC issued an NPRM in November 2025 proposing new call branding rules that would let businesses display their brand name on caller ID. Still in the comment phase, but worth tracking -- branded caller ID could improve answer rates significantly.

Oregon tightened calling restrictions in 2026. More states will follow Florida and Oklahoma's lead with their own mini-TCPA laws. The trend is toward more regulation, not less.

And TCPA litigation isn't slowing down. It's accelerating. The plaintiff's bar has figured out that TCPA cases are profitable, and they're filing more aggressively every quarter. Your compliance posture isn't just about avoiding fines -- it's about not becoming the next $40 million headline.

---

*Running VICIdial and need help getting your compliance settings right? [We configure and audit VICIdial compliance settings as part of our deployment and optimization work.](/contact/) One misconfigured campaign can cost more than a decade of consulting fees.*

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/call-center-compliance-checklist-2026).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
