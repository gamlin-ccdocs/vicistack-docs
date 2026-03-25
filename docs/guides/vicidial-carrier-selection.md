# VICIdial Carrier Selection: SIP Trunks, Rates & Quality

*Published by ViciStack — the managed VICIdial platform built by operators, for operators.*

---

You are paying too much for bad calls. Your [SIP trunk](/glossary/sip-trunk/) provider quoted you a rate that looked competitive, your calls connect most of the time, and you have never thought much about it since. Meanwhile, 15% of your outbound dials are failing silently, your [post-dial delay](/glossary/post-dial-delay/) is adding 3 seconds to every call attempt, your [STIR/SHAKEN](/glossary/stir-shaken/) attestation is sitting at B-level because your [carrier](/glossary/carrier/) cannot sign your numbers properly, and you have no idea because you have never measured any of it.

This is the norm. Most VICIdial operators pick a carrier the same way they pick a cell phone plan — they compare the per-minute rate, pick the cheapest one that does not seem obviously terrible, and move on. The problem is that in high-volume outbound operations, carrier selection is not a billing decision. It is an infrastructure decision that directly determines your answer rates, your call quality, your compliance posture, and ultimately your revenue per seat per hour.

We manage SIP trunk relationships across over 100 VICIdial-based contact centers. We have tested every major carrier that serves the outbound calling market. We track quality metrics across roughly 600,000 dials per month spanning dozens of carrier configurations. What follows is everything we know about choosing, evaluating, and configuring carriers for VICIdial — laid out in the order you need it.

---

## Why Carrier Selection Matters More Than You Think

The math is simple and brutal. Take a 25-seat outbound call center dialing 4,000 calls per day. If your carrier's Answer Seizure Ratio (ASR) is 45% instead of the 52% you would get from a better carrier, that is 280 fewer connected calls per day. At a conservative 10% conversion rate and $50 average revenue per conversion, that is $1,400 per day — $30,800 per month — evaporating because of a carrier problem you never measured.

Now layer on call quality. If your carrier's [MOS score](/glossary/mos-score/) is averaging 3.4 instead of 4.0, your agents are straining to hear leads, leads are straining to hear agents, and conversations that should convert are falling apart 30 seconds in because someone said "can you repeat that?" three times and hung up. You cannot see this in any VICIdial report. It shows up as lower conversion rates that everyone blames on the list, the script, or the agents.

Then there is compliance. If your carrier is handing you B-level STIR/SHAKEN attestation — or worse, C-level gateway attestation — every single call you place starts with a trust deficit. The terminating carrier sees your call as unverified. Analytics engines at AT&T, T-Mobile, and Verizon treat unverified calls with increased scrutiny. Your [DID management](/blog/vicidial-did-management/) strategy becomes twice as hard because your numbers are fighting an uphill battle from the moment the call is placed. For the full breakdown on attestation levels and their impact, see our [STIR/SHAKEN implementation guide](/blog/stir-shaken-vicidial-guide/).

The carrier you choose affects everything downstream. Picking the wrong one — or failing to evaluate the one you have — is the most expensive mistake you can make that does not show up on any line item in your budget.

---

## The Carrier Evaluation Framework: What Actually Matters

Most operators evaluate carriers on two dimensions: price and "does it work." That is not an evaluation framework. That is coin-flipping with extra steps. Here is the framework we use when onboarding a new ViciStack client or auditing an existing carrier relationship.

### 1. Call Quality Metrics

There are four metrics that define whether a carrier is actually delivering quality calls. If your carrier cannot provide these numbers — or if you have never asked for them — you are flying blind.

**ASR (Answer Seizure Ratio)**: The percentage of call attempts that result in a connection (answered by a human or machine, not failed at the network level). For U.S. domestic outbound calling, a healthy ASR ranges from 45-60% depending on the campaign type and list quality. If your carrier's ASR is consistently below 40%, the problem is likely carrier-side — failed routes, overloaded gateways, or poor interconnect agreements with terminating carriers.

```
ASR = (Answered Calls / Total Call Attempts) × 100
```

**ACD (Average Call Duration)**: The average length of connected calls in seconds. ACD itself is more of a campaign metric than a carrier metric, but abnormally low ACD (under 15 seconds average) can indicate carrier-level problems: calls connecting but dropping immediately, one-way audio issues causing immediate hangups, or codec negotiation failures producing garbled audio that leads to quick disconnects.

```
ACD = Total Talk Time (seconds) / Number of Answered Calls
```

**PDD (Post-Dial Delay)**: The time between when VICIdial sends the INVITE and when the caller hears ringing or the call is answered. This is the metric most operators never measure and should. A good PDD is under 3 seconds. An acceptable PDD is 3-5 seconds. Anything over 5 seconds means your carrier is routing calls through too many hops, and those extra seconds compound across thousands of dials per day into hours of lost agent productivity.

For a 25-seat center making 4,000 calls per day, every additional second of PDD costs roughly 67 minutes of aggregate agent wait time. At 5 seconds average PDD versus 2 seconds, that is over 3 hours of agent time per day spent listening to dead air. At $15/hour loaded agent cost, that is $45/day or nearly $1,000/month — just from slow call setup.

```
PDD = Time of first ringback or answer - Time of SIP INVITE sent
```

**MOS (Mean Opinion Score)**: A standardized measurement of voice quality on a scale of 1 (unintelligible) to 5 (perfect). In VoIP, a MOS of 4.0+ is considered good. 3.5-4.0 is acceptable. Below 3.5, your agents and leads are having degraded conversations. MOS is affected by [codec](/glossary/codec/) selection, packet loss, jitter, and the number of transcoding steps between your server and the terminating carrier. Your [Asterisk configuration](/blog/vicidial-asterisk-configuration/) plays a role here, but carrier routing quality is the bigger factor.

