# TCPA 2026 Changes: What Call Centers Need to Know Before They Get Sued

**TCPA class actions spiked 283% in September 2025. The FCC rewrote how consent revocation works. Texas passed a mini-TCPA that covers text messages. And the blanket revocation rule takes effect April 2026. This is what actually changed, what's coming, and how to configure your dialer so you don't become the next seven-figure settlement.**

---

I've been building and maintaining call center infrastructure for over a decade. I've watched the TCPA go from a statute most operators barely thought about to the single biggest litigation risk in the outbound industry. And the 2025-2026 cycle has been the most chaotic regulatory stretch I've seen.

Between January 2025 and March 2026, we got: a major FCC consent rule vacated by a federal court, a new revocation framework that went live, a blanket revocation rule that's about to go live, three new state mini-TCPAs, and an FCC Notice of Proposed Rulemaking that could rewrite the abandonment rules entirely. Oh, and TCPA lawsuit filings hit all-time highs.

If you run outbound campaigns on VICIdial or any predictive dialer, this post walks through every change that matters, what the actual compliance requirements are right now, and the specific configurations that keep your operation out of the crosshairs.

For the detailed VICIdial settings guide, see our [TCPA compliance checklist](/blog/vicidial-tcpa-compliance/). For timezone-specific dialing configuration, see [timezone dialing and TCPA safe hours](/blog/vicidial-timezone-dialing-tcpa/).

---

## The 2025-2026 TCPA Timeline: What Actually Happened

The chronological sequence matters because the order of events created a situation where many operators either over-corrected or under-corrected depending on when they last checked the rules.

### January 2025: One-to-One Consent Rule -- Vacated

In December 2023, the FCC adopted the "one-to-one consent" rule to close what it called the "lead generator loophole." The rule would have required that consumer consent to receive autodialed or prerecorded calls be given to one specific seller at a time. No more comparison-shopping forms where a single checkbox authorizes calls from 10 different companies.

The rule was set to take effect January 27, 2025. On January 24, the FCC issued an order postponing the effective date to January 26, 2026. But the same day, the U.S. Court of Appeals for the Eleventh Circuit vacated the rule entirely in *Insurance Marketing Coalition v. FCC*, finding the FCC exceeded its statutory authority. The court reasoned that "prior express consent" under the TCPA has a plain meaning -- clearly stating willingness to receive calls -- and the FCC can't redefine it to require one-to-one specificity.

In April 2025, the FCC confirmed it would not challenge the ruling. In September 2025, the FCC issued a final rule formally eliminating the one-to-one consent requirement.

**What this means for you right now:** The one-to-one consent rule is dead. Multi-seller lead forms are still legally valid under federal law. A consumer who fills out a form authorizing calls from "Company A and its partners" has given valid prior express consent to all named parties.

**The catch:** Just because the federal rule died doesn't mean you should be sloppy about consent. Several states have their own consent requirements that are stricter than the federal baseline. And plaintiff attorneys don't need the one-to-one rule to sue you -- they'll argue the consent language was too vague, the form was misleading, or the consumer didn't actually see the disclosure. Keep your consent records tight regardless.

### April 2025: Consent Revocation Rules -- Partially Live

This is the change that actually went into effect and matters the most right now.

Effective April 11, 2025, the FCC's new revocation rules established that consumers can revoke consent to receive calls through **any reasonable means**. Not just the specific method your company designates. Any reasonable method.

The FCC specified these keywords as automatic revocation triggers: **stop, quit, revoke, opt out, cancel, unsubscribe, end**. But those aren't the only valid revocations. A consumer saying "take me off your list," "don't call me again," or "I don't want these calls" on a live call with your agent -- all valid.

Additionally, revocations made through any interactive keypress mechanism on an automated call, or through a website or phone number provided by the business, are valid.

**The 10-business-day window:** Once a consumer revokes consent, you have a "reasonable time" to process it. The FCC has indicated this means no more than 10 business days. Most plaintiff attorneys argue 24-48 hours is "reasonable." If you want to stay safe, process revocations within 24 hours.

**What went into effect April 2025:**
- "Any reasonable means" standard for revocation
- Mandatory honoring of the seven standard keywords
- 10-business-day processing window

**What got delayed to April 2026:**
- The "blanket revocation" rule -- where a consumer opting out of one type of message from your company is treated as opting out of ALL types of messages from you. The FCC granted a limited waiver pushing this to April 11, 2026.

