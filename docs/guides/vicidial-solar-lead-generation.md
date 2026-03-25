# VICIdial for Solar Lead Generation: The Complete Guide

**Everything you need to configure VICIdial for solar lead generation — from campaign settings and dialing patterns to TCPA landmines and cost-per-appointment benchmarks. Built from real solar campaign data.**

---

Solar lead generation is one of the most competitive outbound calling verticals in the United States. The math is simple: the average residential solar installation generates $15,000-$40,000 in revenue, which means a qualified appointment is worth $200-$800 to the installer. That margin supports aggressive dialing operations — and it's why solar call centers are some of the highest-volume VICIdial deployments we manage.

It's also one of the most regulated. Solar calling intersects with the [TCPA](/glossary/tcpa/), state mini-TCPAs, the FTC's Telemarketing Sales Rule, state-specific solar regulations, and a patchwork of do-not-call requirements that change faster than most compliance teams can track. Get it wrong, and the penalties aren't hypothetical — TCPA lawsuits in the solar industry hit record numbers in 2024-2025, with per-violation damages of $500-$1,500 stacking up across thousands of calls.

This guide covers everything: VICIdial campaign configuration optimized for solar, [DID management](/blog/vicidial-did-management/) strategies for local presence dialing, compliance frameworks, lead recycling, CRM integration, and the actual cost-per-appointment numbers you should be benchmarking against. For the solar industry overview, see our [solar industry page](/industries/solar/).

---

## Why Solar Lead Gen Is Different From Other Verticals

Before diving into VICIdial settings, you need to understand what makes solar fundamentally different from debt settlement, insurance, home services, or other outbound verticals:

**1. The decision maker is a homeowner.** Not a business. Not a renter. A homeowner with a roof that faces the right direction, lives in a service area, and has a utility bill high enough to make solar economics work. This means your lead data quality requirements are higher than most verticals — you're filtering for property ownership, roof type, utility provider, and credit profile before anyone picks up the phone.

**2. The sale is consultative and local.** Solar isn't a "close on the first call" product for most markets. The typical solar sales cycle involves an initial qualification call, a virtual or in-person consultation, a site survey, a proposal, and then a close. Your VICIdial campaign is generating the appointment — not closing the deal. This changes how you measure success: cost per appointment (CPA) and appointment-to-close rate are your primary KPIs, not cost per sale.

**3. Seasonality is real.** Solar calling volumes follow predictable seasonal patterns. Spring and early summer (March-June) are peak seasons — homeowners are getting higher utility bills, longer days make solar feel tangible, and installers have capacity. Winter months (November-February) see lower contact rates and higher CPA because homeowners aren't thinking about their electric bill. Your VICIdial campaign configurations should shift with the seasons.

**4. Local trust matters enormously.** Solar is a home improvement product. Homeowners are inherently suspicious of out-of-area calls about their roof. Local presence dialing — where your outbound caller ID shows a local area code — isn't a nice-to-have in solar. It's the difference between a 4% contact rate and an 8% contact rate. We'll cover the DID strategy in detail below.