| Metric | Good | Acceptable | Problem | How to Measure |
|---|---|---|---|---|
| **ASR** | 50-60% | 40-50% | Below 40% | VICIdial outbound report, filter by carrier trunk |
| **ACD** | Campaign-dependent | Campaign-dependent | Below 15 sec avg | VICIdial call log, filter by disposition |
| **PDD** | Under 3 sec | 3-5 sec | Over 5 sec | SIP packet capture (Asterisk CLI: `sip set debug on`) |
| **MOS** | 4.0+ | 3.5-4.0 | Below 3.5 | VoIP monitoring tools (VoIPmonitor, Homer) or carrier-provided dashboards |

### 2. STIR/SHAKEN Attestation Capability

This is non-negotiable in 2026. Your carrier must provide A-level [STIR/SHAKEN](/glossary/stir-shaken/) attestation for every DID you use for outbound calling. Here is what that requires:

**The carrier must own their own STIR/SHAKEN certificate.** As of June 2025, the FCC requires all voice service providers with a STIR/SHAKEN implementation obligation to authenticate calls using their own certificates — third-party signing arrangements are no longer compliant. If your carrier is still using a third-party certificate, your calls are at risk of receiving reduced attestation or being flagged as non-compliant at the terminating end.

**The carrier must verify your identity and your authorization to use each DID.** A-level attestation means the carrier knows who you are and has confirmed that you are authorized to use the specific number being displayed. This requires that every DID in your pool is properly registered with your carrier and linked to your account. If you are porting numbers from another provider or using DIDs purchased from a third party, the new carrier must complete their own verification before they can provide A-level attestation.

**The carrier must sign calls in real time.** STIR/SHAKEN signing happens at the SIP level. The carrier's Session Border Controller (SBC) adds Identity headers to the SIP INVITE. If the carrier's signing infrastructure is slow or unreliable, it can add latency to call setup (increasing PDD) or fail silently, resulting in unsigned calls that get treated as suspicious.

Here is how attestation levels break down by carrier type:

| Carrier Type | Typical Attestation | Why | Risk to Your Operation |
|---|---|---|---|
| **Tier 1 carrier with direct certificate** | A-level for verified DIDs | Full identity verification, own certificate | Low — calls are trusted |
| **Mid-tier VoIP carrier with own certificate** | A-level if DIDs are properly registered | Depends on their verification process | Medium — verify their process is thorough |
| **Reseller using upstream provider's certificate** | B or C-level | Cannot fully verify your identity or DID authorization | High — calls start with trust deficit |
| **Offshore carrier routing through U.S. gateway** | C-level or unsigned | Gateway attestation only; no caller identity verification | Very high — calls treated as suspicious by default |

**Ask your carrier directly:** "What attestation level are my calls receiving?" If they cannot answer immediately, that is a red flag. If they say "B-level" or "it depends," that is a problem you need to solve before worrying about anything else.

### 3. Network Quality and Routing

Not all SIP trunks are created equal, even among carriers quoting the same per-minute rate. The difference is in how calls are routed.

**Direct interconnects vs. least-cost routing (LCR)**: Tier 1 carriers maintain direct interconnect agreements with the major terminating carriers (AT&T, Verizon, T-Mobile, etc.). Your call goes from their network directly to the terminating network in one hop. LCR carriers route calls through the cheapest available path, which may involve multiple intermediate carriers. Each hop adds latency, increases the chance of packet loss, and can introduce transcoding that degrades audio quality.

**The transcoding problem**: If your VICIdial server sends audio using the G.711 [codec](/glossary/codec/) and one of the intermediate carriers only supports G.729, the audio gets transcoded. Every transcoding step degrades the [MOS score](/glossary/mos-score/) by approximately 0.2-0.5 points. A call that starts at MOS 4.2 on your server can arrive at the recipient's phone at MOS 3.3 after two transcoding steps. Your [Asterisk codec configuration](/blog/vicidial-asterisk-configuration/) matters, but it only controls the first leg — after that, it is up to your carrier's routing decisions.

**Geographic routing intelligence**: Good carriers route calls based on the terminating number's location and carrier, optimizing for quality. Poor carriers route every call through the same path regardless of destination. The difference is measurable: 1-3 seconds of PDD improvement and 0.3-0.5 MOS improvement on average.

**Network capacity and congestion**: During peak calling hours (typically 10am-2pm and 4pm-7pm in each time zone), carrier networks experience congestion. Carriers with adequate capacity handle this gracefully. Carriers running close to capacity during peaks will show increased PDD, higher call failure rates, and degraded audio quality — exactly when you need performance most.

### 4. Concurrent Channel Model and Burst Capacity

The way your carrier prices and allocates channels directly affects your dialing capacity and your cost structure. There are three primary models:

**Per-channel pricing (committed channels)**: You purchase a fixed number of concurrent channels — say, 100. You can have up to 100 simultaneous calls in progress at any moment. You pay for these channels whether you use them or not. This model is predictable and guarantees capacity, but you pay for idle channels during off-peak hours.

Typical pricing: $1.50-$5.00 per channel per month, plus per-minute usage.

**Pay-per-minute (no channel commitment)**: You pay only for the minutes you use, with no dedicated channels. The carrier allocates channels dynamically from their pool. This is cheaper when you are small or have irregular call volume, but you have no guaranteed capacity. During peak hours, when every outbound call center on that carrier is dialing simultaneously, you may experience channel exhaustion — your calls fail because the carrier has no available channels to allocate.