That April 2026 blanket revocation deadline is 16 days away as of this writing. More on that below.

### September 2025: Texas Mini-TCPA Expansion

Effective September 1, 2025, Texas Senate Bill 140 materially expanded the Texas Business and Commerce Code. The changes:

- **Text messages now covered:** The definition of "telephone solicitation" now includes text messages, graphic messages, and image transmissions. Before SB 140, the Texas mini-TCPA only covered voice calls.
- **Registration required:** Sellers making telephone solicitations from or into Texas must obtain a registration certificate for each physical location.
- **Enhanced penalties:** Violations are now expressly deemed "unfair and deceptive practices" under Texas law, carrying $500-$1,500 per unlawful call or text. The AG can seek up to $5,000 per violation.
- **Stronger private right of action:** Plaintiffs face fewer procedural barriers under the Deceptive Trade Practices Act (DTPA) for violations.

If you send text campaigns into Texas from your dialer or any integrated SMS platform, you need to register and comply with SB 140 or you're exposed.

### October 2025: FCC Proposes Major NPRM

On October 28, 2025, the FCC unanimously adopted a Further Notice of Proposed Rulemaking (FNPRM) that could reshape TCPA compliance. Comments were due January 5, 2026; reply comments due February 3, 2026. The proposals include:

**Possible elimination of abandonment rules:** The FCC is considering removing the rules that require 15-second ring time before disconnect and the 3% abandon rate safe harbor for predictive dialers. The logic is that dialer technology has improved enough that these rules may be unnecessary. This is not yet law -- it's a proposal. But if adopted, it would completely change how predictive dialers are regulated.

**Consent revocation refinements:** The FCC is reconsidering the "revoke-all" rule (the blanket revocation set for April 2026). They're asking whether it "unduly restricts consumers' ability to receive wanted calls" -- like pharmacy reminders, bank alerts, or appointment notifications from businesses with multiple service lines.

**DNC rule changes:** An early draft proposed eliminating company-specific DNC list requirements. The final NPRM text removed that proposal -- internal DNC rules remain in effect. But the FCC did propose deleting certain company-specific DNC provisions where they believe general anti-robocall rules already cover the same ground.

**Caller ID and call branding:** The FCC proposed requiring that whenever a terminating provider displays A-level STIR/SHAKEN attestation, it must also show the caller's verified name. This would make anonymous robocalling significantly harder.

**Status as of March 2026:** The comment period has closed. No final rule has been adopted yet. Plan for the current rules, not the proposed ones.

---

## The Blanket Revocation Deadline: April 11, 2026

This deserves its own section because it's imminent and a lot of operators don't fully grasp the implications.

Starting April 11, 2026, if a consumer revokes consent for one type of communication from your company, that revocation applies to **all** communications from you -- even unrelated ones.

Example: A consumer receives a marketing text from your solar campaign and replies "STOP." Under the current rules (through April 10), that revocation only applies to the solar marketing texts. Under the blanket rule (starting April 11), that "STOP" means you can't call them about home warranties, insurance, or anything else either. One opt-out kills all channels and all campaigns from your organization.

The FCC's October 2025 NPRM floated the idea of narrowing or eliminating this rule, recognizing it could prevent consumers from receiving wanted calls (appointment reminders, fraud alerts, etc.). But as of this writing, no modification has been adopted. The April 11 deadline stands.

### What you need to do before April 11:

1. **Unify your opt-out processing across all campaigns.** If a consumer opts out anywhere -- any campaign, any channel -- that phone number needs to land on a system-wide DNC list, not just a campaign-specific one.

2. **In VICIdial:** Use the **Internal DNC list** (Admin > DNC Lists > Internal DNC), not just campaign-specific DNC lists. When a consumer revokes consent, add the number to the system-wide internal DNC so it blocks across all campaigns.

3. **Audit your multi-campaign operations.** If you run separate campaigns for different products under the same legal entity, all of them need to respect a single unified opt-out list after April 11.

4. **Document everything.** When the blanket revocation rule takes effect, plaintiff attorneys will look for companies that honored an opt-out on Campaign A but continued dialing the same number on Campaign B. That's an easy lawsuit.

---

## State Mini-TCPAs: The Patchwork That Multiplies Your Risk

