# VICIdial Asterisk Configuration: SIP, Codecs & NAT

*You finished the [VICIdial install](/blog/vicidial-setup-guide/), agents can log in, and you've got a SIP trunk registered. Then the real problems start. One-way audio on half your calls. Registration drops every 90 seconds. Your carrier complains about codec mismatches. Remote agents hear nothing. And every forum thread you find says "check your NAT settings" without telling you what that actually means.*

*This is the Asterisk configuration guide that VICIdial operators actually need.*

VICIdial is, at its core, a sophisticated wrapper around [Asterisk](/glossary/asterisk/) — the open-source PBX that handles every call, every transfer, every recording, every piece of audio that flows through your contact center. The VICIdial admin GUI manages campaigns, agents, and lists. But underneath all of that, Asterisk is doing the actual telephony work. And Asterisk's configuration determines whether your calls sound crystal clear or like they're being routed through a tin can on a string.

We've configured Asterisk on hundreds of VICIdial deployments — from 5-agent shops to 500-agent [clusters](/blog/vicidial-cluster-guide/) processing over a million calls per day. This guide covers everything: channel drivers, codec selection, NAT traversal, trunk configuration, dialplan structure, security hardening, module management, and the diagnostic commands that will save you at 2 AM when calls stop connecting.

---

## Channel Drivers: chan_sip vs PJSIP (And Why VICIdial Uses Both)

Asterisk has two SIP channel drivers, and understanding which one VICIdial uses — and when — is foundational to every configuration decision that follows.

### chan_sip: The Legacy Default

VICIdial has used `chan_sip` since the beginning. Every SIP trunk, every phone registration, every [carrier](/glossary/carrier/) connection in a standard VICIdial installation runs through `chan_sip`. The configuration lives in `/etc/asterisk/sip.conf`, and VICIdial's installer generates most of it automatically.

Why does VICIdial still default to chan_sip? Because it works. It has been battle-tested across 14,000+ VICIdial installations for over 15 years. Matt Florell and the VICIdial development team know its behavior intimately — every edge case, every quirk, every workaround. Switching the entire platform to PJSIP would mean regression-testing every call flow, every transfer scenario, every conference bridge interaction. That's a massive engineering effort for a project maintained by a small team.

But here's the critical timeline: **Asterisk 21 removes chan_sip entirely.** It's gone. Not deprecated — removed from the source tree. Asterisk 20 is the last LTS release that includes it, with full support through 2026 and security fixes through 2027. If you're deploying VICIdial today on Asterisk 18 (the recommended version), you have runway. But this is not a problem you can defer indefinitely.

### PJSIP: Required for WebRTC

[PJSIP](/glossary/pjsip/) is Asterisk's modern SIP stack. It runs as `res_pjsip` and its configuration lives in `/etc/asterisk/pjsip.conf`. In VICIdial, PJSIP serves one primary purpose: **[WebRTC](/glossary/webrtc/) support for ViciPhone.**

ViciPhone — VICIdial's browser-based softphone — requires WebSocket transport (WSS) for [SIP](/glossary/sip/) signaling and DTLS-SRTP for encrypted media. chan_sip doesn't support either. So VICIdial runs both channel drivers simultaneously: chan_sip handles trunks and traditional SIP phones, while PJSIP handles WebRTC connections from ViciPhone.

This dual-driver architecture means you need to be careful about port conflicts. By default:

- **chan_sip** binds to UDP port 5060
- **PJSIP** binds to UDP port 5060 *and* TCP port 5060

If both try to bind to the same port, one fails silently and you spend an afternoon wondering why registrations work intermittently. The fix:

```ini
; In sip.conf — keep chan_sip on 5060
[general]
bindport=5060

; In pjsip.conf — move PJSIP transports to different ports
[transport-wss]
type=transport
protocol=wss
bind=0.0.0.0:8089
```

VICIdial's installer handles this correctly on fresh installs with WebRTC enabled. But if you're retrofitting WebRTC onto an existing installation, verify that the ports don't collide. Run `asterisk -rx "sip show settings"` and `asterisk -rx "pjsip show transports"` to confirm what's actually bound.

### The Coexistence Configuration

Here's what a properly configured dual-driver VICIdial system looks like at the transport level:

```
chan_sip (sip.conf):     UDP/5060  — SIP trunks, hardware phones
PJSIP (pjsip.conf):     WSS/8089  — ViciPhone WebRTC clients
                         UDP/5062  — (optional) PJSIP SIP phones
```

The key rule: **never register the same endpoint on both drivers.** A SIP trunk goes in sip.conf OR pjsip.conf, not both. A phone registers to chan_sip OR PJSIP. Mixing creates race conditions where both drivers try to handle the same INVITE and calls fail unpredictably.

