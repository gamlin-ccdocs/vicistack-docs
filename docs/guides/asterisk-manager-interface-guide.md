# Asterisk Manager Interface (AMI): The Complete Developer Guide

**Last updated: March 2026 | Reading time: ~24 minutes**

The Asterisk Manager Interface is a TCP socket API that lets you control Asterisk programmatically. Originate calls. Monitor channels. Transfer calls. Get real-time events for every call that enters, bridges, and hangs up on your system.

If you're building anything on top of Asterisk — a CRM integration, a wallboard, a click-to-call feature, a custom reporting tool — AMI is probably how you're going to do it. VICIdial itself uses AMI heavily. The agent screen, the real-time report, the auto-dial engine — they all talk to Asterisk through AMI.

This guide covers everything from initial setup to production-grade integrations. No fluff. Code that works.

---

## AMI vs. ARI vs. AGI: Which One Do You Need?

Asterisk has three interfaces for external programs. People confuse them constantly.

**AMI (Manager Interface)** — TCP socket on port 5038. Send commands, receive events. Best for: monitoring, originating calls, managing channels, building dashboards. Think of it as the "admin remote control."

**AGI (Asterisk Gateway Interface)** — Runs a script during a live call. The dialplan hits an AGI application, which forks a process (or connects to a FastAGI server), and that process controls the call step by step. Best for: IVR logic, call routing decisions, database lookups during a call. Think of it as "dialplan scripting."

**ARI (Asterisk REST Interface)** — HTTP/WebSocket API introduced in Asterisk 12. Best for: building entire telephony applications where you want full control of every call. Stasis applications. Think of it as "build your own PBX."

For VICIdial integrations, AMI is what you want 95% of the time. VICIdial doesn't use ARI. AGI is used for specific call-flow customizations. But AMI is the backbone.

---

## Setting Up manager.conf

The AMI configuration lives in `/etc/asterisk/manager.conf`. On a fresh Asterisk install, it's usually there but disabled.

```ini
; /etc/asterisk/manager.conf

[general]
enabled = yes
port = 5038
bindaddr = 127.0.0.1    ; IMPORTANT: Only listen on localhost by default
displayconnects = yes
timestampevents = yes

[vicistack]
secret = a_strong_random_password_here
deny = 0.0.0.0/0.0.0.0
permit = 127.0.0.1/255.255.255.255
read = system,call,log,verbose,agent,user,config,dtmf,reporting,cdr,dialplan
write = system,call,agent,user,config,command,reporting,originate
writetimeout = 5000
```

Let's break down the important parts.

### bindaddr

This controls what IP address AMI listens on. **Never set this to 0.0.0.0 on a production system** unless you have firewall rules restricting access. AMI has no rate limiting, no brute force protection, and if someone gets in, they can originate calls, execute shell commands (yes, really), and read your entire Asterisk configuration.

```ini
; GOOD: Only local connections
bindaddr = 127.0.0.1

; ACCEPTABLE: Specific internal network interface
bindaddr = 10.0.0.5

; DANGEROUS: Listen on all interfaces
; bindaddr = 0.0.0.0
```

If you need remote AMI access (e.g., your web server is on a different box than Asterisk), use an SSH tunnel or a VPN. Don't expose port 5038 to the internet.

### User Permissions

The `read` and `write` lines control what the AMI user can do. These are comma-separated permission classes:

| Class | Read (receive events) | Write (send actions) |
|-------|----------------------|---------------------|
| system | System events | Restart, shutdown |
| call | Call events | Originate, redirect, hangup |
| log | Log events | — |
| verbose | Verbose messages | — |
| agent | Agent events | Agent login/logoff |
| user | User events | UserEvent |
| config | — | GetConfig, UpdateConfig |
| dtmf | DTMF events | — |
| reporting | CEL/CDR events | — |
| cdr | CDR events | — |
| dialplan | Dialplan events | — |
| command | — | Execute CLI commands |
| originate | — | Originate calls |

**Principle of least privilege.** If your integration only needs to monitor calls, give it `read = call,cdr` and `write = ` (nothing). If it only needs to originate calls, give it `write = originate` and nothing else.

The `command` permission is particularly dangerous — it lets the AMI user run arbitrary Asterisk CLI commands, some of which can execute shell commands. Only grant `command` to trusted integrations running on the same server.

### IP Restrictions

Always use `deny` and `permit` to restrict which IPs can connect:

```ini
[monitoring_user]
secret = monitor_password_here
deny = 0.0.0.0/0.0.0.0           ; Deny everything first
permit = 127.0.0.1/255.255.255.255   ; Allow localhost
permit = 10.0.1.50/255.255.255.255   ; Allow the monitoring server
read = call,cdr,agent
write =
```

After editing manager.conf, reload:

```bash
asterisk -rx "manager reload"
```

---

## Connecting and Authenticating

AMI is a plain TCP socket. You can test it with telnet:

```bash
telnet localhost 5038
```

You'll see:

```
Asterisk Call Manager/7.0.3
```

Now authenticate:

```
Action: Login
Username: vicistack
Secret: a_strong_random_password_here

```

(Note the blank line at the end — AMI uses blank lines as message terminators.)

Response:

```
Response: Success
Message: Authentication accepted
```

You're in. Now you can send actions and receive events.

### Authentication Failure

If you get:

```
Response: Error
Message: Authentication failed
```

Check:
1. Username matches a section name in manager.conf (case-sensitive)
2. Secret matches exactly
3. Your IP is in the permit list
4. You reloaded manager after editing the config

---

## Core AMI Actions

### Originate — Make a Call

The most used AMI action. This tells Asterisk to call a number and connect it to an extension, application, or another number.

```
Action: Originate
Channel: SIP/carrier/18005551234
Context: from-internal
Exten: 100
Priority: 1
CallerID: "Sales" <18005559999>
Timeout: 30000
ActionID: call-001

```

This tells Asterisk: "Call 18005551234 through the SIP trunk named 'carrier'. When they answer, send them to extension 100 in the from-internal context."

For click-to-call (agent clicks a number in CRM, phone rings, then the lead is dialed):

```
Action: Originate
Channel: SIP/agent_phone_100
Context: click-to-call
Exten: 18005551234
Priority: 1
CallerID: "Agent Smith" <18005559999>
Timeout: 30000
ActionID: c2c-agent100-001

```

This calls the agent's phone first. When the agent picks up, Asterisk dials the lead number.

### Status — Get Channel Information

```
Action: Status
ActionID: status-001

```

Returns a list of all active channels with caller ID, state, duration, context, and extension. This is what powers real-time wallboards.

### Command — Run CLI Commands

```
Action: Command
Command: sip show peers
ActionID: cmd-001

```

Returns the output of the Asterisk CLI command. Use this sparingly — it's a text-based response that you have to parse. For structured data, use dedicated actions instead.

### CoreShowChannels — Active Call Details

```
Action: CoreShowChannels
ActionID: channels-001

```

Returns structured data about every active channel. Better than `Action: Command / Command: core show channels` because the output is event-based and parseable.

### Hangup — Kill a Channel

```
Action: Hangup
Channel: SIP/carrier-00000042
ActionID: hangup-001

```

Immediately hangs up the specified channel. Use the channel name from Status or CoreShowChannels.

### Redirect — Transfer a Call

```
Action: Redirect
Channel: SIP/agent_100-00000042
Context: transfer-destination
Exten: 200
Priority: 1
ActionID: xfer-001

```

Transfers the call to extension 200 in the transfer-destination context. This is how VICIdial's blind transfer works under the hood.

### QueueStatus — Queue Monitoring

```
Action: QueueStatus
ActionID: queue-001

```

Returns members and callers in all queues. If you're building an inbound call center dashboard, this is the action you'll use most.

---

## AMI Events: Real-Time Call Data

When you connect with `read` permissions, Asterisk streams events to your connection in real time. Here are the events you'll see most:

### Newchannel

Fired when a new channel is created (call starts):

```
Event: Newchannel
Privilege: call,all
Channel: SIP/carrier-00000042
ChannelState: 0
ChannelStateDesc: Down
CallerIDNum: 18005551234
CallerIDName: John Doe
AccountCode:
Exten: s
Context: from-carrier
Uniqueid: 1711408200.42
```

### Dial

Fired when one channel dials another:

```
Event: Dial
SubEvent: Begin
Channel: SIP/carrier-00000042
Destination: SIP/agent_100-00000043
CallerIDNum: 18005551234
CallerIDName: John Doe
DialString: agent_100
```

### Bridge

Fired when two channels are connected (call answered):

```
Event: Bridge
Privilege: call,all
Bridgestate: Link
Bridgetype: core
Channel1: SIP/carrier-00000042
Channel2: SIP/agent_100-00000043
Uniqueid1: 1711408200.42
Uniqueid2: 1711408200.43
CallerID1: 18005551234
CallerID2: 100
```

### Hangup

Fired when a channel is destroyed:

```
Event: Hangup
Channel: SIP/carrier-00000042
Uniqueid: 1711408200.42
CallerIDNum: 18005551234
Cause: 16
Cause-txt: Normal Clearing
```

### AgentCalled / AgentConnect / AgentComplete

Queue-specific events that track agent availability and call handling. Essential for building agent performance dashboards.

---

## Building a Real AMI Client

Telnet is fine for testing. For production, you need a real client with connection management, event parsing, and error handling.

### Python Example

```python
import socket
import time


class AMIClient:
    def __init__(self, host='127.0.0.1', port=5038):
        self.host = host
        self.port = port
        self.sock = None
        self.buffer = ''

    def connect(self):
        self.sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self.sock.settimeout(30)
        self.sock.connect((self.host, self.port))
        # Read the banner
        banner = self._read_response()
        return banner

    def login(self, username, secret):
        self._send_action({
            'Action': 'Login',
            'Username': username,
            'Secret': secret,
        })
        response = self._read_response()
        if 'Authentication accepted' not in response:
            raise Exception(f'AMI login failed: {response}')
        return response

    def originate(self, channel, context, exten, priority=1,
                  callerid=None, timeout=30000, action_id=None):
        action = {
            'Action': 'Originate',
            'Channel': channel,
            'Context': context,
            'Exten': exten,
            'Priority': str(priority),
            'Timeout': str(timeout),
        }
        if callerid:
            action['CallerID'] = callerid
        if action_id:
            action['ActionID'] = action_id
        self._send_action(action)
        return self._read_response()

    def command(self, cmd, action_id=None):
        action = {
            'Action': 'Command',
            'Command': cmd,
        }
        if action_id:
            action['ActionID'] = action_id
        self._send_action(action)
        return self._read_response()

    def logoff(self):
        self._send_action({'Action': 'Logoff'})

    def _send_action(self, action):
        msg = ''
        for key, value in action.items():
            msg += f'{key}: {value}\r\n'
        msg += '\r\n'
        self.sock.sendall(msg.encode('utf-8'))

    def _read_response(self):
        while '\r\n\r\n' not in self.buffer:
            chunk = self.sock.recv(4096).decode('utf-8')
            if not chunk:
                raise ConnectionError('AMI connection closed')
            self.buffer += chunk
        response, self.buffer = self.buffer.split('\r\n\r\n', 1)
        return response

    def close(self):
        if self.sock:
            self.logoff()
            self.sock.close()


# Usage
ami = AMIClient('127.0.0.1', 5038)
ami.connect()
ami.login('vicistack', 'your_password')

# Check SIP peers
result = ami.command('sip show peers')
print(result)

# Originate a call
ami.originate(
    channel='SIP/carrier/18005551234',
    context='from-internal',
    exten='100',
    callerid='"Sales" <18005559999>',
    action_id='test-call-001'
)

ami.close()
```

### Node.js Example

```javascript
const net = require('net');

class AMIClient {
  constructor(host = '127.0.0.1', port = 5038) {
    this.host = host;
    this.port = port;
    this.socket = null;
    this.buffer = '';
    this.callbacks = {};
    this.eventHandlers = {};
  }

  connect() {
    return new Promise((resolve, reject) => {
      this.socket = net.createConnection(this.port, this.host);
      this.socket.setEncoding('utf8');

      this.socket.once('data', (data) => {
        // First data is the banner
        resolve(data.trim());
      });

      this.socket.on('data', (data) => {
        this.buffer += data;
        this._processBuffer();
      });

      this.socket.on('error', reject);
    });
  }

  login(username, secret) {
    return this.sendAction({
      Action: 'Login',
      Username: username,
      Secret: secret,
    });
  }

  originate(channel, context, exten, options = {}) {
    return this.sendAction({
      Action: 'Originate',
      Channel: channel,
      Context: context,
      Exten: exten,
      Priority: options.priority || '1',
      Timeout: options.timeout || '30000',
      CallerID: options.callerId || '',
      ActionID: options.actionId || `orig-${Date.now()}`,
    });
  }

  sendAction(action) {
    return new Promise((resolve) => {
      const actionId = action.ActionID || `act-${Date.now()}`;
      action.ActionID = actionId;
      this.callbacks[actionId] = resolve;

      let msg = '';
      for (const [key, value] of Object.entries(action)) {
        if (value) msg += `${key}: ${value}\r\n`;
      }
      msg += '\r\n';
      this.socket.write(msg);
    });
  }

  on(eventName, handler) {
    if (!this.eventHandlers[eventName]) {
      this.eventHandlers[eventName] = [];
    }
    this.eventHandlers[eventName].push(handler);
  }

  _processBuffer() {
    const messages = this.buffer.split('\r\n\r\n');
    this.buffer = messages.pop(); // Keep incomplete message

    for (const msg of messages) {
      if (!msg.trim()) continue;
      const parsed = {};
      for (const line of msg.split('\r\n')) {
        const idx = line.indexOf(': ');
        if (idx > 0) {
          parsed[line.substring(0, idx)] = line.substring(idx + 2);
        }
      }

      // Route to callback or event handler
      if (parsed.ActionID && this.callbacks[parsed.ActionID]) {
        this.callbacks[parsed.ActionID](parsed);
        delete this.callbacks[parsed.ActionID];
      }
      if (parsed.Event) {
        const handlers = this.eventHandlers[parsed.Event] || [];
        for (const h of handlers) h(parsed);
      }
    }
  }

  close() {
    if (this.socket) {
      this.socket.write('Action: Logoff\r\n\r\n');
      this.socket.end();
    }
  }
}

// Usage
async function main() {
  const ami = new AMIClient('127.0.0.1', 5038);
  await ami.connect();
  const loginResult = await ami.login('vicistack', 'your_password');
  console.log('Login:', loginResult);

  // Listen for call events
  ami.on('Newchannel', (event) => {
    console.log('New call:', event.CallerIDNum, event.Channel);
  });

  ami.on('Hangup', (event) => {
    console.log('Call ended:', event.Channel, event['Cause-txt']);
  });

  // Keep running to receive events
  // ami.close() when done
}

main().catch(console.error);
```

