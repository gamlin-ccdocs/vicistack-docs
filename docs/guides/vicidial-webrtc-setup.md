# VICIdial WebRTC Setup Guide for Remote Agents

**The complete technical guide to running VICIdial with WebRTC-based softphones — from SSL certificates to PJSIP configuration to the NAT traversal nightmares that make remote agent deployments interesting.**

---

Remote agents are no longer the exception. They're the majority. In most call center operations we manage, 60-80% of agents work from home or from distributed locations. The days of a physical call center floor with IP desk phones on every desk are fading.

VICIdial supports remote agents through WebRTC — specifically through **ViciPhone**, the built-in browser-based softphone that lets agents take calls directly in their web browser without installing any software. No SIP softphone to configure. No VPN to troubleshoot. No port forwarding on the agent's home router. The agent opens a browser, logs in, and starts taking calls.

When it works, WebRTC is elegant. When it doesn't work, you'll spend hours chasing one-way audio, certificate errors, and the kind of NAT traversal problems that make experienced sysadmins question their career choices.

This guide covers the full WebRTC setup for VICIdial: server requirements, SSL configuration, PJSIP vs. SIP channel driver considerations, Asterisk WebSocket configuration, ViciPhone settings, NAT traversal (STUN/TURN), codec selection, scaling for large remote workforces, and troubleshooting every common issue. For the WebRTC glossary entry, see [/glossary/webrtc/](/glossary/webrtc/).

---

## How WebRTC Works in VICIdial

Before configuring anything, understand the architecture. WebRTC in VICIdial has three components:

**1. ViciPhone (the browser client).** This is a JavaScript-based SIP client built into VICIdial's agent interface. It runs in the agent's browser and handles the audio connection. ViciPhone uses the JsSIP library (or a modified version of it) to implement SIP signaling over WebSocket and audio transport via WebRTC.

**2. Asterisk (the media server).** Asterisk handles the actual SIP registration, call routing, and media processing. For WebRTC, Asterisk must be configured to accept WebSocket connections (WSS — WebSocket Secure) for SIP signaling and DTLS-SRTP for encrypted media transport.

**3. The signaling path.** When an agent loads ViciPhone, the browser establishes a WebSocket connection to Asterisk on a designated port (typically 8089 for WSS). SIP REGISTER and INVITE messages travel over this WebSocket. Audio media (RTP) is negotiated via ICE (Interactive Connectivity Establishment) and flows as encrypted SRTP between the browser and Asterisk.

The key constraint: **WebRTC requires HTTPS.** Modern browsers will not allow WebRTC media access (microphone) on non-secure origins. Your VICIdial web interface must be served over HTTPS with a valid SSL certificate. Self-signed certificates will cause problems — browsers either block WebRTC entirely or require users to manually accept certificate exceptions, which breaks the agent experience.

---

## Server Requirements

### Operating System and Asterisk Version

- **ViciBox 12.0.2+** (AlmaLinux 9 based): Recommended. Comes with Asterisk 18 LTS or 20 and PJSIP compiled.
- **AlmaLinux 9 / Rocky Linux 9 with manual install:** Works, but you must ensure Asterisk is compiled with `res_pjsip`, `res_pjsip_transport_websocket`, and `res_http_websocket` modules.
- **Asterisk version:** 18 LTS or 20. Asterisk 16 works but has known WebSocket stability issues. Asterisk 13 requires significantly more manual configuration and is not recommended for new WebRTC deployments.

**Required Asterisk modules:**
```
res_pjsip.so
res_pjsip_transport_websocket.so
res_http_websocket.so
res_pjsip_dtls.so
res_pjsip_sdp_rtp.so
codec_opus.so (recommended)
```

Verify modules are loaded:
```bash
asterisk -rx "module show like pjsip"
asterisk -rx "module show like websocket"
asterisk -rx "module show like opus"
```

### HTTPS and SSL Certificate

You need a valid, publicly-trusted SSL certificate. Let's Encrypt is free and works perfectly:

```bash
# Install certbot
sudo dnf install -y certbot

# Obtain certificate (your VICIdial server must be publicly accessible on port 80)
sudo certbot certonly --standalone -d vicidial.yourdomain.com

# Certificate files will be at:
# /etc/letsencrypt/live/vicidial.yourdomain.com/fullchain.pem
# /etc/letsencrypt/live/vicidial.yourdomain.com/privkey.pem
```

**Automatic renewal:**
```bash
# Add to crontab
echo "0 3 * * * certbot renew --quiet --post-hook 'systemctl reload httpd && asterisk -rx \"module reload res_pjsip.so\"'" | sudo crontab -
```