> **WebRTC Configuration Giving You Headaches?**
> ViciStack deploys ViciPhone with PJSIP, STUN/TURN, and certificate management pre-configured. No port conflicts, no certificate debugging. [Get Your Free Assessment →](/free-audit/)

---

## Codec Configuration: What to Use and When

[Codecs](/glossary/codec/) determine how voice audio gets compressed for transmission. The wrong codec choice means either wasted bandwidth, degraded audio quality, or both. VICIdial supports every codec Asterisk supports, but in practice you're choosing between four options.

### G.711 ulaw (PCMU) — The North American Default

G.711 ulaw is uncompressed audio at 64 kbps per channel. It's the default for VICIdial installations in North America and the codec your SIP [trunks](/glossary/trunk/) almost certainly prefer.

**Use ulaw when:**
- Your trunks are in North America (carriers optimize for ulaw)
- Bandwidth is not a constraint (LAN or high-bandwidth WAN)
- You want zero transcoding overhead on your Asterisk server
- Call quality is paramount

**Configuration in sip.conf:**

```ini
[general]
disallow=all
allow=ulaw
```

The `disallow=all` line is critical. Without it, Asterisk offers every loaded codec during SDP negotiation, and you might end up transcoding G.729 to ulaw on every call — burning CPU for no benefit.

### G.711 alaw (PCMA) — The International Default

G.711 alaw is the same 64 kbps uncompressed audio, but using the A-law companding algorithm instead of u-law. It's the standard in Europe, Asia, and most countries outside North America.

**Use alaw when:**
- Your trunks terminate in Europe, Asia, or other ITU-T regions
- Your carrier explicitly requires alaw

In sip.conf, just swap the codec:

```ini
[general]
disallow=all
allow=alaw
```

If you're running a global operation with trunks in multiple regions, allow both:

```ini
[general]
disallow=all
allow=ulaw
allow=alaw
```

Asterisk will negotiate the first matching codec. Put your preferred codec first.

### G.729 — For Bandwidth-Constrained Links

G.729 compresses audio to 8 kbps — one-eighth the bandwidth of G.711. The tradeoff is measurable quality loss (especially on conference bridges and transfers where audio gets re-encoded) and CPU overhead for transcoding.

**Use G.729 when:**
- Remote agents connect over low-bandwidth links
- You're paying per-megabyte for transit (some international carriers)
- Inter-server links in a [cluster](/glossary/vicidial-cluster/) cross a WAN

**Critical caveat:** G.729 is a licensed codec. Asterisk's built-in G.729 implementation (`codec_g729`) requires a per-channel license from Digium (now Sangoma). However, the open-source `bcg729` library provides a free G.729 implementation that works with Asterisk. On ViciBox/AlmaLinux:

```bash
yum install bcg729 asterisk-codec-g729
```

Verify it loaded:

```bash
asterisk -rx "core show codecs" | grep g729
```

**Do not mix G.729 and G.711 on the same call path without understanding the transcoding cost.** Every call that enters your system on G.729 from a remote agent and exits on G.711 to a carrier (or vice versa) requires real-time transcoding. At 20 agents, this is negligible. At 200 agents with 5:1 dial ratios, that's potentially 1,000 simultaneous transcoding operations — enough to measurably impact your telephony server's capacity.

### Opus — The WebRTC Codec

Opus is the modern variable-bitrate codec used by WebRTC. When agents connect via ViciPhone, the browser-to-Asterisk leg uses Opus. Asterisk then transcodes to whatever codec your trunks use (almost always G.711).

You don't configure Opus in sip.conf — it's handled in the PJSIP WebRTC endpoint configuration. VICIdial's WebRTC installer sets this up automatically, but here's what the relevant pjsip.conf section looks like:

```ini
[webrtc-endpoint](!)
type=endpoint
transport=transport-wss
context=default
disallow=all
allow=opus
allow=ulaw
dtls_auto_generate_cert=yes
webrtc=yes
```

The `allow=opus` line must come before `allow=ulaw` so Opus is preferred for the WebRTC leg. Asterisk handles the Opus-to-ulaw transcoding internally for the carrier leg.

**Performance note:** Opus transcoding is heavier than G.711 passthrough. Each WebRTC agent on ViciPhone adds roughly 15-20% more CPU load per channel compared to a hardware SIP phone using ulaw. Plan your [telephony server capacity](/blog/vicidial-cluster-guide/) accordingly — if your servers handle 20 agents on SIP phones, expect 15-17 on ViciPhone.

### Codec Priority in Carrier Trunks

When you configure a carrier in VICIdial Admin → Carriers, the codec priority is set in the trunk registration string or the sip.conf peer definition. Here's a real-world example for a SIP trunk:

```ini
[carrier-voip-trunk]
type=peer
host=sip.yourcarrier.com
port=5060
username=your_account
secret=your_password
fromuser=your_account
fromdomain=sip.yourcarrier.com
insecure=port,invite
qualify=yes
disallow=all
allow=ulaw
allow=alaw
dtmfmode=rfc2833
nat=force_rport,comedia
```

The `disallow=all` followed by `allow=ulaw` and `allow=alaw` tells Asterisk to offer ulaw first, fall back to alaw if the carrier doesn't support ulaw. Most North American VoIP carriers support both, but some international routes only do alaw.

**Always match your trunk codec config to your carrier's documentation.** Codec mismatches cause either failed calls (if no common codec exists) or unnecessary transcoding (if Asterisk accepts a codec and then has to transcode for the agent leg).

---

## NAT Traversal: The #1 Source of VICIdial Audio Problems

If you've ever had "one-way audio" on a VICIdial system, you've had a NAT problem. If you've had agents who can hear the customer but the customer can't hear them, that's NAT. If calls connect fine for 30 seconds and then go silent, that's probably NAT. [NAT traversal](/glossary/nat-traversal/) issues account for more VICIdial support tickets than every other Asterisk configuration problem combined.

Here's why: [SIP](/glossary/sip/) was designed in the 1990s when every device had a public IP address. The protocol embeds IP addresses directly into signaling messages (SDP bodies). When a server behind a NAT sends a SIP INVITE, it advertises its *private* IP (10.x.x.x, 172.16.x.x, 192.168.x.x) in the SDP. The remote side tries to send [RTP](/glossary/rtp/) audio to that private IP, the packets hit the NAT boundary, and the audio disappears into the void.

### sip.conf NAT Settings (chan_sip)

The three critical settings in `sip.conf [general]`:

```ini
[general]
externip=203.0.113.50          ; Your server's PUBLIC IP address
localnet=10.0.0.0/8            ; Private network ranges — traffic here stays internal
localnet=172.16.0.0/12
localnet=192.168.0.0/16

nat=force_rport,comedia         ; Force symmetric RTP for NATed clients
```

**`externip`** tells Asterisk what public IP to substitute into SDP bodies when communicating with endpoints outside the `localnet` ranges. Without this, every call to/from a carrier on the public internet will have audio issues.

**`localnet`** defines which networks are "inside" the NAT. Asterisk uses the original private IPs for traffic within these ranges (server-to-server in a cluster) and substitutes `externip` for everything else.

**`nat=force_rport,comedia`** is the nuclear option for NAT — and it's the right one for VICIdial. `force_rport` makes Asterisk ignore the port in the Via header and use the port the packet actually arrived from. `comedia` makes Asterisk send RTP to the address and port where it *receives* RTP from, rather than where the SDP says to send it. Together, these settings handle 95% of NAT scenarios.

**If your server has a dynamic public IP** (cloud instances, some colo providers), use `externhost` instead:

```ini
externhost=myserver.example.com
externrefresh=60
```

Asterisk will resolve the hostname every 60 seconds to get the current IP.

### Per-Peer NAT Settings

For SIP trunks and phones that are behind NAT (which is almost all of them in 2026), set NAT handling on the peer definition:

```ini
[my-carrier]
type=peer
nat=force_rport,comedia
qualify=yes
qualifyfreq=30
```

The `qualify=yes` with `qualifyfreq=30` sends OPTIONS packets every 30 seconds. This serves two purposes: it tells you if the peer is reachable (status shows in `sip show peers`), and — critically — it keeps the NAT pinhole open. Without qualify, many NAT gateways close the UDP mapping after 60-90 seconds of inactivity, causing registration drops and call failures.

### STUN/TURN for Remote Agents

When agents connect from home networks — behind consumer routers, hotel WiFi, corporate firewalls — the NAT situation gets significantly worse. Double NAT, symmetric NAT, and restrictive firewalls can defeat `force_rport,comedia` because the RTP port mapping changes unpredictably.

This is where STUN and TURN servers become essential, particularly for [WebRTC](/glossary/webrtc/) connections via ViciPhone.

**STUN** (Session Traversal Utilities for NAT) helps clients discover their public IP and port mapping. It works for most consumer NAT scenarios but fails with symmetric NAT.

**TURN** (Traversal Using Relays around NAT) relays all media through a server with a public IP. It works in every scenario but adds latency and bandwidth cost because all audio routes through the TURN server.

For ViciPhone/WebRTC, configure STUN/TURN in the VICIdial admin under System Settings → WebRTC:

```
STUN server: stun:stun.l.google.com:19302
TURN server: turn:your-turn-server.example.com:3478
TURN username: vicidial
TURN credential: your-secure-password
```

**Deploy your own TURN server.** Don't rely on public STUN servers for production traffic. Install `coturn` on a dedicated VM with a public IP:

```bash
yum install coturn
```

Minimal `/etc/turnserver.conf`:

```ini
listening-port=3478
tls-listening-port=5349
relay-ip=203.0.113.60
external-ip=203.0.113.60
realm=yourdomain.com
server-name=yourdomain.com
lt-cred-mech
user=vicidial:your-secure-password
total-quota=300
stale-nonce=600
no-multicast-peers
```

The TURN server needs bandwidth — every active WebRTC call relayed through TURN uses roughly 100-200 kbps in each direction. For 50 remote WebRTC agents, provision at least 20 Mbps on the TURN server.

### Cluster NAT: The Private Network Shortcut

In a [VICIdial cluster](/blog/vicidial-cluster-guide/), inter-server communication (IAX2 between dialers, MySQL connections to the database server) should happen on a **private network** with no NAT involved. Use dual NICs:

- **NIC 1 (public):** Carriers, agents, WebRTC — gets the `externip` treatment
- **NIC 2 (private):** Inter-server traffic on 10.x.x.x — stays in `localnet`, no NAT rewriting

This eliminates an entire category of audio issues. Calls that transfer between dialers in a cluster use IAX2 over the private network, and [IAX2](/glossary/iax2/) handles NAT far better than SIP because it multiplexes signaling and media on a single port.

> **One-Way Audio Is a Solved Problem.**
> Every ViciStack deployment ships with externip, localnet, STUN/TURN, and dual-NIC configuration verified before your first call. [See How We Do It →](/pricing/)

---

## Trunk Configuration: From VICIdial Admin to sip.conf

Configuring SIP trunks in VICIdial involves two layers: the VICIdial admin GUI (Carriers page) and the underlying Asterisk configuration files.

### Adding a Carrier in VICIdial Admin

Navigate to Admin → Carriers and create a new entry. The critical fields:

| Field | Purpose | Example Value |
|-------|---------|---------------|
| Carrier ID | Internal identifier | VOIP_TRUNK_1 |
| Carrier Name | Descriptive name | Telnyx Primary |
| Registration String | SIP registration line for sip.conf | `register => user:pass@sip.telnyx.com` |
| Account Entry | Full peer definition for sip.conf | (see below) |
| Protocol | SIP, IAX2, or CUSTOM | SIP |
| Global String | Dialplan variables | (usually empty) |

The **Account Entry** field is where you paste the actual sip.conf peer definition:

```ini
[telnyx-trunk]
type=peer
host=sip.telnyx.com
port=5060
username=+15551234567
secret=your_sip_password
fromuser=+15551234567
fromdomain=sip.telnyx.com
insecure=port,invite
qualify=yes
qualifyfreq=30
disallow=all
allow=ulaw
dtmfmode=rfc2833
nat=force_rport,comedia
context=trunkinbound
```

When you save the carrier, VICIdial writes this configuration to the Asterisk configuration and triggers a SIP reload. You can verify the trunk registered with:

```bash
asterisk -rx "sip show registry"
```

And check the peer status:

```bash
asterisk -rx "sip show peers" | grep telnyx
```

### Registration Strings

Some carriers require SIP registration (your server sends REGISTER to the carrier). Others use IP-based authentication (carrier whitelists your IP, no registration needed).

**Registration-based carrier:**

```
register => username:password@sip.carrier.com
```

**IP-authenticated carrier:** Leave the registration string blank. Just define the peer with `host=sip.carrier.com` and ensure your server's IP is whitelisted on the carrier's portal.

**Registration with specific Contact header** (some carriers require this):

```
register => username:password@sip.carrier.com/username
```

### Codec Priority in Trunk Definitions

Your trunk's codec priority should match what your carrier documents. Most North American VoIP carriers prefer this order:

```ini
disallow=all
allow=ulaw
allow=alaw
```

If your carrier supports G.729 and you want to conserve bandwidth on the carrier leg:

```ini
disallow=all
allow=g729
allow=ulaw
```

**Warning:** Using G.729 on carrier trunks when your agents are on G.711 (or Opus via WebRTC) means every single call gets transcoded. At scale, this CPU cost is significant. Only use G.729 on trunks if bandwidth savings justify it.

### DTMF Mode

[DTMF](/glossary/dtmf/) (touch-tone signals) configuration is a surprisingly common source of issues. If your IVR transfers aren't working, or customers can't navigate phone trees after transfer, DTMF mode mismatch is the likely culprit.

```ini
dtmfmode=rfc2833    ; Most common, most compatible
```

Options:
- **rfc2833** — DTMF sent as RTP events (out-of-band). This is the standard and what 95% of carriers expect.
- **info** — DTMF sent as SIP INFO messages. Some legacy carriers use this.
- **inband** — DTMF tones in the audio stream. Unreliable with compressed codecs. Avoid.
- **auto** — Asterisk tries to detect. Sounds smart, works poorly.

Always set `dtmfmode=rfc2833` unless your carrier specifically documents otherwise.