---

## AMI and VICIdial: How They Work Together

VICIdial's relationship with AMI is deep. Here's what's happening behind the scenes:

**Auto-dial engine (VDauto_dial.pl):** Connects to AMI and sends Originate actions for each lead pulled from the hopper. It monitors Newchannel, Bridge, and Hangup events to track call state.

**Agent interface (vicidial.php):** Uses AMI to transfer calls, hang up channels, initiate manual dials, and monitor agent channel state.

**Real-time reporting:** Queries AMI via the Status action and CoreShowChannels to build the real-time agent display.

**Call recording:** Uses AMI's Monitor/MixMonitor actions to start and stop recordings.

If you're building custom integrations with VICIdial, you don't usually talk to AMI directly. VICIdial has its own [Non-Agent API](/blog/vicidial-api-integration/) that wraps AMI actions. But for direct Asterisk integrations, AMI is the way.

### VICIdial's AMI User

On a ViciBox install, VICIdial creates its own AMI user in manager.conf. Don't modify or share this user. Create a separate AMI user for your integrations:

```ini
; Add below the existing VICIdial user
[myintegration]
secret = separate_strong_password
deny = 0.0.0.0/0.0.0.0
permit = 127.0.0.1/255.255.255.255
read = call,agent,cdr
write = originate
```

---

## Production Hardening

### Connection Pooling

AMI connections are persistent TCP sockets. Opening a new connection for each request wastes time on TCP handshake + authentication. Maintain a pool of authenticated connections.

### Automatic Reconnection

Asterisk restarts, network blips, and TCP timeouts will drop your AMI connection. Your client must detect disconnection and automatically reconnect + re-authenticate.

```python
# Simplified reconnection loop
def get_ami_connection():
    while True:
        try:
            ami = AMIClient()
            ami.connect()
            ami.login('user', 'secret')
            return ami
        except Exception as e:
            print(f'AMI connection failed: {e}, retrying in 5s')
            time.sleep(5)
```

### Event Filtering

A busy Asterisk system generates thousands of events per minute. If your integration only cares about Hangup events, filter server-side:

```
Action: Events
EventMask: call

```

Or more granular filtering with Asterisk 16+:

```
Action: Filter
Operation: Add
Filter: Event: Hangup

```

This tells Asterisk to only send you Hangup events, reducing bandwidth and processing overhead.

### Rate Limiting Your Actions

Don't blast AMI with hundreds of Originate actions per second. Asterisk processes AMI actions sequentially per connection. Flooding it will queue up actions and increase latency for all of them.

For high-volume origination (like a predictive dialer), use multiple AMI connections with load distribution. VICIdial does this — it maintains multiple connections and distributes originate requests across them.

### Security Considerations