Typical pricing: $0.005-$0.015 per minute for U.S. domestic, no monthly minimum or a low minimum ($25-$100).

**Hybrid (base channels + burst)**: You commit to a base number of channels (e.g., 50) and can burst above that up to a maximum (e.g., 150) on a per-minute basis. The base channels are guaranteed. Burst channels are available when the carrier has capacity. This is the model most well-run outbound operations use because it provides guaranteed minimum capacity with flexibility for peak periods.

Typical pricing: $2.00-$4.00 per base channel per month + $0.008-$0.020 per minute for burst channels.

**How to calculate channels needed for VICIdial:**

```
Minimum channels = Agents × Dial Level × 1.2 (safety margin)
```

For a 25-seat center running a predictive dialer at a 3:1 dial ratio:

```
Minimum channels = 25 × 3 × 1.2 = 90 channels
```

But VICIdial's predictive algorithm adjusts the dial level dynamically. During periods of low answer rates, it may push the ratio to 4:1 or 5:1. You need burst capacity for these peaks:

```
Burst capacity = Agents × Max Dial Level × 1.2
             = 25 × 5 × 1.2 = 150 channels
```

If your carrier caps you at 100 channels and VICIdial tries to place 120 simultaneous calls, 20 calls will fail with a SIP 503 (Service Unavailable) response. VICIdial logs these as "CONGESTION" dispositions. If you are seeing CONGESTION dispositions in your call logs, your channel count is too low for your dial level.

| Operation Size | Agents | Recommended Base Channels | Recommended Burst Capacity | Estimated Monthly Channel Cost |
|---|---|---|---|---|
| Small | 5-10 | 20-40 | 40-60 | $40-$200 |
| Medium | 10-25 | 40-90 | 80-150 | $120-$450 |
| Large | 25-50 | 90-180 | 150-300 | $270-$900 |
| Enterprise | 50-100 | 180-360 | 300-600 | $540-$1,800 |

### 5. DID Services and Number Management

Your carrier is also your DID provider in most cases. Evaluate their DID capabilities alongside their trunk services:

- **DID provisioning speed**: How fast can you get new numbers? Same-day provisioning is the standard. If your carrier needs 3-5 business days to provision DIDs, they are not built for outbound operations.
- **Number quality**: Do they provision clean numbers with no prior call history, or are you getting recycled numbers that may already carry spam flags? Always test new numbers before deploying them.
- **Local number availability**: If you run local presence campaigns, your carrier needs broad area code coverage. Can they provide numbers in every area code you need?
- **Porting support**: If you are moving from another carrier, porting speed and accuracy matter. A botched port can leave your numbers in limbo for days.
- **CNAM registration**: Can the carrier register CNAM (Caller Name) records for your DIDs? Some carriers handle this; others require you to use a third-party CNAM provider.

For a complete guide to managing your DIDs once they are provisioned, see our [DID management guide](/blog/vicidial-did-management/).

---

## Rate Comparison: What You Are Actually Paying and What You Should Be

Per-minute rates are the number every operator fixates on. They matter, but they are only one component of your total carrier cost — and often not the largest one.

### Understanding Rate Structures

**Flat rate per minute**: A single per-minute rate for all U.S. domestic calls regardless of destination. Simple, predictable, and the most common model for outbound-focused carriers. Typical range: $0.005-$0.015 per minute.

**Tiered rate by destination**: Different rates for different destination types — landline vs. mobile, local vs. long distance, rate center-based pricing. This model can save money if your traffic is heavily weighted toward cheaper destinations, but it makes cost prediction harder. Some carriers advertise a low rate that only applies to landline terminations, while mobile terminations (which are the majority of outbound consumer calls in 2026) carry a 2-3x premium.

**Blended rate**: A single advertised rate that averages across all destination types. This is functionally similar to flat rate but may be recalculated periodically based on your actual traffic mix.

**Volume commitment pricing**: Lower per-minute rates in exchange for committing to a minimum monthly spend. Common tiers:

| Monthly Commitment | Typical Per-Minute Rate | Annual Savings vs. No Commitment |
|---|---|---|
| No minimum | $0.010-$0.015 | Baseline |
| $500/month | $0.008-$0.012 | $600-$1,800/year |
| $2,000/month | $0.006-$0.010 | $2,400-$6,000/year |
| $5,000/month | $0.004-$0.008 | $7,200-$14,400/year |
| $10,000+/month | $0.003-$0.006 | Negotiate directly |

### The Hidden Costs Most Operators Miss

The per-minute rate your carrier quotes is not your true cost per minute. Here are the line items that inflate your actual cost:

**Billing increment**: Some carriers bill in 6-second increments. Others bill in 1-minute increments. On a 25-second call, a 6-second increment carrier charges you for 30 seconds. A 1-minute increment carrier charges you for 60 seconds — double the cost for the same call. For outbound operations where a huge percentage of calls are short (unanswered calls that ring for 20-30 seconds before dropping), billing increment can increase your effective cost per minute by 20-40%.

**Regulatory surcharges and fees**: USF (Universal Service Fund) contributions, state and local telecom taxes, E911 surcharges, PICC fees, number portability fees — these can add $0.002-$0.005 per minute on top of your quoted rate. A carrier quoting $0.006/min with $0.003 in surcharges costs you more than a carrier quoting $0.008/min with no surcharges.

**Minimum monthly spend**: If you commit to a monthly minimum and your actual usage falls below it, you pay the minimum anyway. Size your commitment accurately based on 3 months of call data, not on what you hope to be doing next quarter.

