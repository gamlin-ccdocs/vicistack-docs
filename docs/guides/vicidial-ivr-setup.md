# VICIdial IVR Setup: Inbound Call Routing & Auto-Attendant

*Customers call your number. They hear silence for three seconds, then a robot voice says "please hold," then they wait in a queue for four minutes while a distorted MP3 loops, then they hang up. Your abandonment rate is 40% and your agents are asking why the phone never rings. Meanwhile, the guy who set up your IVR two years ago no longer works here and nobody knows which call menu routes where.*

*This is the VICIdial IVR guide that fixes all of that.*

VICIdial's [IVR](/glossary/ivr/) system is built around a feature called **Call Menus** — the admin-level configuration that controls what callers hear, which [DTMF](/glossary/dtmf/) key presses route where, and how calls flow from a [DID](/glossary/did/) to an agent's headset. It's powerful, flexible, and — if you've never set one up before — surprisingly unintuitive. The admin GUI gives you a blank form with 20+ fields, no tooltips, and documentation that assumes you already know how IVR routing works.

We've built IVR systems on VICIdial for everything from 3-option auto-attendants to 6-level-deep menu trees with time-of-day routing, language selection, account number lookups, and overflow to external numbers. This guide covers the entire stack: DID configuration, [call menu](/glossary/call-menu/) setup, DTMF routing, [inbound group](/glossary/inbound-group/) integration, audio prompts, time-based routing, multi-level IVR architectures, Asterisk [dialplan](/glossary/dialplan/) integration, and the mistakes that cause callers to end up in the wrong queue — or nowhere at all.

---

## How VICIdial Inbound Call Routing Actually Works

Before touching any configuration, you need to understand the chain of events that happens when someone dials your number. Every inbound call in VICIdial follows this path:

1. **Carrier delivers the call** to your Asterisk server via a SIP trunk
2. **Asterisk receives the INVITE** and matches the called number (DID) against the dialplan
3. **VICIdial's dialplan routes the DID** to the appropriate handler — either a Call Menu (IVR) or directly to an Inbound Group (queue)
4. **If routed to a Call Menu**, the caller hears a prompt, presses a key, and the call routes based on DTMF input
5. **The call reaches an Inbound Group**, which queues it and delivers it to an available agent based on priority, skill, and wait time

The critical concept: **DIDs, Call Menus, and Inbound Groups are three separate objects in VICIdial, and they connect like a chain.** A DID points to a Call Menu OR an Inbound Group. A Call Menu's DTMF options point to other Call Menus OR Inbound Groups. Get one link wrong and calls either dead-end or route to the wrong place.

### The DID: Where Everything Starts

A DID (Direct Inward Dial) is the phone number your customers call. In VICIdial, you configure DIDs under Admin > Inbound DID. Each DID entry tells VICIdial what to do when a call arrives for that number.

When your carrier delivers an inbound call, Asterisk matches the dialed number against VICIdial's DID table. If it finds a match, it routes according to that DID's configuration. If it doesn't find a match, the call hits the default context and typically gets a "number not in service" treatment — or worse, dead air.

Key fields when creating a DID:

| Field | Purpose | Example |
|-------|---------|---------|
| DID Extension | The phone number (E.164 or as delivered by carrier) | 15551234567 |
| DID Route | Where to send the call | CALLMENU, IN_GROUP, or AGENT |
| Call Menu | Which call menu to use (if DID Route = CALLMENU) | MAIN_IVR |
| In-Group | Which inbound group (if DID Route = IN_GROUP) | SALES_QUEUE |
| Filter URL | Optional URL for custom routing logic | (blank for most setups) |

**The DID Extension must match exactly what your carrier sends.** This trips up more people than any other IVR configuration issue. Some carriers send the full E.164 number with country code (15551234567). Others strip the country code (5551234567). Some send just the last 10 digits. Some prepend a "1". Run `asterisk -rx "sip set debug on"` and place a test call to see the exact string in the INVITE's To header — that's what you put in the DID Extension field.

If your carrier sends different formats for different numbers, you can use DID pattern matching with wildcards, but the cleanest approach is to standardize your carrier's DID delivery format. Most SIP carriers let you configure this in their portal.

---

## Call Menus: The Heart of VICIdial's IVR

A [Call Menu](/glossary/call-menu/) is VICIdial's IVR builder. It defines a prompt, a set of DTMF options, timeout behavior, and invalid-input handling. Navigate to Admin > Call Menus to create one.

### Creating Your First Call Menu

Let's build a standard auto-attendant. The caller hears "Press 1 for Sales, Press 2 for Support, Press 3 for Billing." Each key routes to a different inbound group.

**Step 1: Create the Call Menu**

Navigate to Admin > Call Menus > Add A New Call Menu.

| Field | Value | Why |
|-------|-------|-----|
| Menu ID | MAIN_IVR | Unique identifier. Alphanumeric, no spaces. |
| Menu Name | Main Auto-Attendant | Descriptive name for the admin GUI. |
| Menu Prompt | `main-greeting` | Asterisk audio file name (without extension). |
| Menu Timeout | 10 | Seconds to wait for DTMF input. |
| Menu Timeout Prompt | `sorry-try-again` | What to play if caller doesn't press anything. |
| Menu Invalid Prompt | `invalid-option` | What to play if caller presses an unrecognized key. |
| Menu Repeat | 3 | How many times to replay before timeout action. |
| Menu Time Check | DISABLED | We'll cover time-based routing later. |
| Tracking Group | MAIN_IVR_TG | For reporting on IVR traffic. (Optional) |

**Step 2: Define DTMF Options**

In the Call Menu Options section, add one row per key:

| Option | Destination | Route Type |
|--------|-------------|------------|
| 1 | SALES_QUEUE | CALLMENU or IN_GROUP |
| 2 | SUPPORT_QUEUE | IN_GROUP |
| 3 | BILLING_QUEUE | IN_GROUP |
| 0 | OPERATOR_QUEUE | IN_GROUP |
| t | SALES_QUEUE | IN_GROUP |
| i | (replay menu) | CALLMENU: MAIN_IVR |

