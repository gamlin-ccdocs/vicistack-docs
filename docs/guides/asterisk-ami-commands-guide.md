# Asterisk AMI Commands Guide

**Last updated: March 2026 | Reading time: ~26 minutes**

The [Asterisk Manager Interface](/blog/asterisk-manager-interface-guide/) (AMI) is the programmatic backdoor to your PBX. It's a TCP socket protocol that lets external programs control Asterisk — originate calls, redirect channels, read variables, monitor events, and do anything the Asterisk CLI can do, but from code.

VICIdial uses AMI extensively under the hood. The dialer cron scripts originate calls through AMI. The agent screen sends DTMF through AMI. Call monitoring, conference management, and agent session control all go through AMI. If you understand AMI, you understand the engine that drives VICIdial's real-time operations.

The official documentation lists 150+ commands. You'll use maybe 15 of them regularly. Here are those 15, with the exact syntax and practical examples.

---

## Setting Up AMI

### manager.conf

AMI is configured in `/etc/asterisk/manager.conf`. VICIbox ships with this pre-configured for VICIdial, but understanding the config helps when you need to add custom integrations.

```ini
[general]
enabled = yes
port = 5038
bindaddr = 127.0.0.1
displayconnects = yes
timestampevents = yes

[cron]
secret = YOUR_SECRET_HERE
read = system,call,log,verbose,agent,user,config,dtmf,reporting,cdr,dialplan,originate
write = system,call,agent,user,config,command,reporting,originate

[monitoring]
secret = ANOTHER_SECRET
read = system,call,agent,reporting,cdr
write = command
```

Key configuration points:

**`bindaddr = 127.0.0.1`**: Binds AMI to localhost only. This means only programs running on the same server can connect. This is the safe default. If you need remote AMI access (for external dashboards or integrations), change this to `0.0.0.0` and add firewall rules.

**`read` permissions**: What events and data this user can receive.
**`write` permissions**: What actions this user can execute.

The permission categories:
| Permission | What It Controls |
|-----------|-----------------|
| `system` | System status, module management, shutdown |
| `call` | Channel events, call origination |
| `log` | Log events |
| `verbose` | Verbose messages |
| `agent` | Agent-related events and actions |
| `user` | User events |
| `config` | Configuration changes |
| `dtmf` | DTMF events and sending |
| `reporting` | CDR and reporting events |
| `cdr` | CDR data |
| `dialplan` | Dialplan events |
| `originate` | Call origination |
| `command` | CLI command execution through AMI |

After editing, reload:

```bash
asterisk -rx "manager reload"
```

### Testing Connectivity

The quickest way to test AMI:

```bash
telnet 127.0.0.1 5038
```

You'll see:

```
Asterisk Call Manager/6.0.0
```

Authenticate:

```
Action: Login
Username: cron
Secret: YOUR_SECRET_HERE

```