Federal TCPA compliance is the floor. State laws are often the ceiling. And in 2025-2026, more states tightened their telemarketing rules. Below is the current landscape for the states that matter most.

### Florida (FTSA -- Florida Telephone Solicitation Act)

Florida's mini-TCPA has been in effect since July 2021, with 2023 amendments clarifying ATDS definitions. Key requirements:

- **Calling hours:** 8:00 AM to 8:00 PM local time (one hour shorter than federal)
- **Contact limit:** Maximum 3 attempts per recipient per 24-hour rolling period (calls + texts combined)
- **Consent:** Prior express written consent required for automated calls and texts
- **ATDS definition:** After 2023 amendments, ATDS means systems that randomly or sequentially generate numbers -- pre-existing contact lists are excluded
- **Penalties:** $500 per violation, $1,500 for willful violations
- **Private right of action:** Yes, and it's being used aggressively. A South Florida firm filed over 100 TCPA/FTSA timing-violation lawsuits in March 2025 alone.

**VICIdial configuration for Florida:** Set a `state_call_time` entry for Florida with an 8:00 PM cutoff. Set contact attempt limits using VICIdial's [list management](/blog/vicidial-list-management/) and lead recycling rules -- configure `Dial Timeout` and `Daily Call Count Limit` to cap at 3 attempts per 24 hours for Florida leads.

### Oklahoma (OTSA -- Oklahoma Telephone Solicitation Act)

Oklahoma's mini-TCPA has been in effect since November 2022. It's one of the most aggressive state telemarketing laws in the country:

- **Calling hours:** 8:00 AM to 8:00 PM local time
- **Contact limit:** Maximum 3 calls per 24-hour period to the same person regarding the same subject
- **Caller ID:** Must transmit originating phone number and name (if CNAM supported by carrier)
- **Consent:** Prior express written consent required for automated commercial calls and texts
- **ATDS definition:** Broad -- covers any automated system for selecting/dialing numbers or playing recorded messages
- **Penalties:** $500 per violation, $1,500 for willful/knowing violations

**VICIdial configuration for Oklahoma:** Same approach as Florida -- `state_call_time` with 8 PM cutoff, daily call count limits at 3. Verify your CID groups are transmitting real numbers with proper CNAM.

### Texas (Expanded September 2025)

As covered above, SB 140 expanded coverage to text messages, added registration requirements, and increased penalties. The key operational impact:

- **Registration:** Required for each physical location making solicitations from or into Texas
- **Text messages now regulated:** If you're running SMS campaigns alongside your dialer, texts to Texas numbers fall under the mini-TCPA
- **Penalties:** $500-$1,500 per call/text, plus AG enforcement up to $5,000 per violation
- **Sunday restriction:** No telemarketing calls before noon on Sundays

**VICIdial configuration for Texas:** Add a `state_call_time` for Texas with Sunday hours starting at 12:00 PM. Register with the Texas Secretary of State if you haven't already. If you run integrated SMS, make sure your text campaigns also respect Texas opt-out lists.

### Washington (Robocall Scam Protection Act)

- **Calling hours:** 8:00 AM to 8:00 PM local time
- **Identification:** Must identify yourself, company, and purpose within 30 seconds
- **DNC:** Companies must update DNC lists more frequently under the state law
- **Caller ID:** Accurate caller ID required -- spoofing is an independent violation
- **Penalties:** Up to $1,000 per incident; private right of action for repeated violations

### Other States to Watch

Eleven states maintain their own DNC registries separate from the federal list: Colorado, Florida, Indiana, Louisiana, Massachusetts, Missouri, Oklahoma, Pennsylvania, Tennessee, Texas, and Wyoming. If you're calling into any of these states, you need to obtain and scrub against their state DNC lists on top of the federal list.

**The state compliance matrix for VICIdial:**

| State | Hours | Daily Limit | DNC List | Registration | CID Rules |
|-------|-------|-------------|----------|-------------|-----------|
| Federal | 8AM-9PM | None | Federal | FTC TSR | Required |
| Florida | 8AM-8PM | 3/24hr | State + Federal | State | Required |
| Oklahoma | 8AM-8PM | 3/24hr | Federal | None | CNAM required |
| Texas | 8AM-9PM (noon Sun) | None | State + Federal | Per-location | Required |
| Washington | 8AM-8PM | None | Federal | None | Anti-spoof |
| Indiana | Complex (split TZ) | None | State + Federal | State bond | Required |
| Louisiana | 8AM-9PM (holidays) | None | State + Federal | None | Required |