The `--post-hook` restarts Apache and reloads PJSIP after certificate renewal, ensuring Asterisk picks up the new certificate without a service interruption.

**Important:** The SSL certificate must cover the hostname that agents use to access VICIdial. If agents connect to `vicidial.yourdomain.com`, the certificate must be issued for that exact hostname. Wildcard certificates (`*.yourdomain.com`) also work.

### Apache HTTPS Configuration

Configure Apache to serve VICIdial over HTTPS:

```apache
# /etc/httpd/conf.d/ssl.conf (or site-specific config)

<VirtualHost *:443>
    ServerName vicidial.yourdomain.com
    DocumentRoot /var/www/html

    SSLEngine on
    SSLCertificateFile /etc/letsencrypt/live/vicidial.yourdomain.com/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/vicidial.yourdomain.com/privkey.pem

    # Recommended SSL settings
    SSLProtocol all -SSLv3 -TLSv1 -TLSv1.1
    SSLCipherSuite HIGH:!aNULL:!MD5
    SSLHonorCipherOrder on

    # WebSocket proxy (if proxying through Apache)
    ProxyPass /ws wss://127.0.0.1:8089/ws
    ProxyPassReverse /ws wss://127.0.0.1:8089/ws

    # Standard VICIdial config
    <Directory /var/www/html>
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>

# Redirect HTTP to HTTPS
<VirtualHost *:80>
    ServerName vicidial.yourdomain.com
    Redirect permanent / https://vicidial.yourdomain.com/
</VirtualHost>
```

Enable required Apache modules:
```bash
# These are typically already available on AlmaLinux 9
sudo dnf install -y mod_ssl
sudo systemctl restart httpd
```

---

## PJSIP vs. SIP Channel Driver for WebRTC

VICIdial historically used the `chan_sip` channel driver. WebRTC support works with both `chan_sip` and `chan_pjsip`, but **PJSIP is the recommended choice** for WebRTC deployments:

| Feature | chan_sip | chan_pjsip (PJSIP) |
|---|---|---|
| WebSocket transport | Requires `res_http_websocket` + manual config | Native `res_pjsip_transport_websocket` |
| DTLS-SRTP | Limited support | Full native support |
| ICE negotiation | Partial | Full native support |
| Multiple transports per endpoint | No | Yes (UDP + WSS on same endpoint) |
| Asterisk development status | Deprecated (removed in Asterisk 21) | Actively maintained |
| VICIdial compatibility | Full (legacy) | Full (current) |

**If you're setting up a new VICIdial installation or upgrading, use PJSIP.** If you're running an existing `chan_sip` installation and adding WebRTC, you can either migrate to PJSIP or configure WebRTC on `chan_sip` — but be aware that `chan_sip` is deprecated and will not be available in future Asterisk versions.

The rest of this guide assumes PJSIP configuration. For `chan_sip` WebRTC setup, the concepts are similar but the configuration file format differs.

---

## Asterisk Configuration for WebRTC

### http.conf — Enable the HTTP Server and WebSocket

Asterisk's built-in HTTP server handles WebSocket connections. Configure it in `/etc/asterisk/http.conf`:

```ini
[general]
enabled=yes
bindaddr=0.0.0.0
bindport=8088
tlsenable=yes
tlsbindaddr=0.0.0.0:8089
tlscertfile=/etc/letsencrypt/live/vicidial.yourdomain.com/fullchain.pem
tlsprivatekey=/etc/letsencrypt/live/vicidial.yourdomain.com/privkey.pem
```

**Key settings:**
- `tlsenable=yes` — Enables HTTPS/WSS on port 8089
- `tlscertfile` and `tlsprivatekey` — Must point to the same SSL certificate used by Apache. Using a different certificate will cause certificate mismatch errors in the browser.
- `bindport=8088` — HTTP (non-TLS) on port 8088. Used for internal health checks but should not be exposed to the internet.
- Port 8089 (WSS) must be accessible from agents' networks. If agents are remote, this port must be open through your firewall.

Reload the HTTP module:
```bash
asterisk -rx "module reload res_http_websocket.so"
```

Verify the HTTP server is running:
```bash
asterisk -rx "http show status"
```

You should see both HTTP (8088) and HTTPS (8089) listeners active.

### pjsip.conf — WebSocket Transport and Endpoint Configuration

The PJSIP configuration is where most of the WebRTC-specific settings live. On VICIdial/ViciBox installations, PJSIP configuration may be managed through the database (via `table_config` entries) or directly in `/etc/asterisk/pjsip.conf`. Check your installation — ViciBox uses database-driven configuration by default.

#### Transport Configuration

Define a WebSocket Secure (WSS) transport:

```ini
; /etc/asterisk/pjsip.conf (or via VICIdial admin → PJSIP settings)

[transport-wss]
type=transport
protocol=wss
bind=0.0.0.0:8089
cert_file=/etc/letsencrypt/live/vicidial.yourdomain.com/fullchain.pem
priv_key_file=/etc/letsencrypt/live/vicidial.yourdomain.com/privkey.pem
; For self-signed certs during testing (not recommended for production):
; verify_client=no
; verify_server=no

; Also keep your UDP transport for SIP trunks
[transport-udp]
type=transport
protocol=udp
bind=0.0.0.0:5060
```

#### WebRTC Endpoint Template

Create a template for WebRTC agent endpoints:

```ini
[webrtc-endpoint](!)
type=endpoint
transport=transport-wss
context=default
disallow=all
allow=opus
allow=g722
allow=ulaw
dtls_auto_generate_cert=yes
webrtc=yes
; The webrtc=yes shorthand sets:
;   use_avpf=yes
;   media_encryption=dtls
;   dtls_verify=fingerprint
;   dtls_setup=actpass
;   ice_support=yes
;   media_use_received_transport=yes
;   rtcp_mux=yes
```

The `webrtc=yes` setting (available in Asterisk 16+) is a convenience option that configures all the necessary DTLS, ICE, AVPF, and RTCP multiplexing settings in a single line. Without it, you'd need to set each of those individually.

#### Agent Endpoint Configuration

For each VICIdial phone entry used by WebRTC agents, the PJSIP endpoint configuration is typically managed through VICIdial's admin interface under **Admin → Phones**. When creating a phone entry for a WebRTC agent:

- **Server:** Your Asterisk server IP
- **Dialplan Number:** The extension number (e.g., 8000-8999 range for WebRTC phones)
- **Protocol:** PJSIP
- **Registration:** Include WebRTC-specific parameters

VICIdial generates the PJSIP configuration entries from the phone table. After adding or modifying phone entries, reload PJSIP:

```bash
asterisk -rx "pjsip reload"
```

Verify the endpoint is configured:
```bash
asterisk -rx "pjsip show endpoints"
asterisk -rx "pjsip show endpoint [endpoint-name]"
```

---

## VICIdial Admin Configuration for WebRTC

### System Settings

In **Admin → System Settings**, configure WebRTC-related parameters:

- **WebRTC Phone Enabled:** Y
- **WebSocket URL:** `wss://vicidial.yourdomain.com:8089/ws`

The WebSocket URL is what ViciPhone uses to establish the SIP signaling connection. It must match your Asterisk `http.conf` TLS bind address and the hostname on your SSL certificate.

### Phone Configuration

For each remote agent using WebRTC, create a Phone entry under **Admin → Phones:**

- **Extension:** Unique extension number (e.g., 8001)
- **Server IP:** Your Asterisk/dialer server IP
- **Dialplan Number:** Same as extension
- **Phone Type:** Use the WebRTC-compatible type
- **Protocol:** PJSIP (or SIP if using chan_sip)
- **Registration String:** The PJSIP registration parameters. VICIdial auto-generates this for WebRTC-enabled phones.

### Agent Interface — ViciPhone

When an agent logs into the VICIdial agent interface on a WebRTC-enabled phone, ViciPhone automatically loads in the browser. The agent sees:

- A phone status indicator (registered/unregistered)
- Volume controls
- A dial pad (for manual dialing if enabled)
- Call state indicators

**ViciPhone v3.0** (available in recent SVN revisions) includes improvements over earlier versions:

- Better ICE candidate handling for NAT traversal
- Opus codec support for better audio quality at lower bandwidth
- Improved reconnection logic when WebSocket connections drop
- Audio level monitoring (helps diagnose one-way audio issues)
- Echo cancellation improvements

The agent doesn't need to do anything special — ViciPhone initializes automatically when they log in with a WebRTC-enabled phone assignment. The browser will prompt for microphone permission on first use; the agent must click "Allow."

---

## NAT Traversal: STUN and TURN Servers

NAT traversal is the single biggest source of WebRTC problems in VICIdial remote agent deployments. Here's why and how to solve it.

### The Problem

Most remote agents are behind residential NAT routers. Their computer has a private IP address (192.168.x.x or 10.x.x.x) that isn't directly reachable from the internet. WebRTC needs to establish a direct media path between the agent's browser and the Asterisk server. Without NAT traversal assistance, the browser and Asterisk can't figure out how to route audio packets to each other.

This manifests as:
- **One-way audio:** The agent can hear the caller, but the caller can't hear the agent (or vice versa)
- **No audio at all:** Call connects (SIP signaling works over WebSocket) but no audio flows
- **Audio works briefly then drops:** ICE candidates expire or NAT mappings time out

