# VICIdial Training Guide: From Beginner to Expert

**Last updated: March 2026 | Reading time: ~24 minutes**

You got VICIdial installed. Maybe you followed [our setup guide](/blog/vicidial-setup-guide/) and everything went smoothly. Maybe you inherited a running system from the last guy who left without documenting anything. Either way, you're staring at a login page and a hundred menu items, and nobody handed you a training manual.

That's because there isn't one. Not really.

The VICIdial Manager Manual on Amazon covers reference material. The forum has 13,400+ threads with answers scattered across a decade of posts. YouTube has a handful of walkthroughs from 2019 that reference CentOS 7 and interfaces that have since changed. None of it adds up to a structured training path that takes a new admin or agent from "what does this button do" to "I can run this operation with my eyes closed."

This guide is that training path. We're covering admin panel mastery, [campaign](/glossary/campaign/) configuration, [agent screen](/glossary/agent-screen/) training, web form setup, [reporting](/blog/vicidial-reporting-monitoring/), and the community resources that are actually worth your time in 2026. Whether you're training yourself, onboarding a new manager, or building a curriculum for 50 agents, this is the document you send them first.

**Or — if you'd rather skip the learning curve entirely — ViciStack includes full admin and agent training as part of every deployment.** We've onboarded 10,000+ agents across 200+ call centers. We know exactly which concepts trip people up and which settings cause the most damage when misconfigured. But if you want to build your own expertise, keep reading. That's what this guide is for.

---

## The Two Training Tracks: Admin vs. Agent

VICIdial training isn't one thing. It's two completely different skill sets that happen to use the same software.

**Admin training** covers the [admin panel](/glossary/admin-panel/) — the back-end interface where campaigns are built, leads are loaded, carriers are configured, reports are pulled, and every operational decision gets implemented. This is the domain of managers, supervisors, and IT staff. Getting this wrong means agents can't work, calls don't connect, and reports lie to you.

**Agent training** covers the [agent screen](/glossary/agent-screen/) — the front-end interface where calls are taken, dispositions are set, callbacks are scheduled, and customer data is entered. This is simpler in scope but just as critical. A poorly trained agent on VICIdial will misdispose leads, forget to set callbacks, accidentally transfer calls to nowhere, and tank your campaign metrics without realizing it.

Most organizations make the mistake of training everyone on everything. Don't. Your agents don't need to know how dial ratios work. Your admins don't need to practice cold-call rebuttals. Train each role on what they actually touch, and go deep instead of wide.

Here's the training path for each, in the order you should learn it.

---

## Admin Training Path: The Complete Curriculum

This is the big one. VICIdial's admin panel has hundreds of settings spread across dozens of screens. Nobody needs to memorize all of them — but you do need to understand the architecture, know where to find things, and recognize the 20% of settings that cause 80% of the problems.

### Phase 1: Admin Panel Navigation and Mental Model

Before you change a single setting, spend 30 minutes just clicking through the [admin panel](/glossary/admin-panel/). Here's the mental model that makes VICIdial's interface click:

**The admin panel is organized around objects, not workflows.** There's no "set up a new campaign" wizard. Instead, you create individual objects — users, phones, campaigns, lists, carriers, inbound groups, DIDs — and then link them together. Understanding this architecture is the single most important concept for a new VICIdial admin.

The main navigation breaks down like this:

- **Users** — Agent accounts, admin accounts, permissions, user groups
- **Campaigns** — Outbound campaign configurations, dial methods, statuses, [pause codes](/glossary/pause-code/), hopper settings
- **Lists** — Lead data, list loading, duplicate checking, DNC management
- **In-Groups** — Inbound call routing (VICIdial calls these "in-groups," not "queues")
- **DIDs/TFNs** — Inbound phone numbers and their routing rules
- **Carriers** — SIP trunk configurations
- **Phones** — SIP phone registrations for agents
- **Scripts** — Call scripts displayed to agents
- **Filters** — Lead filtering rules for campaigns
- **Call Times** — Schedules that control when campaigns can dial
- **Remote Agents** — External agents using non-VICIdial phones
- **Reports** — Real-time dashboards and historical analytics

New admins: bookmark the Admin → System Settings page. This is where global defaults live — server timezone, recording format, password requirements, and dozens of other settings that silently affect everything else. If something is behaving strangely and you can't find the setting at the campaign level, it's probably in System Settings.

### Phase 2: User and Phone Management

This is where you start building. Every agent needs two things: a **user account** and a **phone**.

**Creating users:** Admin → Users → Add A New User. The fields that matter most:

- **User ID** — Numeric only. Pick a convention (e.g., 1001–1999 for agents, 6001–6999 for admins) and stick with it.
- **User Level** — 1 for agents, 7–8 for supervisors, 9 for full admin. Be stingy with level 9. An undertrained admin with level 9 access can delete campaigns, wipe lists, and change carrier configs. We've seen it happen.
- **User Group** — Groups control permissions and reporting segments. Create groups like SALES_AGENTS, SUPPORT_AGENTS, SUPERVISORS. You'll use these constantly in reports.
- **Phone Login/Pass** — This is separate from the web login. Agents use this to log into a specific phone at the start of their shift.

**Creating phones:** Admin → Phones → Add A New Phone. One phone per agent workstation.

- **Extension** — Unique per server (e.g., 1001, 1002, etc.)
- **Server IP** — The telephony server this phone registers to
- **Dialplan Number** — Typically same as extension
- **Registration Password** — Change this from the default. Always.
- **As Webphone** — Set to Y if you're using WebRTC/ViciPhone (and you should be in 2026)

**The most common new-admin mistake:** creating users but forgetting to create matching phones, then wondering why agents can't log in. VICIdial requires both objects — a user account AND a phone registration — before an agent can take calls.

### Phase 3: Campaign Configuration

This is the heart of VICIdial operations. A [campaign](/glossary/campaign/) is the container that holds everything together — dial method, lead lists, agent assignments, recording rules, [dispositions](/glossary/disposition/), call routing, and about 150 other settings.

**Creating your first campaign:** Admin → Campaigns → Add A New Campaign. The name and ID are permanent, so choose something meaningful (e.g., SOLAR_OB_2026, not TEST123).

The settings that matter most on day one:

**Dial Method** — This is the biggest decision you'll make per campaign. The options:

- **MANUAL** — Agent clicks to dial each call. No automation. Use this for compliance-heavy operations or initial testing.
- **RATIO** — Fixed number of simultaneous calls per agent. Simple and predictable. Set [Auto Dial Level](/settings/auto-dial-level/) to 1.5–2.0 for starters.
- **ADAPT_HARD_LIMIT** — Predictive dialing with an automatic ceiling on drop rate. The system adjusts dial ratio in real-time based on connect rates and agent availability. This is what most outbound campaigns should use.
- **ADAPT_TAPERED** — More aggressive early, pulling back as the day progresses. For experienced operations only.

For the complete breakdown of every dial method setting, intensity level, and the math behind adaptive algorithms, see our [predictive dialer settings guide](/blog/vicidial-predictive-dialer-settings/).

**Statuses/Dispositions** — VICIdial ships with default [dispositions](/glossary/disposition/) (A = Answering Machine, B = Busy, N = No Answer, etc.), but you'll want campaign-specific statuses. Admin → Campaigns → Statuses. Create statuses that match your actual workflow: SALE, CALLBACK, NOT_INTERESTED, DNC, WRONG_NUMBER, etc.

Every status has three critical flags:

- **Human Answered** — Was this a real conversation? Affects reporting accuracy.
- **Callable** — Can this lead be dialed again? Set DNC and SALE to non-callable.
- **Scheduled Callback** — Does this status trigger callback scheduling?

Getting these flags wrong doesn't break anything immediately. It breaks your reports silently. You'll run analytics a month later and wonder why your "contact rate" is 85% when you know it's closer to 40%. It's because every AMD detection was flagged as human-answered.

**Pause Codes** — Admin → Campaigns → Pause Codes, or enable them globally via the [Agent Pause Codes Active](/settings/agent-pause-codes-active/) setting. [Pause codes](/glossary/pause-code/) track why agents aren't taking calls — BREAK, LUNCH, TRAINING, BATHROOM, TECH_ISSUE. Without them, your reports show agents as "paused" for 2 hours with no explanation.

Make pause codes required (Agent Pause Codes Active = FORCE) from day one. If you make them optional, agents won't use them, and you'll have zero visibility into non-talk time. This is one of the most common gaps we see in self-managed VICIdial deployments.

**Recording** — Set [Campaign Recording](/settings/campaign-recording/) to ALLCALLS unless you have a specific reason not to. You can always delete recordings later; you can never recover calls you didn't record. Verify recordings are actually saving by checking Admin → Recordings after a few test calls.

> **ViciStack Deploys Campaigns Pre-Configured for Your Vertical.**
> Solar, insurance, home services, debt — we've tuned campaigns for every major vertical. Dial methods, dispositions, AMD settings, all optimized from call one. [Get Your Free Audit →](/free-audit/)

### Phase 4: Lead List Management

You can build the perfect campaign, but if your lead data is garbage, nothing else matters.

**Creating lists:** Admin → Lists → Add A New List. Assign it to a campaign. Each list has an ID (numeric, up to 14 digits) and belongs to exactly one campaign.