---

## Lawsuit Statistics: Why This Matters More Than Ever

If you think TCPA compliance is a theoretical concern, here are the numbers.

### Filing Volume

- **2024:** 2,788 TCPA federal court filings (up 67% from 2023)
- **Q1 2025:** 507 class actions filed -- a 112% increase over Q1 2024
- **September 2025:** 224 class actions in a single month -- a 283% spike over September 2024
- **2025 full year:** Approximately 3,200 federal TCPA filings
- **78%** of TCPA lawsuits filed in September 2025 were class actions

There are over 10x more TCPA class actions than FDCPA class actions, and 20x more than FCRA class actions. The TCPA is the most litigated consumer protection statute in the country by a wide margin.

### Recent Settlements

Some examples that should keep you awake:

| Company | Settlement | Year | Allegation |
|---------|-----------|------|------------|
| Keller Williams | $40 million | 2025 | Unsolicited calls/texts |
| SiriusXM | $28 million | 2025 | Calls to DNC-registered numbers |
| Kaiser Permanente | $10.5 million | 2025 | Spam telemarketing texts |
| Gen Digital (Norton/LifeLock) | $9.95 million | 2025 | Unsolicited phone calls |
| Fathom Realty | $2.8 million | 2025 | Texts to DNC numbers |
| Nationwide | $1.4 million | 2025 | Robocalls |

### Per-Violation Penalties

- **Federal TCPA:** $500 per violation, $1,500 for willful violations
- **FTC TSR:** Up to $53,088 per call (adjusted for inflation, 2026)
- **State penalties:** Range from $100 to $25,000 per call depending on state

Do the math: a single 8-hour dialing session making 200 calls outside safe hours, at $500 per violation, is $100,000 in statutory damages. At the willful rate, $300,000. Scale that to a class action covering thousands of consumers and you're looking at the settlements above.

### Defense Costs

Even if you win, you lose money. Taking a TCPA case through discovery costs $50,000-$150,000 in legal fees. Trial preparation adds another $100,000-$300,000. Most companies settle because fighting costs more than paying.

---

## The 25-Point TCPA Compliance Checklist for 2026

Print this. Hand it to your compliance officer. Walk through every item.

### Consent

- [ ] All lead sources provide timestamped, traceable consent records with your company name visible in the disclosure
- [ ] Consent language is clear and conspicuous -- not buried in fine print or behind scrollable terms
- [ ] You can produce the specific form, timestamp, IP address, and consent language for any number you dial within 24 hours of a challenge
- [ ] Consent records are stored for at least 5 years (the TCPA recordkeeping requirement under 47 CFR 64.1200(d))
- [ ] Third-party lead vendors provide consent documentation, not just a CSV of phone numbers

### Revocation Processing

- [ ] Agents are trained to recognize revocation in any phrasing -- "stop calling," "take me off your list," "don't call again," "I'm not interested in any more calls"
- [ ] All seven FCC-specified keywords (stop, quit, revoke, opt out, cancel, unsubscribe, end) trigger automatic DNC processing in text/SMS channels
- [ ] Revocations are processed to DNC lists within 24 hours (technically 10 business days, but 24 hours is the safe target)
- [ ] Multi-channel revocations are unified -- a text "STOP" also stops voice calls to the same number
- [ ] After April 11, 2026: Revocation from any campaign/product blocks ALL campaigns/products (blanket revocation)

### DNC Management

- [ ] Federal DNC list scrubbed within the last 31 days
- [ ] State DNC lists obtained and scrubbed for every state you call into (11 states have separate lists)
- [ ] Internal DNC list enabled on all VICIdial campaigns (Admin > Campaigns > [Campaign] > DNC/Filtering > Use Internal DNC List: Y)
- [ ] Campaign DNC lists enabled where needed (Admin > Campaigns > [Campaign] > DNC/Filtering > Use Campaign DNC List: Y)
- [ ] DNC disposition auto-adds to internal DNC list on every campaign
- [ ] Established Business Relationship (EBR) exemptions tracked -- 18 months for transactions, 3 months for inquiries

### Calling Hours and Timezone