The `t` option handles timeout — if the caller doesn't press anything after all repeats, this is where they go. Most operations route timeouts to the default queue (usually Sales) rather than disconnecting. The `i` option handles invalid input — routing back to the same call menu replays the prompt.

**Step 3: Link the DID to the Call Menu**

Go to Admin > Inbound DID, find your DID, set DID Route to `CALLMENU`, and select `MAIN_IVR` from the dropdown. Save.

That's it. Callers to that number now hear your greeting and route based on their key press.

### The Call Menu Option Types

Each DTMF option in a Call Menu can route to one of several destination types:

**IN_GROUP** — Routes the call to a VICIdial [Inbound Group](/glossary/inbound-group/) (agent queue). This is the most common destination for leaf-level menu options. The caller enters the queue and waits for an available agent.

**CALLMENU** — Routes to another Call Menu. This is how you build multi-level IVRs. "Press 1 for Sales" goes to the Sales sub-menu, which offers "Press 1 for New Accounts, Press 2 for Existing Accounts."

**DID** — Routes to another DID entry. Useful for forwarding to external numbers configured as DIDs with external routing.

**EXTENSION** — Routes to a specific Asterisk extension. Use this for direct-to-agent transfers, voicemail boxes, or custom dialplan destinations defined in `extensions_custom.conf`.

**VOICEMAIL** — Sends the caller directly to a voicemail box. Requires voicemail to be configured in Asterisk's `voicemail.conf`.

**AGI** — Executes an Asterisk AGI script. This is the advanced option for custom routing logic — database lookups, API calls, dynamic queue assignment. We cover this in the Asterisk integration section.

**HANGUP** — Disconnects the call. Use sparingly — after playing a final message on options where continuing the call doesn't make sense (like a "store hours" informational option).

### Menu Prompt Configuration

The Menu Prompt field takes either a single Asterisk audio file name or a pipe-delimited sequence:

**Single prompt:**
```
main-greeting
```

Asterisk looks for `main-greeting.wav` (or `.gsm`, `.sln`, etc.) in `/var/lib/asterisk/sounds/`.

**Multi-file prompt sequence:**
```
welcome-to-company|press-one-for-sales|press-two-for-support|press-three-for-billing
```

Asterisk plays each file in order. This is useful because you can swap individual segments without re-recording the entire greeting. Changed your billing department's name? Replace `press-three-for-billing.wav` and leave everything else alone.

**Inline TTS or silence:**
```
main-greeting|silence/2|menu-options
```

The `silence/2` inserts 2 seconds of silence between the greeting and the options. This small touch makes the IVR sound more professional — a wall of unbroken speech is harder for callers to process.

---

## Audio Prompts: Recording, Formatting, and Uploading

Bad audio is the number one thing that makes an IVR sound amateur. A tinny recording with background noise and inconsistent volume tells callers they've reached a company that doesn't care about details. Get this right.

### Recording Format Requirements

Asterisk is flexible about audio formats, but for optimal performance and quality in VICIdial IVR prompts, use:

- **Format:** WAV (PCM)
- **Sample rate:** 8000 Hz (8 kHz)
- **Bit depth:** 16-bit
- **Channels:** Mono
- **Encoding:** Linear PCM (signed, little-endian)

Why 8 kHz mono? Because that's the native format for G.711 telephony audio. If you upload a 44.1 kHz stereo WAV, Asterisk transcodes it on every playback — wasting CPU and often introducing artifacts. Record at the target format and skip the runtime conversion.

### Converting Audio with SOX

If you have a professionally recorded prompt in studio format (44.1 kHz, stereo, 24-bit), convert it:

```bash
sox original-recording.wav -r 8000 -c 1 -b 16 main-greeting.wav
```

For batch conversion of an entire directory:

```bash
for f in /path/to/originals/*.wav; do
  sox "$f" -r 8000 -c 1 -b 16 "/var/lib/asterisk/sounds/$(basename $f)"
done
```

Verify the result:

```bash
soxi /var/lib/asterisk/sounds/main-greeting.wav
```

You should see: `Sample Rate: 8000`, `Channels: 1`, `Precision: 16-bit`.

### Uploading via VICIdial Admin

VICIdial provides an audio upload interface under Admin > Audio Store. Upload your WAV files here, and VICIdial copies them to the correct Asterisk sounds directory. The file name (minus extension) becomes the prompt name you reference in Call Menus.

You can also upload directly via SCP:

```bash
scp main-greeting.wav root@vicidial-server:/var/lib/asterisk/sounds/
```

Set ownership and permissions:

```bash
chown asterisk:asterisk /var/lib/asterisk/sounds/main-greeting.wav
chmod 644 /var/lib/asterisk/sounds/main-greeting.wav
```

### Recording via the Asterisk CLI

For quick test prompts, record directly from a phone connected to your system:

```bash
asterisk -rx "originate SIP/100 application Record test-prompt.wav"
```

This calls extension 100 (an agent phone) and records whatever the person says into `test-prompt.wav`. Press `#` to stop recording. The file lands in `/var/lib/asterisk/sounds/`.

### Professional Recording Tips

If you're investing in professional voice talent (and you should for customer-facing IVRs):

1. **Record all prompts in the same session** with the same talent. Nothing sounds worse than mismatched voices across menu levels.
2. **Record each option as a separate file.** "Press 1 for Sales" is one file. "Press 2 for Support" is another. This modular approach lets you rearrange and update without re-recording everything.
3. **Include 0.5 seconds of clean silence** at the start and end of each file. This prevents audio clipping when Asterisk concatenates prompts.
4. **Normalize volume** across all files. If the greeting is loud and the options are quiet, callers miss the menu choices.

---

## Inbound Groups: Where Calls Meet Agents

Once a caller navigates the IVR and selects an option, they typically land in an [Inbound Group](/glossary/inbound-group/). This is VICIdial's inbound queue — the system that holds callers, plays hold music, and delivers calls to available agents.

Understanding Inbound Groups is essential for IVR setup because they're the destination for most Call Menu options. You can read the deep dive in our [inbound configuration guide](/blog/vicidial-inbound-setup/), but here's what you need to know for IVR integration.