(Note the blank line after Secret — that's required. AMI uses blank lines as packet delimiters.)

Response:

```
Response: Success
Message: Authentication accepted
```

You're in. Now you can type actions directly.

---

## The Core AMI Actions

### 1. Originate — Make a Call

The most powerful AMI action. Tells Asterisk to originate a new call to a destination.

**Two modes:**

**Application mode** — Call a number and run a dialplan application:

```
Action: Originate
Channel: PJSIP/carrier/sip:3125551234@carrier.com
Context: from-internal
Exten: s
Priority: 1
CallerID: "Sales" <8005551234>
Timeout: 30000
Variable: campaign_id=SALES01
ActionID: orig-001

```

**Async mode** — Same but returns immediately instead of waiting for the call to connect:

```
Action: Originate
Channel: PJSIP/carrier/sip:3125551234@carrier.com
Application: MeetMe
Data: 8600100|qF
CallerID: "Sales" <8005551234>
Timeout: 30000
Async: true
ActionID: orig-002

```

VICIdial uses Application mode with `MeetMe` to bridge calls into conference rooms where agents are already waiting. That's how the predictive dialer works — it originates a call, and when it connects, the call lands in a MeetMe room where an agent is listening.

**Important parameters:**
- `Channel`: The outbound SIP channel (format depends on chan_sip vs PJSIP)
- `Context`/`Exten`/`Priority`: Where to send the call in the dialplan after it connects
- `Application`/`Data`: Alternative to Context — run a specific application directly
- `CallerID`: The caller ID to present
- `Timeout`: Ring timeout in milliseconds (30000 = 30 seconds)
- `Async`: Return immediately without waiting for call outcome
- `Variable`: Set channel variables (key=value, comma-separated for multiple)
- `ActionID`: Your identifier for matching the response

### 2. Redirect — Transfer a Call

Move an active channel to a different dialplan destination:

```
Action: Redirect
Channel: PJSIP/carrier-00000042
Context: transfer-dest
Exten: 3125559876
Priority: 1
ActionID: redir-001

```

For attended transfers (redirect two channels simultaneously):

```
Action: Redirect
Channel: PJSIP/carrier-00000042
ExtraChannel: PJSIP/agent-00000043
Context: transfer-dest
Exten: 3125559876
Priority: 1
ExtraContext: from-internal
ExtraExten: hangup
ExtraPriority: 1
ActionID: redir-002

```

VICIdial uses Redirect for blind transfers, warm transfers, and parking. When an agent clicks "Transfer" on the agent screen, the web interface sends an AMI Redirect action to move the call.

### 3. Hangup — End a Call

Terminate a specific channel:

```
Action: Hangup
Channel: PJSIP/carrier-00000042
Cause: 16
ActionID: hangup-001

```

Cause code 16 is "Normal clearing" — the standard hangup cause. Other common codes:
- 17 = User busy
- 19 = No answer
- 21 = Call rejected
- 31 = Normal, unspecified

### 4. Status — Query Active Channels

Get information about active channels:

```
Action: Status
ActionID: status-001

```

Returns a list of all active channels with their state, caller ID, duration, and bridged channel. For a specific channel:

```
Action: Status
Channel: PJSIP/carrier-00000042
ActionID: status-002

```

This is how monitoring systems check how many calls are active on the system at any moment.

### 5. CoreShowChannels — List All Channels

Similar to Status but returns a more detailed listing:

```
Action: CoreShowChannels
ActionID: channels-001

```

Response includes channel name, context, extension, priority, state, application, duration, caller ID, and account code for every active channel. Useful for building real-time call dashboards.

### 6. GetVar — Read a Channel Variable

Read the value of a channel variable or dialplan function:

```
Action: GetVar
Channel: PJSIP/carrier-00000042
Variable: CALLERID(num)
ActionID: getvar-001

```

Response:

```
Response: Success
Variable: CALLERID(num)
Value: 3125551234
ActionID: getvar-001
```

You can read any channel variable or dialplan function this way. Useful for checking call state, reading custom data attached to calls, and debugging dialplan logic.

### 7. SetVar — Write a Channel Variable

Set a variable on an active channel:

```
Action: SetVar
Channel: PJSIP/carrier-00000042
Variable: CUSTOM_DATA
Value: priority_customer
ActionID: setvar-001

```

You can also set global variables (no Channel parameter):

```
Action: SetVar
Variable: GLOBAL_SETTING
Value: enabled
ActionID: setvar-002

```

### 8. Command — Execute CLI Commands

Run any Asterisk CLI command through AMI:

```
Action: Command
Command: core show channels
ActionID: cmd-001

```

Response includes the full CLI output. This is the escape hatch — anything you can type in `asterisk -r` you can execute through AMI.

Common commands:
- `sip show peers` / `pjsip show endpoints` — Check SIP registrations
- `core show channels` — Active channel count
- `queue show` — Queue statistics
- `database show` — AstDB contents
- `module reload res_pjsip.so` — Reload PJSIP without restarting

### 9. QueueStatus — Get Queue Information

Query the current state of Asterisk queues:

```
Action: QueueStatus
Queue: support-queue
ActionID: queue-001

```

Returns queue members, their status (available/paused/busy), and current callers waiting. For VICIdial, queue data is managed differently (through the VICIdial application layer), but this is useful for hybrid setups that use both VICIdial and native Asterisk queues.

### 10. Monitor / StopMonitor — Call Recording

Start recording an active channel:

```
Action: Monitor
Channel: PJSIP/carrier-00000042
File: /var/spool/asterisk/monitor/20260326-call-042
Format: wav
Mix: true
ActionID: monitor-001

```

Stop recording:

```
Action: StopMonitor
Channel: PJSIP/carrier-00000042
ActionID: stopmon-001

```

VICIdial handles recording through its own mechanism (campaign-level recording settings), but Monitor/StopMonitor is useful for on-demand recording of specific calls from external systems.

---

## AMI Events: Real-Time Call Tracking

AMI isn't just for sending commands — it's also a real-time event stream. When you connect and authenticate, Asterisk pushes events to your connection as they happen.

### Key Events

**Newchannel** — A new channel was created (call initiated):

```
Event: Newchannel
Channel: PJSIP/carrier-00000042
ChannelState: 0
ChannelStateDesc: Down
CallerIDNum: 3125551234
CallerIDName: John Smith
Uniqueid: 1711432800.42
```

**Hangup** — A channel was hung up:

```
Event: Hangup
Channel: PJSIP/carrier-00000042
Cause: 16
Cause-txt: Normal Clearing
Uniqueid: 1711432800.42
```

**BridgeEnter** / **BridgeLeave** — Channels joining/leaving bridges:

```
Event: BridgeEnter
Channel: PJSIP/carrier-00000042
BridgeUniqueid: bridge-001
BridgeType: basic
```

**AgentConnect** — An agent connected to a queue call:

```
Event: AgentConnect
Queue: sales-queue
Interface: PJSIP/agent001
MemberName: Agent Smith
HoldTime: 12
```

**DTMFReceived** — A DTMF digit was detected:

```
Event: DTMFReceived
Channel: PJSIP/carrier-00000042
Digit: 5
Direction: Received
```

### Filtering Events

On a busy system, AMI generates thousands of events per minute. Filter them:

```
Action: Events
EventMask: call,agent
ActionID: events-001

```

EventMask options: `on` (all events), `off` (no events), or a comma-separated list of event categories. This reduces noise and processing overhead.

---

## Practical AMI Automation Scripts

### Python: Click-to-Dial

A script that originates a call between an agent's phone and a customer number:

```python
import socket

class AMIClient:
 def __init__(self, host, port, user, secret):
 self.sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
 self.sock.connect((host, port))
 self.sock.settimeout(5)
 # Read banner
 self.read_response()
 # Login
 self.send_action({
 'Action': 'Login',
 'Username': user,
 'Secret': secret,
 })
 resp = self.read_response()
 if 'Success' not in resp:
 raise Exception(f"AMI login failed: {resp}")

 def send_action(self, action):
 msg = ''
 for key, value in action.items():
 msg += f'{key}: {value}\r\n'
 msg += '\r\n'
 self.sock.sendall(msg.encode())

 def read_response(self):
 data = b''
 while True:
 try:
 chunk = self.sock.recv(4096)
 data += chunk
 if b'\r\n\r\n' in data:
 break
 except socket.timeout:
 break
 return data.decode()

 def originate(self, channel, context, exten, callerid, variables=None):
 action = {
 'Action': 'Originate',
 'Channel': channel,
 'Context': context,
 'Exten': exten,
 'Priority': '1',
 'CallerID': callerid,
 'Timeout': '30000',
 'Async': 'true',
 }
 if variables:
 action['Variable'] = ','.join(f'{k}={v}' for k, v in variables.items())
 self.send_action(action)
 return self.read_response()

 def hangup(self, channel):
 self.send_action({
 'Action': 'Hangup',
 'Channel': channel,
 'Cause': '16',
 })
 return self.read_response()

 def close(self):
 self.send_action({'Action': 'Logoff'})
 self.sock.close()

# Usage: click-to-dial between agent extension and customer
ami = AMIClient('127.0.0.1', 5038, 'cron', 'YOUR_SECRET')

# First, call the agent's phone
result = ami.originate(
 channel='PJSIP/agent001',
 context='click-to-dial',
 exten='3125551234',
 callerid='"Click-to-Dial" <8005551234>',
 variables={'customer_name': 'John Smith'}
)
print(f"Originate result: {result}")
ami.close()
```

### Node.js: Real-Time Event Monitor

A script that connects to AMI and streams call events:

```javascript
const net = require('net');

class AMIEventMonitor {
 constructor(host, port, user, secret) {
 this.client = new net.Socket();
 this.buffer = '';

 this.client.connect(port, host, () => {
 console.log('Connected to AMI');
 this.send(`Action: Login\r\nUsername: ${user}\r\nSecret: ${secret}\r\n\r\n`);
 });

 this.client.on('data', (data) => {
 this.buffer += data.toString();
 this.processBuffer();
 });

 this.client.on('error', (err) => {
 console.error('AMI connection error:', err.message);
 });
 }

 send(msg) {
 this.client.write(msg);
 }

 processBuffer() {
 const packets = this.buffer.split('\r\n\r\n');
 this.buffer = packets.pop(); // Keep incomplete packet in buffer

 for (const packet of packets) {
 if (!packet.trim()) continue;
 const event = {};
 for (const line of packet.split('\r\n')) {
 const idx = line.indexOf(': ');
 if (idx > 0) {
 event[line.substring(0, idx)] = line.substring(idx + 2);
 }
 }
 this.handleEvent(event);
 }
 }

 handleEvent(event) {
 if (event.Event === 'Newchannel') {
 console.log(`NEW CALL: ${event.CallerIDNum} -> Channel: ${event.Channel}`);
 } else if (event.Event === 'Hangup') {
 console.log(`HANGUP: ${event.Channel} (Cause: ${event['Cause-txt']})`);
 } else if (event.Event === 'BridgeEnter') {
 console.log(`BRIDGE: ${event.Channel} joined ${event.BridgeUniqueid}`);
 } else if (event.Response === 'Success') {
 console.log(`AUTH: ${event.Message}`);
 // Filter to only call events
 this.send('Action: Events\r\nEventMask: call\r\n\r\n');
 }
 }
}

const monitor = new AMIEventMonitor('127.0.0.1', 5038, 'monitoring', 'YOUR_SECRET');
```

### Bash: Quick Channel Count Check

A one-liner for monitoring scripts:

```bash
#!/bin/bash
# Count active channels via AMI
CHANNELS=$(echo -e "Action: Login\r\nUsername: cron\r\nSecret: YOUR_SECRET\r\n\r\nAction: CoreShowChannels\r\n\r\nAction: Logoff\r\n\r\n" | nc -w 3 127.0.0.1 5038 | grep "ListItems:" | awk '{print $2}')
echo "Active channels: $CHANNELS"
```

---

## AMI Security

### Bind to Localhost

Unless you have a specific need for remote AMI access, keep `bindaddr = 127.0.0.1`. If you need remote access, use SSH tunneling instead:

```bash
ssh -L 5038:127.0.0.1:5038 user@asterisk-server
```

Then connect your client to `127.0.0.1:5038` on your local machine.

### Unique Secrets Per User

Every AMI user should have a different secret. If you use the same secret for all users, you can't revoke one user's access without changing everyone's credentials.

### Minimal Permissions

Give each AMI user only the permissions they need:
- Monitoring system: `read=system,call` only
- Click-to-dial: `read=call`, `write=originate`
- VICIdial cron (needs everything): Full permissions

### Firewall

If AMI is bound to `0.0.0.0`, firewall it:

```bash
iptables -A INPUT -p tcp --dport 5038 -s 10.0.0.0/24 -j ACCEPT
iptables -A INPUT -p tcp --dport 5038 -j DROP
```

Only allow connections from known internal IPs.

### Audit Logging

Enable `displayconnects = yes` in manager.conf to log all AMI connections. Review these logs periodically.

---

## VICIdial-Specific AMI Patterns

VICIdial's backend scripts use AMI in specific patterns worth understanding:

### How VICIdial Dials Calls

The `VDauto_dial_CAMPAIGN.pl` cron script reads the [dial hopper](/blog/vicidial-dial-hopper-guide/), picks leads, and issues AMI Originate commands with Application: `MeetMe`. The call goes to a MeetMe conference room. When the call is answered (detected by answer supervision or AMD), VICIdial routes an available agent into the same MeetMe room via another AMI Originate.

### How Agent Monitoring Works

When a supervisor clicks "Listen" on the real-time report, VICIdial sends an AMI Originate to the supervisor's phone, connecting them to the same MeetMe conference room as the agent. The `q` flag on MeetMe means the supervisor enters in listen-only mode.

### How DTMF Injection Works

The agent screen's DTMF button triggers an AMI `AGI` action that runs `agi-dtmf.agi` on the customer channel. This script uses the `EXEC SendDTMF` command within the AGI session to inject digits into the correct call leg.

Understanding these patterns helps when troubleshooting VICIdial issues — many "VICIdial bugs" are AMI configuration or permission problems.

---

## When AMI Isn't Enough: ARI

Asterisk 12+ includes the Asterisk REST Interface (ARI), a more modern HTTP/WebSocket-based API. ARI gives you finer-grained control over individual channels, bridges, and recordings.

For new development, ARI is the recommended interface. AMI is still fully supported and VICIdial relies on it entirely, so you'll need AMI knowledge for VICIdial administration regardless.

For projects that need both VICIdial integration (AMI) and custom call control (ARI), they can coexist on the same Asterisk instance. Just be careful about conflicting channel manipulations — if VICIdial owns a channel through AMI and your ARI application tries to manipulate the same channel, things get unpredictable.

---

## Getting AMI Right

AMI is the power tool that makes Asterisk programmable and VICIdial possible. Most VICIdial admins never touch it directly because VICIdial abstracts it away. But when you need custom integrations, automated call flows, or advanced monitoring — AMI is where you go.

---

---

**Related reading:**
- [Asterisk Manager Interface (AMI): The Commands You'll Actually Use](https://vicistack.com/blog/asterisk-ami-commands-guide/)
- [Asterisk Manager Interface (AMI): The Complete Developer Guide](https://vicistack.com/blog/asterisk-manager-interface-guide/)
- [Asterisk PJSIP TLS Broken After OpenSSL 3 Upgrade? Here's the Fix for 'Wrong Curve' and Every Other Handshake Failure](https://vicistack.com/blog/asterisk-pjsip-tls-openssl3-guide/)
- [Asterisk Observability with OpenTelemetry and Grafana](https://vicistack.com/blog/asterisk-otel-observability/)
## AMI Troubleshooting

### "Connection Refused on Port 5038"

```bash
# Check if AMI is listening
ss -tlnp | grep 5038
```

If nothing shows, AMI isn't enabled. Check `manager.conf`:
- `enabled = yes` must be set in `[general]`
- `bindaddr` must match where you're connecting from
- Reload: `asterisk -rx "manager reload"`

### "Authentication Failed"

```bash
# Verify the user exists in manager.conf
asterisk -rx "manager show users"
```

Check that:
- Username and secret match exactly (case-sensitive)
- The user isn't disabled
- The connecting IP is allowed (check `permit`/`deny` ACLs in the user section)

### "Permission Denied" on Specific Actions

AMI permission errors mean the user's `write` permissions don't include the required category. For Originate, you need `write = originate`. For Command, you need `write = command`. For Redirect, you need `write = call`.

Check current permissions:
```bash
asterisk -rx "manager show user cron"
```

### Events Not Arriving

If you're connected and authenticated but not receiving events:
1. Check your EventMask — if set to `off`, no events will arrive
2. Verify `read` permissions include the event categories you expect
3. Check your socket buffer — if you're not reading fast enough, the TCP buffer fills and events are dropped
4. On busy systems (200+ concurrent calls), filter events aggressively with EventMask to avoid overwhelming your client

### AMI Performance at Scale

On a 200-agent VICIdial system, AMI handles thousands of actions per minute. Performance considerations:

- Each AMI connection consumes an Asterisk thread. Limit concurrent connections to 5-10
- Use a single persistent connection instead of connecting/disconnecting per action
- Filter events with EventMask — receiving all events on a busy system generates 50+ MB/hour of data
- Use Async:true for Originate actions to avoid blocking the AMI connection for 30+ seconds per call

### Common AMI Client Libraries

Rather than implementing raw TCP socket handling, use a library for your language:

**Python**: `panoramisk` or `starpy` — both handle connection management, authentication, and event parsing. Panoramisk supports async/await patterns for modern Python.

**Node.js**: `asterisk-manager` (npm) — provides an event emitter pattern that works well with Node's event loop.

**PHP**: `PAMI` (PHP Asterisk Manager Interface) — the most mature PHP AMI library.

**Go**: `ari-proxy` or `goami` — for high-performance AMI integrations.

Using a library eliminates the boilerplate of TCP connection management, packet parsing, and authentication handling. Focus on your business logic instead of protocol details.

If you're building AMI integrations for your call center and need the kind of deep Asterisk expertise that comes from hundreds of deployments, that's what [ViciStack](https://vicistack.com/) offers. We've built AMI-based automation for everything from click-to-dial CRM integrations to real-time quality monitoring dashboards. The $5K engagement covers your specific integration needs, not generic documentation.

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/asterisk-ami-commands-guide).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