**Loading leads:** Admin → Lists → Load New Leads. VICIdial accepts CSV/TSV files with a specific field order. The minimum viable lead has: phone_number, first_name, last_name, state. But you should also map: city, zip, address, email, source_id, and any custom fields your scripts or web forms need.

**The lead loader's quirks that will burn you:**

- Fields must be in the exact order VICIdial expects, or you need to use custom field mapping. There's no automatic header detection.
- Phone numbers must be 10 digits (for US). No dashes, no spaces, no +1 prefix.
- The duplicate check option only checks within the list being loaded by default. To check against all lists, you need to set the "Check for Duplicates" option to DUPCAMP or DUPLIST depending on your needs.
- Uploading a file with encoding issues (common when exporting from Excel) will silently corrupt records. Save as UTF-8 CSV.

**List management over time:** Leads cycle through statuses as they're dialed. The hopper (VICIdial's pre-fetch queue) pulls callable leads based on your campaign's dial statuses, call count limits, and filtering rules. When a list runs dry, either recycle leads by resetting statuses (Admin → Lists → List ID → Reset Lead Status) or load fresh data.

For the complete deep-dive on lead loading, deduplication, list recycling, DNC scrubbing, and the custom field system, see our [list management guide](/blog/vicidial-list-management/).

### Phase 5: Inbound Call Routing

Outbound-only operations can skip this. Everyone else: VICIdial's inbound routing uses three linked objects.

**DIDs** — Admin → Inbound → DIDs. This is where an inbound phone number enters the system. The DID points to either an in-group, a campaign, an agent, or a custom extension.

**In-Groups** — Admin → Inbound → In-Groups. VICIdial's term for call queues. In-groups have their own hold music, queue prioritization, routing strategies, agent rankings, and overflow rules.

**Call Menus (IVRs)** — Admin → Inbound → Call Menus. Build press-1-for-sales, press-2-for-support IVR trees here. Each option routes to an in-group, another menu, a voicemail, or a hangup.

The routing chain is: **Carrier delivers call → Asterisk matches DID → DID routes to Call Menu or In-Group → In-Group delivers call to available agent.**

**The most common inbound mistake:** forgetting to add agents to the in-group. Creating a DID and in-group isn't enough — agents must be configured to receive calls from that specific in-group, either through campaign settings (Allowed Inbound Groups) or the closer campaigns feature.

### Phase 6: Reports and Real-Time Monitoring

If you're not checking reports, you're flying blind. VICIdial's reporting is powerful but intimidating — there are 40+ report types, and half of them return confusing results if you don't understand the underlying data model.

The reports every admin should master first:

**Real-Time Report** — Reports → Real-Time Main Report. This is your command center. It shows every logged-in agent, their status (INCALL, PAUSED, READY, DEAD), current call duration, campaign stats, and queue depth. Run this on a dedicated monitor during business hours. Always.

**Agent Performance Detail** — Reports → Agent Performance Detail. Shows per-agent breakdowns: talk time, wait time, pause time, calls handled, [dispositions](/glossary/disposition/). This is how you identify your top performers and your training cases.

**Outbound Calling Report** — Reports → Outbound Calling. Campaign-level metrics: calls dialed, contacts made, drops, abandons, average talk time. Use this to evaluate campaign health, not individual agents.

**Export Calls Report** — Reports → Export Calls Report. Raw call data export. Every call, every field, every timestamp. This is what you hand to your data team or import into your BI tool.

For the full breakdown of every report, what each metric actually means, and the real-time monitoring setup that separates amateur operations from professional ones, see our [reporting and monitoring guide](/blog/vicidial-reporting-monitoring/).

---

## Agent Training Path: What Your Agents Actually Need to Know

Agent training is a different animal. Your agents don't need to understand dial ratios or SIP trunk configuration. They need to know how to log in, handle calls, set the right [disposition](/glossary/disposition/), schedule callbacks, use web forms, and not break anything.

Here's the curriculum, ordered by importance.

### Lesson 1: Logging In (It's Not Obvious)

VICIdial's agent login is a two-step process, and it trips up new agents constantly.

**Step 1: Web login.** Navigate to `http://your-server/agc/vicidial.php`. Enter their VICIdial user ID and password. Select their campaign from the dropdown.

**Step 2: Phone login.** After the web login, a second prompt asks for Phone Login and Phone Password. This is the SIP phone credential, not their user password. If you're using WebRTC/ViciPhone, the phone launches automatically in the browser.

**The most common first-day confusion:** agents entering their user password in the phone password field. These are two different credentials. Label them clearly in your onboarding docs: "Web Login = your employee ID, Phone Login = your workstation number."

Once logged in, the [agent screen](/glossary/agent-screen/) loads. Before any calls come in, walk agents through the layout:

- **Top bar** — Campaign name, agent ID, status indicator, session timer
- **Main panel** — Customer information (name, phone, address, custom fields), call script, web form buttons
- **Bottom controls** — Disposition buttons, hangup, transfer, park, conference, manual dial, callback scheduling
- **Status area** — Current call status, wait timer, pause code selector

For the full guide on [customizing the agent screen](/blog/vicidial-agent-screen-customization/) layout, hiding unnecessary fields, and adding custom buttons, see our dedicated article.

### Lesson 2: Call Handling Workflow

In predictive/adaptive campaigns, calls arrive automatically. The agent's screen flashes, customer info populates, and the agent is live on a call. There's no "answer" button — VICIdial connects the call and the agent simultaneously. New agents need to understand this: **the customer is already on the line the moment their screen changes.**

Train this workflow until it's muscle memory:

1. **Screen pops, call connects** — Greet the customer immediately. Don't wait for the screen to fully load.
2. **Review customer info** — Name, location, source. This data is on-screen. Use it.
3. **Follow the script** — If a script is configured, it appears in the script panel. Read it, but don't sound like you're reading it.
4. **Use the web form** — Click the Web Form button to open the customer-facing form or CRM integration in a new tab. Enter data as the conversation progresses, not after.
5. **Handle the outcome:**
   - **Sale/appointment/transfer** — Follow your operation's procedure. Use the Transfer/Conference button for warm transfers.
   - **Not interested** — Thank them, hang up, disposition as NOT_INT or your equivalent.
   - **Callback requested** — Click the Callback button, set the date/time, then disposition as CALLBK.
   - **Voicemail/no answer** — The system handles most of these automatically in predictive mode. If you reach a voicemail manually, disposition as A (Answering Machine) or your equivalent.
6. **Disposition the call** — This is mandatory. Select the correct [disposition](/glossary/disposition/) and click it. The system will not send the next call until the current one is dispositioned.

**Drill this into every agent: disposition accuracy is not optional.** A wrong disposition doesn't just mess up one record — it corrupts your campaign data, throws off your dial ratios, and can cause DNC violations if a "do not call" request gets dispositioned as "no answer."

### Lesson 3: Pause Codes and Status Management

When agents aren't taking calls, they need to pause. And when they pause, they need to select a [pause code](/glossary/pause-code/).

Train agents on the available pause codes and when to use each one:

- **BREAK** — Scheduled short break
- **LUNCH** — Meal break
- **TRAINING** — In a training session
- **BATHROOM** — Self-explanatory
- **TECH_ISSUE** — Computer/headset/software problem
- **MEETING** — Team meeting or one-on-one

**The cardinal sin:** staying in READY status while not actually ready to take calls. If an agent walks away from their desk in READY status, the dialer sends them a call. Nobody answers. The customer hears silence. The call drops. Your drop rate goes up. Your campaign's predictive algorithm gets thrown off. One absent agent in READY status can degrade the experience for every other agent on the campaign.

Make this a fireable offense from day one. Not kidding.

### Lesson 4: Transfers and Conferences

VICIdial's transfer system is powerful but unintuitive. There are four types:

**Blind Transfer** — Sends the call to another extension, in-group, or external number. The agent's involvement ends immediately. Use for transferring to departments, not to specific people (because you don't know if they're available).

**Warm/Consultative Transfer** — The agent dials the transfer target, speaks to them ("I have John on the line, he's interested in the premium plan"), then connects all three parties. The agent drops off. This is the professional way to transfer.

**Three-Way Conference** — Agent, customer, and third party all on the line simultaneously. Used for verification calls, manager assistance, or three-way closes.

**Park** — Puts the customer on hold while the agent does something else. The call sits in a parking lot until retrieved. Use sparingly — customers hate hold music.

Train the warm transfer process specifically. The button sequence is: **Transfer/Conference → dial the target → wait for answer → announce the call → click Transfer → hang up.** New agents will try to click Hangup before completing the transfer, which disconnects the customer. Practice this on test calls before going live.

### Lesson 5: Callbacks

Callbacks are one of VICIdial's strongest features, and one of the most underused.

When a prospect says "call me back Thursday at 2 PM," the agent clicks the Callback button, sets the date and time, and dispositions the call as CALLBK (or your equivalent callback status). VICIdial's hopper will automatically queue that lead at the specified time.

Two types of callbacks:

- **USERONLY** — Only the agent who set it gets the callback. The system delivers it specifically to them when the time arrives. Good for relationship-based sales.
- **ANYONE** — Any available agent on the campaign gets the callback. Better for high-volume operations where agent-specific callbacks cause scheduling problems.

**Training point:** Always set callbacks in the customer's timezone, not yours. VICIdial can handle timezone-aware dialing, but only if the lead data includes the correct state/timezone information and your Call Time settings are properly configured.