---

## Dialplan Internals: What VICIdial Generates in extensions.conf

VICIdial auto-generates its Asterisk [dialplan](/glossary/dialplan/) — you should never manually edit `extensions.conf` on a VICIdial system. But understanding the structure is essential for troubleshooting call flow issues.

### The Core Contexts

VICIdial creates several contexts in `extensions.conf`. The most important:

**`[default]`** — The catchall context. Inbound calls from carriers land here. VICIdial routes them based on DID matching.

**`[trunkinbound]`** — Where carrier trunks deliver inbound calls. This context is referenced in the carrier peer definition (`context=trunkinbound`).

**`[vicidial-auto]`** — The outbound auto-dial context. When the predictive dialer places a call, it originates through this context. The dialplan here handles answer detection, [AMD](/glossary/amd/) processing, and routing to agents.

**`[vicidial-auto-agent]`** — After a call is answered (and passes AMD if enabled), it gets routed here to connect to the waiting agent via a MeetMe/[ConfBridge](/glossary/confbridge/) conference.

### How an Outbound Call Flows

1. `AST_VDauto_dial.pl` identifies a lead to call and uses AMI to originate a call
2. Asterisk places the call through the carrier trunk
3. The call enters the `[vicidial-auto]` context
4. If AMD is enabled, the call hits the AMD application
5. On answer/human detection, the call gets routed to [AGI](/glossary/agi/) (`agi-VDAD_ALL_outbound.agi`)
6. The AGI script creates a conference room and bridges the agent to the answered call
7. The agent's phone (already in a "listening" conference) hears the call connect

### Custom Dialplan Additions

If you need custom dialplan logic (after-hours routing, custom IVR flows, special transfer destinations), add it to `extensions_custom.conf` — not `extensions.conf`. VICIdial overwrites `extensions.conf` on reload, but `extensions_custom.conf` is included via `#include` and survives updates.

```ini
; extensions_custom.conf
[custom-afterhours]
exten => s,1,Answer()
exten => s,n,Playback(after-hours-message)
exten => s,n,Voicemail(1000@default)
exten => s,n,Hangup()
```

---

## Security Hardening: Protecting Asterisk on the Public Internet

A VICIdial server with SIP ports open to the internet is a target. SIP scanning bots hit every public IP on port 5060 continuously. Without hardening, you'll see registration attempts from random IPs within hours of deployment, and a successful breach means toll fraud — someone making international calls on your trunks at your expense.

### fail2ban for SIP

fail2ban monitors Asterisk logs for failed authentication attempts and blocks offending IPs via iptables. ViciBox includes a pre-configured fail2ban jail for Asterisk, but if you're on a custom install:

```ini
; /etc/fail2ban/jail.local
[asterisk]
enabled = true
port = 5060,5061,8089
protocol = all
filter = asterisk
logpath = /var/log/asterisk/messages
maxretry = 3
findtime = 600
bantime = 86400
action = iptables-allports[name=asterisk, protocol=all]
```

This bans any IP that fails 3 authentication attempts within 10 minutes, for 24 hours. For production systems, consider `bantime = -1` (permanent) and a whitelist for your known IPs.

Verify fail2ban is working:

```bash
fail2ban-client status asterisk
```

### iptables Rules for SIP

Don't rely solely on fail2ban. Use iptables to restrict SIP access to known IP ranges:

```bash
# Allow SIP from your carrier's IP range
iptables -A INPUT -p udp -s 203.0.113.0/24 --dport 5060 -j ACCEPT

# Allow SIP from your office/agent network
iptables -A INPUT -p udp -s 198.51.100.0/24 --dport 5060 -j ACCEPT

# Allow WebSocket from anywhere (for ViciPhone — agents need this)
iptables -A INPUT -p tcp --dport 8089 -j ACCEPT

# Allow RTP from anywhere (audio must flow)
iptables -A INPUT -p udp --dport 10000:20000 -j ACCEPT

# Drop all other SIP
iptables -A INPUT -p udp --dport 5060 -j DROP
iptables -A INPUT -p tcp --dport 5060 -j DROP
```

**RTP ports must be open to all sources.** You can't restrict RTP to known IPs because carriers route media from different IPs than signaling, and NATed agents send RTP from unpredictable source addresses. The SIP layer is where you lock things down.

### SIP Registration ACLs

In `sip.conf`, use `permit` and `deny` to restrict which IPs can register:

```ini
[general]
; Only allow registration from these networks
contactdeny=0.0.0.0/0.0.0.0
contactpermit=10.0.0.0/255.0.0.0
contactpermit=203.0.113.0/255.255.255.0
```

For individual peers, add ACLs:

```ini
[my-carrier]
type=peer
permit=203.0.113.50/32
deny=0.0.0.0/0.0.0.0
```