### STUN Servers

STUN (Session Traversal Utilities for NAT) servers help WebRTC clients discover their public IP address and the type of NAT they're behind. STUN is sufficient for most residential NAT setups (cone NAT, restricted cone NAT).

**Google's public STUN servers** (free, reliable, widely used):
```
stun:stun.l.google.com:19302
stun:stun1.l.google.com:19302
stun:stun2.l.google.com:19302
```

Configure STUN servers in VICIdial's WebRTC/ViciPhone settings. The specific location depends on your VICIdial version — check **Admin → System Settings** or the ViciPhone JavaScript configuration.

In Asterisk's `pjsip.conf`, configure the STUN server for ICE:

```ini
[global]
type=global

[system]
type=system

; Or in the transport section:
[transport-wss]
type=transport
protocol=wss
bind=0.0.0.0:8089
cert_file=/etc/letsencrypt/live/vicidial.yourdomain.com/fullchain.pem
priv_key_file=/etc/letsencrypt/live/vicidial.yourdomain.com/privkey.pem
external_media_address=YOUR_PUBLIC_IP
external_signaling_address=YOUR_PUBLIC_IP
```

The `external_media_address` and `external_signaling_address` settings tell Asterisk to advertise its public IP (not its private IP) in SIP/SDP messages. This is essential when Asterisk is behind a NAT or firewall.

### TURN Servers

STUN doesn't work for symmetric NAT — the kind found in some corporate firewalls and carrier-grade NAT (CGNAT) deployments. For these cases, you need a TURN (Traversal Using Relays around NAT) server.

TURN relays media traffic through a server when direct peer-to-peer connection fails. It's the fallback when STUN can't establish a direct path.

**Options for TURN:**

**1. Self-hosted coturn** (recommended for production):

```bash
# Install coturn on a separate server (or on your VICIdial web server)
sudo dnf install -y coturn

# Configure /etc/turnserver.conf
listening-port=3478
tls-listening-port=5349
listening-ip=0.0.0.0
relay-ip=YOUR_PUBLIC_IP
external-ip=YOUR_PUBLIC_IP
min-port=49152
max-port=65535
fingerprint
lt-cred-mech
realm=vicidial.yourdomain.com
server-name=vicidial.yourdomain.com
user=vicidial:your_turn_password
cert=/etc/letsencrypt/live/vicidial.yourdomain.com/fullchain.pem
pkey=/etc/letsencrypt/live/vicidial.yourdomain.com/privkey.pem
no-stdout-log
syslog

# Start coturn
sudo systemctl enable --now coturn
```

**Firewall requirements for coturn:**
- TCP/UDP 3478 (STUN/TURN)
- TCP 5349 (TURN over TLS)
- UDP 49152-65535 (media relay range)

**2. Cloud TURN services** (simpler, usage-based pricing):
- Twilio Network Traversal: $0.0005/minute
- Cloudflare Calls: Included in Workers plan
- Xirsys: Free tier for testing, paid for production

**Configure TURN in ViciPhone/VICIdial:**

The STUN/TURN configuration is typically set in the ViciPhone JavaScript initialization. Look for ICE server configuration in the agent interface code:

```javascript
// Typical ViciPhone ICE configuration
var iceServers = [
    { urls: "stun:stun.l.google.com:19302" },
    {
        urls: "turn:vicidial.yourdomain.com:3478",
        username: "vicidial",
        credential: "your_turn_password"
    },
    {
        urls: "turns:vicidial.yourdomain.com:5349",
        username: "vicidial",
        credential: "your_turn_password"
    }
];
```

In VICIdial, this configuration may be set through system settings or the ViciPhone configuration section. Check your version's documentation for the exact admin path.

---

## Codec Selection: Opus vs. G.722 vs. G.711

Codec choice directly impacts audio quality and bandwidth usage for remote agents. Here's what works best for WebRTC in VICIdial:

### Codec Comparison

| Codec | Bandwidth (per call) | Audio Quality | Latency | Browser Support | VICIdial Support |
|---|---|---|---|---|---|
| **Opus** | 6-128 kbps (adaptive) | Excellent (wideband/fullband) | Low | All modern browsers | Asterisk 13+ (requires codec_opus) |
| **G.722** | 64 kbps | Good (wideband) | Low | Most browsers | Native in Asterisk |
| **G.711 ulaw** | 64 kbps | Acceptable (narrowband) | Very low | All browsers | Native in Asterisk |
| **G.711 alaw** | 64 kbps | Acceptable (narrowband) | Very low | All browsers | Native in Asterisk |

### Recommended Configuration

**For most remote agent deployments: Opus first, G.722 fallback, ulaw as last resort.**