### Lesson 6: Web Forms and CRM Integration

Web forms are how VICIdial connects to external systems — your CRM, order entry system, appointment scheduler, or lead management platform. When an agent clicks the "Web Form" button, VICIdial opens a URL in a new browser tab, passing customer data as URL parameters.

From the agent's perspective, the training is straightforward:

1. Click **Web Form** (or Web Form 2, if configured) when you need to access the external system
2. The customer's information pre-populates in the form
3. Enter any additional data during the call
4. Save/submit in the external system
5. Return to the VICIdial agent screen to disposition the call

**What agents need to understand:** the web form is a separate window. Don't close the VICIdial agent screen. Don't close the web form before saving. If you have a dual-monitor setup (and you should for any serious operation), put VICIdial on one screen and the web form on the other.

From the admin's perspective, web forms are configured per-campaign at Admin → Campaigns → Web Form. The URL uses VICIdial field variables like `--A--phone_number--B--`, `--A--first_name--B--`, and `--A--vendor_lead_code--B--` that get replaced with actual lead data when the agent clicks the button. You can pass any lead field, custom field, or agent field to your external system this way.

Advanced integrations use VICIdial's Non-Agent API or the Agent API to push and pull data programmatically. If you're building a custom CRM integration, start with the API documentation in the VICIdial SVN at `/agc/vicidial_api.php` and the non-agent API at `/vicidial/non_agent_api.php`.

---

## Building Your Training Program: The Practical Playbook

Knowing the material is one thing. Turning it into a repeatable training program is another. Here's how we structure training at the call centers we build.

### Week 1: Admin Bootcamp (For Managers/Supervisors)

**Day 1–2: Navigation and Architecture**
- Tour every admin panel section without changing anything
- Build a mental map of how objects connect (users → phones → campaigns → lists → carriers)
- Create test users with different permission levels
- Create test phones and verify SIP/WebRTC registration

**Day 3: Campaign Building**
- Create a test campaign from scratch
- Configure dial method, statuses, [pause codes](/glossary/pause-code/), hopper settings
- Set up a call script
- Configure [campaign recording](/settings/campaign-recording/)

**Day 4: Lead Management and Carriers**
- Load test leads from CSV
- Practice list recycling and status resets
- Review carrier configuration (even if someone else set it up, you need to understand it)
- Configure DID routing for inbound test calls

**Day 5: Reports and Real-Time Monitoring**
- Run every major report type against test data
- Set up the real-time report on a dedicated display
- Practice exporting data
- Learn to identify common problems from report data (high pause time, low disposition accuracy, excessive drops)

### Week 2: Agent Training (3-Day Intensive)

**Day 1: Login, Interface, and Call Flow**
- Login process (web + phone)
- Agent screen orientation
- Practice calls with test leads (agents call each other)
- Disposition training — go through every status and when to use it
- Pause code training — practice pausing and unpausing correctly

**Day 2: Advanced Call Handling**
- Transfers — blind, warm, and conference
- Callback scheduling — practice setting callbacks and receiving them
- Web form usage
- Manual dial practice
- Edge cases: what to do when the screen freezes, what to do when audio drops, what to do when you accidentally hang up

**Day 3: Live Fire With Supervision**
- Agents take real calls with a supervisor listening in (Admin → Users → Monitor)
- Real-time coaching via whisper (if configured)
- End-of-day debrief: disposition accuracy review, call quality review, questions

**Ongoing: Weekly Calibration**
- Pull disposition reports weekly and compare to actual call outcomes
- Identify agents who are misdisposing and retrain specifically
- Review callback completion rates
- Review pause-code usage for gaming (agents spending 90 minutes per day in BATHROOM)

---

## Advanced Admin Training: The Settings That Separate Amateurs From Professionals

Once you're comfortable with the basics, these are the settings and techniques that take your operation from "functional" to "optimized."

### AMD Tuning

Answering Machine Detection is the single highest-impact setting in VICIdial for outbound campaigns. The default AMD settings are conservative. Too conservative for most operations — they'll pass through 15–20% of answering machines as live calls, wasting your agents' time.

