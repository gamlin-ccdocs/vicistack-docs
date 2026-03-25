# VICIdial DID Management: How to Stop Your Numbers From Getting Flagged as Spam

*Published by ViciStack — the managed VICIdial platform built by operators, for operators.*

---

Your numbers are getting flagged. You know it. Your answer rates have been sliding for months, your agents are sitting idle while the dialer churns through leads that never pick up, and you have no idea which of your 40 DIDs are the problem. You rotate them, buy new ones, burn through those too, and the cycle repeats.

This is not a mystery. It is a system — a system with specific rules, specific players, and specific levers you can pull. The problem is that most VICIdial operators are pulling the wrong levers, or worse, pulling no levers at all because they don't understand how the system works.

We run over 100 VICIdial-based contact centers. We watch DID reputation data across roughly 600,000 dials per month. We have tracked the correlation between specific VICIdial configurations and spam flag rates across every major carrier. What follows is everything we know, laid out in the order you need it.

---

## The Spam Flagging Crisis Hitting Call Centers in 2026

The numbers tell the story with brutal clarity. Hiya — the analytics engine behind AT&T's spam detection — flagged 13.7 billion suspected spam calls in Q2 2025 alone. That is roughly 150 million calls per day getting tagged before a human even decides whether to answer. By early 2026, the trend has only accelerated.

Here is what that means for your call center: **80% of consumers now avoid answering unknown calls entirely.** More than 65% of U.S. mobile users actively rely on call-screening or spam-blocking tools, and industry projections show that number exceeding 75% by mid-2026. If your outbound number shows up as "Scam Likely" or "Spam Risk" on a recipient's phone, you are not competing against other call centers for that lead's attention — you are invisible.

The financial impact is staggering. Answer rates drop 20-50% overnight following a spam labeling event. We have seen campaigns go from 22% contact rates to under 8% in a single day because a batch of DIDs got flagged simultaneously. For a 30-seat call center running a home services campaign, that translates to roughly $15,000-$25,000 per week in lost revenue from leads that never hear the phone ring.

Three forces are driving the crisis in 2026:

**1. Carrier analytics engines are getting more aggressive.** The three major analytics providers — Hiya (AT&T), First Orion (T-Mobile), and TNS/Transaction Network Services (Verizon) — are flagging calls faster and with lower thresholds than ever before. In independent testing by Numeracle, Hiya rated 32% of tested numbers as spam, TNS flagged 24%, and First Orion flagged the fewest. Each carrier's analytics engine uses different algorithms, which means a number can be clean on T-Mobile and flagged on AT&T simultaneously.

**2. Consumer reporting has compounded.** Call-blocking apps like Hiya, Nomorobo, RoboKiller, and Truecaller now have hundreds of millions of users contributing feedback. A single consumer pressing "Report Spam" on your number feeds into the analytics engine that decides your reputation for every subsequent call. Ten reports in a day can trigger a flag that takes weeks to remove.

**3. STIR/SHAKEN enforcement has created a two-tier system.** Calls with full A-level attestation get a trust advantage. Calls with B or C attestation — or worse, unsigned calls — get treated as suspicious by default. If your carrier is handing you B-level attestation because they cannot verify your relationship to the number, every dial starts at a disadvantage before any behavioral analytics even kick in. For a deep dive on attestation levels and how to ensure A-level signing, see our [STIR/SHAKEN implementation guide](/blog/stir-shaken-vicidial-guide/).

The bottom line: DID management is no longer a "nice to have" operational task. It is the single highest-leverage activity in your call center. A perfectly tuned [predictive dialer](/blog/vicidial-predictive-dialer-settings/) running against a perfectly cleaned list will produce nothing if 40% of your DIDs are flagged and calls never reach the consumer's screen.

> **Not sure how many of your DIDs are already flagged?** [Request a free DID health audit](/free-audit/) — we will scan every number in your pool across all three major carrier analytics engines and tell you exactly where you stand.

---

## How Caller ID Reputation Works: STIR/SHAKEN and Beyond

Most operators think caller ID reputation is a single score. It is not. It is a layered system with at least five independent reputation signals, each controlled by a different entity, and they do not talk to each other.

### Layer 1: STIR/SHAKEN Attestation

STIR/SHAKEN is a cryptographic call authentication framework mandated by the FCC. When your carrier originates a call, they sign it with a digital certificate and assign one of three attestation levels:

| Attestation Level | What It Means | Impact on Your Calls |
|---|---|---|
| **A — Full** | Carrier knows the customer and has verified they are authorized to use the calling number | Lowest risk of spam flags; calls treated as trusted |
| **B — Partial** | Carrier knows the customer but cannot verify authorization for the specific number | Moderate risk; some analytics engines treat this as a yellow flag |
| **C — Gateway** | Carrier does not know the caller's identity; call routed through a gateway | High risk; treated as suspicious by most terminating carriers |

As of June 2025, the FCC requires all voice service providers with a STIR/SHAKEN implementation obligation to obtain their own certificates and authenticate calls directly — third-party certificates are no longer valid. If your VoIP carrier is still using a third-party signing arrangement, your calls may be receiving B or C attestation without you knowing it.