```ini
; In pjsip.conf endpoint configuration
disallow=all
allow=opus
allow=g722
allow=ulaw
```

**Why Opus is the best choice:**

1. **Adaptive bitrate.** Opus automatically adjusts its bitrate based on available bandwidth. If an agent's home internet hiccups, Opus drops to 6-12 kbps and maintains intelligible audio. G.722 and G.711 are fixed bitrate — if bandwidth drops, you get packet loss and audio breakup.

2. **Packet loss resilience.** Opus includes forward error correction (FEC) that can recover from 10-20% packet loss without audible degradation. Home internet connections have variable packet loss — Opus handles this gracefully.

3. **Lower bandwidth in normal conditions.** Opus at its default 24 kbps setting uses 60% less bandwidth than G.722 or G.711 while delivering similar or better perceived audio quality. For 50+ remote agents, the bandwidth savings are meaningful.

**Installing Opus on Asterisk (if not already present):**

```bash
# Check if codec_opus is available
asterisk -rx "core show codecs" | grep opus

# If not available, install the Digium Opus codec
# (Asterisk 18+ on ViciBox 12 typically includes it)
# For manual installation:
cd /usr/lib64/asterisk/modules/
wget https://downloads.digium.com/pub/telephony/codec_opus/asterisk-20.0/x86-64/codec_opus-20.0_current-x86_64.tar.gz
tar xzf codec_opus-*.tar.gz
cp codec_opus.so /usr/lib64/asterisk/modules/
asterisk -rx "module load codec_opus.so"
```

### Bandwidth Planning

Calculate total bandwidth requirements for your remote agent deployment:

| Agents | Opus (24 kbps avg) | G.722 (64 kbps) | G.711 (64 kbps) |
|---|---|---|---|
| 10 | ~480 kbps | ~1.28 Mbps | ~1.28 Mbps |
| 50 | ~2.4 Mbps | ~6.4 Mbps | ~6.4 Mbps |
| 100 | ~4.8 Mbps | ~12.8 Mbps | ~12.8 Mbps |
| 200 | ~9.6 Mbps | ~25.6 Mbps | ~25.6 Mbps |

These are media bandwidth only. Add 10-15% overhead for SIP signaling, WebSocket keepalives, and protocol overhead. Your server's internet connection must handle the total bandwidth for all concurrent WebRTC sessions.

**Per-agent minimum bandwidth requirement:** Each remote agent needs at least 1 Mbps upload and 1 Mbps download for reliable WebRTC audio. This is well within what any modern broadband or cellular connection provides, but can be a problem for agents on satellite internet, congested shared WiFi, or connections with high upload asymmetry (like some DSL plans with 256 kbps upload).

---

## Scaling WebRTC for 50+ Remote Agents

Running WebRTC at scale introduces challenges that don't exist with 5-10 agents:

### Server Capacity Planning

Each WebRTC call requires Asterisk to:
- Maintain a WebSocket connection (persistent TCP)
- Handle DTLS handshake and key exchange
- Process SRTP encryption/decryption for media
- Run ICE candidate negotiation

The SRTP encryption is the most CPU-intensive part. A standard VICIdial dialer server (8-core Xeon, 32GB RAM) can typically handle:
- **200-300 concurrent SIP calls** (no WebRTC, standard RTP)
- **100-150 concurrent WebRTC calls** (SRTP encryption + WebSocket overhead)

The roughly 2:1 ratio means WebRTC calls consume about double the server resources of standard SIP calls. For a 200-agent remote deployment, plan for dedicated WebRTC Asterisk servers in your [cluster architecture](/blog/vicidial-cluster-guide/).

### Cluster Architecture for WebRTC

In a multi-server VICIdial cluster, the recommended approach for large WebRTC deployments:

```
[Web Server] ← HTTPS → [Agents' Browsers]
     |
     ↓
[VICIdial DB Server (MySQL master)]
     |
     ↓
[Dialer Server 1] ← SIP trunks → [Carrier]
[Dialer Server 2] ← SIP trunks → [Carrier]
[WebRTC Server 1] ← WSS → [Remote Agents 1-75]
[WebRTC Server 2] ← WSS → [Remote Agents 76-150]
```

Separate your WebRTC (remote agent) traffic from your outbound dialing traffic. The dialer servers handle SIP trunk connections and predictive dialing algorithms. The WebRTC servers handle agent audio connections. Both connect to the central MySQL database.

This separation ensures that SRTP encryption load from 150 remote agents doesn't degrade your outbound dialing capacity.

### WebSocket Connection Limits

Each WebRTC agent maintains a persistent WebSocket connection to Asterisk. Asterisk's HTTP server has connection limits that may need tuning for large deployments:

In `/etc/asterisk/http.conf`:
```ini
[general]
; Session limit - increase for large WebRTC deployments
session_limit=200
; Session inactivity timeout - keep WebSocket connections alive
session_inactivity=300
```

Also ensure your operating system's file descriptor limits are adequate:

```bash
# Check current limits
ulimit -n

# If below 65535, increase in /etc/security/limits.conf
* soft nofile 65535
* hard nofile 65535
```

And increase Asterisk's connection limit:
```bash
# In /etc/asterisk/asterisk.conf
[options]
maxcalls=1024
```

### Load Balancing WebRTC Connections

For 100+ WebRTC agents, distribute connections across multiple Asterisk servers. VICIdial's phone configuration assigns each phone to a specific server IP, so you can distribute agents across WebRTC servers at the phone entry level.

Alternatively, use an NGINX or HAProxy load balancer for WebSocket connections:

```nginx
# /etc/nginx/conf.d/vicidial-wss.conf
upstream asterisk_wss {
    ip_hash;  # Sticky sessions - same agent always hits same server
    server asterisk-webrtc-1:8089;
    server asterisk-webrtc-2:8089;
}

server {
    listen 8089 ssl;
    server_name vicidial.yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/vicidial.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/vicidial.yourdomain.com/privkey.pem;

    location /ws {
        proxy_pass https://asterisk_wss/ws;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_read_timeout 86400;
    }
}
```

The `ip_hash` directive ensures sticky sessions — once an agent connects to a specific Asterisk server, all subsequent WebSocket connections from the same IP go to the same server. This is important because SIP registration state must stay on one server.

---

## Troubleshooting Common WebRTC Issues

### One-Way Audio

**Symptoms:** Agent can hear the caller but the caller can't hear the agent, or vice versa.

**Causes and fixes:**

1. **Missing `external_media_address` in PJSIP transport.** If Asterisk is behind a NAT/firewall and doesn't know its public IP, it advertises its private IP in SDP. The remote browser can't route media to a private IP.

   **Fix:** Add `external_media_address=YOUR_PUBLIC_IP` to your WSS transport in `pjsip.conf`.

2. **STUN/TURN not configured or unreachable.** The browser can't determine its public IP for ICE negotiation.

   **Fix:** Configure STUN servers in ViciPhone's ICE configuration. Test STUN reachability with: `stunclient stun.l.google.com 19302`

3. **Symmetric NAT without TURN.** Some carrier-grade NAT or strict corporate firewalls block direct media paths even with STUN.

   **Fix:** Deploy a TURN server (coturn) and configure it in ViciPhone's ICE servers.

4. **Firewall blocking RTP port range.** Media packets use high-numbered UDP ports (typically 10000-20000 on Asterisk). If these are blocked by the server's firewall, media can't flow.

   **Fix:** Open UDP ports 10000-20000 in your firewall:
   ```bash
   sudo firewall-cmd --permanent --add-port=10000-20000/udp
   sudo firewall-cmd --reload
   ```

5. **Codec mismatch.** The browser and Asterisk can't agree on a codec. This typically shows as no audio rather than one-way audio.

   **Fix:** Ensure the PJSIP endpoint allows codecs that WebRTC supports (opus, g722, ulaw).

### Connection Drops / WebSocket Disconnects

**Symptoms:** Agent periodically loses connection, phone shows "unregistered," calls drop mid-conversation.

**Causes and fixes:**

1. **SSL certificate expired or mismatched.** The WebSocket connection fails TLS handshake.

   **Fix:** Check certificate expiry: `openssl x509 -in /etc/letsencrypt/live/vicidial.yourdomain.com/fullchain.pem -noout -enddate`. Renew if expired. Verify the hostname matches.

2. **WebSocket keepalive timeout.** The persistent WebSocket connection times out due to idle detection at the firewall, load balancer, or Asterisk level.

   **Fix:** Configure appropriate keepalive intervals. In Asterisk `http.conf`: `session_inactivity=300`. In any intermediary proxy (NGINX, HAProxy): set `proxy_read_timeout 86400;` for WebSocket locations.

3. **Agent's internet connection dropping.** Home WiFi instability, ISP issues, or VPN interference.

   **Fix:** This is outside your control, but ViciPhone v3.0 includes automatic reconnection logic. Ensure agents use wired Ethernet when possible. Run a network quality test: have agents visit `https://speed.cloudflare.com` and verify they have consistent latency below 100ms and packet loss below 1%.

4. **Asterisk HTTP server overwhelmed.** Too many simultaneous WebSocket connections for the server's configuration.

   **Fix:** Increase `session_limit` in `http.conf`. Check system resource usage (`top`, `htop`) for CPU or memory exhaustion. Consider distributing WebRTC agents across multiple Asterisk servers.