### TLS/SRTP for SIP

If your carrier supports SIP over TLS (and most modern carriers do), enable it:

```ini
[general]
tlsenable=yes
tlsbindaddr=0.0.0.0:5061
tlscertfile=/etc/asterisk/keys/asterisk.pem
tlscafile=/etc/asterisk/keys/ca.crt
tlscipher=ALL
tlsclientmethod=tlsv1_2
```

For the carrier peer:

```ini
[secure-carrier]
type=peer
transport=tls
encryption=yes     ; Require SRTP
```

**Note:** TLS encrypts SIP signaling. SRTP (`encryption=yes`) encrypts the audio. For full encryption, you need both. But SRTP adds CPU overhead — roughly 10-15% per channel. Factor this into your server capacity planning.

---

## Asterisk Module Management

Asterisk loads dozens of modules at startup. VICIdial only needs a subset of them, and unnecessary modules waste memory and can cause conflicts. The module configuration lives in `/etc/asterisk/modules.conf`.

### Essential Modules for VICIdial

```ini
[modules]
autoload=yes

; Core telephony
load => res_rtp_asterisk.so
load => chan_sip.so
load => res_pjsip.so           ; Only if using WebRTC
load => app_meetme.so           ; Legacy conferencing
load => app_confbridge.so       ; Modern conferencing (recommended)
load => res_agi.so              ; AGI — VICIdial's brain
load => app_dial.so
load => app_playback.so
load => app_record.so
load => res_musiconhold.so

; Recording
load => app_mixmonitor.so       ; MixMonitor for call recording
load => func_cdr.so

; AMD
load => app_amd.so              ; If using AMD
```

### Modules to Disable

```ini
; Disable unused channel drivers
noload => chan_alsa.so
noload => chan_console.so
noload => chan_oss.so
noload => chan_mgcp.so
noload => chan_skinny.so
noload => chan_unistim.so

; Disable unused protocols
noload => res_xmpp.so
noload => res_http_websocket.so  ; Only noload if NOT using WebRTC

; Disable database backends you're not using
noload => res_config_pgsql.so
noload => cdr_pgsql.so
noload => res_config_sqlite3.so
```

**The ConfBridge migration matters here.** If you're on Asterisk 18 and have enabled ConfBridge (which you should for new deployments), you still need `app_meetme.so` loaded because VICIdial's code references both. Set `mixing_interval=20` in `confbridge.conf` as noted in the [cluster guide](/blog/vicidial-cluster-guide/) to prevent RTP packet ordering issues.

To check what's currently loaded:

```bash
asterisk -rx "module show" | wc -l      # Total module count
asterisk -rx "module show like chan_"    # Channel drivers
asterisk -rx "module show like res_"    # Resource modules
```

---

## Monitoring and Diagnostics

When calls stop working at 2 AM, you need to know exactly which Asterisk commands give you answers fast.

### The Essential CLI Commands

Connect to the Asterisk CLI:

```bash
asterisk -rvvv     # Read-only, verbosity level 3
```

The verbosity level controls how much real-time output you see. Level 3 (`vvv`) shows call setup and teardown. Level 5 (`vvvvv`) shows SIP message details. Level 10+ shows everything including RTP — useful for deep debugging but extremely noisy on a busy system.

**Check SIP trunk status:**

```bash
asterisk -rx "sip show registry"       # Registration status
asterisk -rx "sip show peers"          # Peer status and latency
asterisk -rx "sip show peer telnyx-trunk"  # Detailed peer info
```

**Active call monitoring:**

```bash
asterisk -rx "core show channels"      # All active channels
asterisk -rx "core show channels concise"  # Machine-parseable format
asterisk -rx "core show channel SIP/xxx"   # Specific channel details
```

**Conference monitoring** (essential for VICIdial — every call is a conference):

```bash
asterisk -rx "meetme list"             # All active conferences
asterisk -rx "confbridge list"         # ConfBridge conferences
```

**Codec negotiation debugging:**

```bash
asterisk -rx "sip show channel SIP/trunk-00000001"  # Shows negotiated codec
```

### AMI Events for Monitoring

The [Asterisk Manager Interface (AMI)](/glossary/asterisk-manager-interface/) is how VICIdial communicates with Asterisk programmatically. It's also useful for external monitoring. Connect to AMI on port 5038:

```bash
telnet localhost 5038
```

Login and listen for events:

```
Action: Login
Username: cron
Secret: your_ami_password

Action: Events
EventMask: call
```

Key events to monitor:

| Event | Meaning |
|-------|---------|
| `Registry` | SIP registration state change (trunk up/down) |
| `PeerStatus` | SIP peer reachable/unreachable |
| `Newchannel` | New call created |
| `Hangup` | Call ended (includes cause code) |
| `AGIExec` | AGI script execution (VICIdial call routing) |