**Setup fees and contract terms**: Some carriers charge setup fees ($50-$500) and require 12-24 month contracts. Others are month-to-month with no setup cost. For a growing operation, flexibility has real value — getting locked into a contract with an underperforming carrier is expensive to unwind.

**Channel overage fees**: If you are on a committed channel model and burst above your allocation, overage charges can be 2-5x the base channel rate. Understand the overage structure before you sign.

### How to Compare Carriers Apples-to-Apples

Here is the formula for calculating your true effective rate:

```
True Effective Rate = (Total Monthly Carrier Invoice) / (Total Billed Minutes)
```

Do this calculation for your current carrier using the last 3 months of invoices. Then ask prospective carriers to quote you based on your actual traffic profile — volume, destination mix, average call duration, peak concurrent channels. Compare the projected True Effective Rate, not the headline per-minute rate.

For a detailed breakdown of all VICIdial operating costs including carrier expenses, see our [cost analysis guide](/blog/vicidial-cost-2026/).

---

## STIR/SHAKEN Attestation by Carrier Type: What You Will Actually Get

We touched on attestation levels in the evaluation framework. This section goes deeper, because this is where most outbound operations are unknowingly sabotaging themselves.

### The Attestation Landscape in 2026

The FCC's STIR/SHAKEN mandate has matured enough that nearly all legitimate U.S. voice carriers have implemented signing. But "implemented" does not mean "implemented well." The quality of implementation varies enormously, and the differences directly affect your calls.

**Full A-level attestation** requires three things:
1. The carrier knows the identity of the customer originating the call
2. The carrier has verified the customer is authorized to use the specific calling number
3. The carrier signs the call with their own STI certificate in real time

Most Tier 1 and reputable Tier 2 carriers can deliver this for DIDs they provision directly to your account. The problems start when the DID chain gets complicated.

### Common Scenarios That Degrade Attestation

**Scenario 1: DIDs from one carrier, trunk from another.** You bought 50 DIDs from Carrier A but your SIP trunk is through Carrier B. When Carrier B originates the call, they can verify your identity (they know you as a customer), but they cannot verify that you are authorized to use Carrier A's numbers. Result: B-level attestation at best. This is the single most common attestation problem in VICIdial operations, and most operators do not even know it is happening.

**Scenario 2: Ported numbers with incomplete registration.** You ported numbers from a previous carrier to your current one. The port completed, the numbers work, but the carrier's STIR/SHAKEN system was never updated with the new ownership records. The carrier's signing system sees a number that is not in their internal DID registry and assigns B or C attestation.

**Scenario 3: Carrier reseller arrangement.** Your "carrier" is actually a reseller sitting on top of a Tier 1 carrier's infrastructure. The reseller may not have their own STI certificate. Calls are signed by the upstream carrier, who may only know the reseller — not you. Result: B-level attestation because the signing carrier cannot verify the end customer.

**Scenario 4: Multiple trunks, inconsistent signing.** You have two SIP trunks for redundancy — one through Carrier A and one through Carrier B. Carrier A provides A-level attestation. Carrier B, your backup, provides B-level. When your failover activates, your attestation drops and your answer rates drop with it. You might not notice for days because you are focused on getting calls flowing again, not checking attestation levels.

### How to Verify Your Attestation Level

You cannot see attestation levels in VICIdial. You need to check at the SIP level or through your carrier.

**Method 1: Ask your carrier directly.** Request a report showing the attestation level assigned to your calls. Any carrier with proper STIR/SHAKEN implementation can provide this.

**Method 2: Check SIP headers.** Enable SIP debug logging on your Asterisk server (your [Asterisk configuration](/blog/vicidial-asterisk-configuration/) guide covers this) and look for the `Identity` header in outgoing INVITE messages. The header contains the attestation level. If you do not see an Identity header, the call is being sent unsigned.

**Method 3: Test with a terminating carrier tool.** Some terminating carriers and analytics providers offer tools that show the STIR/SHAKEN verification result for incoming calls. TransNexus, for example, provides a STIR/SHAKEN verification service that can tell you how your calls look from the receiving end.

> **Not sure what attestation level your calls are getting?** [Request a free audit](/free-audit/) — we will analyze your carrier's STIR/SHAKEN signing and tell you exactly what the terminating carriers see when your calls arrive.

---

## Carrier Redundancy and Failover in VICIdial

A single carrier is a single point of failure. When that carrier has an outage — and every carrier has outages — your entire operation stops. For a 25-seat call center, one hour of downtime costs roughly $375-$750 in lost productivity alone, before accounting for lost sales.

VICIdial supports multi-carrier configurations natively through its carrier entry system. Here is how to set it up properly.

### The Two-Carrier Minimum

Every production VICIdial operation should have at least two carriers configured:

- **Primary carrier**: Handles 100% of traffic under normal conditions. Selected for best combination of quality, attestation, and cost.
- **Secondary carrier**: Activated when the primary fails. May have slightly higher rates or slightly lower quality, but must still provide A-level attestation and acceptable call quality.

Some operations run a three-carrier setup: primary (70% of traffic), secondary (30% of traffic), and tertiary (failover only). Splitting traffic across two active carriers provides real-time comparison data — you can see which carrier is performing better on a daily basis.

### Configuring Failover in VICIdial

VICIdial handles carrier failover through its dial prefix and carrier configuration system. Here is the setup:

**Step 1: Create carrier entries for each carrier.**

In Admin → Carriers, create a separate carrier entry for each SIP trunk. Each entry includes the trunk name, SIP peer details, and [dial prefix](/settings/dial-prefix/) configuration.

**Step 2: Configure dial plans with failover logic.**