### Creating an Inbound Group for IVR Routing

Navigate to Admin > Inbound Groups > Add A New Group.

| Field | Value | Notes |
|-------|-------|-------|
| Group ID | SALES_QUEUE | Matches what you entered in the Call Menu option |
| Group Name | Sales Queue | Descriptive name |
| Group Color | #0066CC | Visual identifier in real-time reports |
| Active | Y | Must be active to receive calls |
| Queue Priority | 0 | Higher number = higher priority. See [Inbound Queue Priority](/settings/inbound-queue-priority/). |
| Call Time ID | 24hours | Which call time definition applies |
| After Hours Action | CALLMENU | What happens outside business hours |
| After Hours Call Menu | AFTERHOURS_IVR | Routes to an after-hours menu |

The **After Hours Action** is critical for IVR design. When an inbound group has a call time defined and a call arrives outside that window, the After Hours Action determines what happens. Setting it to CALLMENU lets you route to a separate IVR that plays "We're currently closed" and offers voicemail or callback options.

### Agent Assignment: Closer Campaigns

Agents receive inbound calls through [Closer Campaigns](/settings/closer-campaigns/). An agent logged into a closer campaign with the SALES_QUEUE group selected will receive calls from that queue.

In VICIdial Admin > Campaigns, your inbound-receiving campaign needs:

- **Campaign Type:** Set to allow inbound (or create a dedicated CLOSER campaign type)
- **Allowed Inbound Groups:** Select which inbound groups this campaign's agents can receive
- **Dial Method:** For blended operations (outbound + inbound), agents take inbound calls between outbound attempts

The priority system determines which agents get calls first:

1. **Agent rank** within the inbound group (set per-agent)
2. **Longest wait time** among equal-rank agents
3. **Queue priority** if multiple inbound groups are competing for the same agents

### Hold Music and Queue Announcements

When a caller enters an inbound group, they hear hold music until an agent is available. Configure this in the Inbound Group settings:

| Field | Purpose |
|-------|---------|
| On Hold Prompt | Audio file or music-on-hold class to play while waiting |
| Hold Time Announce | Play estimated hold time ("Your call will be answered in approximately 3 minutes") |
| Queue Position Announce | Play queue position ("You are caller number 5 in the queue") |
| Announce Frequency | How often to repeat queue announcements (in seconds) |

For music-on-hold, you have two options. You can specify a single audio file name in the On Hold Prompt field, or you can use Asterisk's music-on-hold classes defined in `/etc/asterisk/musiconhold.conf`:

```ini
[sales-hold]
mode=files
directory=/var/lib/asterisk/moh/sales
sort=alpha

[support-hold]
mode=files
directory=/var/lib/asterisk/moh/support
sort=random
```

Place your hold music files (8 kHz mono WAV) in the respective directories. Different hold music per queue is a simple way to reinforce branding and set caller expectations.

> **IVR Dropping Calls? Queues Overflowing?**
> ViciStack builds and tests your entire inbound routing chain before go-live. Call menus, queue priorities, overflow rules, time-based routing — all configured and verified. [Get Your Free Assessment -->](/free-audit/)

---

## Time-Based Routing: Business Hours, Holidays, and Overflows

A production IVR can't play the same menu 24/7. During business hours, "Press 1 for Sales" routes to a live queue. After hours, it should route to voicemail or an after-hours message. On holidays, you need a different greeting entirely. VICIdial handles this with Call Time definitions and Call Menu time checks.

### Setting Up Call Times

Navigate to Admin > Call Times to define your business hours.

**Example: Standard US business hours:**

| Field | Value |
|-------|-------|
| Call Time ID | BUSINESS_HOURS |
| Call Time Name | Business Hours M-F 8-6 |
| Monday Start | 0800 |
| Monday End | 1800 |
| Tuesday Start | 0800 |
| Tuesday End | 1800 |
| ... | ... |
| Saturday Start | (blank) |
| Saturday End | (blank) |
| Sunday Start | (blank) |
| Sunday End | (blank) |

Leaving Saturday and Sunday blank means those days are considered "after hours" for the entire 24 hours.

**Holiday handling:**

VICIdial supports holiday definitions via Admin > Call Time Holidays (sometimes listed under Call Times). Define each holiday with a specific date:

| Holiday | Date |
|---------|------|
| New Year's Day | 2026-01-01 |
| Memorial Day | 2026-05-25 |
| Independence Day | 2026-07-04 |
| Labor Day | 2026-09-07 |
| Thanksgiving | 2026-11-26 |
| Christmas | 2026-12-25 |

When a holiday is defined and associated with a Call Time, that day is treated as after-hours regardless of the day-of-week schedule.

### Call Menu Time Check

In the Call Menu configuration, set **Menu Time Check** to the Call Time ID you created. Then configure two sets of DTMF options:

**In-Hours Options:** The normal menu. Press 1 for Sales, Press 2 for Support, etc.

**After-Hours Options:** A different set of destinations. Press 1 for Voicemail, Press 2 to hear store hours, etc.

The Call Menu form gives you separate prompt and option fields for in-hours and after-hours routing. When the time check is enabled, VICIdial checks the current time against the Call Time definition before playing the prompt. If it's within business hours, callers get the in-hours treatment. Outside business hours (or on holidays), they get the after-hours treatment.

**Configuration example:**

```
Menu ID:          MAIN_IVR
Menu Time Check:  BUSINESS_HOURS

--- In-Hours ---
Prompt:           business-hours-greeting
Option 1:         IN_GROUP → SALES_QUEUE
Option 2:         IN_GROUP → SUPPORT_QUEUE
Option 3:         IN_GROUP → BILLING_QUEUE
Timeout:          IN_GROUP → SALES_QUEUE

--- After-Hours ---
Prompt:           after-hours-greeting
Option 1:         VOICEMAIL → 1000
Option 2:         EXTENSION → hours-info
Option 3:         CALLMENU → CALLBACK_MENU
Timeout:          VOICEMAIL → 1000
```

This single Call Menu handles both business hours and after-hours routing without any external scripting or manual switching.