### SIP Debug Mode

When you need to see the actual SIP messages (INVITE, 200 OK, BYE, etc.):

```bash
asterisk -rx "sip set debug on"        # All SIP traffic (very noisy)
asterisk -rx "sip set debug peer telnyx-trunk"  # Just one peer
asterisk -rx "sip set debug off"       # Turn it off when done
```

**Never leave SIP debug on in production.** It writes massive amounts of data to the Asterisk log and can fill your disk in hours on a busy system.

---

## Common Issues and Fixes

After hundreds of deployments and thousands of forum threads, these are the Asterisk configuration problems VICIdial operators actually hit.

### One-Way Audio

**Symptom:** Agent hears customer, customer doesn't hear agent (or vice versa).

**Cause:** NAT. 95% of the time.

**Fix:**
1. Verify `externip` is set correctly in sip.conf
2. Verify `localnet` includes all your private networks
3. Set `nat=force_rport,comedia` on both `[general]` and the carrier peer
4. Ensure UDP ports 10000-20000 are open in iptables (both directions)
5. For remote agents: deploy STUN/TURN

### Registration Failures

**Symptom:** `sip show registry` shows "Rejected" or "Request Sent" that never resolves.

**Troubleshooting:**
```bash
asterisk -rx "sip set debug peer carrier-name"
# Watch for 401/403 responses — credentials wrong
# Watch for 408 Timeout — network/firewall issue
# Watch for 503 Service Unavailable — carrier side problem
```

Common causes:
- Wrong username/password in the registration string
- Carrier requires `fromuser` or `fromdomain` that doesn't match
- Firewall blocking outbound UDP 5060
- DNS resolution failure (check with `dig sip.carrier.com`)

### Certificate Issues (WebRTC)

**Symptom:** ViciPhone connects but no audio, or connection fails entirely.

WebRTC requires valid TLS certificates. Self-signed certs are rejected by browsers. Use Let's Encrypt:

```bash
certbot certonly --standalone -d vicidial.yourdomain.com
```

Then create the combined PEM that Asterisk needs:

```bash
cat /etc/letsencrypt/live/vicidial.yourdomain.com/fullchain.pem \
    /etc/letsencrypt/live/vicidial.yourdomain.com/privkey.pem \
    > /etc/asterisk/keys/asterisk.pem
```

Set up a cron job to renew and restart Asterisk:

```bash
0 3 * * 1 certbot renew --quiet && cat /etc/letsencrypt/live/vicidial.yourdomain.com/fullchain.pem /etc/letsencrypt/live/vicidial.yourdomain.com/privkey.pem > /etc/asterisk/keys/asterisk.pem && asterisk -rx "module reload res_pjsip.so"
```

### Audio Quality Degradation

**Symptom:** Calls sound robotic, choppy, or have echo.

**Checklist:**
1. Check for packet loss: `asterisk -rx "rtp set debug on"` — look for gaps in sequence numbers
2. Check for codec transcoding you didn't intend: `sip show channel` on an active call
3. Check server CPU: if Asterisk is above 80% CPU utilization, audio quality drops
4. Check for jitter buffer settings: VICIdial default is usually fine, but for high-jitter links:

```ini
[general]
jbenable=yes
jbmaxsize=200
jbimpl=adaptive
```

### "All Circuits Busy" on Outbound Calls

**Symptom:** Dialer shows trunk errors, agents get no calls.

**Check trunk capacity:**
```bash
asterisk -rx "core show channels" | grep -c "SIP/carrier"
```

If this number equals your trunk's channel limit, you need more capacity. VICIdial's [predictive dialer](/glossary/predictive-dialing/) at a 5:1 ratio with 50 agents tries to maintain 250 simultaneous outbound channels. Make sure your SIP trunk supports that volume.

---

## astguiclient.conf: The Bridge Between VICIdial and Asterisk

Every VICIdial server has `/etc/astguiclient.conf` — the configuration file that tells VICIdial how to talk to Asterisk. Key Asterisk-related settings:

```ini
# AMI credentials — must match manager.conf
VARserver_ip=10.0.0.1
VARDB_server=10.0.0.2
VARactionURL=
VARfastagi_log_min_servers=3
VARfastagi_log_server_ip=10.0.0.1
VARactive_keepalives=123456789
```

The `VARserver_ip` is the IP this server uses for Asterisk operations. In a cluster, this must be the IP in the `servers` table in the VICIdial database. Get it wrong and calls originate but can't be tracked back to the right server, causing orphaned channels and ghost calls.

---

## Where ViciStack Fits In

This guide gives you everything you need to configure Asterisk correctly in VICIdial. But knowing what to configure and keeping it configured correctly across updates, certificate renewals, carrier changes, and capacity scaling are two very different problems.