The settings that matter: AMD Active, AMD Type (the VICIdial-native AMD is generally better than Asterisk's built-in AMD), AMD Threshold (start at 120ms and adjust down based on your testing), and AMD Message Duration (the cutoff between "hello" and a voicemail greeting).

We get AMD accuracy to 92–96% on our managed deployments. The default out-of-the-box accuracy is typically 80–85%. That difference means your agents spend 7–16% more of their time talking to real humans instead of answering machines. Over a day, that's hours of productive talk time recovered. For the full AMD tuning guide, including the specific values we use, see our [AMD guide](/blog/vicidial-amd-guide/).

### Dial Level Optimization

The [Auto Dial Level](/settings/auto-dial-level/) determines how many calls VICIdial places per available agent. Set it too low and agents sit idle. Set it too high and you drop calls (which means TCPA fines and angry customers).

In adaptive modes (ADAPT_HARD_LIMIT, ADAPT_TAPERED), VICIdial automatically adjusts the dial level based on real-time metrics. But "automatically" doesn't mean "optimally." The system needs guardrails:

- **Drop Rate Max** — Set to 3% for TCPA compliance. The system will throttle dialing to stay under this number.
- **Dial Level Difference Target** — How aggressively the system adjusts. Start at 0 and increase by 0.5 increments.
- **Available Only Tally** — Set to Y. Always. This tells the system to only count truly idle agents when calculating dial ratios, not agents who are paused or wrapping up.
- **Hopper Level** — Keep this at 100–200. The hopper pre-fetches leads so the dialer doesn't have to wait on database queries. Too low = dialing gaps. Too high = wasted resources on leads that won't get called.

For the complete breakdown of every setting that affects dialing behavior, see our [predictive dialer settings guide](/blog/vicidial-predictive-dialer-settings/).

### Call Time and State Restrictions

This is compliance territory. VICIdial's Call Times system controls when campaigns can dial based on day-of-week, time-of-day, and state-level rules. The TCPA and individual state laws have specific windows for when you can call consumers.

Admin → Call Times. Create a call time entry for each regulatory window you need. The default "9am-9pm" is a starting point, but individual states have stricter rules (e.g., some states prohibit calls before 10 AM or after 8 PM local time).

**State call time enforcement** requires that your lead data includes the state field. If state is blank, VICIdial can't apply state-specific rules and will fall back to the campaign default. Garbage lead data = compliance exposure.

### Quality Assurance and Monitoring

VICIdial supports three levels of live monitoring:

- **Listen** — Supervisor hears both sides of the conversation. Agent doesn't know.
- **Whisper** — Supervisor can speak to the agent. Customer can't hear.
- **Barge** — Supervisor joins the call. Everyone can hear everyone.

Set this up at Admin → Users → modify the supervisor account → set Monitor to Y. The supervisor accesses monitoring through the real-time report — click on an active agent and choose the monitoring mode.

Combine live monitoring with call recording review for a complete QA program. Pull recordings from the Export Calls Report, score them against your quality rubric, and feed the results back into targeted training.

> **ViciStack Includes QA Dashboards and Training Support.**
> We build custom real-time monitoring dashboards and provide ongoing training for your supervisors. Stop guessing which agents need help. [See What We Build →](/free-audit/)

---

## Community Resources: What's Actually Worth Your Time

VICIdial's community is its greatest asset and its biggest frustration. The information exists — it's just scattered across a dozen platforms with no curation. Here's what's worth bookmarking and what you can skip.

### Must-Have Resources

**The VICIdial Forum (forum.vicidial.org)** — 13,400+ topics and the single most authoritative source of VICIdial knowledge outside the source code itself. Search before posting. The two people whose answers carry the most weight: Matt Florell (mflorell, the creator of VICIdial) and William Conley (williamconley, long-time community admin). If they say something, it's correct.

**The VICIdial Manager Manual (Amazon, $45–65)** — Matt Florell's comprehensive reference book. It's not a training guide — it's a reference manual. Think of it as VICIdial's equivalent of a man page. Dry but authoritative. Worth owning if you're administering VICIdial full-time.

**ViciBox Documentation (docs.vicibox.com)** — Hardware requirements, installation phases, networking, firewall rules. Specifically relevant if you're on ViciBox (which you should be for new installs).

**VICIdial SVN Source Code** — `svn://svn.eflo.net:3690/agc_2-X/trunk`. When all documentation fails, the source code is the final truth. The API documentation in particular lives in the source files themselves (`vicidial_api.php`, `non_agent_api.php`). If you're building integrations, you'll be reading PHP source.

**VICIdial YouTube Channels** — A handful of creators maintain useful walkthroughs. Filter for videos from the last 18 months. Anything older likely references deprecated interfaces or dead operating systems.

### The ViciStack Knowledge Base

This is us. We publish guides because we run VICIdial every day and the existing documentation has gaps you could drive a truck through.

- [VICIdial Setup Guide](/blog/vicidial-setup-guide/) — Complete installation walkthrough for ViciBox 12 and AlmaLinux 9
- [Predictive Dialer Settings](/blog/vicidial-predictive-dialer-settings/) — Every dial setting that actually matters, with recommended values
- [Agent Screen Customization](/blog/vicidial-agent-screen-customization/) — Layout changes, custom buttons, field visibility, branding
- [Reporting and Monitoring](/blog/vicidial-reporting-monitoring/) — Every report type explained, real-time dashboards, KPI tracking
- [AMD Guide](/blog/vicidial-amd-guide/) — Answering machine detection tuning for 92–96% accuracy
- [STIR/SHAKEN Guide](/blog/stir-shaken-vicidial-guide/) — Caller ID attestation compliance
- [DID Management](/blog/vicidial-did-management/) — Number reputation, rotation strategies, spam flag avoidance

### Resources That Waste Your Time

**Outdated Udemy courses** — Most VICIdial courses on Udemy haven't been updated since 2019–2020. They reference CentOS 7, old admin panel layouts, and deprecated PHP functions. The instructors may know VICIdial, but the content is fossil.

**Random blog posts about "installing VICIdial"** — If the post mentions CentOS 7, ViciBox 8/9, or Asterisk 13, close the tab. It will actively mislead you.

**Facebook groups** — There are a few VICIdial-related Facebook groups. The signal-to-noise ratio is abysmal. You'll get 10 wrong answers for every correct one, and the correct one is usually "ask on the forum instead."

---

## ViciStack Training Services: What We Actually Offer

We wrote this guide to be genuinely useful on its own. But we'd be lying if we said self-training gets you to the same level as structured, expert-led training on a properly optimized system.

Here's what ViciStack's training includes as part of our managed deployments:

**Admin Training (included with every deployment):**
- Two-hour live session covering your specific campaign configuration
- Custom documentation for your operation's workflows
- Recorded walkthrough of every report you'll use
- Direct Slack/phone access to our engineering team for questions

**Agent Training (included with every deployment):**
- Pre-built training curriculum tailored to your vertical
- Recorded video walkthroughs of the agent interface
- Quick-reference cards for dispositions, pause codes, and transfer procedures
- First-week monitoring and coaching support

**Ongoing Support:**
- Campaign optimization reviews (monthly)
- Agent performance analysis with specific training recommendations
- New feature training as VICIdial updates roll out
- Emergency support for "the dialer is doing something weird and I don't know why" situations

The difference between self-training and ViciStack-supported training isn't the information — it's the context. We know which settings interact with each other in non-obvious ways. We know which "recommended" forum advice actually causes problems in production. We've seen every mistake because we've helped 200+ call centers through them.

---

## The 30-Day Training Timeline: Putting It All Together

Here's the realistic timeline for getting a new VICIdial operation from "installed" to "running smoothly with trained staff."

**Days 1–5: Admin foundation.** One designated admin works through Phases 1–4 of the admin training path. Build test campaigns, load test leads, make test calls. Don't let agents near the system yet.

**Days 6–10: Admin advanced + agent prep.** Admin works through Phase 5 (inbound routing) and Phase 6 (reports). Simultaneously, build your agent training materials: login instructions, disposition guides, pause code cheat sheets, transfer procedures.

**Days 11–13: Agent training.** Run the 3-day agent training intensive. Use test campaigns with real phone numbers but internal-only calls for days 1–2.

**Days 14–20: Supervised live calling.** Agents take real calls with supervisors monitoring. Daily debriefs. Daily disposition accuracy reviews. Fix problems immediately — bad habits that survive the first week become permanent.

**Days 21–30: Optimization.** Pull your first month's reports. Identify patterns: Which agents have the highest pause time? Which dispositions are being used incorrectly? Is your dial level too aggressive (high drop rate) or too conservative (high agent idle time)? Make adjustments and continue monitoring.

**Day 31+: Ongoing.** Weekly report reviews. Monthly campaign audits. Continuous agent coaching based on data. This never stops. The operations that plateau are the ones that stop looking at their numbers.

---

## Common Training Mistakes (And How to Avoid Them)

We've trained 10,000+ agents. These are the mistakes we see most often.

**Mistake 1: Training agents on admin concepts.** Your agents don't need a 2-hour lecture on how predictive dialing algorithms work. They need to know: calls come in automatically, answer immediately, use the script, disposition correctly, pause when you're not ready. That's it.

**Mistake 2: Not making pause codes mandatory.** If pause codes are optional, agents won't use them. If agents don't use them, your reports are incomplete. If your reports are incomplete, you can't optimize. Enable FORCE from day one. See the [Agent Pause Codes Active](/settings/agent-pause-codes-active/) setting.

**Mistake 3: Skipping disposition training.** New admins create 15 disposition statuses and then tell agents "just pick the right one." Agents don't know what "right" means. Create a one-page disposition guide with clear rules: "Use SALE when the customer verbally commits AND provides payment. Use CALLBACK when the customer says 'call me later' AND you set the callback time. Use DNC when the customer says 'do not call' OR any variation thereof."

**Mistake 4: No test environment.** Training new agents on your production campaign means their practice calls go to real customers. Build a test campaign with internal-only numbers for training. Repurpose old lists that have been fully dialed for practice.

**Mistake 5: Training once and forgetting.** VICIdial gets SVN updates regularly. Your campaigns evolve. New features get added. Schedule quarterly refresher training for agents and monthly admin skill-ups. The operations that plateau are the ones that trained once in 2023 and haven't revisited since.

**Mistake 6: Ignoring the reports.** VICIdial generates an enormous amount of data. If nobody is reviewing it, you have no feedback loop. Assign someone — a supervisor, a QA lead, someone — to review the real-time report daily and pull performance reports weekly. No exceptions.

---

## Frequently Asked Questions

### How long does it take to fully train a new VICIdial admin?

Expect 2–4 weeks for a new admin to become comfortable with daily operations — campaign management, lead loading, basic reporting, user management. True proficiency with advanced features like carrier configuration, cluster management, API integrations, and performance optimization takes 2–3 months of hands-on experience. VICIdial has a deep feature set, and there's no shortcut to learning the edge cases. The forum becomes your best friend during this phase. If you're working with ViciStack, we compress this timeline significantly through structured training and direct access to engineers who've configured hundreds of systems.

### What's the minimum training an agent needs before taking live calls?

Three days is the minimum we'd recommend. Day one covers login, interface orientation, and practice calls. Day two covers transfers, callbacks, and web forms. Day three is supervised live calling. Some operations try to get agents on the phone after a 30-minute screen share. Those operations have the highest turnover, the worst disposition accuracy, and the most compliance incidents. Investing three days upfront saves weeks of cleanup later.

### Do I need to buy the VICIdial Manager Manual?

It's helpful but not essential. The manual is a reference guide, not a training program. If you're a full-time VICIdial admin, the $45–65 is worth it for having an authoritative offline reference. If you're a supervisor who occasionally adjusts campaign settings, the combination of this guide, our other blog posts, and the VICIdial forum will cover 95% of what you need.

### How do I train remote agents who I can't sit next to?

Record your training sessions. Every one of them. Use screen-sharing software (Zoom, Google Meet, whatever) for live training and save the recordings. Build a shared folder with: (1) login instructions with screenshots, (2) a disposition guide, (3) a pause code cheat sheet, (4) a transfer procedure document, (5) a "what to do when something breaks" FAQ. For the first week, use VICIdial's built-in monitoring to listen to remote agents' calls and provide feedback via a separate chat channel. WebRTC/ViciPhone makes remote agent deployment trivially easy from a technology standpoint — the challenge is entirely in training and supervision.

### What VICIdial certifications exist?

There is no official VICIdial certification program. The software is open-source and community-maintained. Some third-party training providers offer "VICIdial certified" designations, but these carry no official weight. What matters is hands-on experience: Can you build a campaign from scratch? Can you troubleshoot one-way audio? Can you read a real-time report and identify problems? Can you optimize dial ratios based on performance data? Those skills are your certification. If you want formal validation, ViciStack provides documentation of training completion for our managed clients, which at least confirms you trained with people who build VICIdial systems for a living.

### How often should I retrain agents?

Formally, quarterly. Informally, continuously. Quarterly retraining sessions should cover: new features or interface changes, disposition accuracy review (pull the reports and show agents where they're misdisposing), compliance updates (TCPA rules evolve), and technique refreshers. Informally, your supervisors should be providing daily coaching based on monitoring — "Hey, you dispositioned that last call as NOT_INT but the customer said to call back next week, that should be a CALLBK." The best call centers treat training as a continuous process, not a one-time event.

### What's the biggest training gap you see in self-managed VICIdial operations?

Reporting. Every time. The admin knows how to create campaigns and load leads. The agents know how to take calls and set dispositions. But nobody knows how to read the reports, and nobody is reviewing them regularly. This means problems go undetected for weeks or months: agents gaming pause codes, disposition accuracy declining, drop rates creeping above compliance thresholds, callbacks not being completed. VICIdial gives you incredible data visibility. But data that nobody looks at is just wasted disk space. Make report review a non-negotiable part of your weekly operations rhythm.

### Can ViciStack train my existing team on a system we already run?

Yes. We do this regularly. You don't have to migrate to ViciStack-managed infrastructure to get training support. We offer standalone training engagements where we audit your current configuration, identify optimization opportunities, and train your admin and agent teams on best practices. That said, we usually find 5–10 configuration changes during the audit that produce immediate performance improvements — so training often turns into training plus optimization. [Start with a free audit](/free-audit/) and we'll tell you exactly where you stand.

---

*This guide is maintained by ViciStack and updated as the VICIdial ecosystem evolves. Last verified against VICIdial SVN trunk 2.14b0.5, March 2026. Found something outdated? [Let us know.](/free-audit/)*

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-training-guide).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