In your Asterisk dialplan (extensions.conf), you can configure failover using the `|` (pipe) delimiter in the Dial() application, or by using sequential priority steps:

```
; Primary carrier attempt
exten => _1NXXNXXXXXX,1,Dial(SIP/${EXTEN}@carrier_primary,60,tT)
; If primary fails, try secondary
exten => _1NXXNXXXXXX,2,Dial(SIP/${EXTEN}@carrier_secondary,60,tT)
; If both fail, set disposition
exten => _1NXXNXXXXXX,3,Hangup()
```

**Step 3: Set the [Force Dial Prefix](/settings/force-dial-prefix/) appropriately.**

The [Force Dial Prefix](/settings/force-dial-prefix/) setting in VICIdial determines which carrier entry processes the call. If you are using dial prefix routing, each carrier gets a unique prefix, and VICIdial's campaign configuration determines which prefix (and therefore which carrier) is used.

**Step 4: Test failover before you need it.**

Simulate a primary carrier failure by temporarily disabling the primary SIP trunk registration. Verify that calls automatically route to the secondary carrier. Check that caller ID, STIR/SHAKEN attestation, and call quality are acceptable on the failover path. Do this during a maintenance window, not during a production emergency.

### Monitoring Carrier Health

VICIdial does not have built-in carrier health monitoring. You need to build or implement external monitoring:

**SIP OPTIONS polling**: Configure your Asterisk server to send periodic SIP OPTIONS messages to each carrier's SBC. If the OPTIONS response stops arriving, the trunk is down. Asterisk's `qualify=yes` setting in sip.conf handles this natively — it sends periodic checks and marks the peer as unreachable if responses stop.

**CDR-based monitoring**: Parse VICIdial's call detail records (CDRs) and track failure rates per carrier trunk. A sudden spike in SIP 503 (Service Unavailable), 408 (Request Timeout), or 487 (Request Terminated) responses indicates a carrier problem. Set up alerting thresholds: if failure rate exceeds 10% in a 15-minute window, trigger a notification.

**Real-time dashboard**: VICIdial's real-time report shows active calls and waiting agents. A sudden drop in active calls with agents piling up in "WAIT" status is a strong indicator of a carrier outage. Watch the ratio — if agents are available but calls are not connecting, check your carrier trunks immediately.

### Carrier Failover Decision Matrix

| Scenario | Action | Automation Possible? |
|---|---|---|
| Primary carrier total outage | Route 100% to secondary | Yes — Asterisk failover dialplan |
| Primary carrier degraded (high PDD, low ASR) | Split traffic or route to secondary | Partial — requires monitoring + manual switch |
| Primary carrier attestation degraded | Route to secondary immediately | No — requires manual detection and switch |
| Both carriers degraded | Reduce dial level, alert management | No — requires human judgment |
| Primary carrier maintenance window (scheduled) | Pre-route to secondary | Yes — time-based dialplan rules |

---

## Rate Negotiation: How to Get Better Pricing

Carriers expect you to negotiate. The first rate they quote is never the best rate they can offer. Here is how to negotiate effectively.

### Know Your Leverage

Your negotiating power comes from three factors:

**Volume**: The more minutes you push, the better your rate. Carriers have fixed infrastructure costs and want to fill their pipes. A commitment of 100,000 minutes per month gets a significantly better rate than 10,000 minutes per month.

**Consistency**: Carriers prefer predictable, steady traffic over spiky, unpredictable volume. If you can commit to a consistent monthly volume (even with seasonal variation), you are a more attractive customer than someone whose usage swings 3x month to month.

**Payment reliability**: Carriers extending credit to VoIP customers take on risk. If you pay on time, have been with your current carrier for 12+ months without payment issues, and have good business credit, mention it. Some carriers offer rate reductions for prepayment or auto-pay.

### The Negotiation Playbook

**Step 1: Get quotes from at least three carriers.** Do not negotiate with your current carrier until you have competing quotes in hand. The quotes do not need to be from carriers you would actually switch to — they just need to be legitimate alternatives.

**Step 2: Provide your actual traffic data.** Give each carrier your CDR summary for the past 3 months: total minutes, peak concurrent channels, destination mix (% mobile vs. landline), average call duration, and geographic distribution. Carriers quote better rates when they can model your traffic accurately.

**Step 3: Ask for the right things.** Beyond the per-minute rate, negotiate:
- Billing increment (push for 6-second)
- Regulatory surcharge cap or inclusion in the per-minute rate
- Free or reduced-cost DID provisioning
- Channel burst capacity at base rates (not overage rates)
- Month-to-month terms instead of annual contracts
- Free porting for your existing DIDs
- Dedicated support contact (not a general support queue)

**Step 4: Use competing quotes transparently.** Tell your current carrier and the new prospects what you have received. "Carrier X quoted me $0.006 per minute with 6-second billing and no regulatory surcharges. Can you match or beat that?" This is not aggressive — it is how business works. Carriers expect it.

**Step 5: Negotiate quality guarantees.** The best deal in the world means nothing if call quality degrades. Ask for SLA (Service Level Agreement) commitments on:
- ASR minimum (e.g., 45% floor)
- PDD maximum (e.g., 5 seconds)
- Uptime guarantee (e.g., 99.95%)
- STIR/SHAKEN A-level attestation guarantee
- Remediation timeline for quality issues (e.g., 4-hour response for degradation events)

If a carrier will not commit to quality SLAs in writing, they are telling you something about their confidence in their own network.

---

## Configuring Carriers in VICIdial: The Technical Details