- [ ] Local Call Time set on all campaigns (not just Call Time -- Local Call Time enforces timezone-aware dialing)
- [ ] `state_call_time` entries configured for Florida (8PM), Oklahoma (8PM), Washington (8PM), and Texas (noon Sunday)
- [ ] Postal code-based timezone lookup enabled where ZIP codes are available (more accurate than area code for ported numbers)
- [ ] DST transitions tested and verified (see our [timezone dialing guide](/blog/vicidial-timezone-dialing-tcpa/) for the edge cases)

### Abandon Rate and Safe Harbor

- [ ] Adaptive Dropped Percentage set to 2.5% or lower (safety margin below the 3% legal max)
- [ ] Drop Percentage Max set to 3.0% (emergency brake)
- [ ] Safe Harbor Audio configured and enabled (plays within 2 seconds of abandoned call -- must include company name, callback number, and purpose of call)
- [ ] 30-day rolling abandon rate monitored weekly per campaign via VICIdial reports or the query below

**Quick check -- 30-day rolling abandon rate per campaign:**

```sql
SELECT
    campaign_id,
    COUNT(CASE WHEN status = 'DROP' THEN 1 END) AS drops_30d,
    COUNT(CASE WHEN status NOT IN ('NA','B','DC','N','AFTHRS','XDROP')
          THEN 1 END) AS answered_30d,
    ROUND(
      COUNT(CASE WHEN status = 'DROP' THEN 1 END) /
      NULLIF(COUNT(CASE WHEN status NOT IN
        ('NA','B','DC','N','AFTHRS','XDROP') THEN 1 END), 0)
      * 100, 2
    ) AS drop_pct
FROM vicidial_log
WHERE call_date >= DATE_SUB(NOW(), INTERVAL 30 DAY)
GROUP BY campaign_id;
```

Any campaign showing above 3.0% needs immediate attention -- reduce `Adaptive Dropped Percentage` to 1.5% and lower `Adaptive Maximum Dial Level` until the 30-day window rolls clean.

### Recording and Caller ID

- [ ] ALLFORCE recording enabled for all-party consent states (California, Connecticut, Delaware, Florida, Illinois, Maryland, Massachusetts, Montana, Nevada, New Hampshire, Oregon, Pennsylvania, Washington)
- [ ] Recording disclosure played at call start for two-party consent states
- [ ] STIR/SHAKEN A-level attestation confirmed with your SIP provider for all outbound DIDs
- [ ] Caller ID transmitting real, dialable numbers -- no spoofed or invalid CIDs

---

## VICIdial-Specific Configuration Quick Reference

Every compliance-critical setting, accessible through the admin GUI. No command-line access needed, no database edits -- everything through the web interface.

### Campaign-Level Settings

**Path: Admin > Campaigns > [Campaign Name] > Campaign Detail**

| Setting | Path | Compliant Value | Purpose |
|---------|------|----------------|---------|
| Local Call Time | Campaign Detail | Your TCPA-compliant call_time | Timezone-aware hour enforcement |
| Use Internal DNC | DNC/Filtering | Y | System-wide opt-out blocking |
| Use Campaign DNC | DNC/Filtering | Y | Campaign-level opt-out blocking |
| Adaptive Dropped Pct | Dialing | 2.5% | Abandon rate target |
| Drop Pct Max | Dialing | 3.0% | Hard ceiling on drops |
| Safe Harbor Audio | Dialing | [your file] | Compliance message on drops |
| Safe Harbor Message on Drop | Dialing | Y | Enable safe harbor playback |
| Drop Action | Dialing | AUDIO | Play message vs. disconnect |

### Call Time Configuration

**Path: Admin > Call Times**

Create a federal-compliant call time:
```
Call Time ID: TCPA_FEDERAL
Monday-Friday: 08:00 - 21:00
Saturday: 08:00 - 21:00
Sunday: 08:00 - 21:00
```

Create state-specific overrides:
```
State Call Time ID: TCPA_FL
State: FL
Monday-Saturday: 08:00 - 20:00
Sunday: 08:00 - 20:00

State Call Time ID: TCPA_OK
State: OK
Monday-Saturday: 08:00 - 20:00
Sunday: 08:00 - 20:00

State Call Time ID: TCPA_TX
State: TX
Monday-Saturday: 08:00 - 21:00
Sunday: 12:00 - 21:00
```