### Time Zone Considerations

VICIdial uses the server's system time for Call Time evaluations. If your server is in US Eastern but your business operates in US Pacific, your "9 AM open" will actually trigger at noon Eastern. Set the server timezone correctly:

```bash
timedatectl set-timezone America/Los_Angeles
```

For operations spanning multiple time zones, you have two options: set the server to the timezone of your primary business location, or use custom AGI scripts that evaluate multiple time zones and route accordingly.

---

## Multi-Level IVR: Building Complex Menu Trees

Real-world IVR systems rarely stop at one level. A caller presses 1 for Sales, then hears "Press 1 for New Customers, Press 2 for Existing Accounts, Press 3 for Partner Inquiries." That's a two-level IVR. Some operations go three or four levels deep — language selection, then department, then sub-department, then product line.

### Architecture: Call Menu Chains

Multi-level IVRs in VICIdial are built by chaining Call Menus. Each DTMF option set to route type CALLMENU points to another Call Menu, which has its own prompt and options.

**Example: Two-level IVR**

```
MAIN_IVR (Level 1)
├── Press 1 → SALES_MENU (Level 2)
│   ├── Press 1 → NEW_CUSTOMERS_QUEUE (Inbound Group)
│   ├── Press 2 → EXISTING_ACCOUNTS_QUEUE (Inbound Group)
│   ├── Press 3 → PARTNERS_QUEUE (Inbound Group)
│   └── Press 0 → MAIN_IVR (back to main menu)
├── Press 2 → SUPPORT_MENU (Level 2)
│   ├── Press 1 → TECH_SUPPORT_QUEUE (Inbound Group)
│   ├── Press 2 → BILLING_SUPPORT_QUEUE (Inbound Group)
│   └── Press 0 → MAIN_IVR (back to main menu)
├── Press 3 → BILLING_QUEUE (Inbound Group — direct, no sub-menu)
├── Press 9 → DIRECTORY_EXTENSION (Asterisk extension for dial-by-name)
└── Press 0 → OPERATOR_QUEUE (Inbound Group)
```

Each box in this tree is a separate Call Menu in VICIdial Admin. To build this:

1. Create the leaf-level Call Menus first (SALES_MENU, SUPPORT_MENU)
2. Create the top-level Call Menu (MAIN_IVR) that references them
3. Link the DID to MAIN_IVR

**Always include a "return to previous menu" option.** Press 0 or Press * to go back. Callers who land in the wrong sub-menu need an escape route. Without it, they hang up and call back, which wrecks your IVR completion metrics and frustrates customers.

### The Three-Level Trap

Three levels is the practical maximum for phone-based IVR. At four levels, callers have been pressing buttons for over a minute before they reach a human. Customer satisfaction drops sharply after 45 seconds of IVR navigation. If your routing logic requires four levels, you probably need to rethink the tree structure — consolidate options, use ANI-based routing to skip levels for known callers, or implement a speech-recognition front end.

If you absolutely must go deeper, consider this pattern: collect the most critical routing decision at level 1 (language or department), then route to an inbound group with skill-based routing rather than adding more menu levels. Let the agent handle the sub-categorization. A skilled agent asks "Are you calling about a new account or an existing one?" faster than a caller can navigate two more IVR levels.

### Language Selection IVR

A common multi-level pattern for bilingual operations:

```
LANGUAGE_SELECT (Level 1)
├── Press 1 (or timeout) → MAIN_IVR_EN (English menus)
│   ├── Press 1 → SALES_EN
│   ├── Press 2 → SUPPORT_EN
│   └── ...
├── Press 2 → MAIN_IVR_ES (Spanish menus)
│   ├── Press 1 → SALES_ES
│   ├── Press 2 → SUPPORT_ES
│   └── ...
```

Each language gets its own set of Call Menus with localized prompts and potentially different inbound groups (so Spanish-speaking agents are separate from English-speaking agents). The timeout on the language selection menu should default to the primary language — don't leave callers in limbo because they didn't understand the language prompt.

---

## DTMF Routing: Beyond Simple Key Presses

VICIdial's Call Menu options handle single DTMF digits (0-9, *, #) as well as multi-digit input. Understanding the nuances of [DTMF](/glossary/dtmf/) handling prevents routing failures.

### Single-Digit vs. Multi-Digit Input

Most IVRs use single-digit options: Press 1, Press 2, etc. But VICIdial Call Menus also support multi-digit input for scenarios like:

- **Extension dialing:** Caller enters a 3 or 4-digit extension number
- **Account lookup:** Caller enters an account number followed by #
- **PIN verification:** Caller enters a numeric code

For multi-digit input, set the Menu Timeout appropriately — callers need enough time between digits. A timeout of 3-5 seconds per expected digit is reasonable. The `#` key typically serves as the "submit" delimiter.

### DTMF Mode Must Match End-to-End

If callers press keys but nothing happens, the problem is almost always a DTMF mode mismatch somewhere in the call chain. VICIdial's IVR relies on Asterisk receiving DTMF events. If the carrier sends DTMF in-band (as audio tones) but your trunk is configured for RFC 2833, Asterisk never registers the key press.

Verify your trunk's DTMF mode matches your carrier's:

```bash
asterisk -rx "sip show peer your-carrier" | grep -i dtmf
```

You should see `dtmfmode=rfc2833` for most modern carriers. If DTMF isn't working, test with:

```bash
asterisk -rx "core set verbose 5"
```

Then place a test call and press keys. In the CLI output, you should see lines like:

```
DTMF/SIP-xxx-1 '1' received on SIP/trunk-00000001
```

If you don't see DTMF events, the issue is between the carrier and Asterisk. Check the DTMF mode setting in your carrier peer definition. Refer to the [DTMF section in our Asterisk configuration guide](/blog/vicidial-asterisk-configuration/) for the full breakdown.

### Special Options: t, i, and *

Three Call Menu options have special meaning:

- **`t` (timeout):** Triggered when the caller doesn't press anything within the Menu Timeout period, after all repeats are exhausted. Always configure this. Never let callers sit in silence.
- **`i` (invalid):** Triggered when the caller presses a key that isn't defined as an option. Route this back to the same Call Menu to replay the prompt, or to a catch-all queue.
- **`*` (star key):** Often used as "go back" or "return to main menu" in multi-level IVRs. Callers intuitively try the star key when they want to start over.

---

## Asterisk Dialplan Integration for Advanced IVR

VICIdial's Call Menu system covers 90% of IVR needs through the admin GUI alone. But for advanced scenarios — database lookups, API-driven routing, dynamic prompt generation — you need to drop into Asterisk's [dialplan](/glossary/dialplan/) and AGI.

### Custom Dialplan Extensions

As covered in the [Asterisk configuration guide](/blog/vicidial-asterisk-configuration/), custom dialplan logic goes in `extensions_custom.conf` — never in `extensions.conf`, which VICIdial overwrites.

**Example: After-hours voicemail with custom greeting**

```ini
; /etc/asterisk/extensions_custom.conf
[custom-afterhours-vm]
exten => s,1,Answer()
 same => n,Wait(1)
 same => n,Playback(afterhours-message)
 same => n,Playback(beep)
 same => n,VoiceMail(1000@default,u)
 same => n,Hangup()
```

Reference this from a Call Menu option by setting the route type to EXTENSION and the destination to `custom-afterhours-vm,s,1`.

**Example: Dynamic routing based on caller ID**

```ini
[custom-vip-check]
exten => s,1,Set(CALLER=${CALLERID(num)})
 same => n,AGI(vip-lookup.agi,${CALLER})
 same => n,GotoIf($["${VIP}" = "yes"]?vip:regular)
 same => n(vip),Goto(default,8300,1)     ; Route to VIP inbound group
 same => n(regular),Goto(default,8301,1) ; Route to standard queue
```

This checks the caller's number against a VIP list (via AGI script) and routes high-value callers to a priority queue — skipping the IVR entirely.

### AGI Scripts for Dynamic IVR

Asterisk Gateway Interface (AGI) lets you execute external scripts (Perl, PHP, Python) during call processing. VICIdial itself is built on AGI — all its call routing logic runs through AGI scripts in `/var/lib/asterisk/agi-bin/`.

**Example: Account number lookup AGI (Perl)**

```perl
#!/usr/bin/perl
# /var/lib/asterisk/agi-bin/account-lookup.agi
use strict;
use DBI;
use Asterisk::AGI;

my $AGI = new Asterisk::AGI;
my %input = $AGI->ReadParse();

# Get DTMF input (account number) from the IVR
my $account = $input{'arg_1'};

# Database lookup
my $dbh = DBI->connect("DBI:mysql:your_crm_db:localhost", "user", "pass");
my $sth = $dbh->prepare("SELECT department FROM accounts WHERE account_num = ?");
$sth->execute($account);
my ($dept) = $sth->fetchrow_array();

# Set channel variable for dialplan routing
if ($dept eq 'enterprise') {
    $AGI->set_variable('ROUTE_GROUP', 'ENTERPRISE_QUEUE');
} else {
    $AGI->set_variable('ROUTE_GROUP', 'STANDARD_QUEUE');
}

$dbh->disconnect();
```

Make it executable:

```bash
chmod 755 /var/lib/asterisk/agi-bin/account-lookup.agi
chown asterisk:asterisk /var/lib/asterisk/agi-bin/account-lookup.agi
```

Then reference it in the dialplan:

```ini
[custom-account-route]
exten => s,1,Answer()
 same => n,Playback(enter-account-number)
 same => n,Read(ACCOUNT,,10,,2,10)   ; Read up to 10 digits, 2 attempts, 10s timeout
 same => n,AGI(account-lookup.agi,${ACCOUNT})
 same => n,Goto(default,${ROUTE_GROUP},1)
```

### Integrating Custom Dialplan with VICIdial Call Menus

The cleanest way to blend VICIdial Call Menus with custom dialplan is:

1. Build the standard IVR tree using Call Menus in the VICIdial admin
2. For options that need custom logic, set the route type to EXTENSION
3. Point the extension to a context and extension in `extensions_custom.conf`
4. Have the custom dialplan do its work, then route back to a VICIdial inbound group using `Goto(default,INGROUP_DID,1)`

This keeps VICIdial's call tracking intact. If you route calls entirely through custom dialplan without going through VICIdial's normal inbound handling, the calls won't show up correctly in real-time reports or historical reporting.

---

## DID-to-IVR Mapping: Managing Multiple Numbers

Operations with dozens or hundreds of DIDs need a strategy for mapping numbers to IVR trees. You don't want to create a unique Call Menu for every phone number — that's unmaintainable.

### The Hub-and-Spoke Model

Use a small number of Call Menus as "hubs" and map multiple DIDs to the same menu:

```
DID 15551001000 → MAIN_IVR
DID 15551001001 → MAIN_IVR
DID 15551001002 → MAIN_IVR
DID 18001234567 → SALES_DIRECT (skips menu, goes straight to Sales queue)
DID 18007654321 → SUPPORT_DIRECT (skips menu, goes straight to Support queue)
```

The first three DIDs are general numbers that all hit the same IVR. The toll-free numbers are campaign-specific DIDs that skip the IVR entirely and route straight to the relevant queue. This is common for marketing campaigns where you want to track which ad generated the call — each campaign gets its own DID, all routed to the same queue, with the DID visible in VICIdial's reporting.

### DID-Specific Greetings with Shared Menus

Sometimes you want different greetings for different numbers but the same routing logic. Options:

**Option A: Separate Call Menus with identical options.** Create MAIN_IVR_BRAND_A and MAIN_IVR_BRAND_B with different prompts but the same DTMF destinations. Simple but creates duplication you need to maintain.

**Option B: Custom dialplan greeting + shared menu.** Route the DID to custom dialplan that plays a DID-specific greeting, then transfers to the shared Call Menu:

```ini
[custom-branded-greeting]
exten => 15551001000,1,Answer()
 same => n,Playback(greeting-brand-a)
 same => n,Goto(default,MAIN_IVR,1)

exten => 15551001001,1,Answer()
 same => n,Playback(greeting-brand-b)
 same => n,Goto(default,MAIN_IVR,1)
```