Getting a carrier contract signed is half the job. Configuring it properly in VICIdial is the other half. Misconfiguration here leads to failed calls, wrong caller IDs, and compliance problems. For a complete guide to SIP and Asterisk settings, see our [Asterisk configuration guide](/blog/vicidial-asterisk-configuration/).

### Carrier Entry Configuration

In VICIdial Admin → Carriers, each carrier entry defines how VICIdial communicates with a SIP trunk. The critical fields:

**Carrier ID and Name**: Use descriptive naming. "CARRIER_PRIMARY_THINQ" is better than "CARRIER1." When something breaks at 2am, clear naming saves time.

**Registration**: Most carriers use IP authentication (your server's IP is whitelisted on their SBC) or SIP registration (your server registers with a username and password). IP authentication is preferred for production environments because it eliminates registration failures as a point of failure.

**Dial Prefix and Dialplan**: The [Dial Prefix](/settings/dial-prefix/) setting determines what gets prepended to the number before dialing. The dialplan in the carrier entry controls how Asterisk processes the call. A common mistake: the dialplan overwrites the caller ID set by VICIdial's campaign or CID Group configuration. If your caller ID is not transmitting correctly, check the carrier entry's dialplan first.

Example of a carrier dialplan entry that preserves VICIdial's caller ID:

```
exten => _91NXXNXXXXXX,1,AGI(agi://127.0.0.1:4577/call_log)
exten => _91NXXNXXXXXX,2,Dial(SIP/${FILTER(0-9,${EXTEN:1})}@carrier_trunk,,tTor)
exten => _91NXXNXXXXXX,3,Hangup()
```

The key: do not include any `Set(CALLERID(num)=...)` commands in the carrier dialplan unless you specifically need to override VICIdial's caller ID. If the carrier requires a specific format for the caller ID header (e.g., E.164 format with country code), handle it in the `fromuser` or `sendrpid` settings in sip.conf, not in the dialplan.

### SIP Trunk Configuration in sip.conf

Each carrier needs a SIP peer definition in `/etc/asterisk/sip.conf`. The critical parameters:

```
[carrier_primary]
type=peer
host=sip.carrier.com           ; Carrier's SBC address
port=5060                       ; SIP port (5060 standard, some carriers use 5080)
context=trunkinbound            ; Context for inbound calls from this carrier
dtmfmode=rfc2833                ; DTMF handling — rfc2833 is standard
disallow=all                    ; Reset codec preferences
allow=ulaw                      ; G.711 ulaw — best quality, highest bandwidth
allow=alaw                      ; G.711 alaw — fallback
allow=g729                      ; G.729 — lower bandwidth, acceptable quality
insecure=port,invite            ; Required for most IP-authenticated trunks
qualify=yes                     ; Enable SIP OPTIONS health checks
qualifyfreq=60                  ; Check every 60 seconds
nat=no                          ; Set based on your network topology
sendrpid=yes                    ; Send Remote-Party-ID header (some carriers need this for CID)
trustrpid=yes                   ; Trust incoming RPID from carrier
```

**Codec order matters.** List your preferred [codec](/glossary/codec/) first. G.711 (ulaw/alaw) provides the best audio quality but uses 87.2 kbps per call. G.729 uses only 31.2 kbps but has slightly lower quality. For most VICIdial operations on modern bandwidth, G.711 is the right choice. Only use G.729 if you are bandwidth-constrained.

**The NAT question**: If your VICIdial server is behind a NAT firewall (common in cloud deployments), set `nat=force_rport,comedia` to handle NAT traversal. Incorrect NAT settings are the number one cause of one-way audio — calls connect but the lead cannot hear the agent, or vice versa.

### Testing a New Carrier Before Production

Never cut over to a new carrier without testing. Here is the process:

1. **Configure the carrier entry in VICIdial** as a secondary carrier with a unique dial prefix
2. **Place 50-100 test calls** to numbers you control across all three major carriers
3. **Verify caller ID transmission**: Does the correct CID display on the receiving phone?
4. **Verify STIR/SHAKEN**: Check that calls arrive with A-level attestation at the terminating end
5. **Measure PDD**: Time the delay from dial to first ringback on at least 20 calls. Calculate the average.
6. **Test audio quality**: Have a two-way conversation on at least 10 calls. Listen for echo, one-way audio, choppy audio, or latency.
7. **Test under load**: Run a small campaign (5-10 agents) through the new carrier for a full day. Compare ASR, ACD, and connect rates against your primary carrier's baseline.
8. **Test failover**: Disable the new carrier's trunk and verify that calls route correctly to your existing carrier.

Only after all tests pass should you begin migrating production traffic to the new carrier.

> **Want help evaluating a new carrier or optimizing your existing trunk configuration?** [Request a free audit](/free-audit/) — we will review your SIP configuration, measure your call quality metrics, and identify specific changes that will improve your connect rates.

---

## The Carrier Evaluation Scorecard

Here is the complete scorecard we use when evaluating carriers for ViciStack-managed operations. Score each category 1-5, then weight by importance.

| Category | Weight | What to Evaluate | Red Flags |
|---|---|---|---|
| **STIR/SHAKEN attestation** | 25% | A-level for all DIDs, own certificate, real-time signing | B/C attestation, third-party certificate, no clear answer |
| **Call quality (ASR/PDD/MOS)** | 25% | ASR above 45%, PDD under 3 sec, MOS above 3.8 | No quality metrics available, PDD over 5 sec, frequent audio issues |
| **Pricing** | 15% | True effective rate including all fees, billing increment, volume pricing | 1-minute billing, hidden surcharges, long-term contract required |
| **Channel capacity** | 10% | Adequate base + burst channels, no congestion during peaks | CONGESTION dispositions, no burst option, overage penalties |
| **DID services** | 10% | Fast provisioning, clean numbers, CNAM support, broad area code coverage | Slow provisioning, recycled numbers, limited area codes |
| **Redundancy/uptime** | 10% | 99.95%+ uptime SLA, redundant infrastructure, geographic diversity | No SLA, single data center, history of extended outages |
| **Support** | 5% | Dedicated contact, fast response times, technical competence | Ticket-only support, 24+ hour response, non-technical support staff |

**Scoring guidelines:**
- **5**: Exceptional — best-in-class for this category
- **4**: Good — meets all requirements with minor gaps
- **3**: Acceptable — meets minimum requirements
- **2**: Below standard — noticeable gaps that affect operations
- **1**: Unacceptable — fails to meet minimum requirements

**Minimum passing score: 3.5 weighted average.** Any carrier scoring below 3.5 is not suitable for a production outbound VICIdial operation. Any carrier scoring below 2.0 in STIR/SHAKEN attestation or call quality should be disqualified regardless of overall score.

---

## Common Carrier Problems in VICIdial and How to Fix Them

These are the carrier-related issues we see most frequently across ViciStack-managed operations. If you are experiencing any of these, the fix is almost always carrier-side, not VICIdial-side.

### Problem 1: Calls Connecting But No Audio (One-Way or No-Way Audio)

**Symptoms**: Agents hear ringing, the call appears to connect, but either the agent cannot hear the lead, the lead cannot hear the agent, or neither can hear the other.

**Cause**: Almost always a NAT/firewall issue. The SIP signaling (call setup) passes through correctly, but the RTP media stream (actual audio) is being blocked or misrouted. This happens when your VICIdial server is behind a NAT and the SIP configuration does not account for it.

**Fix**: Set `nat=force_rport,comedia` in sip.conf for the carrier peer. Ensure your firewall allows RTP traffic on ports 10000-20000 (or whatever range is configured in rtp.conf). If you are in a cloud environment, verify that your provider's security groups allow UDP traffic on both SIP and RTP port ranges. See the [Asterisk configuration guide](/blog/vicidial-asterisk-configuration/) for detailed NAT troubleshooting.

### Problem 2: Caller ID Not Transmitting Correctly

**Symptoms**: Recipients see the wrong number, a blank caller ID, or your carrier's default number instead of the CID set in VICIdial.

**Cause**: Usually one of three things:
1. The carrier entry's dialplan is overwriting the CID with a `Set(CALLERID(num)=...)` command
2. The carrier's SBC is stripping or replacing the From header based on their trunk configuration
3. The `fromuser` setting in sip.conf is overriding the CID

**Fix**: Check the carrier entry dialplan in VICIdial Admin → Carriers for any CALLERID manipulation. Check sip.conf for `fromuser` settings on the carrier peer. If both are clean, contact your carrier — they may be stripping CID at their SBC level. Some carriers require you to register each DID in their portal before they will pass it through as caller ID. The [Force Dial Prefix](/settings/force-dial-prefix/) setting can also affect CID transmission if configured incorrectly.

### Problem 3: High Failure Rate During Peak Hours

**Symptoms**: Call failures spike between 10am-2pm and 5pm-8pm. VICIdial logs show CONGESTION or CHANUNAVAIL dispositions. Agents sit in WAIT status despite having leads to call.

**Cause**: Channel exhaustion. Your carrier does not have enough capacity to handle your peak traffic, or you have hit your concurrent channel limit.

**Fix**: Check your carrier's channel allocation against your actual peak usage. In VICIdial's real-time report, note the maximum number of simultaneous calls during peak hours. If that number is at or near your channel limit, you need more channels. If it is well below your limit and calls are still failing, the problem is on the carrier's side — their infrastructure cannot handle the load. Time to escalate or switch carriers.

### Problem 4: Intermittent Call Quality Degradation

**Symptoms**: Audio quality is good most of the time but periodically degrades — choppy audio, echo, robotic-sounding voices, or noticeable delay in conversation.

**Cause**: Network congestion or routing changes. Carriers sometimes shift traffic to alternate routes during peak periods, and those alternate routes may have more hops, more transcoding, or more congestion.

**Fix**: Deploy a VoIP monitoring tool (VoIPmonitor is open-source and works well with Asterisk) to measure MOS, jitter, and packet loss per call. Correlate quality degradation with time of day and carrier trunk. If the pattern is consistent, present the data to your carrier and request routing optimization. If they cannot or will not fix it, this is a strong argument for a carrier switch.

### Problem 5: STIR/SHAKEN Attestation Drops

**Symptoms**: Answer rates decline gradually over 1-2 weeks with no change in list quality, dial strategy, or DID pool. DID reputation checks come back clean.

**Cause**: Your carrier's STIR/SHAKEN signing may have changed — certificate renewal issues, system updates that altered signing behavior, or changes to their verification database that downgraded your attestation level. This is insidious because nothing in VICIdial will tell you it is happening.

**Fix**: Contact your carrier and ask for a current attestation report for your DIDs. If attestation has dropped from A to B, demand immediate remediation. While waiting for the fix, consider routing traffic through your secondary carrier if they provide A-level attestation. For ongoing protection, build attestation verification into your weekly monitoring routine.

---

## Frequently Asked Questions

### How many SIP trunk providers should I use for VICIdial?

At minimum, two. One primary and one secondary for failover. The cost of maintaining a second carrier relationship — even if it only carries 5-10% of your traffic under normal conditions — is trivial compared to the cost of a total outage when your single carrier goes down. For enterprise operations (50+ seats), consider three carriers: primary (60-70% of traffic), secondary (30-40%), and tertiary (failover only). Distributing traffic across two active carriers also gives you real-time quality comparison data that is invaluable for ongoing carrier evaluation.

### What is a good per-minute rate for outbound SIP trunking in 2026?

For U.S. domestic outbound calling at moderate volume (50,000-200,000 minutes per month), expect to pay $0.005-$0.010 per minute with 6-second billing increments. At higher volume (500,000+ minutes), rates of $0.003-$0.006 are achievable. But remember: the per-minute rate is not the number that matters. The True Effective Rate — total invoice divided by total minutes — is what you should track. A carrier quoting $0.005/min with 1-minute billing and $0.003 in surcharges costs you more than a carrier quoting $0.008/min with 6-second billing and no surcharges. Do the math on your actual traffic pattern before choosing based on headline rate.

### How do I know if my carrier is providing A-level STIR/SHAKEN attestation?

Ask them directly. Any carrier with a proper STIR/SHAKEN implementation can tell you the attestation level assigned to your calls. If they cannot answer the question, that is a problem. You can also verify by capturing SIP headers on your Asterisk server — enable SIP debug logging and look for the Identity header on outgoing calls. The header contains the attestation level. If there is no Identity header, your calls are being sent unsigned, which is worse than any attestation level. For a deeper technical walkthrough, see our [STIR/SHAKEN implementation guide](/blog/stir-shaken-vicidial-guide/).

### Can I use different carriers for different campaigns in VICIdial?

Yes. VICIdial's carrier system supports this through [dial prefixes](/settings/dial-prefix/). Each carrier entry is associated with a dial prefix, and each campaign can be configured with a specific [Force Dial Prefix](/settings/force-dial-prefix/) that routes calls through the designated carrier. This is useful when you have campaigns with different quality requirements — for example, a high-value sales campaign routed through your premium carrier and a survey campaign routed through your lower-cost carrier. Just ensure that both carriers provide A-level STIR/SHAKEN attestation, or your lower-tier campaigns will suffer from reputation damage that can spill over to your DIDs.

### What causes CONGESTION dispositions in VICIdial call logs?

CONGESTION means VICIdial (via Asterisk) attempted to place a call but the carrier returned a SIP 503 (Service Unavailable) response — typically because you have exceeded your concurrent channel limit or the carrier's network is overloaded. Check your peak concurrent call count against your channel allocation. If you are hitting the limit during peak hours, you need more channels or burst capacity. If CONGESTION dispositions appear randomly throughout the day at low call volumes, the problem is on the carrier's side — their infrastructure may be experiencing issues. A small number of CONGESTION dispositions (under 1% of total attempts) is normal. Above 3% is a problem that needs immediate attention.

### How do I test call quality on my SIP trunks?

Start with manual testing: place calls to phones on each major carrier and listen for audio quality, echo, delay, and one-way audio issues. For ongoing monitoring, deploy VoIPmonitor or a similar open-source tool that captures RTP streams and calculates [MOS scores](/glossary/mos-score/) per call. You can also use Asterisk's built-in `sip set debug on` command to inspect SIP signaling in real time. For PDD measurement, timestamp the SIP INVITE and the first 180 Ringing or 200 OK response — the difference is your [post-dial delay](/glossary/post-dial-delay/). Run these measurements across at least 100 calls at different times of day to get a reliable baseline. Compare your results against the benchmarks in the quality metrics table earlier in this guide.

### Should I choose a carrier that specializes in outbound calling?

Yes, with a caveat. Carriers that specialize in outbound calling understand the needs of high-volume operations: they offer appropriate channel models, they are experienced with STIR/SHAKEN for outbound caller ID, and their support teams understand predictive dialer traffic patterns. General-purpose VoIP carriers may offer lower rates but lack the outbound-specific infrastructure and expertise. The caveat: some "outbound specialist" carriers have reputations with terminating carriers as spam originators. If a carrier's other customers are burning DIDs and generating complaints, the carrier's IP ranges and network may carry negative reputation that affects your calls. Ask potential carriers about their acceptable use policies and how they handle customers who violate calling regulations.

### What happens if my SIP trunk provider goes out of business or gets shut down?

This is not hypothetical — several VoIP carriers have been shut down by the FCC in recent years for facilitating illegal robocalling. If your sole carrier disappears, your operation stops immediately. This is the strongest argument for maintaining at least two active carrier relationships. Keep your secondary carrier configured, tested, and carrying at least some regular traffic so you know it works. Keep copies of your DID ownership records and be prepared to port your numbers to a new carrier on short notice. Porting typically takes 5-10 business days for standard requests, though emergency ports can sometimes be expedited. Having your DIDs registered on [FreeCallerRegistry.com](https://freecallerregistry.com/) and your CNAM records documented independently of your carrier makes the transition smoother. For a complete view of operational risk management, see our [cost analysis guide](/blog/vicidial-cost-2026/).

---

### The Bottom Line

Your carrier is not a commodity vendor. It is the foundation of your entire dialing operation — the infrastructure that determines whether your calls connect, whether they sound good, whether they are trusted by terminating carriers, and whether your DIDs survive long enough to be useful. Choosing a carrier on rate alone is like choosing an office building on rent alone without checking whether it has electricity.

The operators who are winning in 2026 are the ones who treat carrier selection as a strategic decision. They measure quality metrics. They verify attestation levels. They maintain redundancy. They negotiate from a position of data, not desperation. They know exactly what they are paying and exactly what they are getting for it.

You now have the framework, the metrics, and the technical knowledge to evaluate your current carrier — or choose a better one. The only question is whether you will actually measure what matters or keep assuming the cheapest rate is the best deal.

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-carrier-selection).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