1. **Never expose port 5038 to the internet.** SSH tunnels or VPN for remote access.
2. **Use strong passwords.** AMI has no lockout after failed attempts.
3. **Restrict permissions.** Don't give `command` or `system` write access unless absolutely necessary.
4. **Log AMI connections.** Enable `displayconnects = yes` in manager.conf and monitor `/var/log/asterisk/messages` for login events.
5. **Consider TLS.** Asterisk 16+ supports AMI over TLS. Configure it if your AMI client connects over a network.

```ini
; manager.conf — TLS support (Asterisk 16+)
[general]
enabled = yes
port = 5038
tlsenable = yes
tlsbindaddr = 0.0.0.0:5039
tlscertfile = /etc/asterisk/keys/asterisk.pem
tlsprivatekey = /etc/asterisk/keys/asterisk.key
```

---

## Common AMI Pitfalls

**ActionID is your friend.** Always include an ActionID in your actions. AMI responses don't always come back in order, especially with async actions like Originate. The ActionID is the only way to correlate a response with the action that triggered it.

**Originate is async by default.** When you send an Originate action, the Response: Success just means Asterisk accepted the request. It doesn't mean the call connected. You need to listen for Newchannel, Dial, Bridge, and Hangup events to track the call's lifecycle.

```
Action: Originate
Channel: SIP/carrier/18005551234
Context: from-internal
Exten: 100
Priority: 1
Async: true
ActionID: call-001

```

**Buffer carefully.** AMI messages are terminated by `\r\n\r\n`. But messages can arrive fragmented across multiple TCP reads. Your parser must buffer data and only process complete messages.

**Handle multi-line responses.** Some responses (like Command output) span multiple lines. They end with `--END COMMAND--` before the `\r\n\r\n` terminator. Your parser needs to handle this.

**Event floods during peak hours.** On a system with 100+ concurrent calls, AMI can generate 50+ events per second. If your client can't keep up processing events, the TCP buffer fills and you lose data. Use event filtering to only subscribe to events you actually need.

---

## Useful AMI One-Liners

For quick operations from the command line without building a full client:

```bash
# Quick call origination via netcat
echo -e "Action: Login\r\nUsername: user\r\nSecret: pass\r\n\r\nAction: Originate\r\nChannel: SIP/carrier/18005551234\r\nContext: default\r\nExten: 100\r\nPriority: 1\r\n\r\nAction: Logoff\r\n\r\n" | nc localhost 5038

# Check how many active channels
echo -e "Action: Login\r\nUsername: user\r\nSecret: pass\r\n\r\nAction: Command\r\nCommand: core show channels count\r\n\r\nAction: Logoff\r\n\r\n" | nc localhost 5038

# Reload SIP configuration remotely
echo -e "Action: Login\r\nUsername: user\r\nSecret: pass\r\n\r\nAction: Command\r\nCommand: sip reload\r\n\r\nAction: Logoff\r\n\r\n" | nc localhost 5038
```

---

## AMI for Real-Time Dashboards

The killer use case for AMI in a call center is [real-time monitoring](/blog/vicidial-realtime-agent-dashboard/). Here's the event flow for building a dashboard:

1. Connect to AMI, authenticate, subscribe to `call` and `agent` events
2. On **Newchannel**: New call starting, add to active calls list
3. On **Dial**: Call is ringing, show in "ringing" state
4. On **Bridge**: Call answered, start timer, show in "connected" state
5. On **Hangup**: Call ended, calculate duration, remove from active calls
6. On **AgentConnect**: Agent took a call, update agent status
7. On **AgentComplete**: Agent finished a call, show wrap-up time

Feed these events into a WebSocket server and push to a browser-based dashboard. That's exactly how VICIdial's Real-Time Report works, though it goes through PHP + database polling rather than direct WebSocket.

For a more modern approach, push AMI events into Redis or a message queue and have your dashboard consume from there. This decouples the AMI connection from the web layer and handles connection drops gracefully.

---

**Building custom integrations on top of VICIdial?** ViciStack has done this for dozens of operations — CRM integrations, custom wallboards, automated quality monitoring, and click-to-call systems. We know where AMI works well and where you'll hit walls. Our engagement is $5K ($1K deposit, $4K on completion) and includes architecture review, implementation guidance, and AMI security hardening. [Let's talk](/contact/).

---

*Related: [VICIdial API Integration](/blog/vicidial-api-integration/) | [VICIdial Grafana Dashboards](/blog/vicidial-grafana-dashboards/) | [VICIdial Asterisk Configuration](/blog/vicidial-asterisk-configuration/) | [VICIdial CRM Integration](/blog/vicidial-crm-integration/)*

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/asterisk-manager-interface-guide).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