**5. The regulatory environment is a minefield.** Solar calling is under more scrutiny than almost any other outbound vertical. The [FCC's one-to-one consent rule](/blog/vicidial-tcpa-compliance/) (effective January 2025) specifically targeted lead generators in home improvement verticals — including solar. State attorneys general in California, Florida, Texas, and New York have all increased enforcement against solar telemarketing in 2025-2026.

---

## VICIdial Campaign Configuration for Solar

### Campaign Type and Dial Method

For solar lead generation, your primary campaign should be configured as:

- **Campaign Type:** Outbound
- **Dial Method:** RATIO or ADAPT_TAPERED
- **Auto Dial Level:** Start at 1.5-2.0, adjust based on contact rate

**Why RATIO or ADAPT_TAPERED instead of pure predictive?** Solar lead lists tend to have lower contact rates than B2B lists (you're calling residential numbers during the day when many homeowners are at work). Pure predictive mode can over-dial when contact rates are low, leading to excessive abandoned calls. RATIO mode gives you direct control over the dial ratio. ADAPT_TAPERED starts at a lower dial level and gradually increases as the system learns the list's contact rate, reducing early-campaign abandons.

For experienced operations with well-scrubbed lists and stable contact rates (typically the second or third pass through a list), switching to full [ADAPT_AVERAGE_BALANCE](/settings/power-dial-mode/) predictive dialing is appropriate. First passes through new lists should use RATIO at 1.5-2.5 until you have baseline data.

### Dial Level Settings

```
Adaptive Maximum Dial Level: 3.0
Adaptive Dropped Percentage: 2.5
Adaptive Maximum Abandoned: 2.5
Dial Timeout: 26 seconds
```

**Adaptive Maximum Dial Level at 3.0** keeps the dialer from getting overly aggressive. Solar residential lists have variable answer patterns — you might hit a stretch of numbers where nobody answers (homeowners at work), then suddenly get a cluster of pickups. Capping at 3.0 prevents the dialer from spinning up to 5x or 6x during low-contact stretches and then slamming agents with simultaneous connects when the list warms up.

**Adaptive Dropped Percentage at 2.5%** gives you margin below the TCPA-mandated [3% abandon rate maximum](/glossary/abandon-rate/). The TCPA measures abandon rate over a 30-day rolling period per campaign. Running at 2.5% gives you buffer for statistical variance. Some operations run at 2.0% for extra safety. Never run above 2.8% — one bad hour can push your 30-day average over the line.

**Dial Timeout at 26 seconds** is optimized for residential numbers. Residential voicemail systems typically pick up at 25-30 seconds. Setting timeout at 26 catches most live answers while avoiding voicemail greetings, which waste AMD processing time and can generate false positive dispositions.

### AMD Configuration for Solar

AMD in solar campaigns requires careful calibration because residential answering patterns differ from business lines. See our [complete AMD guide](/blog/vicidial-amd-guide/) for the full technical breakdown, but here are the solar-specific recommendations:

```
AMD Type: AMD
AMD Send to Agent: Y (for HUMAN results)
AMD Inbound: N
```

**For the AMD parameters** (configured in the Asterisk dialplan or via Settings Container):

```
initialSilence: 2500
greeting: 2000
afterGreetingSilence: 1200
totalAnalysisTime: 5000
minimumWordLength: 120
betweenWordsSilence: 50
maximumNumberOfWords: 5
silenceThreshold: 256
```

Key adjustments from default:

- **afterGreetingSilence increased to 1200ms** (from the typical 1000ms). Residential callers often have a longer pause after saying "Hello?" before they speak again. The extra 200ms reduces false MACHINE classifications on legitimate homeowner answers.
- **maximumNumberOfWords increased to 5** (from 4). Some homeowners answer with longer greetings: "Hello, this is the Johnson residence" is 6 words. Setting this to 5 catches most of those without losing too many actual voicemail greetings to false HUMAN classifications.

**Important:** If you're running campaigns in markets with high cell phone penetration (essentially all markets in 2026), expect AMD accuracy to be lower. Cell phone audio characteristics (variable latency, background noise, hands-free audio) confuse the silence-based algorithm. Monitor your AMD HUMAN → agent disconnect rate. If agents are getting more than 15% machine/dead connects on AMD-classified HUMAN calls, your AMD settings need adjustment.

### Call Time and Time Zone Configuration

Solar calling hours should be configured precisely in VICIdial's [call time settings](/settings/call-timeout/):

**Optimal solar calling windows:**

| Day | Hours (Local Time) | Why |
|---|---|---|
| Monday-Thursday | 10:00 AM - 1:00 PM | Catch homeowners on lunch, retired homeowners at home |
| Monday-Thursday | 4:30 PM - 7:30 PM | After-work window, highest contact rates |
| Friday | 10:00 AM - 5:00 PM | Shorter evening window (people go out) |
| Saturday | 10:00 AM - 4:00 PM | Weekend homeowners (strong contact rates) |
| Sunday | Noon - 5:00 PM | Limited calling, some states restrict |

**VICIdial call time configuration:**

Create a custom Call Time in Admin → Call Times with the appropriate start/stop hours for each day. Assign this call time to your solar campaigns.

**Time zone enforcement** is critical for solar. VICIdial uses the [Hopper](/glossary/hopper/) to manage time-zone-aware dialing. Ensure these settings are correct:

- **Local Call Time:** Set to the call time you created above
- **Hopper level:** 200-500 (enough to keep the dialer fed without loading numbers that won't be callable for hours)
- **Time Zone Checking:** ENABLED — this cross-references lead phone numbers against the [phone_code and gmt_offset fields](/settings/timezone-filtering/) to ensure you're only dialing numbers within legal calling hours for their time zone

**Seasonal adjustments:** During peak solar season (March-June), extend your evening window to 8:00 PM local time where legally permitted. Homeowners are more receptive when it's light outside and they can look at their roof. During winter months (November-February), tighten to core hours only — contact rates drop and cost-per-dial increases, so you want to concentrate agent time in the highest-probability windows.

---

## DID Management for Solar: Local Presence Is Everything

If there's one single factor that separates profitable solar campaigns from money-losing ones, it's local presence dialing. The data is unambiguous:

- **Calls from local area codes:** 7-12% answer rate
- **Calls from toll-free numbers:** 3-5% answer rate
- **Calls from out-of-area numbers:** 2-4% answer rate

That's a 2-3x difference in contact rate from the same list, same agents, same script. In a business where cost per appointment runs $15-$80, doubling your contact rate can cut your CPA in half.

### DID Rotation Strategy

VICIdial's [DID management system](/blog/vicidial-did-management/) supports caller ID rotation at the campaign level. Here's the optimal setup for solar:

**1. Acquire DIDs for every target market.** For each metro area you're calling, purchase 5-10 local DIDs from your SIP provider. If you're calling the Phoenix metro area, you need 480, 602, and 623 area code numbers. If you're calling the Dallas-Fort Worth market, you need 214, 469, 817, 682, and 972 numbers.

**2. Configure CID Group Rotation.** In VICIdial Admin → Caller ID Groups, create a CID group for each target market. Add your local DIDs to each group. Set the rotation method to:

```
CID Group Type: AREACODE
Rotation: SEQUENTIAL (not RANDOM)
```

AREACODE-based CID groups automatically match the outbound caller ID to the area code of the number being dialed. This is the simplest and most effective local presence implementation.

**3. Manage DID reputation aggressively.** Solar DIDs get flagged as spam faster than almost any other vertical because of the high dial volume and consumer complaint rates. Implement:

- **Per-DID daily limits:** Cap each DID at 75-100 outbound calls per day. High-volume DIDs trigger carrier analytics flags.
- **DID health monitoring:** Use a spam reputation checking service (CallerID Reputation, Numeracle, Free Caller Registry) to monitor flag status weekly.
- **Rotation on flag detection:** The moment a DID shows up as "Spam Likely" or "Scam Likely" on any major carrier analytics database, pull it from rotation. Let it rest for 30-60 days or replace it.
- **Inbound functionality:** Every outbound DID should route to a voicemail or IVR that identifies your company. Homeowners call back solar numbers — if they reach a dead line, they'll report it as spam. Configure VICIdial inbound routes for all outbound DIDs.

**4. STIR/SHAKEN attestation.** Ensure your SIP provider delivers A-level [STIR/SHAKEN](/blog/stir-shaken-vicidial-guide/) attestation on your solar DIDs. B or C attestation will result in "Spam Likely" labeling on major carriers within days of high-volume dialing. This is non-negotiable for solar.

---

## TCPA Compliance for Solar Campaigns

Solar lead generation is ground zero for TCPA enforcement. The FCC and plaintiff attorneys have specifically targeted solar telemarketing in recent enforcement actions. Here's what you must configure in VICIdial to stay compliant.

### The FCC One-to-One Consent Rule (January 2025)

This rule changed everything for solar lead generators. Previously, a consumer could fill out a web form and their consent could be shared with multiple solar companies (the "comparison shopping" model). Under the new rule, consent must be:

- **One-to-one:** Each consent authorizes calls from only one specific seller
- **Topically related:** The consent must relate to the specific product/service being marketed
- **Clear and conspicuous:** The consent language must be clear to the consumer
- **Logged and retrievable:** You must be able to produce the specific consent record for any called number

**VICIdial implementation:** Your lead intake system must capture and store the consent record (timestamp, IP address, specific consent language, the specific seller authorized). This data should be loaded into VICIdial as custom fields on the lead record. Configure your VICIdial lead loading process to reject any lead that doesn't include a valid consent record.

### State-Specific Rules That Affect Solar Dialing

**Florida FTSA (Florida Telephone Solicitation Act):**
- Maximum 3 calls per 24-hour period to the same residential number
- Calling hours: 8:00 AM - 8:00 PM local time (stricter than federal)
- Requires registration with the Florida Do Not Call list
- VICIdial config: Set [Auto Alt Dial Limit](/settings/auto-alt-dial/) to 3 and use the daily call limit feature in your filter settings for Florida area codes

**California (CCPA + state telemarketing laws):**
- Opt-out must be processed within 15 business days
- Written consent required for automated calls to cell phones
- California's CCPA applies to the personal data you collect during solar calls
- VICIdial config: Maintain a California-specific internal DNC list; process removals within the required timeframe

**Texas SB 140:**
- Requires registration with the Texas Secretary of State before telemarketing in Texas
- $10,000 per violation penalties
- VICIdial config: Ensure your company is registered before loading any Texas leads; maintain proof of registration

**Virginia (10-year opt-out requirement):**
- Once a Virginia consumer opts out, you must maintain that opt-out for 10 years
- VICIdial config: Your [internal DNC list](/glossary/dnc/) must be persistent and maintained across system migrations. Do not purge DNC entries for Virginia numbers.

**Oklahoma OTSA (Oklahoma Telephone Solicitation Act):**
- Registration required; strict calling hour restrictions
- VICIdial config: Similar to Florida, configure separate call times for Oklahoma area codes

Configure VICIdial's campaign settings to enforce these state-specific rules:

```
Drop Percentage Maximum: 2.5 (federal 3% with safety margin)
Safe Harbor Message: Y (plays a recorded message for abandoned calls)
Safe Harbor Audio File: [your compliance recording]
DNC List: [your internal DNC list IDs]
```

For detailed compliance configuration across all VICIdial settings, see our [TCPA compliance checklist](/blog/vicidial-tcpa-compliance/).

---

## Lead Management and Recycling for Solar

Solar leads have a longer lifecycle than most outbound verticals. A homeowner who doesn't answer today might be a qualified appointment next week. Your lead recycling strategy directly impacts CPA.

### Lead Source Hierarchy

Not all solar leads are created equal. Structure your VICIdial campaigns to prioritize by lead quality:

| Lead Source | Typical CPA | Priority |
|---|---|---|
| Inbound web form (exclusive) | $15-$30 | Highest — dial within 5 minutes |
| Shared web form (aged 0-24 hrs) | $25-$50 | High — dial same day |
| Purchased list (qualified homeowner) | $40-$80 | Medium — standard campaign |
| Aged leads (7-30 days) | $20-$45 | Medium — second/third pass |
| Aged leads (30-90 days) | $30-$60 | Lower — recycled campaign |
| Cold list (homeowner filter only) | $60-$120 | Lowest — prospecting campaign |

**VICIdial implementation:** Create separate campaigns or list groups for each lead tier. Use VICIdial's [list mix](/settings/list-order/) or [hopper priority](/settings/hopper-level/) settings to weight higher-quality leads. Fresh exclusive web leads should be in a speed-to-lead campaign with agents dedicated to immediate response.

### Speed-to-Lead Configuration

For inbound web form leads, speed is everything. The probability of qualifying a solar lead drops by 80% after the first 5 minutes. Configure a dedicated VICIdial speed-to-lead campaign:

```
Dial Method: RATIO
Auto Dial Level: 1.0
Lead Filter: [web_form_leads_last_30_minutes]
Hopper Level: 50
Drop Percentage: 0 (no drops — every call gets an agent)
```

Use VICIdial's API to load leads in real-time as web forms are submitted. The Non-Agent API `add_lead` function with `duplicate_check=DUPLIST` and `hopper_priority=99` ensures the lead enters the hopper immediately at highest priority.

### Recycling Rules

Configure disposition-based recycling to maximize lead utilization without over-calling:

| Disposition | Recycle After | Max Attempts | Notes |
|---|---|---|---|
| NA (No Answer) | 2 hours | 6 attempts | Vary time of day |
| B (Busy) | 30 minutes | 4 attempts | Likely screening |
| AM (Answering Machine) | 4 hours | 3 attempts | Leave VM on 2nd attempt |
| NI (Not Interested) | 14 days | 2 attempts | Different agent, different pitch |
| CB (Callback) | Scheduled time | 3 attempts | Agent-specific callback |
| DNC (Do Not Call) | Never | N/A | Add to internal DNC |
| NQ (Not Qualified) | Never | N/A | Wrong homeowner type, renter, etc. |

Configure these in VICIdial under Campaign → Disposition Recycle settings. The key is varying the time of day across attempts — if someone didn't answer at 11 AM on Tuesday, try 5:30 PM on Wednesday.

---

## Solar CRM Integration

VICIdial doesn't replace your solar CRM — it feeds it. The integration between VICIdial and your solar-specific CRM is where appointments become proposals become installations.

### Common Solar CRM Integrations

**Salesforce:** The most common integration for mid-to-large solar operations. Use VICIdial's API to push qualified appointments into Salesforce as Leads or Opportunities. Key fields to sync: homeowner name, address, phone, utility provider, average bill, roof type (if captured), appointment date/time, agent notes.

VICIdial's custom fields can be mapped to Salesforce fields via a middleware script (PHP, Python, or a tool like Zapier/Make). The typical flow:

1. Agent qualifies the homeowner in VICIdial
2. Agent dispositions the call as SET (appointment set)
3. VICIdial webhook or API trigger sends lead data to Salesforce
4. Salesforce creates an Opportunity and assigns to the solar sales team
5. Sales team confirms the appointment and runs the consultation

**Aurora Solar / EnergySage:** These platforms handle proposal generation and solar design. Integration is typically one-directional — VICIdial feeds qualified leads into the proposal pipeline. Use their APIs to push appointment data after disposition.

**GoHighLevel (GHL):** Increasingly popular with smaller solar operations. VICIdial can push leads via GHL's API, and GHL handles nurture sequences (SMS follow-up, email drips) for leads that didn't convert on the first call.

### VICIdial API Integration Example

Here's the typical API flow for pushing a set appointment to your CRM:

```bash
# VICIdial Non-Agent API call to retrieve lead data after SET disposition
curl "https://your-vicidial-server/vicidial/non_agent_api.php?\
source=crm_sync&\
function=lead_field_info&\
user=api_user&\
pass=api_pass&\
lead_id=12345678&\
custom_fields=Y"
```

Your middleware parses the response and pushes to your CRM. For real-time integration, configure a VICIdial URL that fires on specific dispositions — under Campaign → Dispo Call URL, set a URL that triggers when agents disposition as SET or CALLBACK.

```
Dispo Call URL: https://your-middleware.com/vicidial-webhook?lead_id=--A--lead_id--B--&status=--A--dispo--B--&phone=--A--phone_number--B--&agent=--A--user--B--
```

The `--A--` and `--B--` tags are VICIdial's variable substitution markers — they get replaced with actual values when the URL fires.

---

## Campaign Settings Recommendations: The Complete Configuration

Here's a consolidated VICIdial campaign configuration for a solar lead generation operation. These are starting points — adjust based on your specific list quality, market, and agent skill level.

### Primary Solar Campaign Settings

| Setting | Value | Reason |
|---|---|---|
| Dial Method | ADAPT_TAPERED | Safe ramp-up, adjusts to list quality |
| Auto Dial Level | 1.5 (starting) | Conservative start for new lists |
| Adaptive Max Level | 3.0 | Prevents over-dialing |
| Adaptive Drop % | 2.5 | Below 3% TCPA threshold |
| Dial Timeout | 26 sec | Catches answers, avoids most VMs |
| AMD | ON (with tuned parameters) | Filters machines for agent efficiency |
| Drop Percentage Max | 3.0 | Hard ceiling |
| Safe Harbor | ENABLED | Required for compliance |
| Local Caller ID | AREACODE CID Group | Critical for solar contact rates |
| Call Time | Custom solar hours | Market-optimized windows |
| Answering Machine Message | Y | Drop VM on machine detection |
| DNC Checking | CAMPAIGN AND SYSTEM | Checks both internal lists |
| Hopper Level | 300 | Keeps dialer fed |
| Lead Filter | Homeowner qualification filter | Filters unqualified leads pre-dial |
| Lead Recycling | Disposition-based (see table above) | Maximizes list utilization |
| Auto Alt Dial | ALT_AND_ADDR3 | Tries alt phone numbers |
| Auto Alt Dial Limit | 3 per number per day | Florida FTSA compliance |

### Agent Interface Configuration

Configure the VICIdial agent screen for solar-specific workflow:

- **Custom fields visible on agent screen:** Homeowner name, property address, utility provider, average monthly bill, roof type, credit range
- **Script:** Solar qualification script with branching logic (homeowner Y/N → utility provider → bill amount → roof condition → appointment pitch)
- **Disposition options:** SET, CB (Callback), NI (Not Interested), NQ (Not Qualified - Renter), NQ2 (Not Qualified - Low Bill), NQ3 (Not Qualified - Bad Roof), DNC, NAS (Not Available - Schedule callback)
- **Transfer options:** Configure warm transfer to senior closer or appointment confirmation team

---

## Cost-Per-Appointment Benchmarks for Solar

Based on aggregated data from solar lead generation operations running VICIdial, here are the CPA benchmarks you should be measuring against. These assume a VICIdial deployment with proper local presence, AMD configuration, and TCPA-compliant dialing practices.

| Metric | Bottom Quartile | Median | Top Quartile |
|---|---|---|---|
| Cost per appointment (exclusive web leads) | $35-$50 | $20-$35 | $12-$20 |
| Cost per appointment (aged/shared leads) | $65-$90 | $40-$65 | $25-$40 |
| Cost per appointment (cold list) | $100-$150 | $70-$100 | $45-$70 |
| Contact rate (local presence) | 5-7% | 7-9% | 9-14% |
| Contact rate (no local presence) | 2-4% | 4-6% | 6-8% |
| Qualification rate (contacts to appointments) | 5-10% | 10-18% | 18-28% |
| Appointment show rate | 50-65% | 65-78% | 78-90% |
| Agent appointments per hour | 0.3-0.6 | 0.6-1.2 | 1.2-2.0 |

For a deeper dive into cost benchmarks across all outbound verticals, see our [cost per lead benchmarks analysis](/blog/call-center-cost-per-lead-benchmarks/).

**What separates top-quartile operations from the rest:**

1. **Speed-to-lead under 5 minutes** for web form leads
2. **Local presence dialing** with clean DID reputation
3. **Properly tuned AMD** reducing wasted agent time on machines by 35-50%
4. **Aggressive lead recycling** with time-of-day variance
5. **Quality lead data** — pre-filtered for homeownership, property type, utility provider
6. **Strong scripts** with branching qualification logic
7. **Agent compensation tied to appointment quality** (not just quantity)

---

## Seasonal Campaign Adjustments

Solar calling is seasonal. Your VICIdial configuration should adapt:

### Peak Season (March - June)

- Increase agent staffing 25-40%
- Extend evening calling hours to 8:00 PM where legally permitted
- Increase dial levels (lists are more responsive)
- Focus on new/exclusive leads (highest conversion rates in peak)
- Increase DID pool size (more volume = faster DID burnout)
- Tighten appointment windows (installers have capacity, close the loop fast)

### Summer (July - August)

- Maintain steady volume (still good season but homeowners travel)
- Emphasize "beat the rate hike" messaging (utility rates typically increase in summer)
- Saturday calling is productive (homeowners doing yard work, thinking about their home)
- Watch for utility company incentive deadlines — these create urgency

### Fall Transition (September - October)

- Begin reducing agent count 10-20%
- Focus on lead recycling and aged leads (lower cost, longer nurture)
- Shift messaging to "lock in before year-end tax credits"
- Federal tax credit (30% ITC) deadlines create Q4 urgency

### Winter (November - February)

- Minimum viable staffing for calling operations
- Heavy focus on aged lead recycling (low cost per dial)
- Use this period for data cleanup, DNC scrubbing, list preparation for spring
- Train new agents during slow months so they're ready for March ramp-up
- Negotiate better SIP rates during low-volume months (carriers are more flexible)

---

## Integrating VICIdial With Solar-Specific Tools

### Proposal and Design Software

After VICIdial generates the appointment, the solar sales workflow moves to proposal tools:

- **Aurora Solar:** Cloud-based solar design platform. Push lead address data from VICIdial to Aurora's API to pre-generate roof models before the consultation.
- **Helioscope:** Alternative design tool popular with commercial solar. Less relevant for residential lead gen but worth noting for operations that do both.
- **OpenSolar:** Free design tool that some smaller installers use. Manual data entry from VICIdial dispositions.

### Utility Bill Analysis

- **UtilityAPI:** Automated utility bill retrieval. If you capture the homeowner's utility provider and account number during the VICIdial qualification call, you can push this to UtilityAPI to automatically pull 12 months of usage data for accurate proposal generation.
- **Manual upload:** Many operations have homeowners text or email a photo of their utility bill after the qualification call. Configure a VICIdial email or SMS follow-up (via Dispo Call URL to your SMS platform) to request the bill immediately after appointment set.

### Appointment Confirmation

Automate appointment confirmation to improve show rates:

1. VICIdial agent sets appointment → Dispo Call URL fires
2. Middleware sends SMS confirmation: "Your solar consultation is confirmed for [date/time]. Reply YES to confirm."
3. 24 hours before appointment: automated reminder SMS
4. 1 hour before appointment: final reminder
5. No-shows: automatically reload into VICIdial reschedule campaign

This workflow significantly improves the 65-78% median show rate toward the 78-90% top-quartile range.

---

## Common Mistakes in Solar Lead Gen Campaigns

After managing solar calling operations across dozens of VICIdial deployments, here are the mistakes we see most often:

**1. Not using local presence.** This is the single biggest CPA killer. Operations that dial from a single toll-free number or a fixed out-of-area caller ID are leaving 30-50% of their potential contacts on the table. Set up AREACODE-based CID groups. Period.

**2. Over-dialing Florida.** Florida is the largest residential solar market in the US. It's also the most heavily regulated for telemarketing (FTSA). The 3-calls-per-24-hours rule is strictly enforced. We've seen operations get hit with six-figure FTSA penalties because their VICIdial recycling settings allowed 5-6 attempts per day to Florida numbers. Configure daily attempt limits at the list or lead filter level.

**3. Ignoring AMD accuracy.** Operations that turn on AMD with default settings and never tune it are either routing 20%+ machine calls to agents (wasting time) or misclassifying 15%+ of humans as machines (losing leads). Monitor your AMD performance weekly. Check the ratio of AMD-classified HUMAN calls that agents immediately disposition as machine. If it's above 10%, your settings need adjustment.

**4. No speed-to-lead for web leads.** Fresh web form leads that sit in a queue for 30 minutes are worth half what they were at minute one. If you're buying exclusive web leads, you need a dedicated VICIdial campaign that dials them within 5 minutes of form submission. The API integration to make this work takes a day to build and pays for itself in the first week.

**5. Flat calling schedules.** Calling residential solar leads at the same hours every day of the week ignores the data on when homeowners actually answer. The 4:30-7:30 PM weekday window and 10 AM-4 PM Saturday window consistently outperform midday weekday calling by 40-60% on contact rate. Structure your agent shifts around peak contact hours.

**6. Neglecting DID health.** Solar DIDs burn out fast. If you're not monitoring spam reputation and rotating flagged numbers out of service, your contact rates will degrade week over week. Build DID health checks into your weekly operations routine. Budget for DID replacement — expect to rotate 20-30% of your DID pool every quarter in high-volume solar campaigns.

---

## Frequently Asked Questions

### What's a good cost per appointment for solar lead generation?

For exclusive web form leads run through a well-configured VICIdial operation with local presence dialing, the median CPA is $20-$35. For aged or shared leads, expect $40-$65. For cold list prospecting, $70-$100 is typical. Top-performing operations push CPAs 30-40% below these medians through optimized dialing settings, strong scripts, and aggressive lead recycling. See our [cost per lead benchmarks](/blog/call-center-cost-per-lead-benchmarks/) for cross-vertical comparison.

### How many DIDs do I need for a solar campaign?

Plan for 5-10 DIDs per target area code, with each DID capped at 75-100 outbound calls per day. If you're calling the Phoenix metro area (480, 602, 623 area codes) with 20 agents, you'd want approximately 15-30 DIDs across those area codes. This gives you enough rotation to keep reputation clean while maintaining local presence for every call.

### Is AMD worth using for solar campaigns?

Yes, with proper tuning. Solar campaigns calling residential numbers during the day hit voicemail on 40-60% of connected calls. Without AMD, agents spend nearly half their time listening to voicemail greetings. With properly tuned AMD, you recover 30-45% of that wasted agent time. The key is tuning — default AMD settings have 15-20% false classification rates on residential numbers. Use the adjusted parameters in this guide and monitor weekly.

### How do I handle the FCC one-to-one consent rule for solar?

Every lead you dial must have a consent record showing the consumer specifically authorized your company (not a lead aggregator, not "partner companies") to call them about solar. Your lead intake must capture: the specific consent language shown to the consumer, the consumer's affirmative action (checkbox, signature), the timestamp, and a clear identification of your company as the authorized caller. Store this data with the lead record in VICIdial and be prepared to produce it if challenged.

### Can I use VICIdial for solar appointment setting and sales?

VICIdial is ideal for the appointment-setting phase. For the sales/consultation phase, most operations use a separate CRM (Salesforce, GoHighLevel, etc.) that handles proposal generation, contract management, and project tracking. VICIdial feeds qualified appointments into the sales CRM via API integration. Some smaller operations handle the entire cycle in VICIdial using custom fields and dispositions, but this becomes unwieldy as you scale.

### What's the best time to call solar leads?

The highest contact rates for residential solar leads are weekday evenings (4:30-7:30 PM local time) and Saturday mornings (10:00 AM-2:00 PM). Weekday midday (10 AM-1 PM) catches retirees and work-from-home homeowners. The worst times are weekday mornings (8-10 AM, homeowners are commuting) and Sunday (lower contact rates, some state restrictions). Structure your agent shifts to maximize staffing during the 4:30-7:30 PM window — that's your highest-ROI calling time.

### How does VICIdial compare to dedicated solar dialers like PhoneBurner or Mojo?

VICIdial offers significantly more configuration control and lower per-agent costs than SaaS solar dialers. PhoneBurner and Mojo charge $100-$165/agent/month and provide limited customization. VICIdial's all-in cost is $25-$80/agent/month depending on scale, with full control over dialing algorithms, AMD settings, CID rotation, and campaign logic. The trade-off is that VICIdial requires more technical setup — you need someone who understands the system. For operations with 10+ agents, the cost savings justify the setup effort.

### What VICIdial version should I use for solar lead gen?

Use the latest SVN build on ViciBox 12.x or a clean AlmaLinux 9 installation. Recent SVN revisions include improvements to AMD handling, WebRTC agent support ([ViciPhone](/blog/vicidial-webrtc-setup/) is critical for remote agents), and API enhancements that improve CRM integration. Avoid running on outdated installations — the security and compliance features in recent builds are important for solar operations where regulatory scrutiny is high.

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-solar-lead-generation).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