Option B is cleaner when you have many brands/campaigns sharing the same routing structure.

### Tracking DID Performance

Every DID in VICIdial can be associated with a campaign or custom field for reporting purposes. In the DID configuration:

- **DID Description:** Use a naming convention like `BRAND-A_GOOGLE_ADS_2026Q1` so you can identify the source in reports
- **Custom Fields:** VICIdial 3.14+ supports custom DID fields that flow into call records

Use VICIdial's inbound reporting (Reports > Inbound Report) filtered by DID to see which numbers drive the most calls, best conversion rates, and lowest abandonment. This data feeds directly into marketing ROI analysis.

---

## Overflow and Failover Routing

What happens when all agents are busy, the queue is full, or your primary site goes down? Without overflow routing, callers either wait forever or get disconnected. Both options destroy customer satisfaction.

### Queue Overflow Settings

In the Inbound Group configuration:

| Field | Purpose | Recommended Value |
|-------|---------|-------------------|
| Max Queue Size | Maximum callers waiting | 20-50 depending on agent count |
| Max Queue Time | Maximum wait time (seconds) | 300-600 (5-10 minutes) |
| Overflow Action | What to do when max is reached | CALLMENU or EXTENSION |
| Overflow Destination | Where to send overflow calls | OVERFLOW_IVR or voicemail |

**Overflow Call Menu example:**

```
Menu ID:    OVERFLOW_IVR
Prompt:     all-agents-busy|please-leave-message-or-call-back

Option 1:   VOICEMAIL → 2000 (leave voicemail)
Option 2:   CALLMENU → CALLBACK_MENU (schedule callback)
Option *:   CALLMENU → MAIN_IVR (return to main menu)
Timeout:    VOICEMAIL → 2000
```

### Inter-Site Failover

For operations with multiple VICIdial servers or sites, configure failover at the carrier level. If your primary site's trunk goes down, the carrier routes calls to a backup number that hits a secondary VICIdial installation with its own IVR.

In your carrier portal, set up failover forwarding:

```
Primary:  SIP trunk to 203.0.113.50 (main VICIdial)
Failover: SIP trunk to 203.0.113.60 (backup VICIdial)
Timeout:  5 seconds (if primary doesn't answer in 5s, try backup)
```

On the backup VICIdial, configure the same DIDs and IVR structure. The backup system can be a smaller deployment — it just needs to handle the overflow, not your full capacity.

### After-Hours Failover to External Numbers

For after-hours emergency support, route IVR options to external numbers:

In `extensions_custom.conf`:

```ini
[custom-oncall-forward]
exten => s,1,Dial(SIP/carrier-trunk/15559876543,30)   ; On-call manager's cell
 same => n,GotoIf($["${DIALSTATUS}" = "ANSWER"]?done)
 same => n,Dial(SIP/carrier-trunk/15559876544,30)      ; Backup on-call
 same => n(done),Hangup()
```

Set a Call Menu option to route to this extension for after-hours urgent calls.

---

## Testing Your IVR

Never deploy an IVR without testing every path. "I set it up and it should work" is how you lose calls in production.

### The Complete Test Checklist

For every Call Menu in your IVR tree:

1. **Call in and test every DTMF option.** Press 1, verify it routes correctly. Press 2. Press 3. All of them.
2. **Test timeout behavior.** Call in and don't press anything. Verify the prompt repeats the correct number of times, then routes to the timeout destination.
3. **Test invalid input.** Press 7 when only 1-3 are valid. Verify the invalid prompt plays and the caller gets another chance.
4. **Test the * and # keys** if they're assigned.
5. **Test time-based routing.** Temporarily change your Call Time definition to make the current time "after hours." Call in and verify the after-hours path works. Then change it back.
6. **Test from a cell phone.** DTMF behavior differs between VoIP phones and cellular networks. Some cell carriers send DTMF slightly differently — test to make sure key presses register.
7. **Test multi-level navigation.** Go through the full tree. Press 1 at level 1, then 2 at level 2, then 0 to go back, then 3 at level 1. Verify every path.
8. **Test overflow.** If you have queue overflow configured, fill the queue and verify overflow routing works.
9. **Verify reporting.** After test calls, check VICIdial's inbound reports. Make sure calls show the correct DID, inbound group, and call menu path.

### Monitoring IVR Performance

After deployment, track these metrics:

- **IVR completion rate:** What percentage of callers successfully navigate to an agent queue? Low completion means confusing menus.
- **Option distribution:** Which options do callers press most? If 80% press 1, consider making that the default (timeout) option.
- **Average IVR time:** How long do callers spend in the IVR before reaching a queue? Under 30 seconds is the target.
- **Abandonment by menu level:** Where in the IVR tree do callers hang up? That level has a problem — unclear prompt, too many options, or too long.

VICIdial's built-in reporting shows inbound call flow data. For detailed IVR path analysis, you can query the `vicidial_closer_log` and `vicidial_did_log` tables directly:

```sql
SELECT vid.did_pattern, vdl.did_route, COUNT(*) as calls,
       AVG(vcl.queue_seconds) as avg_queue_time
FROM vicidial_did_log vdl
JOIN vicidial_inbound_dids vid ON vdl.did_id = vid.did_id
JOIN vicidial_closer_log vcl ON vdl.uniqueid = vcl.uniqueid
WHERE vdl.call_date >= '2026-03-01'
GROUP BY vid.did_pattern, vdl.did_route
ORDER BY calls DESC;
```

> **Want an IVR That Actually Works on Day One?**
> ViciStack designs, builds, and tests your complete IVR system — call menus, routing logic, audio prompts, time-based rules, overflow handling — as part of every deployment. No guesswork. [Schedule Your Free Audit -->](/free-audit/)

---

## Real-World IVR Configurations

Theory is useful. Working configs are better. Here are three IVR architectures we deploy regularly.

### Configuration 1: Simple Auto-Attendant (5-Agent Shop)

One DID, one menu level, three options.

