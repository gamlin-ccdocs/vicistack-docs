# VICIdial for Political Campaigns & Voter Contact

**Everything you need to run voter contact operations on VICIdial — campaign configuration, volunteer phone banks, GOTV dialing, robocall exemptions, survey scripting, voter file segmentation, and the compliance rules that actually matter for political callers. Built from real campaign cycle data.**

---

Political campaigns are the most time-compressed, highest-stakes outbound calling operations that exist. A commercial call center has months or years to optimize. A political campaign has weeks — sometimes days — before Election Day, and every unanswered call is a voter who doesn't hear your message. The margin between winning and losing a competitive race can be a few hundred contacts in the right precinct.

VICIdial is the dialer of choice for political operations that need scale without SaaS per-seat pricing eating their budget. A congressional campaign running 50 volunteer phone bank seats for 8 weeks would pay $40,000-$80,000 on a platform like Five9 or Convoso. On VICIdial, the all-in infrastructure cost is $3,000-$8,000 for the same period. When you're spending donor money, that math matters.

But political dialing is different from commercial dialing in ways that catch people off guard. The regulatory framework is different (political calls get specific FCC exemptions that commercial callers don't). The workforce is different (volunteers behave nothing like paid agents). The data is different (voter files have fields that no commercial lead list contains). And the operational tempo is different — you're ramping from zero to maximum volume and back to zero in a single election cycle.

This guide covers all of it: VICIdial configuration for political voter contact, volunteer vs. paid phone bank setup, the FCC exemptions and limitations specific to political calling, [predictive](/glossary/predictive-dialing/) vs. [preview dialing](/glossary/preview-dialing/) for different political use cases, [voicemail drop](/glossary/voicemail-drop/) for political messaging, voter file segmentation, GOTV operations, and compliance. For the political industry overview, see our [political campaigns industry page](/industries/political-campaigns/).

---

## Why Political Dialing Is Fundamentally Different

Before you configure anything in VICIdial, you need to understand what makes political calling a distinct animal from commercial outbound operations.

**1. You have FCC exemptions that commercial callers don't.** Political calls are partially exempt from the [TCPA](/glossary/tcpa/). Specifically, live calls from political campaigns to landlines and cell phones do not require prior express consent under federal law. Pre-recorded political messages (robocalls) to landlines are also exempt. However — and this is where campaigns get into trouble — pre-recorded calls and autodialed calls to cell phones still require prior express consent even for political messages. We'll break down the full exemption matrix below.

**2. Your workforce is mostly volunteers.** A commercial call center hires experienced agents, trains them for days or weeks, and expects consistent performance. A political phone bank onboards 30 new volunteers on a Tuesday night, most of whom have never used a dialer before, and needs them making calls within 15 minutes. Your VICIdial agent interface must be radically simplified compared to a commercial deployment.

**3. The data comes from voter files, not lead vendors.** Every state maintains a voter registration database. These files contain fields that don't exist in commercial data: party registration, voting history (which elections the person voted in, not who they voted for), precinct, congressional district, state legislative district, age, and sometimes ethnicity modeling scores. Your VICIdial list structure needs custom fields to capture this data and filter on it.

**4. The timeline is compressed and non-negotiable.** Election Day doesn't move. A commercial campaign can extend timelines if results are lagging. A political campaign that's behind on voter contacts in October can't push the election to December. This means your VICIdial deployment must be operational immediately — there's no luxury of weeks-long testing and optimization cycles.

**5. Contact goals are different from commercial goals.** A commercial campaign wants to sell something. A political campaign wants one or more of the following: identify supporters (voter ID), persuade undecided voters, mobilize known supporters to vote (GOTV), recruit volunteers, or raise money (fundraising calls). Each goal requires different VICIdial campaign configurations, scripts, and dispositions.

**6. You're dialing the same universe repeatedly.** A commercial operation buys new lists. A political campaign works the same voter file for months. A registered voter in a target precinct might get a voter ID call in August, a persuasion call in September, two GOTV calls in October, and a reminder on Election Day. Your list management and disposition tracking must account for this multi-touch lifecycle without over-calling individuals.

---

## VICIdial Campaign Configuration for Political Voter Contact

### Campaign Types You'll Need

Most political operations need 3-5 separate VICIdial campaigns running simultaneously, each configured differently:

| Campaign | Purpose | Dial Method | Workforce |
|---|---|---|---|
| Voter ID | Identify supporter/undecided/opponent | Predictive (RATIO) | Volunteers or paid |
| Persuasion | Deliver message to undecided voters | Preview | Trained volunteers or paid |
| GOTV | Mobilize known supporters to vote | Predictive (ADAPT_TAPERED) | Volunteers or paid |
| Fundraising | Solicit donations | Preview | Paid agents (experienced) |
| Robocall/VM Drop | Deliver recorded message at scale | Broadcast/AMD + VM Drop | No live agents |

Create each as a separate VICIdial campaign. Do not try to run voter ID and persuasion out of the same campaign with different scripts — the dial method, pacing, and agent workflow are too different.

### Voter ID Campaign Configuration

Voter identification is the bread and butter of political phone banking. The goal is simple: call through a voter file and categorize each contact as a supporter, opponent, undecided, or lean. This data drives every subsequent campaign decision — who gets persuasion calls, who gets GOTV contact, where the campaign allocates canvassing resources.

```
Campaign Type: Outbound
Dial Method: RATIO
Auto Dial Level: 1.0 (volunteer) or 1.5-2.5 (paid agents)
Dial Timeout: 26 seconds
AMD: OFF (for volunteer campaigns) or ON (for paid agent campaigns)
```

**Why RATIO instead of full predictive for voter ID?** Two reasons. First, volunteer phone banks can't handle the pacing of aggressive predictive dialing. When a volunteer finishes a call, they need a few seconds to disposition and mentally prepare for the next one. If the dialer immediately slams another call into their headset, volunteers get flustered, make mistakes, and quit. Second, voter ID calls are short (60-90 seconds average), which means the dialer can over-predict and generate abandoned calls at rates that burn through your voter file wastefully.

For paid agent voter ID operations with experienced dialers, you can push to ADAPT_TAPERED with an auto dial level starting at 2.0 and an adaptive maximum of 3.0. Paid agents can handle the pace.

**AMD for voter ID:** Turn it off for volunteer campaigns. Volunteers don't know what to do when they get connected to a call that was AMD-classified as HUMAN but is actually a voicemail greeting mid-sentence. They freeze, they hang up, they get confused. For volunteer phone banks, just let every answered call go to the volunteer and let them handle voicemails manually — it's simpler and the volunteer experience matters more than raw efficiency.

For paid agent voter ID, turn AMD on with the same residential-optimized parameters used in commercial campaigns:

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

### GOTV Campaign Configuration

Get Out The Vote operations are the final push — typically the last 4-7 days before Election Day, including Election Day itself. You're calling known supporters (identified during voter ID and persuasion phases) and making sure they have a plan to vote. This is maximum-volume, maximum-urgency dialing.

```
Campaign Type: Outbound
Dial Method: ADAPT_TAPERED
Auto Dial Level: 2.0 (starting)
Adaptive Maximum Dial Level: 4.0
Adaptive Dropped Percentage: 2.5
Dial Timeout: 22 seconds
AMD: ON (even for volunteers — GOTV is about volume)
```

**Key differences from voter ID:**

- **Higher adaptive maximum (4.0).** GOTV lists are pre-filtered to known supporters, which means smaller lists with higher contact rates. You need the dialer to be aggressive because you're racing the clock.
- **Shorter dial timeout (22 seconds).** During GOTV, you'd rather move on quickly to the next number than wait for a slow pickup. Every second counts when you're trying to reach thousands of voters before polls close.
- **AMD ON even for volunteers.** During GOTV, explain to volunteers that some calls will be brief or sound odd — the efficiency gain is worth the minor confusion. Alternatively, configure AMD to send machine-detected calls to a voicemail drop with a "remember to vote" message rather than filtering them entirely.

### Persuasion Campaign Configuration

Persuasion calls target undecided voters or soft leaners identified during voter ID. These are longer conversations (3-7 minutes) that require the caller to deliver talking points, handle objections, and read the voter's response. This is not a volume game — it's a quality game.

```
Campaign Type: Outbound
Dial Method: MANUAL or RATIO at 1.0
Auto Dial Level: 1.0
Dial Timeout: 30 seconds
AMD: OFF
```

**Manual or 1:1 ratio dialing** is correct for persuasion. The caller needs to see the voter's information before the call connects — their name, party registration, voting history, any notes from prior contacts. [Preview dialing](/glossary/preview-dialing/) lets the volunteer review this data and mentally prepare the approach. A registered independent who voted in the last three generals but skipped the primary is a different conversation than a registered Democrat who hasn't voted since 2020.

Configure VICIdial's preview mode by setting the dial method to MANUAL with the "Manual Dial Preview" option enabled, or use RATIO at 1.0 with a preview timeout that gives the agent time to review the screen before the call launches.

---

## Volunteer Phone Bank Setup

Volunteer phone banks are the backbone of political calling operations. Getting VICIdial configured correctly for volunteers is the difference between a productive evening of calls and a room full of frustrated people who never come back.

### Agent Interface Simplification

The default VICIdial agent interface has dozens of buttons, fields, and options. A volunteer needs to see exactly five things:

1. **The voter's name** (big, prominent)
2. **The script** (with clear talking points and response options)
3. **Disposition buttons** (4-6 options maximum)
4. **A "Next Call" button** (or auto-advance)
5. **A "Problem/Help" button** (flags a supervisor)

Everything else is noise. Use VICIdial's agent screen customization to hide fields and buttons that volunteers don't need. Strip out transfer options, manual dial, callback scheduling, and any commercial features. The agent screen should be so simple that someone with zero dialer experience can make calls within 5 minutes of sitting down.

**VICIdial settings for volunteer simplification:**

```
Agent Screen: Custom (use screen labels to hide unnecessary fields)
Disable Disposition Abort: Y (prevent accidental skips)
Agent Screen Labels: Custom label set hiding commercial fields
Script: Auto-load (script appears automatically, no clicks required)
Manual Dial Enabled: N (volunteers should not freestyle-dial)
Transfer Options: DISABLED (no transfers for basic phone banks)
Pause Codes: Simplified (BREAK, BATHROOM, DONE — that's it)
```

### Volunteer Onboarding Workflow

Here's the practical workflow for getting 30 volunteers operational in 15 minutes:

**Pre-event setup (done by campaign staff/IT):**

1. Create VICIdial user accounts in bulk (volunteer01 through volunteer50)
2. Assign all accounts to the correct campaign
3. Set a universal password that's easy to communicate verbally (change after the event)
4. Pre-load the voter list and verify the hopper is populated
5. Test the campaign with 2-3 test calls
6. Print a one-page instruction sheet: login URL, username, password, "click green button to start, click disposition when done"

**Event execution:**

1. Volunteers arrive, sit at stations (laptops with headsets or personal cell phones)
2. Brief 10-minute orientation: why we're calling, what the script says, how to disposition
3. Volunteers log in and click "Resume" to start receiving calls
4. Campaign staff (logged in as VICIdial managers) monitor the real-time report for stuck agents, long pauses, and errors
5. Walk the room continuously for the first 30 minutes — most volunteer confusion happens in the first 10 calls

### WebRTC vs. Softphones vs. Personal Phones

Volunteers can connect to VICIdial calls three ways:

**ViciPhone (WebRTC) — Recommended:** Volunteers use their laptop's browser and a headset. No software installation required. VICIdial's built-in WebRTC support means volunteers just log in and their browser becomes their phone. This is the easiest setup for ad-hoc phone banks in community centers, campaign offices, or living rooms.

**SIP Softphones:** More reliable audio quality than WebRTC but requires software installation (Zoiper, MicroSIP, etc.). Only practical if you control the machines — asking 30 random volunteers to install SIP software is a non-starter.

**Personal cell phone callback:** VICIdial can call the volunteer's personal cell phone and bridge them into campaign calls. This works for remote/distributed phone banks where volunteers call from home. Configure the agent phone as an external number and VICIdial will call the volunteer first, then bridge the voter call. The downside: it uses two call legs per contact (doubling your SIP costs) and adds 5-10 seconds of latency per call connection.

For most in-person phone bank events, WebRTC with decent headsets is the right answer. Budget $15-$25 per USB headset and buy 30-50 of them. They're reusable across the entire campaign cycle.

---

## Paid Agent Operations for Political Campaigns

Some political operations — particularly large statewide races, presidential campaigns, national party committees, and political consulting firms — run paid agent operations alongside or instead of volunteer phone banks. The configuration differs significantly.

### When to Use Paid Agents vs. Volunteers

| Factor | Volunteers | Paid Agents |
|---|---|---|
| Cost | Free (labor), but need space/equipment | $12-$25/hour loaded cost |
| Consistency | Highly variable | Predictable output |
| Training depth | Minimal (15-minute orientation) | Full training (scripts, objection handling) |
| Availability | Evenings and weekends, unreliable | Scheduled shifts, reliable |
| Best for | Voter ID, simple GOTV | Fundraising, complex persuasion, high-volume GOTV |
| Scale | Limited by volunteer recruitment | Limited by budget |
| Quality control | Difficult | Standard QA processes |

Most competitive campaigns use a hybrid: volunteers for basic voter ID and GOTV, paid agents for fundraising calls and targeted persuasion in the final weeks.

### Paid Agent Campaign Settings

Paid agents can handle more aggressive dialing configurations:

```
Dial Method: ADAPT_TAPERED
Auto Dial Level: 2.0 (starting)
Adaptive Maximum Dial Level: 3.5
Adaptive Dropped Percentage: 2.5
AMD: ON (with tuned parameters)
Wrap-Up Time: 5 seconds (paid agents disposition quickly)
Agent Pause After Call: N (auto-advance to next call)
```

Paid agents also unlock VICIdial features that volunteers can't use effectively:

- **Warm transfers** to candidate or senior staff for high-value donor calls
- **Scheduled callbacks** for voters who request a specific time
- **Three-way calling** for connecting voters with volunteer coordinators
- **Complex disposition trees** with 10-15 outcome codes for detailed voter modeling

---

## Political Robocall and Autodialer Exemptions: What the FCC Actually Says

This is the section that matters most from a compliance standpoint, and it's the section that campaigns get wrong most often. The rules for political calling are different from commercial calling — but "different" does not mean "anything goes."

### The Exemption Matrix

Here's the actual regulatory framework as of 2026, broken down by call type and phone type:

| Call Type | Landlines | Cell Phones |
|---|---|---|
| Live call, no autodialer | No consent needed | No consent needed |
| Live call, autodialer-assisted | No consent needed | **Consent required** |
| Pre-recorded (robocall) | No consent needed (exempt) | **Prior express consent required** |
| Text message (SMS/MMS) | N/A | **Prior express consent required** |

**The critical distinction:** Live calls made by actual humans — even when VICIdial's predictive dialer is selecting and dialing the number — to landlines require no prior consent for political messaging under 47 U.S.C. Section 227. The FCC has historically treated predictive dialers used for live-agent political calls as exempt when the call is delivered to a human agent before or simultaneously with the called party answering.

**However, autodialed calls to cell phones are a different story.** The TCPA's autodialer restrictions apply to cell phones regardless of whether the call is political. If VICIdial is dialing cell phone numbers using predictive or automatic dialing features and delivering a pre-recorded message (or if the call is abandoned and the voter hears silence or a recording), you need prior express consent for that cell phone number.

**What this means practically:**

1. **Predictive dialing to landlines for live voter contact** — permitted without consent under FCC political exemption
2. **Predictive dialing to cell phones for live voter contact** — the FCC's position has shifted over time, but most campaign compliance attorneys recommend obtaining consent for autodialed calls to cell phones, even with a live agent
3. **Robocalls/pre-recorded messages to landlines** — permitted without consent (political exemption)
4. **Robocalls/pre-recorded messages to cell phones** — requires prior express consent, period
5. **Voicemail drops to cell phones** — treated as a call to a cell phone; consent recommended
6. **SMS to cell phones** — requires prior express consent

### The National DNC List Exemption

Political calls are **exempt from the National [Do Not Call](/glossary/dnc/) Registry**. You can legally call voters who are on the federal DNC list for political purposes. This is a significant advantage — the DNC list covers roughly 245 million numbers. Commercial callers can't touch them. Political callers can.

**However:** State do-not-call lists may not provide the same exemption. Some states (Indiana, Colorado, Missouri, and others) have state-level DNC lists that may apply to political calls or have separate registration requirements for political callers. Check your target states. When in doubt, consult your campaign's legal counsel.

Also, if a voter specifically requests not to be called by your campaign, you must honor that request. The federal DNC exemption doesn't override an individual's direct opt-out to your organization. Add them to your campaign's internal DNC list in VICIdial immediately.

### State-Level Restrictions That Override Federal Exemptions

Federal exemptions are the floor, not the ceiling. Several states impose additional restrictions on political calling:

**Calling hours:** Most states restrict telemarketing calls (including political calls) to 8:00 AM - 9:00 PM local time. Some states are stricter — North Dakota restricts to 8:00 AM - 9:00 PM but has pushed for 8:00 AM - 8:00 PM for automated calls. Always check current state law for every state you're dialing.

**Registration requirements:** Some states require political callers to register with the state before making calls. Florida, Indiana, Kentucky, and others have registration or disclosure requirements specific to political telemarketing.

**Caller ID disclosure:** Federal law requires accurate caller ID information. For political calls, the caller ID must identify either the campaign, the political committee, or provide a callback number where voters can reach the organization. Never spoof caller ID on political calls — it's both illegal and politically disastrous if exposed.

**Robocall restrictions by state:** Several states have their own robocall laws that are stricter than federal law. Some states require consent for pre-recorded calls to landlines (overriding the federal exemption), some ban robocalls during certain hours, and some require specific disclosures at the beginning of the recorded message. Check every target state before launching automated message campaigns.

For a complete compliance configuration walkthrough, see our [TCPA compliance checklist](/blog/vicidial-tcpa-compliance/).

> **Not sure if your VICIdial setup is compliant for political calling?** Our team has configured dialing operations for campaigns at every level — from city council to presidential. [Request a free audit](/free-audit/) and we'll review your campaign configuration, DID setup, and compliance settings before you start dialing.

---

## Preview Dialing for Door-Knock Follow-Up

One of the most effective political calling use cases is following up with voters who were contacted during canvassing (door-knocking) operations. A canvasser knocks on a door, the voter isn't home, and the campaign needs a phone follow-up to complete the contact.

### Configuration for Canvass Follow-Up

```
Campaign Type: Outbound
Dial Method: MANUAL (true preview)
Auto Dial Level: 1.0
AMD: OFF
Script: Canvass follow-up script with canvasser notes visible
```

[Preview dialing](/glossary/preview-dialing/) is essential here because the caller needs to see the canvasser's notes before the call connects. If the canvasser noted "spoke with spouse, voter at work, call back after 5 PM," the phone caller needs that context. Predictive dialing would connect the call before the agent reads the notes, resulting in a poorly handled contact.

**VICIdial custom fields for canvass integration:**

Load canvass data into VICIdial with these custom fields mapped from your canvassing app (MiniVAN, Reach, ThruTalk, etc.):

| Custom Field | Data |
|---|---|
| canvass_date | Date of door knock |
| canvass_result | Home/not home/refused/supportive/undecided |
| canvasser_notes | Free-text notes from the canvasser |
| canvasser_name | Who knocked |
| support_score | 1-5 scale from canvass interaction |

When the phone bank volunteer opens the call in preview mode, they see the canvasser's notes and can tailor the conversation: "Hi, I'm calling from [Campaign] — I believe one of our volunteers stopped by your home last Saturday. We just wanted to follow up..."

This integration between field canvassing data and phone banking is one of the highest-ROI uses of VICIdial in political campaigns. Voters who've had a door knock followed by a phone call are significantly more likely to vote (academic studies show 3-8 percentage point increases in turnout among voters who receive multiple contact types).

---

## Predictive Dialing for Large-Scale Voter ID

When the campaign needs to identify supporters across an entire congressional district (400,000-700,000 registered voters), precinct-level canvassing and preview dialing are too slow. This is where VICIdial's [predictive dialing](/glossary/predictive-dialing/) earns its place.

### Configuration for High-Volume Voter ID

```
Campaign Type: Outbound
Dial Method: ADAPT_TAPERED
Auto Dial Level: 2.0 (starting)
Adaptive Maximum Dial Level: 3.5
Adaptive Dropped Percentage: 2.5
Dial Timeout: 24 seconds
AMD: ON
Hopper Level: 500
```

At 20 agents running predictive with a 2.5x dial ratio, you're pushing through roughly 4,000-6,000 dials per hour. Over a 4-hour evening phone bank session, that's 16,000-24,000 dials with a contact rate of 6-10%, yielding 1,000-2,400 voter contacts per night. Across a 6-week voter ID phase, that's 30,000-100,000 identified voters — enough to build a meaningful targeting model for a congressional race.

**List loading for large voter ID operations:**

Voter files are large. A single congressional district file might contain 500,000 records. Don't load the entire file into a single VICIdial list — break it into manageable segments:

- **By precinct or ward:** Create separate VICIdial lists per precinct. This lets you prioritize high-turnout precincts or target competitive areas.
- **By voter score:** If your campaign uses voter modeling (most do), segment by support score. Call persuadable voters (scores 40-60) before strong supporters (scores 80+) or strong opponents (scores 0-20).
- **By contact priority:** Voters with cell phones on file should be in a separate list with different dialing rules than landline-only voters (see compliance section above).

Use VICIdial's [list management](/blog/vicidial-list-management/) features to organize and prioritize these segments. Load lists in batches and use the list mix feature to weight priority segments higher in the hopper.

---

## Voicemail Drop for Political Messaging

[Voicemail drop](/glossary/voicemail-drop/) — also called ringless voicemail (RVM) — is enormously popular in political campaigns because it delivers a recorded message at scale without requiring live agents. A candidate's 60-second message can reach 50,000 voters in an afternoon.

### How It Works in VICIdial

VICIdial's AMD system classifies answered calls as HUMAN or MACHINE. When a call is classified as MACHINE (answering machine/voicemail), VICIdial can automatically play a pre-recorded message — the voicemail drop. The voter checks their voicemail and hears the candidate's message.

**Configuration:**

```
Answering Machine Message: Y
AMD Send to Agent: N (for machine-classified calls)
Answering Machine Message Filename: candidate_message_v3
Answering Machine Message Delay: 1200 (milliseconds — wait for the beep)
```

**Recording best practices for political VM drops:**

- Keep it under 60 seconds. Voicemail systems cut off at 60-90 seconds, and voter attention drops after 30.
- The candidate records the message personally. A message from "John Smith, your state representative" in the candidate's own voice dramatically outperforms a staff-recorded message.
- Include a clear call to action: "Vote on November 5th," "Visit our website," or "Call us back at [number]."
- State the campaign's identity at the beginning, not the end. "Hi, this is Jane Doe, running for City Council in District 4" should be the first thing the voter hears.
- Record at high audio quality (16-bit WAV, converted to the appropriate format for Asterisk — typically 8kHz mono WAV or GSM).

### Compliance for Political Voicemail Drops

The legal landscape for voicemail drop is still evolving. Key points:

- **Landlines:** Pre-recorded political messages to landlines are exempt from TCPA consent requirements. VM drops to landlines are broadly considered permissible for political messaging.
- **Cell phones:** This is the gray area. If the VM drop involves dialing the cell phone number (even if the call goes directly to voicemail), it likely constitutes a call to a wireless number under the TCPA. Most campaign compliance attorneys recommend treating VM drops to cell phones the same as robocalls to cell phones — obtain prior express consent.
- **"Ringless" voicemail services** that bypass the carrier network and deposit messages directly into voicemail systems are a separate technology from VICIdial's AMD-based VM drop. The FCC has not issued a definitive ruling on whether RVM constitutes a "call" under the TCPA. Campaign attorneys are split on this. Use at your own risk.

**Practical recommendation:** Use VICIdial's VM drop feature for landline voters from your voter file. For cell phone voters, stick to live agent calls or obtain consent before sending pre-recorded messages.

---

## Survey Campaigns and Data Collection

Political survey calls (polling, issue surveys, candidate favorability testing) are another high-value VICIdial use case. The system's custom fields, scripting, and disposition capabilities can replace expensive polling firm services for basic voter surveys.

### Survey Campaign Configuration

```
Campaign Type: Outbound
Dial Method: RATIO at 1.5
AMD: ON (survey campaigns need efficiency)
Script: Survey script with embedded response fields
Custom Fields: survey_q1, survey_q2, survey_q3, etc.
```

**Script design for political surveys:**

VICIdial's scripting engine supports embedded form fields within the script text. Design your survey script so the agent (or volunteer) reads the question and clicks a response option that's recorded directly into the lead's custom fields:

```
"On a scale of 1 to 5, how would you rate the job performance of Governor [Name]?"
[1 - Poor] [2 - Fair] [3 - Neutral] [4 - Good] [5 - Excellent] [R - Refused]
```

Each response option writes to a custom field on the lead record. After the campaign completes, export the data from VICIdial's custom fields for analysis.

**Survey-specific considerations:**

- **Randomize question order** if your survey methodology requires it. This can be handled via script branching in VICIdial or by creating multiple script versions assigned to different sub-campaigns.
- **Capture demographics** (age, gender, party ID) at the beginning or end of the survey for cross-tabulation.
- **Include a voter ID question** to double-purpose the call: "If the election were held today, would you vote for [Candidate A] or [Candidate B]?" Now your survey call also produces voter ID data.
- **Weight your sample.** VICIdial will connect with whoever answers. If your list skews toward older voters (who answer phones at higher rates), your survey data will skew the same way. Account for this in analysis or use list segmentation to oversample underrepresented demographics.

---

## Campaign Segmentation by Voter File Data

The voter file is the most powerful targeting tool in political dialing. Unlike commercial lead lists, voter files contain granular data that enables precise segmentation. Your VICIdial list structure should fully exploit this data.

### Key Voter File Fields to Load Into VICIdial

| Field | VICIdial Location | Use |
|---|---|---|
| Voter name | First/Last Name (standard) | Personalization |
| Phone number | Phone Number (standard) | Primary dial target |
| Cell phone | Alt Phone (standard) | Secondary dial target |
| Address | Address fields (standard) | Geo-targeting |
| Party registration | Custom field: party_reg | Targeting and scripting |
| Voting history | Custom field: vote_history | Turnout modeling |
| Precinct/Ward | Custom field: precinct | Geographic segmentation |
| Congressional district | Custom field: cong_dist | Campaign segmentation |
| State leg district | Custom field: state_dist | Down-ballot targeting |
| Age/DOB | Custom field: voter_age | Demographic targeting |
| Gender | Custom field: voter_gender | Demographic targeting |
| Support score | Custom field: support_score | Prioritization |
| Last contact date | Custom field: last_contact | Recency tracking |
| Last contact result | Custom field: last_result | Contact history |

### Segmentation Strategies

**By party registration:**

- **Own-party voters:** Focus on GOTV and volunteer recruitment. These voters support your candidate but need to be reminded/motivated to vote.
- **Opposite-party voters:** Generally skip. Calling opposition voters for persuasion is expensive and rarely effective except in specific scenarios (crossover appeal candidates, ballot measures).
- **Independents/unaffiliated:** Primary persuasion targets. These voters are reachable and worth the investment in longer, more careful calls.

Create separate VICIdial lists for each party segment and assign different campaigns (with different scripts and dial methods) to each.

**By voting history:**

- **Frequent voters (voted in 3+ of last 4 elections):** Likely to vote regardless. Low-priority for GOTV, high-priority for voter ID (you want to know who they support).
- **Occasional voters (voted in 1-2 of last 4 elections):** The GOTV sweet spot. These voters are persuadable AND need motivation to show up. Highest-ROI contacts for turnout operations.
- **Registered but rarely/never vote:** Hard to mobilize. Only worth calling if the campaign has budget surplus and needs to expand the electorate.

**By precinct and district:**

For a congressional campaign, you might have 200-400 precincts. Not all precincts deserve equal calling resources. Prioritize:

1. **Competitive precincts** where the margin in the last comparable election was under 10 points
2. **High-population precincts** where raw vote totals matter
3. **Precincts with high unaffiliated registration** (persuasion opportunity)
4. **Precincts with low turnout relative to registration** (GOTV opportunity)

Use VICIdial's list structure to create separate lists per precinct group (top-priority precincts, medium-priority, low-priority) and use hopper weighting to dial high-priority precincts first.

---

## GOTV Operations: The Final 72 Hours

GOTV is the culmination of everything the campaign's calling operation has built. In the final 72 hours before Election Day, your VICIdial deployment shifts into maximum-output mode. Everything else stops — voter ID, persuasion, fundraising — and every agent and volunteer focuses on turning out identified supporters.

### GOTV List Preparation

Your GOTV list is built from the voter ID and persuasion data collected over the preceding weeks/months:

- **Include:** Voters identified as supporters (support score 4-5) or strong leaners (score 3 with positive persuasion contact)
- **Exclude:** Voters identified as opponents, voters who've already voted (if early voting data is available), voters who've requested no further contact
- **Prioritize:** Occasional voters who support the candidate but might not vote without a nudge

**Early voting integration:** Many states publish early/absentee voting data daily during the early voting period. If your state provides this data, cross-reference it against your VICIdial GOTV list daily and remove voters who've already cast a ballot. There's nothing that annoys a supporter more than getting a "remember to vote" call when they voted last week. Your data team should be exporting early voter lists and updating VICIdial lead statuses (set to DNC or a custom "VOTED" disposition) every morning during the early voting period.

### GOTV Call Schedule

| Day | Priority | Agent Deployment |
|---|---|---|
| E-7 to E-4 | Early vote push (states with early voting) | 50% of capacity |
| E-3 (Saturday before) | Heavy GOTV dialing | 100% capacity |
| E-2 (Sunday before) | Continue GOTV, absentee ballot follow-up | 80% capacity |
| E-1 (Monday before) | Final full-day push | 100% capacity |
| Election Day (E-0) | All-day blitz until polls close | Every available agent/volunteer |

**Election Day dialing is the most intense phone banking of the cycle.** Many campaigns run three shifts:

1. **Morning (8 AM - 12 PM):** Remind supporters to vote before/during lunch
2. **Afternoon (12 PM - 5 PM):** Target voters who haven't been reached yet
3. **Evening (5 PM - poll close):** Final push to every un-contacted supporter

VICIdial campaign settings should be adjusted for Election Day:

```
Dial Method: ADAPT_TAPERED
Adaptive Maximum Dial Level: 4.0
Dial Timeout: 20 seconds (faster cycling)
AMD: ON with VM drop ("Polls close at [time]. Don't forget to vote!")
Call Time: 8:00 AM to poll close (check state law)
```

### GOTV Script Structure

GOTV scripts are the shortest in the political calling toolkit. Under 30 seconds:

"Hi, this is [Name] calling from [Campaign/Committee]. We're reaching out to remind you that Election Day is [today/tomorrow]. Your polling location is [Location]. Polls are open from [time] to [time]. Can we count on you to vote [today/tomorrow]?"

Dispositions for GOTV:

| Code | Meaning |
|---|---|
| WILL_VOTE | Confirmed will vote / has voted |
| MAYBE | Uncertain, might need follow-up |
| ALREADY | Already voted (early/absentee) |
| NO | Will not vote |
| NA | No answer |
| AM | Answering machine (VM dropped) |
| WN | Wrong number |
| DNC | Do not call |

---

## DID Management for Political Campaigns

Political campaigns burn through DIDs differently than commercial operations. The volume is intense but short-lived — you might push 100,000 calls through a DID pool in 6 weeks and then never use those numbers again.

### DID Strategy

**Local area codes matter — but differently.** In commercial calling, local presence drives answer rates because consumers distrust out-of-area calls. In political calling, local presence matters for the same reason, but voters also respond to recognizable campaign phone numbers. Some campaigns publicize a single callback number (on mailers, yard signs, ads) and want that number to appear on caller ID for recognition.

**Recommended approach:**

- **Voter ID and persuasion:** Use AREACODE-based CID groups for local presence, just like commercial campaigns. Voters are more likely to answer a local number. See our [DID management guide](/blog/vicidial-did-management/) for setup details.
- **GOTV:** Use the campaign's publicized number as the primary CID if voters will recognize it from mailers/ads. If the campaign hasn't built number recognition, use local presence.
- **Robocall/VM drop:** Use a dedicated DID that routes callbacks to a campaign IVR: "Thank you for calling [Campaign]. To learn more, press 1. To be removed from our call list, press 2."

**DID volume for political:** Plan for 5-8 DIDs per target area code, capped at 100-150 calls per DID per day. Political calling generates callbacks and complaints just like commercial calling, and carriers flag high-volume numbers regardless of whether the calls are political. Monitor spam reputation using the same tools commercial operations use — Numeracle, Free Caller Registry, or manual testing with carrier apps.

**Post-election:** After Election Day, retire all campaign DIDs. Don't reuse them for future campaigns or repurpose them for commercial use. They've been flagged by voters and carriers during the campaign and their reputation is burned. The cost of new DIDs for the next cycle is negligible compared to the contact rate penalty of using flagged numbers.

---

## VICIdial Integration With Political Tech Stack

Political campaigns use specialized technology platforms that VICIdial needs to integrate with.

### Voter Database Platforms (VAN/VoteBuilder, PDI, L2)

**NGP VAN / VoteBuilder** is the dominant voter data platform for Democratic campaigns. **Data Trust / i360** serves Republican campaigns. Both maintain voter files, contact history, and support scores.

VICIdial integration typically flows in two directions:

1. **VAN → VICIdial:** Export voter lists from VAN with phone numbers, custom fields (party, vote history, precinct, support score), and load into VICIdial as campaign lists. Most exports are CSV-based and can be loaded via VICIdial's list upload or API.

2. **VICIdial → VAN:** After a phone banking session, export VICIdial dispositions and custom field data (survey responses, support scores) and import back into VAN to update voter records. This is typically a nightly batch process: export VICIdial call results via the Non-Agent API or direct database query, map dispositions to VAN canvass results, and upload via VAN's API or manual import.

**Automation tip:** Write a simple script (Python or PHP) that runs nightly, queries VICIdial for the day's call results, maps dispositions to VAN result codes, and pushes updates. This eliminates manual data entry and ensures voter records stay current across both systems.

### Texting Platforms (ThruText, Hustle, Strive)

Many campaigns pair phone banking with peer-to-peer texting. VICIdial handles the voice calls; a texting platform handles SMS outreach. Coordinate the two to avoid over-contacting voters:

- Share disposition data between platforms. If a voter was identified as a supporter via phone, the texting platform should know.
- Stagger outreach. Don't call and text the same voter on the same day unless it's GOTV weekend.
- Use VICIdial's DNC list to suppress voters who opt out via text and vice versa.

### Digital Advertising Integration

Advanced campaigns use phone banking data to build digital advertising audiences. Export VICIdial voter ID results (supporters and undecideds) and upload to Facebook/Meta or Google Ads as custom audiences for targeted digital persuasion. This creates a multi-channel voter contact strategy that starts with phone identification and continues with digital reinforcement.

> **Running a political operation this cycle?** Whether it's a state legislative race or a national campaign, we'll configure your VICIdial deployment for maximum voter contact efficiency. [Request a free audit](/free-audit/) — we'll review your infrastructure, compliance posture, and campaign configuration before you start dialing.

---

## Budgeting a Political VICIdial Deployment

Campaign managers want to know what this costs. Here's a realistic budget for a congressional-level campaign running VICIdial over a 10-week general election period.

### Infrastructure Costs

| Item | Cost | Notes |
|---|---|---|
| VICIdial server (cloud) | $200-$500/month | 2-4 vCPU, 8-16GB RAM, dedicated instance |
| SIP trunking | $0.008-$0.015/minute | 50,000-200,000 minutes over the campaign |
| DIDs (local numbers) | $1-$3/month per DID | 40-80 DIDs across target area codes |
| STIR/SHAKEN attestation | Included with quality SIP providers | Verify A-level attestation |
| Headsets (USB) | $15-$25 each, one-time | 30-50 headsets for phone bank stations |
| Laptops (if needed) | $200-$400 each used | Or use volunteer personal devices |

**Total infrastructure for 10-week campaign with 30 phone bank stations:** $2,500-$8,000 depending on call volume.

**Comparison to SaaS alternatives:** A comparable deployment on Five9, Convoso, or LiveVox would run $4,000-$12,000 per month (30 seats at $130-$400/seat/month), totaling $10,000-$30,000 for the same 10-week period. VICIdial saves the campaign $7,000-$22,000 — money that goes to voter contact instead of dialer vendor profit margins.

### Staffing Costs (If Using Paid Agents)

| Role | Hours/Week | Rate | 10-Week Cost |
|---|---|---|---|
| Phone bank manager | 30-40 | $25-$35/hr | $7,500-$14,000 |
| Paid callers (10) | 20-30 each | $15-$20/hr | $30,000-$60,000 |
| IT/VICIdial admin | 5-10 | $50-$100/hr (contract) | $2,500-$10,000 |

Most campaigns reduce paid agent costs by using volunteers for the majority of calling hours and reserving paid agents for specialized tasks (fundraising, persuasion to high-value voter segments).

---

## Common Mistakes in Political VICIdial Deployments

**1. Not separating cell phones from landlines.** The compliance rules are different for each. Load cell phone numbers and landline numbers into separate VICIdial lists so you can apply different dialing rules and ensure you're not running robocalls or VM drops to cell phones without consent.

**2. Using aggressive predictive dialing with volunteers.** Volunteers are not call center agents. Setting an adaptive dial level of 3.0+ with volunteers results in abandoned calls, confused volunteers, and people who don't come back for the next phone bank. Keep volunteer campaigns at 1:1 or low-ratio dialing.

**3. Forgetting to scrub against internal opt-outs.** The federal DNC exemption for political calls does not override individual opt-out requests. If a voter tells your campaign to stop calling, add them to your internal DNC list in VICIdial. Calling them again isn't just legally risky — it's politically damaging. One angry voter complaining to local media about harassment calls can generate a news cycle that costs the campaign more than 10,000 successful contacts.

**4. Not syncing VICIdial data back to your voter file.** If your phone banking results aren't flowing back into VAN/VoteBuilder or your campaign's voter database, you're wasting half the value of the calls. The point of voter ID is to build a targeting model. If that data lives in VICIdial but not in your voter file, your canvassers, texters, and mail program are flying blind.

**5. Ignoring early voting data.** Calling voters who already voted is a waste of money and annoying to the voter. During the early voting period, update your GOTV lists daily by removing voters who've already cast ballots.

**6. Setting up VICIdial the week before Election Day.** A VICIdial deployment for political calling needs 1-2 weeks of setup and testing before go-live. Server provisioning, SIP trunking, DID acquisition, list loading, script configuration, volunteer training — none of this should happen the week you need to start dialing. Plan for your VICIdial infrastructure to be operational at least 2 weeks before your first phone bank.

**7. Not configuring callback routing on outbound DIDs.** Voters call back political numbers. If your outbound DIDs ring to dead air or a disconnected message, voters report them as spam, carriers flag them faster, and you've lost a potential supporter contact. Configure VICIdial inbound routes on every outbound DID that at minimum play a campaign identification message and offer a callback or volunteer signup option.

---

## Frequently Asked Questions

### Do political campaigns need TCPA consent to call voters?

For live calls made by actual humans — no, under federal law. Political campaigns are exempt from the TCPA's prior consent requirements for live calls to both landlines and cell phones. However, pre-recorded messages (robocalls) and autodialed calls to cell phones do require prior express consent even for political purposes. Additionally, state laws may impose additional requirements beyond the federal exemption. The safest approach is to use live agents for all cell phone contacts and reserve robocalls/VM drops for landline-only segments of your voter file. See our [TCPA compliance guide](/blog/vicidial-tcpa-compliance/) for the full breakdown.

### Can I use VICIdial's predictive dialer for political calls without consent?

For live-agent calls to landlines, yes — the political exemption covers autodialer-assisted live calls to wireline numbers. For cell phones, the answer is more nuanced. The TCPA restricts autodialed calls to cell phones regardless of content, and VICIdial's predictive dialing mode meets most definitions of an autodialer. Most campaign attorneys recommend one of two approaches: (1) segregate cell phones into separate lists and use manual/preview dialing for cell contacts, or (2) obtain consent through voter registration data, petition signatures, or campaign website opt-ins before predictive-dialing cell numbers. The FCC has not provided a blanket exemption for predictive-dialed live calls to cell phones for political purposes.

### How many voter contacts per hour can VICIdial handle?

It depends on the operation type. A single volunteer in preview mode completes 15-25 voter contacts per hour (mostly short voter ID calls). A paid agent on predictive dialing completes 25-45 contacts per hour. For GOTV operations with very short scripts and aggressive dialing, paid agents can push 40-60 contacts per hour. A 30-seat phone bank running voter ID with volunteers will produce 450-750 contacts in a 3-hour evening session. A 30-seat paid operation on predictive will produce 750-1,350 contacts in the same window. Over a 6-week voter ID phase with 4 evenings per week of phone banking, a 30-seat volunteer operation identifies roughly 10,000-18,000 voters.

### What's the best VICIdial dial method for volunteer phone banks?

RATIO at 1.0 (effectively one-to-one dialing) or MANUAL with preview. Volunteers need time to read the voter's information before the call connects, and they need a natural pace between calls. Predictive dialing overwhelms volunteers — they get calls before they're ready, calls drop because volunteers haven't finished dispositioning, and the experience drives people away. The slight efficiency loss of 1:1 dialing is more than offset by volunteer retention. If a bad experience causes 5 of your 30 volunteers to leave early, you've lost more contacts than the efficiency gain from predictive dialing would have produced.

### How do I load a voter file into VICIdial?

Export your voter file from VAN/VoteBuilder, L2, or your state's voter registration database as a CSV. Map the standard fields (name, phone, address) to VICIdial's standard lead fields, and create VICIdial custom fields for political-specific data (party registration, vote history, precinct, support score). Use VICIdial's Admin list upload (for files under 100,000 records) or the Non-Agent API `add_lead` function (for larger files or automated loading). Break large voter files into separate lists by precinct group, party registration, or priority tier — this gives you granular control over which segments get dialed first. See our [list management guide](/blog/vicidial-list-management/) for the full process.

### Should I use VICIdial or a political-specific dialer like ThruTalk or LiveVox?

VICIdial is the right choice if your campaign needs cost efficiency, configuration control, and scale. ThruTalk and similar political-specific platforms charge $0.05-$0.15 per call attempt or $200-$500 per seat per month — costs that add up fast across a campaign cycle. VICIdial's infrastructure cost is a fraction of that. The trade-off is setup complexity: VICIdial requires technical expertise to deploy and configure, while political SaaS platforms offer turnkey onboarding. If your campaign has access to a VICIdial administrator (in-house or contracted), VICIdial will deliver equal or better performance at 30-60% of the cost. If nobody on the campaign can manage VICIdial, a managed platform may be worth the premium.

### How do I handle voters who ask to be removed from the call list?

Immediately. Add the voter to your campaign's internal [DNC](/glossary/dnc/) list in VICIdial — train every volunteer and agent on the disposition code for removal requests (typically DNC). While political campaigns are exempt from the National Do Not Call Registry, you are legally and ethically required to honor individual removal requests. VICIdial's internal DNC list function handles this: once a number is on the campaign's DNC list, the system will not load that number into the hopper for any campaign that references that DNC list. Sync removals back to your voter file so other campaign contact methods (text, mail, canvass) also respect the opt-out.

### Can I use VICIdial for political fundraising calls?

Yes, and many campaigns do. Fundraising calls are typically handled by paid agents (not volunteers) because they require script discipline, objection handling, and the ability to process credit card donations over the phone. Configure a separate VICIdial campaign for fundraising with preview dialing (the agent needs to see the donor's history — previous donations, event attendance, relationship to the campaign — before the call connects). Integrate with your campaign's fundraising platform (ActBlue, WinRed, NGP) via Dispo Call URL to record pledges and donations in real time. Note that fundraising calls are still subject to the political exemptions described above, but if a third-party vendor is making the calls on behalf of the campaign, additional telemarketing registration requirements may apply at the state level.

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-political-campaigns).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