**The critical point:** A-level attestation is necessary but not sufficient. It tells the terminating carrier "this caller is who they claim to be." It does not tell the terminating carrier "this caller is not a spammer." A fully authenticated call with A-level attestation can still get flagged as spam based on behavioral analytics.

### Layer 2: Carrier Analytics Scoring

This is where most operators lose the game. Each major wireless carrier contracts with an analytics engine that scores every inbound call in real time:

- **AT&T** uses **Hiya** for spam detection and caller ID display
- **T-Mobile** uses **First Orion** for call labeling and branded calling
- **Verizon** uses **TNS (Transaction Network Services)** for spam analytics

Each engine maintains its own scoring model based on factors including:

- **Call volume patterns**: How many calls originate from this number per hour, per day
- **Short duration call ratio**: High percentages of calls lasting under 6 seconds (immediate hangups) are a strong spam signal
- **Consumer complaints**: Direct "Report Spam" actions from recipients
- **Answer rate**: Numbers where most calls go unanswered get scored lower
- **Call frequency to unique numbers**: Calling hundreds of unique numbers per day from a single DID is a robocall signal
- **Number age and history**: Brand-new numbers with no call history start with a neutral-to-negative reputation

The scoring models are proprietary and opaque. Hiya does not publish its algorithm. First Orion does not share its thresholds. This means you cannot game the system — you have to operate within broadly understood best practices and monitor results.

### Layer 3: CNAM (Caller Name Delivery)

CNAM is the system that determines what name displays alongside your number on the recipient's phone. When you make a call, the terminating carrier performs a "CNAM dip" — a database lookup to retrieve the name associated with your number.

If your CNAM record is blank, recipients see "Unknown Caller" or just the raw number. If it shows a business name, you get a marginal trust advantage. CNAM is not a reputation signal per se, but it influences consumer behavior: calls displaying a recognizable business name get answered at higher rates than anonymous numbers.

You can register your CNAM through your carrier or through third-party services. The record propagates across CNAM databases, though not instantly — allow 48-72 hours for full propagation.

### Layer 4: Free Caller Registry