Assign the federal call time as the campaign's `Local Call Time`, and add state call times as `State Call Time` entries.

**Audit query -- find calls placed outside legal hours (catches timezone misconfigurations):**

```sql
SELECT
    vl.campaign_id,
    vl.phone_number,
    vl.call_date,
    vl.status,
    vpc.state,
    vpc.GMT_offset,
    CONVERT_TZ(vl.call_date, 'US/Eastern',
      CASE vpc.GMT_offset
        WHEN '-5' THEN 'US/Eastern'
        WHEN '-6' THEN 'US/Central'
        WHEN '-7' THEN 'US/Mountain'
        WHEN '-8' THEN 'US/Pacific'
        ELSE 'US/Eastern'
      END) AS lead_local_time
FROM vicidial_log vl
JOIN vicidial_list vli ON vl.lead_id = vli.lead_id
JOIN vicidial_phone_codes vpc
  ON vli.phone_code = vpc.country_code
  AND LEFT(vli.phone_number, 3) = vpc.areacode
WHERE vl.call_date >= DATE_SUB(NOW(), INTERVAL 7 DAY)
  AND vl.status NOT IN ('NA','B','DC','N')
HAVING HOUR(lead_local_time) >= 21
    OR HOUR(lead_local_time) < 8
LIMIT 100;
```

If that query returns rows, you have calls landing outside the federal 8AM-9PM window. For states with 8PM cutoffs (FL, OK, WA), adjust the `HAVING` clause to `>= 20` and filter on the state column.

### DNC List Import

**Path: Admin > DNC Lists > Load New DNC File**