**ViciStack ships every deployment with Asterisk pre-configured**: correct NAT settings for your network topology, carrier trunks tested and verified, codec optimization for your specific carriers, fail2ban and iptables hardened, WebRTC with valid certificates and TURN servers, and ConfBridge ready for the Asterisk 21 migration.

When your Asterisk configuration is right, calls connect cleanly, audio is clear, and your agents hear a human voice instead of dead air. When it's wrong, you're debugging SIP traces at 2 AM.

We'd rather you not do that.

[Get a free Asterisk configuration audit →](/free-audit/)

---

*This guide is maintained by the ViciStack team and updated as VICIdial, Asterisk, and best practices evolve. Last updated: March 2026.*

---

## Frequently Asked Questions

### Should I switch from chan_sip to PJSIP for all my trunks?

Not yet. VICIdial's codebase is still primarily built around chan_sip for trunk and phone registration. Switching all trunks to PJSIP requires testing every call flow — outbound, inbound, transfers, three-way calls, and conferencing. The VICIdial development team is working on full PJSIP migration, but for production systems in 2026, the recommended approach is chan_sip for trunks and phones, PJSIP only for WebRTC/ViciPhone endpoints. Start testing PJSIP trunks in a lab environment so you're ready when the transition becomes necessary.

### What's the best codec for VICIdial?

G.711 ulaw for North American operations, G.711 alaw for international. These provide the best call quality with zero transcoding overhead. Only use G.729 when bandwidth constraints genuinely require it (remote offices on limited WAN links, international trunks with per-MB billing). For WebRTC agents on ViciPhone, Opus is used automatically and transcoded to G.711 for the carrier leg.

### Why do my remote agents keep getting one-way audio?

Remote agents behind consumer NAT are the hardest audio scenario. The fix is layered: (1) `externip` and `localnet` correctly set on your Asterisk server, (2) `nat=force_rport,comedia` on both general and peer sections, (3) RTP ports 10000-20000 open, (4) for WebRTC agents, a properly configured TURN server. If you're using hardware SIP phones at remote locations, a SIP-aware router (like those from Grandstream or Ubiquiti) that handles SIP ALG correctly makes a significant difference. Alternatively, move remote agents to ViciPhone — WebRTC with TURN solves NAT problems more reliably than SIP through consumer routers.

### How many SIP channels does my trunk need?

Multiply your agent count by your expected dial ratio. For predictive dialing at a 5:1 ratio with 50 agents, you need at least 250 simultaneous channels. Add 20% headroom for spikes, so 300 channels. Most SIP trunk providers sell in blocks or offer unlimited channels with per-minute billing. Check your provider's concurrent channel limit — it's often lower than you'd assume on basic plans.

### How do I debug SIP registration failures?

Run `asterisk -rx "sip set debug peer your-carrier-name"` and then trigger a registration with `asterisk -rx "sip reload"`. Watch the output for the SIP exchange. A 401 response means authentication failure (wrong credentials). A 403 means forbidden (IP not whitelisted, or account issue). A 408 timeout means the packets aren't reaching the carrier (firewall, DNS, or routing issue). Check DNS resolution with `dig sip.carrier.com` and verify outbound UDP 5060 isn't blocked.

### Does Asterisk configuration differ in a VICIdial cluster?

Yes. In a cluster, each telephony server runs its own Asterisk instance with its own `sip.conf`. Key differences: (1) each server has its own `externip` set to its own public IP, (2) `localnet` includes the private cluster network, (3) IAX2 trunks between dialers use the private network IPs, (4) each server registers independently to SIP carriers (or a subset of carriers for redundancy). The database server and web servers don't run Asterisk at all. See the [cluster guide](/blog/vicidial-cluster-guide/) for the complete architecture.

### How do I monitor Asterisk performance in production?

Beyond the CLI commands covered above, set up automated monitoring. Use `asterisk -rx "core show channels count"` in a cron job piped to your monitoring system (Nagios, Zabbix, or Prometheus via the Asterisk exporter). Alert on: channel count approaching trunk limits, peer status changes (trunk going unreachable), and AMI connection failures. For call quality monitoring, enable RTCP statistics in `rtp.conf` with `rtcpinterval=5000` and collect jitter/packet loss metrics per call.

### What happens to my Asterisk configuration when I update VICIdial via SVN?

VICIdial's SVN update process (`svn update` followed by running the install scripts) regenerates `extensions.conf` and may update other Asterisk config files. This is why custom dialplan changes go in `extensions_custom.conf` — it survives updates. Your `sip.conf` trunk definitions are managed through the VICIdial admin Carriers page and get regenerated from the database, so they're safe. Manual edits to `sip.conf` outside of VICIdial's management (like custom global settings) should be documented so you can reapply them after updates. Better yet, put them in `sip_custom.conf` if your VICIdial version supports custom includes.

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-asterisk-configuration).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