```
DID: 15551234567 → CALLMENU: MAIN_MENU

MAIN_MENU:
  Prompt:     "Thank you for calling [Company]. Press 1 for Sales,
               Press 2 for Support, or press 0 for the operator."
  Option 1:   IN_GROUP → SALES
  Option 2:   IN_GROUP → SUPPORT
  Option 0:   IN_GROUP → OPERATOR
  Timeout:    IN_GROUP → SALES
  Invalid:    CALLMENU → MAIN_MENU (replay)
  Repeat:     2
  Time Check: BUSINESS_HOURS

  After-Hours Prompt: "Our offices are currently closed. Please leave a
                       message after the tone and we will return your
                       call on the next business day."
  After-Hours Timeout: VOICEMAIL → 1000
```

Total Call Menus: 1. Total Inbound Groups: 3. Setup time: 20 minutes.

### Configuration 2: Bilingual Multi-Department (50-Agent Center)

Two languages, three departments, sub-menus for each.

```
DID: 18001234567 → CALLMENU: LANG_SELECT

LANG_SELECT:
  Prompt:     "For English, press 1. Para espanol, oprima el 2."
  Option 1:   CALLMENU → MAIN_EN
  Option 2:   CALLMENU → MAIN_ES
  Timeout:    CALLMENU → MAIN_EN (default to English)

MAIN_EN:
  Prompt:     "Press 1 for Sales... Press 2 for Customer Service...
               Press 3 for Billing... Press 0 for operator."
  Option 1:   CALLMENU → SALES_EN_MENU
  Option 2:   IN_GROUP → CUSTSERV_EN
  Option 3:   IN_GROUP → BILLING_EN
  Option 0:   IN_GROUP → OPERATOR_EN
  Option *:   CALLMENU → LANG_SELECT

SALES_EN_MENU:
  Prompt:     "For new accounts, press 1. For existing accounts, press 2.
               For partner inquiries, press 3."
  Option 1:   IN_GROUP → SALES_NEW_EN
  Option 2:   IN_GROUP → SALES_EXISTING_EN
  Option 3:   IN_GROUP → SALES_PARTNERS_EN
  Option 0:   CALLMENU → MAIN_EN (go back)

MAIN_ES:
  (Mirror of MAIN_EN with Spanish prompts and Spanish agent queues)

SALES_ES_MENU:
  (Mirror of SALES_EN_MENU with Spanish prompts and Spanish agent queues)
```

Total Call Menus: 6. Total Inbound Groups: 10. Setup time: 2-3 hours including audio recording.

### Configuration 3: Enterprise Multi-Site with Overflow (200+ Agents)

Multiple DIDs, time-based routing, overflow between sites, VIP detection.

```
DID: 18005551000 → Custom Dialplan (VIP check) → CALLMENU: MAIN_IVR
DID: 18005552000 → CALLMENU: MAIN_IVR (standard callers)
DID: 18005553000 → IN_GROUP: RETENTION (direct, no IVR)

Custom VIP Check:
  AGI: vip-lookup.agi
  If VIP → IN_GROUP: VIP_QUEUE (skip IVR, priority routing)
  If not VIP → CALLMENU: MAIN_IVR

MAIN_IVR:
  Time Check: MULTI_SITE_HOURS
  In-Hours: Standard 4-option menu → Department queues
  After-Hours: Overflow to offshore partner via external SIP trunk

Department Queues (each):
  Max Queue Time: 300 seconds
  Overflow: CALLMENU → OVERFLOW_MENU

OVERFLOW_MENU:
  Prompt:     "All agents are currently assisting other callers.
               Press 1 to leave a callback number.
               Press 2 to continue holding.
               Press 3 to leave a voicemail."
  Option 1:   AGI → callback-scheduler.agi (writes callback to VICIdial)
  Option 2:   IN_GROUP → SAME_QUEUE (re-enter queue)
  Option 3:   VOICEMAIL → department mailbox
```

Total Call Menus: 8+. Total Inbound Groups: 12+. AGI scripts: 2. Setup time: 1-2 days including testing.

---

## Common IVR Mistakes and How to Fix Them

After building hundreds of VICIdial IVR systems, these are the errors we see operators make most frequently.

### Mistake 1: Too Many Options Per Menu

"Press 1 for Sales, Press 2 for Support, Press 3 for Billing, Press 4 for Shipping, Press 5 for Returns, Press 6 for Technical Support, Press 7 for Hours and Directions, Press 8 for the Company Directory, Press 9 to hear this message again."

Nine options. The caller stopped listening at option 4. Research consistently shows callers retain a maximum of 4-5 options. Beyond that, they either mash 0 for the operator or hang up.

**Fix:** Limit each menu to 4-5 options maximum. Use sub-menus for additional routing. Group related options ("Press 2 for Customer Service" covers both Support and Billing, with a sub-menu to differentiate).

### Mistake 2: No Timeout Handler

The caller is driving and can't press a key immediately. The menu plays once, silence, call drops. Or the menu plays once, silence forever — caller eventually hangs up.

**Fix:** Always configure the `t` (timeout) option. Set Menu Repeat to 2-3 so the prompt plays multiple times. Route the final timeout to the most common destination (usually Sales or a general queue).

### Mistake 3: Wrong Audio Format

The prompt sounds like it's being played underwater. Or there's a pop at the beginning. Or the volume is dramatically different from the hold music.

**Fix:** Convert all audio to 8 kHz, 16-bit, mono WAV. Normalize levels across all prompt files. Test by calling in, not just by listening to the files on your computer — speakers and phone earpieces have very different frequency responses.

### Mistake 4: Not Testing After Changes

You change one Call Menu option and don't test. Two weeks later, a customer complains that "Press 2 for Support" has been going to the Sales queue since you made that change.

**Fix:** Test every path after every change. Log changes in a changelog. Better yet, have a second person verify the routing after you modify it.

### Mistake 5: Ignoring the Caller Experience

Your IVR works perfectly from a routing perspective — every call reaches the right queue. But callers hate it because the greeting is 45 seconds of legal disclaimers before they hear the first option, or the hold music is a 10-second loop that drives people insane after 2 minutes.