### Certificate Errors in Browser

**Symptoms:** Browser console shows SSL/TLS errors, ViciPhone fails to connect, microphone permission prompt doesn't appear.

**Causes and fixes:**

1. **Self-signed certificate.** Browsers block WebRTC on pages served with untrusted certificates.

   **Fix:** Use a CA-signed certificate (Let's Encrypt is free). Self-signed certificates are not viable for production WebRTC deployments.

2. **Certificate hostname mismatch.** The certificate was issued for a different hostname than what the agent is connecting to.

   **Fix:** Ensure the certificate covers the exact hostname in the browser's URL bar. If agents connect via IP address, you need a certificate for that IP (or better, use a hostname).

3. **Mixed content.** The page is served over HTTPS but ViciPhone is trying to connect to a non-TLS WebSocket (ws:// instead of wss://).

   **Fix:** Verify the WebSocket URL in VICIdial system settings uses `wss://` not `ws://`.

### Echo / Audio Feedback

**Symptoms:** Agent or caller hears echo of their own voice.

**Causes and fixes:**

1. **Agent using speakers instead of headset.** Audio from speakers is picked up by the microphone and sent back.

   **Fix:** Require agents to use headsets. USB headsets with built-in sound cards provide the best echo cancellation. Bluetooth headsets introduce additional latency and are not recommended for call center work.

2. **Browser echo cancellation not engaging.** Chrome and Firefox have built-in acoustic echo cancellation (AEC), but it can fail with certain audio hardware.

   **Fix:** Ensure agents use Chrome (best WebRTC implementation). In ViciPhone configuration, verify that audio constraints include `echoCancellation: true`.

3. **Asterisk-side echo.** The SIP trunk connection to the carrier introduces echo (common with poorly configured T1/PRI connections, less common with well-configured SIP trunks).

   **Fix:** Enable echo cancellation in Asterisk: add `echocancel=yes` and `echocancelwhenbridged=yes` in the channel configuration.

---

## Security Considerations

WebRTC adds network attack surface to your VICIdial deployment. Mitigate with:

### Transport Security

- **TLS everywhere.** HTTPS for the web interface, WSS for WebSocket signaling, DTLS-SRTP for media. No unencrypted transport paths.
- **Certificate pinning.** Consider certificate pinning in ViciPhone for additional protection against man-in-the-middle attacks.
- **Cipher suite hardening.** Disable weak ciphers in both Apache and Asterisk SSL configurations. TLS 1.2 minimum, TLS 1.3 preferred.

### Access Control

- **Firewall rules.** Only expose the ports that need to be public: 443 (HTTPS), 8089 (WSS), 3478 (STUN/TURN), and your RTP port range. Block everything else.
- **Asterisk ACLs.** Configure PJSIP ACLs to restrict WebSocket connections to expected IP ranges (if agents use fixed IPs or VPNs):

```ini
[acl]
type=acl
permit=0.0.0.0/0  ; Allow from anywhere (for remote agents)
; Or restrict to known ranges:
; permit=203.0.113.0/24
; deny=0.0.0.0/0
```

- **Rate limiting.** Protect against SIP brute-force attacks on the WebSocket port. Use fail2ban with Asterisk log monitoring:

```bash
# /etc/fail2ban/jail.local
[asterisk]
enabled = true
port = 8089,5060
filter = asterisk
logpath = /var/log/asterisk/messages
maxretry = 5
bantime = 3600
```

### SRTP Enforcement

Never allow unencrypted RTP for WebRTC connections:

```ini
; In pjsip.conf endpoint template
media_encryption=dtls
media_encryption_optimistic=no  ; Fail the call if encryption can't be negotiated
```

The `media_encryption_optimistic=no` setting ensures that if DTLS negotiation fails for any reason, the call is rejected rather than falling back to unencrypted RTP. This prevents accidental cleartext media transmission.

---

## Agent Setup Requirements and Best Practices

Provide these requirements to remote agents before they go live:

### Minimum Agent Requirements

- **Browser:** Chrome 90+ (recommended), Firefox 85+, or Edge 90+. Safari support is limited — avoid it for production.
- **Internet:** 5 Mbps download, 1 Mbps upload minimum. Wired Ethernet strongly preferred over WiFi.
- **Headset:** USB headset with noise cancellation. Recommended: Jabra Evolve2 40, Plantronics Blackwire 3220, or similar. Avoid Bluetooth.
- **Operating system:** Windows 10/11, macOS 12+, or Linux with Chrome. Chromebooks work.
- **Microphone permissions:** Must allow browser microphone access for vicidial.yourdomain.com.

### Agent Troubleshooting Checklist

When an agent reports audio issues, walk through:

1. **Is ViciPhone showing "Registered"?** If not, it's a WebSocket/signaling issue. Check certificate, firewall, connectivity.
2. **Did they allow microphone permission?** Check browser → Site Settings → Microphone for your VICIdial URL.
3. **Are they using a headset?** Speakers + built-in microphone = echo and feedback.
4. **Are they on WiFi or Ethernet?** WiFi packet loss causes audio breakup. Switch to Ethernet.
5. **Run a speed test.** `speed.cloudflare.com` — look for latency under 100ms and packet loss under 1%.
6. **Clear browser cache and restart.** WebRTC sessions can accumulate stale ICE candidates.
7. **Test with a different browser.** If Chrome fails, try Firefox (or vice versa) to isolate the issue.

---

## Frequently Asked Questions

### Do agents need to install anything for VICIdial WebRTC?

No. ViciPhone runs entirely in the browser — no plugins, no extensions, no desktop software. The agent navigates to the VICIdial agent interface URL (over HTTPS), logs in with their credentials, and ViciPhone initializes automatically. The only agent action required is clicking "Allow" when the browser requests microphone permission on first use.

### Can I use WebRTC and traditional SIP phones in the same VICIdial deployment?

Yes. VICIdial supports mixed phone environments. Some agents can use WebRTC (ViciPhone in browser), while others use SIP desk phones, SIP softphones, or IAX phones. Each phone is configured independently in Admin → Phones. The dialer treats all phone types identically — it doesn't care whether the agent's audio is coming from WebRTC or a Polycom desk phone.

### What's the maximum number of WebRTC agents VICIdial can support?

On a single Asterisk server (8-core, 32GB RAM), expect 100-150 concurrent WebRTC sessions before performance degrades. For larger deployments, distribute WebRTC agents across multiple Asterisk servers in a [cluster configuration](/blog/vicidial-cluster-guide/). We've run VICIdial deployments with 300+ concurrent WebRTC agents using 3 dedicated WebRTC Asterisk servers behind a WebSocket load balancer.

### Why does ViciPhone show "Unregistered" even though I can access the VICIdial web interface?

The web interface (HTTPS) and ViciPhone (WSS) are separate connections. "Unregistered" means the WebSocket connection to Asterisk failed. Check: (1) Is port 8089 open on the server firewall? (2) Is `http.conf` configured with `tlsenable=yes`? (3) Does the SSL certificate in `http.conf` match the one used by Apache? (4) Is the WebSocket URL in VICIdial system settings correct (`wss://hostname:8089/ws`)? (5) Run `asterisk -rx "http show status"` to verify the HTTPS listener is active.

### Is WebRTC audio quality good enough for a production call center?

With Opus codec, WebRTC audio quality exceeds traditional G.711 SIP calls. Opus provides wideband (16kHz) or fullband (48kHz) audio compared to G.711's narrowband (8kHz). The key variable is the agent's internet connection quality — WebRTC audio is excellent on any connection with under 100ms latency and under 1% packet loss. That covers virtually all broadband connections. Agents on satellite internet (500ms+ latency) or severely congested connections will have problems regardless of codec.

### Do I need a TURN server, or is STUN enough?

For most remote agents on residential internet (cable, fiber, DSL), STUN is sufficient. STUN handles cone NAT and restricted NAT, which covers 80-90% of home networking equipment. You need a TURN server if: (1) agents are behind corporate firewalls with symmetric NAT, (2) agents are on carrier-grade NAT (CGNAT) — common with some mobile carriers and fixed wireless ISPs, or (3) you're seeing persistent one-way audio that STUN doesn't resolve. Our recommendation: deploy a TURN server (coturn) as insurance. It's cheap to run and only gets used when STUN fails, so it adds almost zero cost in normal operation.

### Can I use WebRTC for outbound calls or only inbound?

WebRTC in VICIdial works for all call types — outbound predictive/progressive, inbound, manual dial, transfers, and conferences. The agent's WebRTC connection is their phone endpoint. Once registered, it functions identically to a SIP phone for all VICIdial operations. The predictive dialer connects answered calls to the agent's WebRTC session the same way it would connect to a SIP extension.

### What browsers work best with VICIdial WebRTC?

Chrome is the best and most widely tested browser for WebRTC with VICIdial. Google maintains the WebRTC implementation in Chrome and it has the most reliable ICE negotiation, echo cancellation, and audio processing. Firefox works well as a secondary option. Edge (Chromium-based) works the same as Chrome. Safari has improved its WebRTC support significantly but still has occasional issues with SRTP negotiation and ICE candidate gathering — use it only if Chrome and Firefox are not options.

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-webrtc-setup).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
