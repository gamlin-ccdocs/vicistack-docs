# VICIdial Predictive Dialer Settings: The 15 Configuration Changes That Actually Matter

**VICIdial's campaign detail screen has over 170 configurable fields. Most admins change three of them, leave the rest at defaults, and then wonder why their agents are sitting idle 40% of the shift.** The predictive dialer algorithm is only as good as the settings feeding it. Default VICIdial is tuned conservatively — designed not to break, not to perform. The difference between a campaign burning through $0.03/minute trunk costs with agents staring at screens and one sustaining 48 minutes of talk time per hour comes down to roughly 15 settings that interact with each other in non-obvious ways. This guide covers every one of them: the specific values to set, why those values work, and how to iterate without nuking your abandon rate or catching an FCC fine.

## The Problem With Default VICIdial Settings

VICIdial ships with settings designed for the broadest possible use case: a small team running a single campaign with modest call volume. That is a reasonable engineering decision and a terrible operational one. Here is what the defaults actually look like in practice:

- **auto_dial_level** defaults to `1.0` — one line per agent, which is barely faster than manual dialing.
- **adaptive_dl_level** (maximum adapt dial level) defaults to `3.0`, but the dial method defaults to `RATIO`, meaning the adaptive algorithm never kicks in at all.
- **hopper_level** defaults to a low value and auto_hopper_level is set to `Y`, but with a multiplier of `1` the hopper runs lean — fine for 5 agents, starved for 50.
- **dial_timeout** defaults to `26 seconds`, which means agents wait through full ring cycles on numbers that are never going to answer.
- **available_only_ratio_tally** defaults to `N`, so the dialer counts paused agents as available when calculating how many lines to dial.
- **AMD (Answering Machine Detection)** is off entirely by default. Every answering machine rings through to an agent, who wastes 15-20 seconds listening to a greeting and dispositioning it.

The cumulative effect is brutal. In a 50-agent campaign running defaults, you can expect 28-32 minutes of agent talk time per hour. That means roughly 40% of your labor cost is paying people to wait. On a $15/hour fully loaded agent cost, that is $300/hour in wasted capacity across the floor — $2,400 per 8-hour shift, $12,000 per week.

The fix is not a single magic setting. It is a coordinated set of 15 changes that align the dialer algorithm, call handling pipeline, and agent routing into a system that actually predicts.

Here is what you need to understand first: VICIdial's predictive algorithm runs as a Perl script called `AST_VDadapt.pl`. Every 15 seconds, it recalculates the dial level for each campaign based on current drop rates, agent availability, call connection rates, and average talk time. The settings on this list are the inputs to that calculation. Change one and the algorithm shifts. Change all 15 correctly and you get a dialer that genuinely anticipates agent availability instead of reacting to it.

The 15 settings break into four groups:

1. **Core dial level controls** (Settings #1-3) — how aggressively the dialer places calls
2. **Hopper and queue configuration** (Settings #4-7) — ensuring the dialer has leads ready and calls route efficiently
3. **AMD and call handling** (Settings #8-11) — filtering out non-human answers before they reach agents
4. **Agent performance and scheduling** (Settings #12-15) — optimizing the human side of the equation

Let's walk through each one.

## Setting #1: adaptive_dl_level — The Most Misunderstood Setting

**Admin path:** `Admin → Campaigns → [Campaign] → Detail → Adaptive Maximum Dial Level`

**Database field:** `vicidial_campaigns.adaptive_maximum_level`

The `adaptive_dl_level` field sets the ceiling — the maximum number of lines the adaptive algorithm is allowed to dial per active agent. It does not set the actual dial level. That distinction trips up more VICIdial admins than any other setting.

When you set `adaptive_dl_level` to `3.0` (the default), you are telling the algorithm: "You can dial up to 3 lines per agent, but figure out the right number based on current conditions." The algorithm will ramp the actual dial level up and down within that ceiling every 15 seconds based on abandon rates, connection rates, and agent availability.

**The problem with 3.0:** For campaigns with fewer than 15 agents, a ceiling of 3.0 is too low. The algorithm needs headroom to compensate for statistical volatility. With 10 agents, a single long call can create a temporary surplus of available agents, and the algorithm needs room to dial ahead. With a 3.0 ceiling, it cannot.

**The problem with setting it too high:** A ceiling of 10.0 on a 50-agent campaign with a 40% contact rate will produce abandon rates north of 8% within the first hour. The algorithm will use the headroom, and with high contact rates it will overshoot.

**Recommended values by team size:**

| Agents | adaptive_dl_level | Why |
|--------|-------------------|-----|
| 5-10 | 4.0-5.0 | Small teams need higher ceilings to compensate for variance |
| 11-25 | 3.0-4.0 | Medium teams start stabilizing; algorithm needs moderate headroom |
| 26-50 | 2.5-3.5 | Large teams produce predictable patterns; less headroom needed |
| 51-100 | 2.0-3.0 | Very large teams need tighter control to prevent abandon spikes |
| 100+ | 1.5-2.5 | At scale, small dial level changes produce large call volume swings |

The key insight: `adaptive_dl_level` is a safety valve, not a performance dial. Set it high enough that the algorithm can do its job during volatile periods, but low enough that a contact rate spike does not blow through your abandon rate target.

For a deep dive on this specific setting, including the math behind the adaptive algorithm's ceiling behavior, see our complete guide at [adaptive_dl_level configuration](/settings/adaptive-dl-level/).

> **Your Settings Are Costing You Money.** ViciStack auto-tunes every setting on this list — including adaptive_dl_level — in real time based on your campaign's actual performance data. [Get Your Free Audit →](/free-audit/)

## Setting #2: abandon_rate_target — Legal Compliance and Performance

**Admin path:** `Admin → Campaigns → [Campaign] → Detail → Drop Percentage Limit`

**Database field:** `vicidial_campaigns.adaptive_dropped_percentage`

In VICIdial, "dropped" and "abandoned" mean the same thing: a call connected to a live person but no agent was available within two seconds. The `adaptive_dropped_percentage` setting tells the adaptive algorithm what abandon rate ceiling to enforce.

**The legal floor:** Under current FTC Telemarketing Sales Rule (TSR) requirements, predictive dialers must maintain an abandon rate below 3% measured across the campaign day. The FCC's TCPA rules have historically enforced the same 3% threshold, though a 2025 NPRM proposed revisiting this limit for modern dialer technology. Until that rulemaking concludes, 3% is the number you must hit.

**Why you should not set it to 3%:** If you set `adaptive_dropped_percentage` to `3`, the algorithm will treat 3% as its target and oscillate around it. Some 15-second intervals will be above 3%. VICIdial measures drop rates across five time windows — since midnight, past hour, past 30 minutes, past 5 minutes, and past 1 minute — and the algorithm considers all of them. But an aggressive target means the short windows will regularly spike above the limit.

**Recommended values:**

| Compliance requirement | adaptive_dropped_percentage | Buffer |
|------------------------|-----------------------------|--------|
| FTC/TCPA 3% rule | 1.5-2.0 | 50-33% safety margin |
| State-level 1% rules (some states) | 0.5-0.8 | 50-20% safety margin |
| Zero-tolerance (B2B, financial) | 0.5 | Minimal drops |
| Maximum aggression (no regulatory concern) | 2.5 | Still below 3% |

Setting this to `1.5` is the sweet spot for most US-based outbound campaigns. It gives the algorithm room to optimize throughput while maintaining enough buffer that even your worst 5-minute window stays under 3%.

VICIdial calculates drop rates using this basic formula:

```
drop_percentage = (total_drops / total_calls) * 100
```

The adaptive script checks this at five intervals and uses the most conservative (highest) value to determine whether to increase or decrease the dial level. With `ADAPT_TAPERED` as your dial method (more on that in Setting #3), the algorithm gets stricter as the shift progresses, which means your effective abandon rate typically decreases throughout the day.

You can monitor your live drop rate directly from the database:

```sql
SELECT campaign_id, drops_today, calls_today,
       ROUND((drops_today / calls_today) * 100, 2) AS drop_pct
FROM vicidial_campaign_stats
WHERE campaign_id = 'YOUR_CAMPAIGN';
```

For the complete compliance and configuration breakdown, see our [abandon rate target guide](/settings/abandon-rate-target/).

## Setting #3: auto_dial_level — Starting Point Optimization

**Admin path:** `Admin → Campaigns → [Campaign] → Detail → Auto Dial Level`

**Database field:** `vicidial_campaigns.auto_dial_level`

This is where most admins start — and where most stop, which is the problem. The `auto_dial_level` sets the starting dial level when the campaign launches. Once the adaptive algorithm takes over (assuming you are using an adaptive dial method), this value becomes the baseline that the algorithm adjusts from.

**Critical distinction:** If your dial method is `RATIO`, the `auto_dial_level` is the *only* dial level — it stays fixed and the adaptive algorithm never runs. If your dial method is `ADAPT_HARD_LIMIT`, `ADAPT_TAPERED`, or `ADAPT_AVERAGE`, the `auto_dial_level` is just the starting point.

**Which dial method to use:**

| Method | Behavior | Best for |
|--------|----------|----------|
| RATIO | Fixed dial level, no adaptation | Testing, very small teams (<5 agents) |
| ADAPT_HARD_LIMIT | Dials up to drop limit, hard stops when limit hit | Strict compliance environments |
| ADAPT_TAPERED | Allows overshoot early in shift, tightens over time | Most production campaigns |
| ADAPT_AVERAGE | Maintains average drop rate, loose enforcement | High-volume campaigns with large teams |

**Use `ADAPT_TAPERED` for 90% of campaigns.** It is the most forgiving method for real-world conditions. Early in the shift, when you have a full day to recover, it allows the dialer to be more aggressive. As the shift progresses and you have less time to bring averages down, it tightens. This mirrors how actual call center operations work — you want to build momentum early and protect your numbers late.

**Recommended starting auto_dial_level values:**

| Scenario | auto_dial_level | Notes |
|----------|-----------------|-------|
| New campaign, unknown contact rate | 1.0 | Start conservative, let adaptive ramp up |
| Established campaign, 30-40% contact rate | 1.5-2.0 | Algorithm will adjust within first 5 minutes |
| Clean list, high contact rate (50%+) | 1.0-1.5 | High contact rate needs a lower starting point |
| Aged list, low contact rate (<20%) | 2.5-3.0 | More dials needed to find live contacts |
| Callback/hot lead campaign | 1.0 | High answer rate requires minimal over-dialing |

The mistake most admins make is setting `auto_dial_level` to `3.0` on a campaign with `RATIO` dial method. That means three simultaneous outbound lines per agent, with zero adaptation. On a list with a 40% contact rate, that produces roughly 1.2 live connections per agent at any moment — which means 20% of connected calls have no agent available. Your abandon rate will be catastrophic.

Switch to `ADAPT_TAPERED`, set your starting `auto_dial_level` to `1.5`, your `adaptive_dl_level` to `3.5`, and your `adaptive_dropped_percentage` to `1.5`. The algorithm will find the right dial level within the first 10-15 minutes and maintain it.

For the full configuration walkthrough including interaction with other settings, see our [auto_dial_level guide](/settings/auto-dial-level/).

> **Stop Guessing Your Dial Level.** ViciStack monitors your campaign in real time and adjusts dial levels faster than any human can. [Get Your Free Audit →](/free-audit/)

## Settings #4-7: Hopper and Queue Configuration

The hopper is VICIdial's lead staging queue — a buffer of phone numbers pre-loaded from your lists, ready for the dialer to fire off. If the hopper runs dry, your dialer stalls regardless of how perfectly you tuned your dial level. These four settings control how that pipeline operates.

### Setting #4: hopper_level

**Admin path:** `Admin → Campaigns → [Campaign] → Detail → Hopper Level`

**Database field:** `vicidial_campaigns.hopper_level`

The `hopper_level` defines how many leads are staged and ready to dial at any given time. VICIdial's hopper loading script (`AST_VDhopper.pl`) runs regularly and tops off the hopper to this level.

The formula VICIdial uses for automatic hopper calculation is:

```
hopper_level = active_agents * auto_dial_level * (60 / dial_timeout) * auto_hopper_multiplier
```

For a 30-agent campaign with a dial level of 2.0, a 26-second dial timeout, and a multiplier of 1:

```
30 * 2.0 * (60 / 26) * 1 = 138 leads in the hopper
```

**The problem:** With the default multiplier of `1`, the hopper runs just enough leads to cover one cycle of dialing. If the hopper script runs slightly late, or if a batch of leads gets filtered out (DNC, timezone, etc.), the hopper drops below the dial demand and agents sit idle waiting for leads to load.

**Recommended configuration:**

Set `auto_hopper_level` to `Y` and adjust the `auto_hopper_multiplier` to `2` for campaigns with 25+ agents. This doubles the buffer, giving the system headroom to handle load spikes. For smaller campaigns (under 15 agents), a multiplier of `1.5` is sufficient.

If you prefer manual control, set `hopper_level` to at least 4x your agent count multiplied by your dial level. For a 30-agent campaign at dial level 2.0, that means a manual hopper level of `240`.

For the complete hopper tuning guide, see [hopper level configuration](/settings/hopper-level/).

### Setting #5: dial_timeout

**Admin path:** `Admin → Campaigns → [Campaign] → Detail → Dial Timeout`

**Database field:** `vicidial_campaigns.dial_timeout`

The `dial_timeout` defines how many seconds VICIdial waits for a call to be answered before hanging up. The default is 26 seconds, which means agents are waiting through 5-6 full ring cycles for numbers that will never answer.

**Why this matters for throughput:** Every second of ring time on an unanswered call is a trunk channel occupied doing nothing productive. With a 26-second timeout and a 60% no-answer rate, you are burning 15.6 seconds of trunk time per unanswered call. Cut the timeout to 18 seconds and that drops to 10.8 seconds — a 30% reduction in wasted trunk capacity.

**Recommended values:**

| Scenario | dial_timeout | Rationale |
|----------|--------------|-----------|
| General outbound (US) | 18-22 seconds | Most people answer by ring 3-4 (12-16 seconds) |
| B2B daytime calling | 20-24 seconds | Business phones may route through PBX |
| Mobile-heavy lists | 16-20 seconds | Mobile users answer fast or not at all |
| Callback campaigns | 22-26 seconds | Callbacks deserve more patience |
| High-velocity cold calling | 15-18 seconds | Speed matters more than catching stragglers |

**The 23-second rule:** In North America, most carrier voicemail systems pick up after approximately 24-25 seconds. If you want to avoid voicemail pickups (and you are not running AMD), set your `dial_timeout` to `23` seconds. This hangs up just before voicemail answers, reducing the number of voicemail connections your agents handle.

However, if you are running AMD (Settings #8-11), you can extend the timeout to 22-26 seconds and let AMD handle the voicemail filtering.

### Setting #6: available_only_ratio_tally

**Admin path:** `Admin → Campaigns → [Campaign] → Detail → Available Only Tally`

**Database field:** `vicidial_campaigns.available_only_ratio_tally`

This is a Y/N toggle that fundamentally changes how VICIdial counts agents when calculating how many lines to dial.

- **N (default):** VICIdial counts all non-paused agents (including those currently on calls) when calculating dial volume.
- **Y:** VICIdial only counts agents who are actually available (not on a call, not in wrap-up) when calculating dial volume.

**When to use N:** For campaigns with 25+ agents where you have statistical stability. The algorithm predicts that agents currently on calls will become available soon, so it pre-dials for them. This is how true predictive dialing works — anticipating availability, not reacting to it.

**When to use Y:** For small teams (under 10 agents) where a single long call can throw off predictions. With 5 agents and `available_only_ratio_tally` set to N, the dialer might place 10 calls because all 5 agents are technically non-paused — even though 4 of them are mid-conversation. Those 10 calls connect to 4 live people, but only 1 agent is actually free. Three calls get abandoned.

**The threshold approach:** VICIdial offers a middle ground with the `available_only_tally_threshold` setting. Set the threshold type to "Non-Paused Agents" and the threshold value to `10`. This makes `available_only_ratio_tally` behave as `Y` when fewer than 10 non-paused agents are active, and `N` when 10 or more are active. This is the recommended approach for campaigns that scale up and down throughout the day.

For the full guide on this setting and its interaction with dial level calculation, see [available only ratio tally configuration](/settings/available-only-ratio-tally/).

### Setting #7: dial_level_difference_target

**Admin path:** `Admin → Campaigns → [Campaign] → Detail → Dial Level Difference Target`

**Database field:** `vicidial_campaigns.dial_level_difference_target`

This setting tells the adaptive algorithm whether you want to target having agents waiting for calls (negative value) or calls waiting for agents (positive value). The default is `0`, which targets equilibrium.

**How it works:**

- **-1:** The algorithm tries to keep 1 agent available at all times. This minimizes abandoned calls but reduces agent utilization slightly.
- **0:** The algorithm targets a balance between waiting agents and waiting calls.
- **1:** The algorithm tries to keep 1 call in queue at all times. This maximizes agent utilization but increases abandon risk.

**Recommended values:**

| Priority | dial_level_difference_target | Effect |
|----------|------------------------------|--------|
| Compliance-first | -1 to -2 | Always have an agent ready; near-zero abandons |
| Balanced (recommended) | 0 | Let the algorithm find equilibrium |
| Throughput-first | 1 | Keep calls queued; agents never idle |

For most campaigns, leave this at `0` and let the adaptive algorithm and your `adaptive_dropped_percentage` setting control the balance. Only deviate if you have specific operational needs — for example, setting it to `-1` for compliance-sensitive financial services campaigns where even a single abandoned call triggers a complaint.

> **Hopper Problems Kill Campaigns Silently.** ViciStack monitors your hopper depth, lead availability, and queue health 24/7 — catching stalls before your agents even notice. [Get Your Free Audit →](/free-audit/)

## Settings #8-11: AMD and Call Handling

Answering machine detection is one of VICIdial's highest-impact features — and one of its most poorly configured. When AMD works correctly, it intercepts voicemail pickups and dead air before they ever reach an agent, recovering 15-25% of agent time that would otherwise be wasted on non-human answers. When it is configured poorly, it misclassifies live humans as machines (dropping good calls) or lets machines through to agents (wasting time). These four settings control the pipeline.

For our complete deep-dive on AMD tuning, see the [VICIdial AMD guide](/blog/vicidial-amd-guide/).

### Setting #8: AMD Type (amd_type)

**Admin path:** `Admin → Campaigns → [Campaign] → Detail → AMD Type`

VICIdial supports AMD through Asterisk's built-in AMD application. To enable it, you must configure the campaign to use an AMD-enabled routing extension.

**AMD routing extensions:**

| Extension | Behavior |
|-----------|----------|
| 8368 | Standard call routing, no AMD |
| 8369 | AMD enabled, routes detected machines to AMD action |
| 8366 | Standard routing with different ring handling |
| 8373 | AMD enabled with alternate routing behavior |

For campaigns where you want AMD active, set the Dial Prefix to route through extension `8369` or `8373`. The choice between them depends on your call flow — 8369 mirrors 8368 behavior with AMD added, while 8373 mirrors 8366.

**When to enable AMD:**

- Lists with high voicemail rates (>30% of connects) — enables AMD, saves massive agent time
- Clean, verified lists with high live-answer rates — keep AMD off to avoid false positives on live humans
- B2B campaigns to direct-dial numbers — AMD off (low voicemail rate, high false-positive risk)
- B2C campaigns to mobile numbers — AMD on (high voicemail rate)

### Setting #9: AMD Configuration (amd.conf)

**Server path:** `/etc/asterisk/amd.conf`

This is not a VICIdial admin setting — it is an Asterisk configuration file that controls the AMD algorithm's sensitivity. The parameters here determine how Asterisk distinguishes between a human answering and a voicemail greeting.

**Default amd.conf values vs. recommended tuning:**

| Parameter | Default | Recommended | What it does |
|-----------|---------|-------------|--------------|
| initial_silence | 2500 | 2000 | ms of silence before classifying as machine |
| greeting | 1500 | 1200 | ms of continuous speech indicating a greeting |
| after_greeting_silence | 800 | 300 | ms of silence after greeting to confirm machine |
| total_analysis_time | 5000 | 3500 | Maximum ms to analyze before deciding |
| min_word_length | 100 | 120 | ms minimum for a "word" |
| between_words_silence | 50 | 50 | ms of silence between words |
| maximum_number_of_words | 3 | 3 | Words before classifying as machine |
| silence_threshold | 256 | 256 | Audio level below which counts as silence |

**Why reduce total_analysis_time:** The default 5000ms (5 seconds) means AMD holds every call for up to 5 seconds before routing it to an agent. Live humans hear silence or dead air during this period, and many hang up. Reducing to 3500ms gets live calls to agents faster at the cost of slightly lower machine detection accuracy.

**Critical warning:** AMD tuning is carrier-dependent. The settings that work on SIP trunks with one carrier may produce 30% false positives on another carrier's T1 lines. Always test AMD changes with control numbers — known voicemail boxes and known live answers — before rolling to production.

```
; /etc/asterisk/amd.conf - Optimized for SIP trunks
[general]
initial_silence=2000
greeting=1200
after_greeting_silence=300
total_analysis_time=3500
min_word_length=120
between_words_silence=50
maximum_number_of_words=3
silence_threshold=256
```

After changing amd.conf, reload Asterisk:

```bash
asterisk -rx "module reload res_speech.so"
asterisk -rx "core reload"
```

For the deep-dive on AMD parameter tuning and carrier-specific profiles, see our [AMD optimization guide](/features/amd-optimization/).

### Setting #10: AMD Send to Action (amd_send_to_vmx)

**Admin path:** `Admin → Campaigns → [Campaign] → Detail → AMD Send to Action`

**Database field:** `vicidial_campaigns.amd_send_to_vmx`

When AMD detects an answering machine, this setting controls what happens next. The two primary options:

- **N:** The call is dispositioned as an answering machine and dropped. No voicemail is left.
- **Y:** The call is routed to the campaign's Answering Machine Message, which plays a pre-recorded audio file into the voicemail box.

**When to use voicemail drops (Y):**

Voicemail drops are a powerful callback generator. A well-crafted voicemail message produces callback rates of 2-8% depending on the industry and message quality. For campaigns where you want every possible touchpoint, enable this.

**Audio requirements:** VICIdial requires voicemail drop files in one of two formats:
- WAV: PCM Mono, 16-bit, 8kHz
- GSM: 8-bit, 8kHz

Upload your audio files through `Admin → Audio Store` and select them in the campaign's Answering Machine Message field.

**When to keep it off (N):**

If you are running compliance-sensitive campaigns or calling in jurisdictions where pre-recorded voicemail messages have specific consent requirements, keep this off. Also keep it off if your AMD accuracy is questionable — dropping voicemails on live humans who were misclassified is a terrible customer experience.

For the full voicemail drop configuration guide, see [amd_send_to_vmx settings](/settings/amd-send-to-vmx/).

### Setting #11: Wait For Silence (campaign-level AMD tuning)

**Admin path:** `Admin → Campaigns → [Campaign] → Detail → Wait For Silence Options`

This campaign-level setting overrides the server-level amd.conf for the specific campaign. The format is `milliseconds,iterations` — for example, `2000,2` means "wait for 2000ms of silence, detected 2 times, before classifying as machine."

**Recommended starting values:**

| Campaign type | Wait For Silence | Notes |
|---------------|-----------------|-------|
| B2C mobile-heavy | 2000,2 | Catches most voicemail greetings |
| B2C landline-heavy | 2500,2 | Landline voicemail greetings tend to be longer |
| B2B | 1500,2 | Shorter analysis for faster routing |
| Mixed | 2000,2 | Good general-purpose starting point |

The second parameter (iterations) should almost always be `2`. Setting it to `1` produces too many false positives; setting it to `3` lets too many machines through.

> **AMD Is Not Set-and-Forget.** Carrier changes, list composition shifts, and seasonal patterns all affect AMD accuracy. ViciStack continuously monitors your AMD hit rates and adjusts parameters automatically. [Get Your Free Audit →](/free-audit/)

## Settings #12-15: Agent Performance and Scheduling

The dialer can be perfectly tuned, but if agent-side settings create bottlenecks, throughput collapses. These four settings control the human side of the predictive equation — how agents are routed calls, how much wrap time they get, and how the system tracks their availability.

### Setting #12: next_agent_call

**Admin path:** `Admin → Campaigns → [Campaign] → Detail → Next Agent Call`

**Database field:** `vicidial_campaigns.next_agent_call`

This setting determines which agent receives the next connected call. VICIdial offers several routing algorithms:

| Method | Behavior | Best for |
|--------|----------|----------|
| random | Random available agent | Default, simple, fair |
| oldest_call_start | Agent who started their last call earliest | Even distribution by call start |
| oldest_call_finish | Agent who finished their last call earliest | Longest-idle-agent routing |
| longest_wait_time | Agent who has been waiting the longest | Maximizes fairness, minimizes idle time |
| fewest_calls | Agent with the fewest calls today | Balances call count across team |
| campaign_rank | Agent with highest campaign-specific rank | Skills-based routing |
| overall_user_level | Agent with highest user level | Seniority-based routing |

**Recommended: `longest_wait_time`** for most outbound campaigns. This method ensures that the agent who has been idle the longest gets the next call, which produces the most even utilization across the team and minimizes the perception of unfair call distribution. Agents who feel the system is fair complain less and perform better.

**When to use `campaign_rank`:** If you have agents with different skill levels and you want your best closers getting first crack at live connects, assign campaign ranks and use this method. Top performers get rank `9`, average performers get `5`, trainees get `1`. The dialer will route to higher-ranked agents first.

### Setting #13: wrapup_seconds

**Admin path:** `Admin → Campaigns → [Campaign] → Detail → Wrap-Up Seconds`

**Database field:** `vicidial_campaigns.wrapup_seconds`

Wrap-up time is the pause between when an agent dispositions a call and when the dialer sends them the next one. The default is `0` — no wrap time. The next call fires immediately after disposition.

**Why zero is almost never right:** Agents need time to complete notes, update CRM records, and mentally reset between calls. With zero wrap time, agents develop bad habits: they skip notes, mis-disposition calls, or disposition as fast as possible without recording useful information. Your data quality degrades silently.

**Why high values kill throughput:** Setting wrap-up to 60 seconds on a 50-agent campaign removes 50 agent-minutes per cycle from your available capacity. If your average call lasts 180 seconds and wrap is 60 seconds, you have just added 33% overhead to every call.

**Recommended values:**

| Campaign type | wrapup_seconds | Rationale |
|---------------|---------------|-----------|
| Simple disposition (A/B leads) | 5-10 | Just enough to click and breathe |
| CRM note entry required | 15-25 | Time for 1-2 sentences of notes |
| Complex data entry | 30-45 | Multi-field updates, but keep it tight |
| Post-call survey entry | 20-30 | Structured data entry |

The optimal approach for most campaigns is to set `wrapup_seconds` to `10` and combine it with `agent_pause_codes_active` set to `FORCE` (Setting #14). This gives agents a brief default pause, but if they need more time for notes they can manually pause with a code that explains why.

### Setting #14: agent_pause_codes_active

**Admin path:** `Admin → Campaigns → [Campaign] → Detail → Agent Pause Codes Active`

**Database field:** `vicidial_campaigns.agent_pause_codes_active`

This setting controls whether agents must provide a reason when they pause. The three options:

- **N:** Agents can pause freely with no accountability. No reason required.
- **Y:** Agents can select a pause code when pausing, but it is optional.
- **FORCE:** Agents must select a pause code every time they pause. They cannot resume dialing without explaining why they paused.

**Use `FORCE`. Always.** This is non-negotiable for any campaign that cares about performance data. Here is why:

Without forced pause codes, you have no visibility into why agents are not taking calls. Is the agent on a bathroom break? In a coaching session? Taking an unauthorized 20-minute personal call? Your real-time reports show "paused" but tell you nothing useful.

With `FORCE`, every pause is categorized. You can run reports that show:

```sql
SELECT sub_status, COUNT(*) AS pause_count,
       SUM(pause_sec) AS total_seconds,
       ROUND(AVG(pause_sec), 1) AS avg_seconds
FROM vicidial_agent_log
WHERE campaign_id = 'YOUR_CAMPAIGN'
  AND event_time >= CURDATE()
GROUP BY sub_status
ORDER BY total_seconds DESC;
```

This query reveals exactly where your agent time is going. If 15% of total shift time is in a "BREAK" pause code but your break policy allows 10%, you have found your leak.

**Standard pause codes to create for every campaign:**

| Code | Description | Expected usage |
|------|-------------|----------------|
| BREAK | Scheduled break | Per policy (typically 15 min/4 hrs) |
| LUNCH | Lunch break | Per policy |
| BIO | Restroom | Short, <5 min |
| TRAIN | Training/coaching | Supervisor-initiated |
| TECH | Technical issue | IT troubleshooting |
| ADMIN | Administrative task | Data entry, email |
| MEET | Meeting | Scheduled |

### Setting #15: calls_per_second (campaign and server level)

**Admin path:** `Admin → Campaigns → [Campaign] → Detail → Calls Per Second` and `Admin → Servers → [Server] → Max Trunks`

**Database field:** `servers.max_vicidial_trunks` (max concurrent channels at server level)

The `calls_per_second` setting controls how quickly the dialer fires off calls. This is not about how many calls you place — that is controlled by the dial level. This is about the rate at which those calls hit your Asterisk server and your SIP trunks.

**Why this matters:** If your dial level says "place 60 calls" and your CPS is set to `1`, those 60 calls take 60 seconds to initiate. If your CPS is `10`, they fire in 6 seconds. The difference affects trunk utilization patterns, Asterisk load, and how your carrier handles the burst.

**Recommended CPS by team size:**

| Agents | Recommended CPS | Rationale |
|--------|-----------------|-----------|
| 5-15 | 1-2 | Low volume, no need for speed |
| 16-30 | 2-5 | Moderate, smooth trunk utilization |
| 31-50 | 5-10 | Higher throughput, monitor server load |
| 51-100 | 10-20 | Requires properly sized Asterisk server |
| 100+ | 20+ (multi-server) | Split across dialer servers |

**A rule of thumb from the VICIdial community:** 1 CPS per 5 agents is a safe starting point. So a 50-agent campaign starts at 10 CPS.

**Server-level capacity:** Your Asterisk server has a finite number of concurrent channels. Set `Max Trunks` in the server configuration to limit concurrent calls. A well-provisioned single VICIdial dialer server handles 150-200 concurrent calls comfortably. Push beyond 250 and you risk audio quality degradation and Asterisk instability. For larger deployments, add dialer servers — see our [VICIdial cluster guide](/blog/vicidial-cluster-guide/) for the architecture.

**Never blindly increase CPS.** Higher CPS means higher burst load on Asterisk. If your server load average exceeds 50% of your CPU core count during peak dialing, you are at risk of audio quality issues or outright crashes.

> **Agent-Side Settings Are Half the Battle.** Most VICIdial admins optimize the dialer and ignore the agent interface. ViciStack tunes both sides — dialer aggression and agent routing — as a unified system. [Get Your Free Audit →](/free-audit/)

## Creating a Tuning Workflow: How to Iterate Safely

Changing 15 settings at once is a recipe for disaster. You will not know which change helped, which hurt, and which did nothing. Here is the iteration workflow we use for every ViciStack client deployment.

### Phase 1: Baseline (Day 1)

Before changing anything, capture your current metrics. Run this query at the end of a full production day:

```sql
SELECT
    c.campaign_id,
    cs.calls_today,
    cs.drops_today,
    ROUND((cs.drops_today / cs.calls_today) * 100, 2) AS drop_pct,
    cs.answers_today,
    ROUND(cs.answers_today / NULLIF(cs.calls_today, 0) * 100, 1) AS contact_rate,
    ROUND(AVG(vl.length_in_sec), 0) AS avg_talk_sec
FROM vicidial_campaigns c
JOIN vicidial_campaign_stats cs ON c.campaign_id = cs.campaign_id
JOIN vicidial_log vl ON vl.campaign_id = c.campaign_id
    AND vl.call_date >= CURDATE()
    AND vl.length_in_sec > 0
WHERE c.campaign_id = 'YOUR_CAMPAIGN'
GROUP BY c.campaign_id;
```

Record these numbers. They are your baseline:

- **Calls placed**
- **Drop percentage**
- **Contact rate (answers/calls)**
- **Average talk time**
- **Agent talk time per hour** (from realtime report)

### Phase 2: Core Dialer Settings (Days 2-3)

Change Settings #1-3 first:

1. Switch dial method to `ADAPT_TAPERED`
2. Set `auto_dial_level` to `1.5`
3. Set `adaptive_dl_level` to `3.5` (adjust per team size table above)
4. Set `adaptive_dropped_percentage` to `1.5`

Run for two full production days. Compare metrics to baseline. You should see:

- Talk time per hour increase by 15-25%
- Drop rate stay under 2%
- Agent idle time decrease noticeably on real-time reports

### Phase 3: Hopper and Queue (Days 4-5)

Change Settings #4-7:

1. Set `auto_hopper_level` to `Y` with multiplier `2`
2. Reduce `dial_timeout` to `20` seconds
3. Configure `available_only_ratio_tally` with threshold approach
4. Set `dial_level_difference_target` to `0`

Run for two days. You should see:

- Fewer "hopper empty" events in the logs
- Slightly higher call volume (shorter timeout = faster trunk recycling)
- More stable agent utilization patterns

### Phase 4: AMD (Days 6-8)

Change Settings #8-11:

1. Enable AMD routing (switch to extension 8369)
2. Tune `amd.conf` with recommended values
3. Configure `amd_send_to_vmx` based on your voicemail strategy
4. Set `Wait For Silence` to `2000,2`

**This phase requires the most monitoring.** Watch for:

- AMD false positive rate (live calls classified as machines) — pull AMD disposition reports and spot-check recordings
- AMD false negative rate (machines getting through to agents) — listen to agent call recordings for voicemail greetings
- Overall contact rate change — AMD should increase effective agent contact rate significantly

### Phase 5: Agent Settings (Days 9-10)

Change Settings #12-15:

1. Set `next_agent_call` to `longest_wait_time`
2. Set `wrapup_seconds` to `10`
3. Set `agent_pause_codes_active` to `FORCE` and create standard pause codes
4. Set `calls_per_second` per team size recommendations

### Phase 6: Fine-Tuning (Ongoing)

After the initial 10-day rollout, you enter continuous optimization. Review metrics weekly:

- If drop rate is consistently under 1%, your dialer is too conservative — increase `adaptive_dl_level` by 0.5
- If drop rate is hovering at 2%+, decrease `adaptive_dl_level` by 0.5
- If AMD false positives exceed 5%, reduce `total_analysis_time` in amd.conf
- If agents report frequent "dead air" calls, check AMD settings and carrier audio quality
- If hopper empties regularly, increase multiplier or check list availability

## Benchmark Values by Campaign Type

After tuning hundreds of VICIdial installations, here are the benchmark ranges you should target for common campaign types. These assume properly configured settings using the recommendations in this guide.

### Outbound Sales (B2C)

| Metric | Poor | Average | Good | Excellent |
|--------|------|---------|------|-----------|
| Agent talk time/hour | <30 min | 30-38 min | 38-45 min | 45-52 min |
| Drop rate | >3% | 2-3% | 1-2% | <1% |
| Contact rate | <15% | 15-25% | 25-35% | 35%+ |
| Agent idle time/hour | >15 min | 10-15 min | 5-10 min | <5 min |
| AMD accuracy | <80% | 80-85% | 85-92% | 92%+ |
| Calls placed per agent/hour | <30 | 30-50 | 50-80 | 80+ |

### Outbound Sales (B2B)

| Metric | Poor | Average | Good | Excellent |
|--------|------|---------|------|-----------|
| Agent talk time/hour | <25 min | 25-32 min | 32-40 min | 40-48 min |
| Drop rate | >2% | 1-2% | 0.5-1% | <0.5% |
| Contact rate | <10% | 10-18% | 18-28% | 28%+ |
| Agent idle time/hour | >18 min | 12-18 min | 7-12 min | <7 min |
| Calls placed per agent/hour | <25 | 25-40 | 40-65 | 65+ |

### Collections

| Metric | Poor | Average | Good | Excellent |
|--------|------|---------|------|-----------|
| Agent talk time/hour | <28 min | 28-35 min | 35-44 min | 44-50 min |
| Right party contact rate | <8% | 8-14% | 14-22% | 22%+ |
| Drop rate | >3% | 2-3% | 1-2% | <1% |
| Promise-to-pay rate | <5% | 5-10% | 10-18% | 18%+ |
| Calls placed per agent/hour | <35 | 35-55 | 55-85 | 85+ |

### Political/Survey

| Metric | Poor | Average | Good | Excellent |
|--------|------|---------|------|-----------|
| Agent talk time/hour | <20 min | 20-28 min | 28-38 min | 38-45 min |
| Drop rate | >5% | 3-5% | 1-3% | <1% |
| Contact rate | <12% | 12-20% | 20-30% | 30%+ |
| Survey completion rate | <15% | 15-25% | 25-40% | 40%+ |
| Calls placed per agent/hour | <40 | 40-65 | 65-100 | 100+ |

### Quick-Reference: All 15 Settings in One Table

| # | Setting | Database field | Default | Recommended starting value |
|---|---------|---------------|---------|---------------------------|
| 1 | Adaptive Max Dial Level | adaptive_maximum_level | 3.0 | 3.0-4.0 (adjust by team size) |
| 2 | Drop Percentage Limit | adaptive_dropped_percentage | 3.0 | 1.5 |
| 3 | Auto Dial Level + Method | auto_dial_level / dial_method | 1.0 / RATIO | 1.5 / ADAPT_TAPERED |
| 4 | Hopper Level / Multiplier | hopper_level / auto_hopper_multiplier | auto/1 | auto/2 |
| 5 | Dial Timeout | dial_timeout | 26 | 20 |
| 6 | Available Only Tally | available_only_ratio_tally | N | Threshold at 10 agents |
| 7 | Dial Level Difference Target | dial_level_difference_target | 0 | 0 |
| 8 | AMD Type | (dial prefix/routing) | Off | Extension 8369 |
| 9 | AMD Sensitivity | amd.conf | Conservative | See tuned values above |
| 10 | AMD Action | amd_send_to_vmx | N | Y (if voicemail drops desired) |
| 11 | Wait For Silence | (campaign setting) | Not set | 2000,2 |
| 12 | Next Agent Call | next_agent_call | random | longest_wait_time |
| 13 | Wrap-Up Seconds | wrapup_seconds | 0 | 10 |
| 14 | Pause Codes Active | agent_pause_codes_active | N | FORCE |
| 15 | Calls Per Second | (campaign/server) | 1 | 1 per 5 agents |

> **Want This Table Pre-Loaded Into Your VICIdial?** ViciStack clients get these optimized values configured and continuously tuned from day one — no manual iteration required. [Get Your Free Audit →](/free-audit/)

## The ViciStack Approach: Automated Configuration Intelligence

Everything in this guide works. The 15 settings, the phased rollout, the benchmark targets — they are based on hundreds of real deployments. But there is a problem: the optimal values are not static.

Your contact rate changes as lists age. Your agent count fluctuates throughout the day. Carrier routing changes affect AMD accuracy. New DNC regulations require tighter abandon rate buffers. The dial level that was perfect at 10 AM with 40 agents on the floor is wrong at 2 PM with 25 agents after lunch breaks.

Manual tuning means someone — usually a campaign manager or IT admin — is watching real-time reports and adjusting settings reactively. By the time they notice a problem, it has been costing money for 15-30 minutes. And most organizations do not have someone watching the real-time screen all day.

ViciStack solves this with [automated dialer tuning](/features/dialer-tuning/) that monitors every metric in this guide continuously and adjusts settings in real time:

**What ViciStack monitors every 15 seconds:**

- Actual vs. target abandon rate across all five VICIdial time windows
- Agent utilization and idle time distribution
- AMD accuracy (false positive and false negative rates)
- Hopper depth relative to dial demand
- Per-carrier connection rates and audio quality metrics
- Agent pause code patterns and wrap-time distribution

**What ViciStack adjusts automatically:**

- `adaptive_dl_level` ceiling based on team size fluctuations throughout the day
- `adaptive_dropped_percentage` based on rolling compliance calculations
- AMD sensitivity parameters based on per-carrier detection accuracy
- Hopper multipliers based on lead availability and filter rates
- Agent routing algorithms based on real-time skill performance data

**The result:** ViciStack clients see average agent talk time improvements of 20-35% within the first week, with abandon rates held consistently under 1.5%. That is not a theoretical number — it is the median across our client base.

The settings in this guide get you 80% of the way there. Automated tuning gets you the last 20% and keeps you there without human intervention.

## Frequently Asked Questions

### What is the single most impactful setting to change first?

Switch your dial method from `RATIO` to `ADAPT_TAPERED`. This single change activates VICIdial's adaptive algorithm, which automatically adjusts your dial level every 15 seconds based on real campaign conditions. With RATIO, your dial level is static — set it too low and agents idle, set it too high and calls get abandoned. With ADAPT_TAPERED, the system self-corrects.

### How do I know if my predictive settings are working?

Check three metrics at the end of every production day: agent talk time per hour (target: 38+ minutes for B2C), abandon/drop rate (target: under 2%), and average agent wait time between calls (target: under 15 seconds). If all three are in range, your settings are working. If talk time is low but abandon rate is also low, your dialer is too conservative — increase `adaptive_dl_level`. If talk time is high but abandon rate is also high, you are too aggressive — decrease `adaptive_dl_level` or lower `adaptive_dropped_percentage`.

### What happens if I set auto_dial_level too high?

If you are using an adaptive dial method (ADAPT_TAPERED, etc.), the `auto_dial_level` is just the starting point and the algorithm will correct it quickly. The risk is in the first few minutes of the campaign before the algorithm has enough data to adjust. If you set it to 5.0 on a campaign with a 50% contact rate, you will get a burst of abandoned calls in the first 2-3 minutes. Start at 1.5 and let the algorithm ramp up.

### Should I use AMD for every campaign?

No. AMD adds 2-4 seconds of analysis time to every connected call, which means live humans experience a brief silence before they hear an agent. On campaigns with high live-answer rates (callbacks, warm leads, B2B direct dial), that silence kills your connect quality. Use AMD on campaigns with voicemail rates above 25-30% where the time savings from filtering machines outweighs the quality hit on live calls.

### How does VICIdial's predictive algorithm actually work?

VICIdial runs a Perl script (`AST_VDadapt.pl`) every 15 seconds. It calculates the current drop percentage across five time windows (today, past hour, past 30 minutes, past 5 minutes, past 1 minute), compares them to your `adaptive_dropped_percentage` target, and raises or lowers the effective dial level within the bounds of your `adaptive_dl_level` ceiling. The `adapt_intensity_modifier` accelerates or dampens these adjustments. The entire system is designed to converge on the optimal dial level that maximizes agent utilization while keeping abandons under your target.

### What is the adapt_intensity_modifier and should I change it?

The `adapt_intensity_modifier` controls how aggressively the adaptive algorithm adjusts the dial level in each 15-second cycle. The default is `0`. Positive values make adjustments larger (more aggressive), negative values make them smaller (more conservative). **Leave it at 0 or set it to a small negative value like -1.** Increasing the intensity modifier causes the algorithm to overshoot in both directions, creating oscillation between too many and too few calls. A slightly negative value produces smoother, more stable dial level curves.

### What is the difference between hopper_level and auto_hopper_level?

`hopper_level` is a fixed number — "keep this many leads staged." `auto_hopper_level` set to Y tells VICIdial to calculate the hopper level dynamically based on agent count, dial level, and dial timeout. For most campaigns, use `auto_hopper_level = Y` with `auto_hopper_multiplier = 2`. This ensures the hopper scales automatically as your agent count changes throughout the day. Manual `hopper_level` is only useful when you want to override the automatic calculation for specific operational reasons.

### How do I handle DID/caller ID rotation with predictive dialing?

Predictive dialing at volume burns through caller IDs quickly — high call volumes from a single number get flagged by carriers and labeled as spam. Rotate your outbound caller IDs using VICIdial's CID Group feature, assigning a pool of numbers to the campaign. For detailed strategies on avoiding spam flags, see our [VICIdial DID management guide](/blog/vicidial-did-management/).

### Can I run predictive dialing across multiple servers?

Yes, and you should for campaigns over 100 agents. VICIdial's multi-server architecture distributes dialing load across multiple Asterisk servers while maintaining centralized campaign management. Each dialer server handles a portion of the outbound call volume, and the adaptive algorithm accounts for the aggregate performance across all servers. See our [VICIdial cluster guide](/blog/vicidial-cluster-guide/) for architecture details and capacity planning.

### How often should I review and adjust these settings?

After the initial 10-day tuning period described in this guide, review your core metrics weekly. Check abandon rates, agent talk time, and AMD accuracy. Adjust settings when you see sustained deviation from your targets — not after a single bad hour. List quality changes (aging leads, new data sources), agent count changes, and carrier changes all warrant a settings review. Or let ViciStack handle it automatically with [continuous dialer tuning](/features/dialer-tuning/).

> **Ready to Stop Tweaking and Start Performing?** Every setting in this guide is something ViciStack monitors and optimizes automatically, 24/7. No more guessing, no more spreadsheet tracking, no more settings that worked last month but are costing you money today. [Get Your Free Audit →](/free-audit/)

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-predictive-dialer-settings).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