**Fix:** Front-load the useful information. "Press 1 for Sales, Press 2 for Support" should be the first thing callers hear, within 5 seconds. Move legal disclaimers to the queue hold message if required. Use real hold music, not a 10-second loop — and for the love of your callers, don't use a hold message that starts with "Your call is important to us" because nobody believes that anymore.

---

## Where ViciStack Fits In

VICIdial's IVR system is powerful, but power without proper configuration means callers routing into dead ends, agents getting calls meant for another department, and after-hours calls going nowhere. The difference between a working IVR and a good IVR is testing, refinement, and operational experience.

**ViciStack builds your IVR as part of every deployment.** We design the call flow with you, record or source professional audio, configure every Call Menu and Inbound Group, set up time-based routing and overflow, and test every DTMF path before your first live call. When your business changes — new department, new hours, new overflow rules — we update the IVR and re-test.

Your customers' first impression of your company is your IVR. Make it count.

[Get a free IVR design consultation -->](/free-audit/)

---

*This guide is maintained by the ViciStack team and updated as VICIdial, Asterisk, and best practices evolve. Last updated: March 2026.*

---

## Frequently Asked Questions

### How many Call Menu levels can I create in VICIdial?

There's no hard limit in VICIdial on Call Menu depth — you can chain as many as you want. But the practical limit is 2-3 levels. Every additional level adds 10-20 seconds to the caller's journey, and abandonment rates climb sharply after 45 seconds of IVR navigation. If your routing logic needs more than 3 levels, restructure: use broader top-level options, leverage skill-based routing within inbound groups, or use AGI scripts for intelligent routing that skips menu levels based on caller data.

### Can I use VICIdial's IVR for outbound call transfers?

Yes. When an agent transfers a call, the transfer destination can be a Call Menu. This is useful for scenarios like transferring a customer to a payment IVR, a survey system, or a different department's auto-attendant. In the campaign's Transfer settings, set up a preset with the Call Menu's extension. The transferred call enters the IVR just like an inbound call. Note that the original agent disconnects from the call at transfer — if you need the agent to stay on during the IVR (warm transfer into a menu), use a three-way call instead.

### Why aren't DTMF key presses being recognized in my IVR?

Nine times out of ten, this is a [DTMF](/glossary/dtmf/) mode mismatch. Your carrier sends DTMF one way and your trunk is configured for another. Check your carrier peer's dtmfmode setting — it should almost always be `rfc2833`. Run `asterisk -rx "sip show peer your-carrier"` and verify. If DTMF works on some calls but not others, the issue might be codec-related — DTMF in-band detection fails with compressed codecs like G.729. Switch to RFC 2833 on both your trunk and your carrier's configuration. Also check that Asterisk's `features.conf` isn't intercepting DTMF meant for the IVR — if `*` or `#` are configured as transfer keys, they'll get consumed before reaching the Call Menu.

### How do I set up different IVR greetings for different times of day?

Use Call Menu Time Checks combined with Call Time definitions. Create a Call Time in Admin > Call Times that defines your business hours schedule (including holidays). Then in your Call Menu, set the Menu Time Check to that Call Time ID. VICIdial gives you separate prompt and option fields for in-hours and after-hours. During business hours, callers hear the in-hours prompt and get routed to live agent queues. Outside business hours, they hear the after-hours prompt and route to voicemail, callback scheduling, or an after-hours team. You can get more granular by creating multiple Call Times (morning shift, evening shift, weekends) and nesting Call Menus that each check a different time window.

### Can I route calls differently based on the caller's phone number?

VICIdial supports ANI-based (caller ID) routing at the DID level. In the DID configuration, you can set up Filter rules that match caller ID patterns and route to different destinations. For more sophisticated logic — like VIP detection, account lookups, or geographic routing based on area code — use AGI scripts in `extensions_custom.conf`. The AGI script checks the caller's number against your database and sets a channel variable that the [dialplan](/glossary/dialplan/) uses for routing. This approach lets you skip the IVR entirely for known callers or route them to priority queues, significantly improving the experience for your best customers.

### What's the difference between a Call Menu and an Inbound Group?

A [Call Menu](/glossary/call-menu/) is the IVR itself — it plays prompts, accepts DTMF input, and routes calls based on key presses. An [Inbound Group](/glossary/inbound-group/) is a queue — it holds calls and delivers them to available agents. They serve fundamentally different purposes but connect in sequence: a DID routes to a Call Menu, the Call Menu routes to an Inbound Group, and the Inbound Group delivers to an agent. You can also skip the Call Menu entirely and route a DID straight to an Inbound Group if no IVR is needed for that number. Think of Call Menus as the decision layer and Inbound Groups as the delivery layer.

### How do I add a dial-by-name directory to my VICIdial IVR?

Asterisk has a built-in `Directory()` application that provides dial-by-name functionality. Add a custom dialplan extension in `extensions_custom.conf` that calls `Directory(default,default,bf)` — the `b` flag searches by first name and last name, and `f` reads the names using the first name first. Then point a Call Menu option (e.g., "Press 9 for our company directory") to that custom extension. For this to work, your Asterisk voicemail.conf must have entries for each user with their name recorded. Callers spell the name using the phone keypad, Asterisk matches it, and connects the call. This is best suited for smaller organizations — with 200+ employees, the directory becomes unwieldy and an operator queue is more practical.

### How do I monitor IVR performance and identify bottlenecks?

Start with VICIdial's built-in inbound reports. The Inbound Report shows call volume, handle time, and abandonment by inbound group — which tells you if specific queues (IVR destinations) have problems. For IVR-specific analysis, query the `vicidial_did_log` table to see which DIDs receive traffic and the `call_menu_log` entries to trace DTMF selections. Key metrics to watch: IVR completion rate (percentage of callers who reach a queue), option distribution (which keys are pressed most), and time-in-IVR (how long callers spend navigating menus before reaching a queue). If your IVR completion rate is below 85%, your menu is too complex, your prompts are unclear, or your timeout handling is broken. Check option distribution — if 60%+ of callers press the same key, consider making that option the default for timeout.

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-ivr-setup).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