1. Download the federal DNC list from [telemarketing.donotcall.gov](https://telemarketing.donotcall.gov) (requires registration and annual fee)
2. Format as a text file with one phone number per line, 10 digits, no formatting
3. Upload via the admin DNC import (Admin > DNC Lists > Load New DNC File)
4. Select "Internal DNC" as the target list
5. Repeat every 31 days maximum -- set a calendar reminder

For state DNC lists, follow the same process but note that each state has its own registration portal and data format.

### Revocation Workflow in VICIdial

When an agent receives a verbal opt-out request on a live call:

1. Agent selects the **DNC** disposition (or whatever disposition you've mapped to auto-DNC)
2. VICIdial automatically adds the number to the campaign DNC list
3. For blanket revocation compliance (post-April 11, 2026), you also need the number on the **Internal DNC list**

To ensure verbal revocations land on the internal DNC (not just campaign DNC):

- **Path:** Admin > Campaigns > [Campaign] > DNC/Filtering
- Set **DNC Internal DNC** to **Y** -- this makes DNC dispositions add to both the campaign DNC and the system-wide internal DNC

If your VICIdial version supports it, configure the DNC disposition to trigger both lists simultaneously. If not, set up a nightly script that syncs campaign DNC entries to the internal DNC list. Check with your VICIdial admin for the exact approach that works with your version.

**Revocation audit -- find numbers that got DNC'd on one campaign but were still dialed on another (this is exactly what plaintiff attorneys look for after April 11):**

```sql
SELECT
    vl.phone_number,
    vl.campaign_id AS dialed_campaign,
    vl.call_date AS dialed_at,
    dnc_log.campaign_id AS dnc_campaign,
    dnc_log.dnc_date AS opted_out_at
FROM vicidial_log vl
JOIN (
    SELECT phone_number, campaign_id,
           MIN(call_date) AS dnc_date
    FROM vicidial_log
    WHERE status = 'DNCL'
    GROUP BY phone_number, campaign_id
) dnc_log ON vl.phone_number = dnc_log.phone_number
WHERE vl.call_date > dnc_log.dnc_date
  AND vl.campaign_id != dnc_log.campaign_id
  AND vl.call_date >= DATE_SUB(NOW(), INTERVAL 90 DAY)
ORDER BY vl.call_date DESC
LIMIT 50;
```

If this returns results, those are potential violations under the blanket revocation rule. Each row represents a call placed to a number that had already opted out on a different campaign.

---

## What's Coming Next: The FCC NPRM and What to Watch For

The October 2025 FNPRM is still pending final action. These are the proposed changes worth tracking -- and how to prepare for each.

### Possible Elimination of Abandon Rate Rules

The FCC proposed removing the 15-second ring requirement and possibly the 3% abandon rate safe harbor for predictive dialers. If adopted, this would remove a major compliance constraint on predictive dialing operations.

**How to prepare:** Don't change your settings yet. Keep your 2.5% target and safe harbor message. If the rule is adopted, you'll have a transition period. If it isn't, you'll be glad you didn't relax your controls.

### Possible Narrowing of Blanket Revocation

The FCC acknowledged that the "revoke-all" approach may prevent consumers from receiving wanted communications. They may carve out exceptions for appointment reminders, fraud alerts, and account notifications.

**How to prepare:** Implement blanket revocation by April 11 anyway. If the FCC later carves out exceptions, you can relax your configuration. It's much harder to retroactively tighten compliance than to start strict and loosen.

### Caller ID and Call Branding

The FNPRM proposed mandatory display of verified caller names alongside A-level STIR/SHAKEN attestation. This would make it much harder to call from anonymous or unverified numbers.

**How to prepare:** Make sure your SIP provider supports STIR/SHAKEN with A-level attestation for every outbound DID. See our [STIR/SHAKEN implementation guide](/blog/stir-shaken-vicidial-guide/) for the full setup.

---

## The Real-World Cost of Getting This Wrong

Let me walk through three scenarios I've seen firsthand.

### Scenario 1: The Timezone Mistake

A 30-seat outbound shop running solar campaigns. Their VICIdial instance had `Local Call Time` set to the server's timezone (Central), not the lead's timezone. They dialed a batch of New York leads at 9:30 PM Central -- 10:30 PM Eastern. A plaintiff attorney bought the call records during discovery and found 847 calls placed after 9 PM Eastern over a three-month period.

Settlement: $380,000. The fix would have taken 15 minutes in the admin GUI.

### Scenario 2: The Lead Vendor Problem

A home improvement lead buyer purchasing from an aggregator. The aggregator's form said "By submitting, you consent to be contacted by our partners." No company names listed. When the operator got sued, they couldn't produce consent records showing the consumer had authorized their calls. The aggregator's "consent" didn't meet the TCPA's prior express written consent standard for prerecorded or autodialed calls.

Settlement: $1.2 million across a class of 4,000 consumers.

### Scenario 3: The DNC Failure

An insurance telemarketer scrubbing against the federal DNC list quarterly instead of every 31 days. During the gap between scrubs, they dialed 2,300 numbers that had been added to the DNC registry. When sued, they couldn't demonstrate the 31-day scrub cadence required by the TSR.

Legal fees to settle: $280,000. Annual cost of monthly DNC scrubbing: about $2,000.

---

## The Compliance Mindset Shift

Nobody talks about this in compliance articles: the TCPA isn't just a regulatory checklist. It's a business risk framework.

Every setting in your dialer, every lead source you buy from, every agent who takes a call -- they're all potential violation vectors. The companies that don't get sued aren't the ones with the biggest legal teams. They're the ones that built compliance into their daily operations:

- Quarterly audits against every item on the checklist above
- Weekly monitoring of abandon rates and calling hour logs
- Documented consent for every lead, retrievable within hours
- Multi-channel revocation processing that actually works
- State-specific configurations that actually reflect current law

That's not glamorous work. It's the operational blocking and tackling that keeps you out of the 3,200-lawsuits-per-year grinder.

---

## Need Help Getting Compliant?

If you're running VICIdial and your compliance configuration hasn't been audited since the 2025 rule changes, you're exposed. We've seen operations running on default settings that violate three or four TCPA provisions simultaneously -- not out of malice, just because nobody went through the admin GUI systematically.

**ViciStack's call center optimization service** includes a full TCPA compliance audit as part of our engagement. We go through every campaign setting, every DNC configuration, every call time entry, and every consent workflow. We identify the gaps, fix them, and document the changes for your compliance records.

The offer: **increase your call center conversions by 50% in 2 weeks, or you don't pay.** $5K total ($1K down, $4K on completion), with $1,500/month continuity for ongoing optimization. The compliance audit comes bundled -- because there's no point optimizing a dialer that's going to get shut down by a lawsuit.

[Talk to us about getting your operation compliant and converting better.](/contact/)

---

*This post reflects TCPA regulations and FCC rulings as of March 2026. Laws change. Consult with a telecommunications attorney for advice specific to your operation. This is not legal advice -- it's the operational reality of running compliant outbound campaigns.*

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/tcpa-compliance-2026).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