In a rare moment of industry cooperation, Hiya, First Orion, and TNS jointly launched [FreeCallerRegistry.com](https://freecallerregistry.com/) — a single portal where businesses can register their phone numbers with all three analytics engines simultaneously. Registration is free. It takes about 60 seconds per batch of up to 20 numbers. The analytics engines independently vet the information, and within 4 business days, registered numbers receive a presumption of legitimacy that reduces (but does not eliminate) the likelihood of spam flags.

**Every VICIdial operator should register every DID they own on FreeCallerRegistry.com.** There is no cost, no contract, and no downside. If you have not done this, stop reading and go do it now. We will wait.

### Layer 5: Consumer-Level App Databases

Beyond carrier analytics, third-party call-blocking apps maintain their own databases. Truecaller, RoboKiller, Nomorobo, and others crowdsource spam reports from their user bases. A number flagged in Truecaller's database (which has over 400 million users globally) will show a spam warning to every Truecaller user regardless of what the carrier analytics engine says.

These app-level databases are the hardest to remediate because each app has its own dispute process, and many rely heavily on user reports rather than carrier-level data.

---

## How VICIdial Handles DIDs: Current Capabilities and Gaps

VICIdial has a functional but limited DID management system. Understanding what it can and cannot do is essential to building a real DID strategy around it.

### What VICIdial Can Do

**Campaign CallerID**: The most basic setting. In Campaign Detail, the "Campaign CallerID" field sets the outbound caller ID for all calls placed by that campaign. This is a single number, applied to every dial.

**CID Groups**: This is VICIdial's built-in DID rotation mechanism, and it is more capable than most operators realize. CID Groups support three rotation modes:

1. **NONE (Sequential Rotation)**: Cycles through a list of DIDs in order. Two sub-settings control the rotation behavior:
   - **CID Auto Rotate Minutes**: How often the system rotates to the next DID. Set to 0 to rotate with every single call. Set to 30 to use one DID for 30 minutes before switching.
   - **CID Auto Rotate Minimum**: Minimum number of calls that must be placed from the active DID before rotation occurs. The system only rotates when both the Minutes threshold and the Minimum calls threshold have been reached.

2. **AREACODE**: Matches the outbound DID to the area code of the lead being called. This is VICIdial's implementation of local presence dialing.

3. **STATE**: Matches the outbound DID to the state of the lead being called.

**Group Alias**: An alternative to CID Groups that allows agents to manually select different caller IDs. When a Group Alias is selected on the agent interface, it overrides the Campaign CallerID. This is useful for operations where agents need to present different numbers based on the type of call.

**AC-CID (Area Code CID)**: A campaign-level setting that allows you to assign specific caller IDs to specific area codes directly within the campaign configuration, without creating a separate CID Group.

**Dial Prefix and Manual Dial Settings**: The [Force Dial Prefix](/settings/force-dial-prefix/) setting controls how numbers are formatted before they hit the carrier, which can affect caller ID transmission. Understanding your dialplan's interaction with CID settings is critical — a misconfigured dialplan can overwrite the CID your campaign is trying to send.

### What VICIdial Cannot Do

Here is where the gaps become painful:

**No reputation monitoring**: VICIdial has zero visibility into whether your DIDs are flagged as spam. It will happily dial 500 calls from a number that T-Mobile is labeling as "Scam Likely" and never tell you.

**No automated remediation**: When a number gets flagged, VICIdial cannot detect it, cannot pull it from rotation, and cannot alert you. The flagged DID keeps burning through leads until you manually discover the problem.

**No carrier-level health data**: VICIdial tracks call disposition (answered, no answer, busy, etc.) but does not correlate answer rate drops with specific DIDs. You can pull reports and do the math yourself, but there is no built-in dashboard showing "DID 555-0100 has dropped from 18% answer rate to 4% over the past 3 days."

**No integration with analytics engines**: There is no API connection to Hiya, First Orion, TNS, or any spam detection service. VICIdial operates blind to the reputation ecosystem.

**No DID lifecycle management**: There is no built-in concept of DID aging, cooling periods, or automated provisioning of replacement numbers when existing ones degrade.

These gaps are not design flaws — VICIdial is an open-source dialer, and reputation management is a relatively new operational requirement. But they mean that any serious DID management strategy requires external tooling layered on top of VICIdial.

> **Running VICIdial without DID monitoring is like driving with no dashboard gauges.** [Get a free audit](/free-audit/) to find out which of your numbers are already hurting your campaigns.

---

## Monitoring DID Reputation: Tools and Methods

You cannot manage what you cannot measure. Here are the tools available for monitoring DID reputation, ranked by capability and cost.

### DID Monitoring Tool Comparison

| Tool | What It Does | Carrier Coverage | Pricing | Best For |
|---|---|---|---|---|
| **FreeCallerRegistry.com** | Registers DIDs with all 3 analytics engines; reduces initial flag risk | Hiya, First Orion, TNS | Free | Every call center — no reason not to use it |
| **Caller ID Reputation (calleridreputation.com)** | Daily scans across carriers; alerts on flag events; remediation support | All 3 major carriers | Custom pricing, month-to-month, no contracts | Mid-size operations (20-100 DIDs) that need daily monitoring |
| **Numeracle Entity Identity Management** | Full-stack identity management; branded calling; managed remediation; direct carrier relationships | All 3 major carriers + third-party apps | Enterprise pricing (contact sales) | Large operations (100+ DIDs) with complex caller ID needs |
| **Hiya Connect Branded Call** | Branded caller ID display on Hiya-powered devices; reputation insights | AT&T (Hiya network) | Self-service plans from ~250 calls/mo; $25 setup fee; enterprise plans available | Operations primarily targeting AT&T subscribers |
| **First Orion INFORM** | Branded calling and reputation management for T-Mobile network | T-Mobile (First Orion) | Enterprise pricing (contact sales) | Operations primarily targeting T-Mobile subscribers |
| **Quality Voice Data** | Free CNAM lookup and caller ID testing; number health assessment | CNAM databases | Free for basic lookups | Quick spot-checks on individual numbers |
| **Readymode iQ** | Built-in DID monitoring, managed remediation, and automated rotation | All 3 major carriers | $249/license/month (5+ licenses); includes 75 DIDs per license | Operations using Readymode as their dialer platform |

### DIY Monitoring Methods for VICIdial Operators

If you are not ready to invest in a paid monitoring tool, you can build a basic monitoring system using VICIdial's existing reporting and some manual processes:

**1. Track answer rates per DID weekly.** Pull VICIdial's outbound calling report and break it down by caller ID. Any DID showing a week-over-week answer rate decline of more than 5 percentage points should be investigated immediately. A healthy DID on a good list should produce 15-25% contact rates. If a number drops below 10%, assume it is flagged on at least one carrier.

**2. Test-call your DIDs from each carrier.** Get a prepaid phone on each of the three major carriers (AT&T, T-Mobile, Verizon). Once a week, call each test phone from every DID in your pool. Note what displays: your business name, the raw number, "Spam Risk," "Scam Likely," or nothing at all. This takes time but gives you ground truth that no API can replace.

**3. Use free lookup tools.** Quality Voice Data offers a free caller ID testing tool that performs a CNAM dip and basic health assessment. Run your numbers through it monthly. FreeCallerRegistry.com does not provide monitoring, but registering your numbers there reduces baseline flag risk.

**4. Monitor consumer complaint patterns.** If your agents report that leads are answering and immediately saying "stop calling me" or "why does my phone say this is spam," log which DID was used for that call. Build a simple spreadsheet tracking complaints per DID per week.

**5. Watch for sudden volume drops.** In VICIdial's real-time report, if a campaign's contact rate drops sharply mid-day with no list change, the most likely cause is one or more DIDs getting flagged. Swap to a different CID Group and see if rates recover.

---

## DID Rotation Strategy: How Many Numbers Do You Actually Need?

This is the most common question we get, and the answer requires math, not guessing.

### The Core Formula

The fundamental constraint: **keep each DID under 80-100 calls per day.** Beyond that threshold, carrier analytics engines start treating the number as high-volume and increase scrutiny. The safe operating range is 50-80 calls per DID per day. Below 50, you are underutilizing your number pool. Above 100, you are asking to get flagged.

Here is the formula:

```
DIDs needed = (Total daily dials) / (Target calls per DID per day)
```

For a 20-seat call center running a predictive dialer at moderate pace:
- Each agent completes roughly 150-200 dials per day
- 20 agents × 175 average dials = 3,500 total daily dials
- At 70 calls per DID per day (conservative target): **3,500 / 70 = 50 DIDs**
- At 100 calls per DID per day (aggressive target): **3,500 / 100 = 35 DIDs**

**Our recommendation: plan for 1 DID per 50-80 daily dials per number.** For the 20-seat example above, that means 44-70 DIDs in your active pool.

### The Reserve Pool

You also need reserve numbers — DIDs that are not currently in active rotation but are registered, aged, and ready to swap in when active numbers get flagged. Plan for a reserve pool equal to 25-30% of your active pool size.

For the 20-seat example:
- Active pool: 50 DIDs
- Reserve pool: 13-15 DIDs
- **Total DID inventory: 63-65 numbers**

### Scaling by Operation Size

| Operation Size | Agents | Est. Daily Dials | Active DIDs Needed | Reserve DIDs | Total Inventory |
|---|---|---|---|---|---|
| Small | 5-10 | 875-1,750 | 12-25 | 3-8 | 15-33 |
| Medium | 10-25 | 1,750-4,375 | 25-63 | 7-19 | 32-82 |
| Large | 25-50 | 4,375-8,750 | 63-125 | 19-38 | 82-163 |
| Enterprise | 50-100 | 8,750-17,500 | 125-250 | 38-75 | 163-325 |

### Configuring Rotation in VICIdial

To set up DID rotation in VICIdial:

1. **Create a CID Group**: Navigate to Admin → CID Groups. Enter a CID Group ID (e.g., "SOLAR_ROTATION_01") and set CID Group Type to "NONE" for basic sequential rotation.

2. **Set rotation timing**: For high-volume campaigns, set CID Auto Rotate Minutes to 0 (rotate every call) or a low value like 15-30 minutes. Set CID Auto Rotate Minimum to ensure each number gets enough calls to avoid looking suspicious — a minimum of 5-10 calls per rotation cycle is reasonable.

3. **Add your DIDs**: Enter each DID in the CID Group entries. Order them so that geographically diverse numbers alternate (e.g., don't put five 212 numbers in a row followed by five 310 numbers).

4. **Assign to campaign**: In Campaign Detail, set "Custom CallerID" to Y and select your CID Group.

5. **Monitor and adjust**: Pull weekly reports broken down by caller ID. Remove underperforming DIDs from the group and replace them with reserve numbers.

### The Rotation Debate

There is a legitimate counterargument to aggressive rotation: carriers want to see numbers with established, consistent usage patterns. A number that places 60 calls per day, every day, for 90 days builds a positive history. A number that appears for 3 days, disappears for a week, and reappears with a burst of 200 calls looks suspicious.

The best practice is **controlled, consistent rotation** — not random number swapping. Every DID in your pool should be in regular use, placing a consistent number of calls per day. "Rotation" means distributing volume evenly across your pool, not rapidly cycling through throwaway numbers. PhoneBurner and Caller ID Reputation have both published research showing that aggressive number-swapping degrades consumer trust and carrier reputation over time.

> **Want us to calculate the exact DID pool size for your operation?** [Request a free audit](/free-audit/) — we will analyze your dial volume, agent count, and campaign mix to recommend the right number of DIDs.

---

## How to Remediate a Flagged Number

Your number just got flagged. The answer rate dropped from 19% to 6% overnight. Here is exactly what to do.

### Step 1: Confirm the Flag (Day 1)

Do not assume — verify. Call your DID from test phones on all three major carriers. Check what the recipient sees:

- **AT&T**: If Hiya is flagging the call, recipients see "Spam Risk" or "Fraud Risk"
- **T-Mobile**: If First Orion is flagging it, recipients see "Scam Likely"
- **Verizon**: If TNS is flagging it, recipients may see "Potential Spam"

Also run the number through Quality Voice Data's free caller ID testing tool and check FreeCallerRegistry.com to confirm your registration is active.

### Step 2: Pull the Number Immediately (Day 1)

Remove the flagged DID from your active CID Group in VICIdial. Do not wait to see if the flag "resolves itself." Every call you place from a flagged number generates more unanswered calls and potential consumer complaints, which deepens the negative reputation score. Replace it with a reserve DID from your pool.

### Step 3: Submit Remediation Requests (Days 1-2)

File disputes with each analytics engine directly:

- **Hiya (AT&T)**: Submit a dispute through Hiya's feedback form or through your Numeracle/Caller ID Reputation account if you have one. Provide your business name, the flagged number, and evidence of legitimate calling practices.
- **First Orion (T-Mobile)**: Submit through First Orion's business portal. If you have a relationship with Numeracle, they have a direct remediation path with First Orion that is significantly faster.
- **TNS (Verizon)**: Submit through TNS's enterprise portal or through a managed remediation provider.

**Also file with third-party apps**: Submit correction requests to Truecaller, Nomorobo, RoboKiller, and any other call-blocking app where your number may be listed.

### Step 4: Rest the Number (Days 2-14)

After filing disputes, do not use the number for outbound calling for a minimum of 7-14 days. This "cooling period" allows the analytics engines to process your dispute, reduces the call volume signal that may have triggered the flag, and lets any consumer complaint momentum die down.

During the cooling period, you can still receive inbound calls on the number. Inbound call activity with normal durations and engagement actually helps rebuild the number's reputation.

### Step 5: Verify Remediation (Day 14-21)

After the cooling period, test-call the number from all three carrier test phones again. If the flag has been removed:

- Re-register the number on FreeCallerRegistry.com (re-registration reinforces the legitimate caller status)
- Re-add the number to your CID Group but at a reduced volume — start with 20-30 calls per day for the first week
- Gradually increase to normal volume (50-80 calls per day) over the following two weeks

If the flag persists after 14 days, escalate through your managed remediation provider or re-submit disputes with additional documentation. Some flags, particularly those driven by a high volume of consumer complaints, can take 30+ days to fully clear.

### Step 6: Investigate Root Cause

A flagged number is a symptom. Ask why it happened:

- **Was the number exceeding 100 calls per day?** Adjust your rotation to lower per-DID volume.
- **Was the number calling a bad list segment?** Old lists with high disconnect rates generate short-duration calls that trigger spam algorithms. Review your [list management practices](/blog/vicidial-list-management/).
- **Was the number's STIR/SHAKEN attestation level dropping?** Check with your carrier to confirm A-level attestation. See our [STIR/SHAKEN guide](/blog/stir-shaken-vicidial-guide/) for troubleshooting.
- **Was your [AMD (Answering Machine Detection)](/blog/vicidial-amd-guide/) too aggressive?** Overly sensitive AMD settings that drop calls within 1-2 seconds create a pattern of very short call durations that looks identical to robocall behavior to analytics engines.

> **Dealing with flagged numbers right now?** [Let us diagnose the problem for free](/free-audit/) — we will identify which carriers are flagging you, why, and give you a remediation plan.

---

## Local Presence Dialing: Benefits, Risks, and Best Practices

Local presence dialing — presenting a caller ID with an area code matching the recipient's location — is one of the most effective and most misunderstood tools in outbound calling.

### The Benefits Are Real

The data is unambiguous: calls displaying a local area code get answered at 2-4x the rate of calls from out-of-area or toll-free numbers. In our data across ViciStack-managed centers, local presence DIDs produce average answer rates of 18-24%, compared to 8-12% for non-local numbers. That is the difference between a profitable campaign and a money loser.

VICIdial supports local presence through CID Groups set to AREACODE mode. You load the CID Group with DIDs covering each target area code, and VICIdial automatically selects the matching DID for each lead. If no exact area code match exists, VICIdial falls back to a default number.

### The Legal Framework

The Truth in Caller ID Act of 2009 prohibits transmitting misleading or inaccurate caller ID information **with the intent to defraud, cause harm, or wrongly obtain anything of value.** Penalties can reach $10,000 per violation.

Local presence dialing is legal when:
- You own or lease the DID being displayed
- The number can receive return calls (it is a real, working number)
- You are not using the local number to misrepresent your identity or location
- Your business has a legitimate reason to present a local number (e.g., you serve customers in that area)

Local presence dialing becomes problematic when:
- You display numbers you do not own or control
- The number cannot receive return calls
- You are presenting a local number for a business that has no connection to that geography
- You are cycling through local numbers specifically to evade spam detection (this is called "neighbor spoofing" and is exactly what robocallers do)

### The 2026 Compliance Risk

The FCC has proposed new rules that would require originating providers to employ reasonable measures to verify the accuracy of caller ID information they transmit. Additionally, proposals to require foreign-originated calls to be labeled as "international" could impact offshore call centers using U.S.-based local presence numbers. While these rules are still in the comment period (comments closed February 2026), the regulatory direction is clear: increased scrutiny of caller ID accuracy.

### Best Practices for Local Presence in VICIdial

**1. Own enough local numbers to be credible.** Do not try to cover 50 area codes with 50 DIDs (one each). Cover your top 10-15 area codes with 3-5 DIDs per area code. It is better to have strong coverage in your primary markets than thin coverage everywhere.

**2. Ensure every DID can receive return calls.** Configure VICIdial inbound routing for every local presence DID. When a lead calls back, it should route to an available agent or a professional voicemail. A "this number is not in service" message destroys trust and generates complaints.

**3. Match your CNAM to the local DID.** Each local DID should have a CNAM record showing your business name. "Unknown Caller" from a local number is almost as suspicious as "Spam Risk" from an 800 number.

**4. Do not use local presence as camouflage.** If you are a solar company based in Phoenix serving Phoenix customers, presenting a 602 number is natural and appropriate. If you are a lead gen shop in Boca Raton presenting a 602 number to seem local, that is the kind of practice that attracts both carrier flags and regulatory attention.

**5. Rotate local DIDs with the same discipline as general DIDs.** Local numbers are not immune to spam flagging. Apply the same 50-80 calls per day per DID threshold to local presence numbers.

---

## Carrier-Level DID Management vs. VICIdial Configuration

There are two layers to DID management: what your carrier does and what VICIdial does. Most operators focus exclusively on VICIdial configuration and ignore the carrier layer. This is a mistake.

### What Your Carrier Controls

**STIR/SHAKEN attestation**: Your carrier determines the attestation level for every call. If they are giving you B-level attestation because your DID registration with them is incomplete, no amount of VICIdial configuration will fix your spam problem. Confirm with your carrier that every DID in your pool receives A-level attestation. If they cannot provide A-level attestation for your numbers, switch carriers.

**Trunk configuration**: How your SIP trunk is configured determines whether your chosen caller ID actually transmits correctly. In VICIdial, the carrier entry's dialplan can override the CID set by your campaign or CID Group. Check your carrier config's `fromuser` parameter and any CALLERID(num) manipulations in your dialplan. If the carrier's trunk config is stripping or replacing your CID, nothing you set in VICIdial will matter.

**Number provisioning and porting**: Carriers differ dramatically in the quality of DIDs they provision. Some carriers sell "recycled" numbers that already have spam flags from their previous owner. Always request the history of a DID before provisioning it, and test-call new numbers from all three carrier test phones before putting them into production.

**E911 registration**: Some carriers require E911 registration for every DID. Failure to register can result in delayed number activation or reduced attestation levels.

### What VICIdial Controls

**DID selection logic**: Through CID Groups, AC-CID, Group Alias, and campaign-level CallerID settings, VICIdial determines which number appears on the recipient's phone. The hierarchy matters:

1. Group Alias (if selected by the agent) overrides everything
2. CID Group (if assigned to the campaign) overrides Campaign CallerID
3. Campaign CallerID is the default fallback

**Call pacing and volume**: Your [predictive dialer settings](/blog/vicidial-predictive-dialer-settings/) directly affect DID reputation. An overly aggressive dial level that burns through 200+ calls per hour per agent creates call patterns that trigger spam algorithms. Dial level optimization and DID management are inseparable concerns.

**Dial prefix configuration**: The [Force Dial Prefix](/settings/force-dial-prefix/) setting and Manual Dial CID settings control how outbound calls are formatted and which CID is attached. Misconfigured dial prefixes are one of the most common causes of caller ID transmission failures in VICIdial.

### Carrier Selection Checklist

When evaluating carriers for your VICIdial operation, ask these questions:

- Do you provide A-level STIR/SHAKEN attestation for all my DIDs?
- Do you own your STIR/SHAKEN certificate directly (not through a third party)?
- Can you provide the call history and reputation status of DIDs before I provision them?
- Do you support CNAM registration for outbound caller ID?
- What is your process for handling spam flag disputes on my behalf?
- Do you provide any DID reputation monitoring or alerting?
- What is your DID replacement policy when numbers get flagged?

If your carrier cannot answer these questions satisfactorily, they are not a carrier built for outbound calling operations. The cheapest per-minute rate means nothing if your calls are not connecting.

> **Your carrier might be the problem.** [Request a free audit](/free-audit/) and we will evaluate your carrier's STIR/SHAKEN compliance and DID quality alongside your VICIdial configuration.

---

## How ViciStack's DID Hygiene Module Automates Reputation Management

Everything described above — monitoring, rotation, remediation, carrier management — is the work that VICIdial operators need to do manually every day. [ViciStack's DID Hygiene module](/features/did-hygiene/) automates the entire process.

### Automated Reputation Scanning

The DID Hygiene module scans every number in your pool across all three major carrier analytics engines daily. It does not wait for answer rates to drop — it detects spam flags the moment they appear, often before they impact your campaigns. Each DID receives a health score on a 0-100 scale, updated every 24 hours.

### Intelligent Rotation Management

Rather than simple sequential rotation, the DID Hygiene module implements reputation-weighted rotation. High-scoring DIDs get more call volume. DIDs showing early signs of reputation degradation get automatically throttled. Flagged DIDs get pulled from rotation entirely without manual intervention.

The module integrates directly with VICIdial's CID Group system, so you do not need to modify your campaign configuration. It manages the CID Group membership dynamically based on real-time health data.

### Automated Remediation Workflow

When a DID is flagged, the module automatically:

1. Removes the DID from all active CID Groups
2. Submits remediation requests to the relevant analytics engines
3. Places the DID in a cooling queue with a configurable rest period (default: 14 days)
4. Activates a reserve DID from your pool to maintain campaign volume
5. Monitors the cooling DID and re-tests it at the end of the rest period
6. Returns cleared DIDs to the reserve pool for gradual reintroduction

### DID Lifecycle Dashboard

The module provides a dashboard showing:

- Real-time health scores for every DID in your pool
- Historical reputation trends per DID
- Active flags by carrier (so you can see if a number is flagged on AT&T but clean on T-Mobile)
- Reserve pool status and availability
- Remediation queue with estimated clearance dates
- Per-DID call volume and answer rate correlation

### Proactive Number Provisioning

When your active pool shrinks below a configurable threshold (because too many DIDs are in cooling or remediation), the module can automatically request new DIDs from your carrier through API integration. New numbers are pre-registered on FreeCallerRegistry.com, CNAM-registered, and aged for a configurable period before entering active rotation.

### What This Means in Practice

A typical 30-seat ViciStack-managed call center with the DID Hygiene module active maintains:

- 55-65 active DIDs at any given time
- 15-20 reserve DIDs ready for deployment
- Average DID health score above 85/100
- Less than 5% of DIDs flagged at any given time
- Answer rates 30-45% higher than unmanaged VICIdial operations

The DID Hygiene module eliminates the most time-consuming and error-prone operational task in outbound calling. Instead of spending 3-5 hours per week manually checking DIDs, filing disputes, and adjusting CID Groups, operators spend zero time — the system handles it.

> **See DID Hygiene in action.** [Schedule a free audit](/free-audit/) and we will show you exactly how the module would work with your current DID inventory and campaign configuration.

---

## Action Plan: 30-Day DID Health Improvement Protocol

Stop reading and start doing. Here is your day-by-day plan for getting your DID management under control in 30 days.

### Week 1: Assessment and Registration (Days 1-7)

**Day 1 — Inventory your DIDs**
- Log every DID your operation uses across all campaigns. Include numbers in active campaigns, inactive campaigns, and any parked numbers.
- Create a spreadsheet with columns: DID, Carrier, Campaign, Daily Call Volume (avg), Current Answer Rate, CNAM Status, FreeCallerRegistry Status, Last Tested Date.

**Day 2 — Register on FreeCallerRegistry.com**
- Go to FreeCallerRegistry.com and register every DID in your inventory. You can submit batches of 20 at a time. For larger inventories, download the bulk upload template.
- Select the appropriate call category for your business (e.g., "Sales," "Customer Service," "Appointment Reminders").
- Save confirmation of registration for each batch.

**Day 3 — Test every DID across carriers**
- Get prepaid test phones on AT&T, T-Mobile, and Verizon (if you do not already have them — this is a one-time $75-100 investment that pays for itself immediately).
- Call each test phone from every DID in your pool. Record what displays: business name, number only, "Spam Risk," "Scam Likely," etc.
- Mark flagged DIDs in your spreadsheet with the carrier(s) showing the flag.

**Day 4 — Pull flagged DIDs**
- Remove every flagged DID from active CID Groups in VICIdial.
- Submit remediation requests to the relevant analytics engines for each flagged number (Hiya for AT&T flags, First Orion for T-Mobile, TNS for Verizon).
- Move flagged DIDs to a "Cooling" tab in your spreadsheet with the date they were pulled.

**Day 5 — Verify CNAM registration**
- Run every active DID through Quality Voice Data's free CNAM lookup tool.
- For any DID without a proper CNAM record, submit a CNAM update through your carrier. Allow 48-72 hours for propagation.

**Day 6 — Verify STIR/SHAKEN attestation**
- Contact your carrier and confirm the attestation level for every DID. Request written confirmation of A-level attestation.
- If any DIDs are receiving B or C attestation, work with your carrier to resolve immediately or plan to move those DIDs to a carrier that provides A-level attestation.

**Day 7 — Calculate your DID gap**
- Using the formula (Total Daily Dials / 70 calls per DID), calculate how many active DIDs you actually need.
- Add 25-30% for your reserve pool.
- Compare to your current inventory minus flagged numbers. The difference is your DID gap — the number of new DIDs you need to provision.

### Week 2: Optimization (Days 8-14)

**Day 8 — Provision new DIDs**
- Order enough new DIDs from your carrier to fill the gap identified on Day 7.
- Request that your carrier provide DIDs with no prior call history (new numbers, not recycled).
- Before activating, test-call each new DID from your three carrier test phones to verify they are clean.

**Day 9 — Configure CID Groups**
- In VICIdial Admin → CID Groups, create organized CID Groups by campaign type or geography.
- Set CID Auto Rotate Minutes to 0 (per-call rotation) or 15-30 minutes depending on your volume.
- Set CID Auto Rotate Minimum to 5-10 calls.
- Add all clean, active DIDs to the appropriate CID Groups.

**Day 10 — Implement per-DID volume tracking**
- Set up a weekly reporting process: pull VICIdial's outbound call report grouped by caller ID. Track calls per DID per day and answer rate per DID.
- Flag any DID exceeding 100 calls per day for immediate volume reduction.
- Set a target range of 50-80 calls per DID per day across all campaigns.

**Day 11 — Register new DIDs**
- Register all newly provisioned DIDs on FreeCallerRegistry.com.
- Submit CNAM registration for all new DIDs.
- Add new DIDs to your master inventory spreadsheet.

**Day 12 — Configure inbound routing for all DIDs**
- Ensure every DID in your pool can receive inbound calls. Configure VICIdial inbound routing so return calls go to available agents or professional voicemail.
- Test inbound routing by calling each DID.

**Day 13 — Review AMD settings**
- Overly aggressive AMD settings create short-duration call patterns that trigger spam algorithms. Review your [AMD configuration](/blog/vicidial-amd-guide/) and ensure you are not dropping calls within the first 1-2 seconds.
- Target AMD detection windows that give the system enough time to distinguish between voicemail greetings and human answers without creating suspicious call patterns.

**Day 14 — First remediation check**
- Re-test DIDs that were flagged and pulled on Day 4. Call from all three carrier test phones.
- Any DIDs that are now clean can be moved to the reserve pool (not active rotation yet — they need gradual reintroduction).
- DIDs still showing flags stay in the cooling queue. Re-submit remediation requests if needed.

### Week 3: Monitoring and Refinement (Days 15-21)

**Day 15 — Pull first weekly DID performance report**
- Generate your first weekly per-DID report from VICIdial. Compare answer rates, call volumes, and talk times across all DIDs.
- Identify the top 10 and bottom 10 performers. Investigate what makes the bottom 10 different (higher volume? older numbers? specific carrier issues?).

**Day 16 — Adjust rotation based on data**
- Remove bottom-performing DIDs from active rotation if their answer rates are below 10%.
- Increase volume allocation to top-performing DIDs (but keep them under 80 calls/day).
- Add reserve DIDs to replace any removed numbers.

**Day 17 — Contact carrier about underperformers**
- For DIDs with consistently low answer rates that are not showing spam flags on test calls, contact your carrier. The issue may be attestation-related, trunk configuration-related, or CNAM-related.
- Verify that your carrier's trunk config is not overriding the caller ID VICIdial sends.

**Day 18 — Register with third-party apps**
- Beyond FreeCallerRegistry.com, register your business and primary DIDs with Truecaller for Business, Hiya Connect, and any other caller ID app that offers business registration.
- This is a time investment (1-2 hours) that reduces the chance of app-level flagging.

**Day 19 — Evaluate DID monitoring tool needs**
- Based on your first two weeks of manual monitoring, decide whether the time investment justifies a paid monitoring tool.
- For operations with 30+ DIDs, paid monitoring almost always pays for itself in saved labor and faster flag detection.

**Day 20 — Reintroduce cleared DIDs**
- DIDs that cleared their flags (from the Day 14 check) can now enter active rotation at reduced volume: 20-30 calls per day for the first week.
- Monitor their answer rates daily. If rates hold steady, gradually increase to normal volume.

**Day 21 — Second weekly DID performance report**
- Generate week 2 performance data. Compare to week 1 baseline.
- You should see improvement in overall answer rates if you have successfully pulled flagged DIDs and distributed volume properly.

### Week 4: Systematization (Days 22-30)

**Day 22 — Document your DID management SOP**
- Write a standard operating procedure covering: how often DIDs are tested, who is responsible, what constitutes a flag, what the remediation process is, how new DIDs are provisioned and aged, and how CID Groups are configured.
- This SOP should be followed by whoever manages your VICIdial campaigns going forward.

**Day 23 — Set up automated reporting**
- Configure VICIdial scheduled reports (or external scripts) to automatically generate per-DID performance data weekly.
- Set up email alerts for any DID whose answer rate drops below a threshold (e.g., 10%).

**Day 24 — Establish DID provisioning pipeline**
- Set up a standing relationship with your carrier for ongoing DID provisioning. Know how to order new numbers quickly when you need them.
- Establish a process for aging new DIDs: register on FreeCallerRegistry.com, set CNAM, and rest for 7 days before entering active rotation.

**Day 25 — Final remediation check**
- Re-test all DIDs that have been in the cooling queue for 21+ days.
- Clear DIDs go to the reserve pool. Still-flagged DIDs may need to be retired permanently if remediation has failed after 30 days.

**Day 26-27 — Run your first full-cycle review**
- Compile all 4 weeks of data. Calculate:
  - Total DIDs in inventory vs. active vs. reserve vs. cooling
  - Average answer rate week 1 vs. week 4
  - Number of flag events detected and remediated
  - Average time to remediation
  - DID cost per month vs. revenue impact of improved answer rates

**Day 28 — Adjust pool sizing**
- Based on a month of data, refine your DID pool size. You may need more DIDs than initially calculated (if flags are frequent) or fewer (if your volume is lower than expected).
- Order or release DIDs to match your optimized pool size.

**Day 29 — Carrier performance review**
- Evaluate your carrier's performance over the past month. Are they providing consistent A-level attestation? Are newly provisioned DIDs clean? Is their support responsive when you report issues?
- If your carrier is underperforming, begin evaluating alternatives. Carrier selection is a DID management decision.

**Day 30 — Ongoing cadence**
- Set your ongoing DID management cadence:
  - **Daily**: Check real-time report for sudden answer rate drops
  - **Weekly**: Pull per-DID performance report; test-call 10-20 DIDs across carriers; review and adjust CID Group membership
  - **Monthly**: Full inventory audit; re-register any new DIDs on FreeCallerRegistry.com; review carrier attestation levels; provision replacement DIDs as needed
  - **Quarterly**: Full carrier review; DID pool size recalculation; SOP review and update

> **Skip the 30 days of manual work.** [ViciStack's DID Hygiene module](/features/did-hygiene/) automates every step of this protocol. [Request a free audit](/free-audit/) to see how much time and revenue you are leaving on the table.

---

### The Bottom Line

DID management is not glamorous work. It is not the part of running a call center that anyone gets excited about. But it is the part that determines whether your leads actually hear the phone ring. A perfectly configured VICIdial server running against a perfectly cleaned list with a perfectly tuned predictive dialer will produce nothing if your numbers are flagged.

The operators who are winning in 2026 are the ones who treat their DID inventory as a strategic asset — monitored daily, maintained proactively, and replaced before problems compound. The operators who are struggling are the ones who still think of DIDs as commodity phone numbers that you buy in bulk and burn through.

You now have the knowledge, the tools, and the 30-day plan to move from the second group to the first. The only question is whether you will actually do it.

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-did-management).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
