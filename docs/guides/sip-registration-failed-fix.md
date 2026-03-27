# SIP Registration Failed: Every Error Code Explained With Fixes

**Last updated: March 2026 | Reading time: ~22 minutes**

You're looking at the Asterisk CLI and you see it. Over and over:

```
[2026-03-26 08:14:32] NOTICE[12847]: chan_sip.c:24022 handle_response_register: Registration for 'trunk_voipcarrier' failed
```

Or maybe you're not even using Asterisk. Maybe it's a Grandstream phone, a softphone, FreePBX, or some other SIP endpoint. The message is always some variation of the same thing: **SIP registration failed**.

This is the most common SIP error in existence. It has about 15 different causes. The error message itself tells you almost nothing. The actual answer is hiding in the SIP response code, and most admin interfaces don't show it to you.

This guide covers every SIP registration failure mode I've encountered across thousands of VoIP deployments. Real packet captures, real Asterisk output, real fixes.

---

## How SIP Registration Works (30-Second Version)

Before troubleshooting, you need to know what's happening behind the scenes. SIP registration is a four-step process:

```
Endpoint                          Registrar (carrier/PBX)
   │                                    │
   │──── REGISTER (no auth) ──────────→│
   │                                    │
   │←─── 401 Unauthorized ─────────────│
   │     (includes nonce/challenge)     │
   │                                    │
   │──── REGISTER (with digest auth) ─→│
   │     (username + MD5 response)      │
   │                                    │
   │←─── 200 OK ───────────────────────│
   │     (registered, expires: 3600)    │
```

Step 1: Your phone/PBX sends a REGISTER request without credentials.
Step 2: The registrar replies with 401 Unauthorized, including a challenge (nonce).
Step 3: Your device computes an MD5 digest from the nonce + your password and sends a second REGISTER.
Step 4: Registrar verifies the digest and replies 200 OK. You're registered.

If anything goes wrong in steps 1-4, registration fails. The response code tells you WHERE it failed. Let's go through every one.

---

## Capturing the SIP Response Code

You can't fix what you can't see. Most SIP devices just show "registration failed" without the response code. Here's how to get it.

### On Asterisk/VICIdial

```bash
# Live SIP debugging — shows every SIP message
asterisk -rx "sip set debug on"

# Check current registration status of all peers
asterisk -rx "sip show peers"

# Check a specific peer's registration
asterisk -rx "sip show peer trunk_name"

# For PJSIP (Asterisk 16+, ViciBox 12+)
asterisk -rx "pjsip set logger on"
asterisk -rx "pjsip show registrations"
```

### On the Network (Any Device)

```bash
# sngrep — the best SIP debugging tool
sngrep -d eth0 port 5060

# tcpdump if sngrep isn't available
tcpdump -i eth0 -n -s 0 port 5060 -w /tmp/sip.pcap
# Then open in Wireshark on your local machine
```

sngrep is invaluable. It shows SIP message flows in real time with a curses UI. If you don't have it installed:

```bash
# AlmaLinux / Rocky / CentOS Stream
dnf install sngrep

# Debian / Ubuntu
apt install sngrep

# OpenSuSE (ViciBox)
zypper install sngrep
```

---

## Every SIP Registration Error Code

### 400 Bad Request

**What the registrar is saying:** "Your REGISTER message is malformed. I can't parse it."

**Common causes:**
- Malformed SIP URI in the From or To header
- Invalid characters in the username or domain
- Broken SIP stack on the endpoint (rare, but happens with cheap hardware)
- SIP ALG on a NAT router rewriting headers incorrectly

**The fix:**

First, disable SIP ALG on your router/firewall. SIP ALG is supposed to help SIP through NAT. In reality, it corrupts SIP messages about 80% of the time.

```bash
# Check if SIP ALG is active (Linux iptables)
lsmod | grep nf_nat_sip

# Disable it
modprobe -r nf_nat_sip nf_conntrack_sip
echo "blacklist nf_nat_sip" >> /etc/modprobe.d/blacklist.conf
echo "blacklist nf_conntrack_sip" >> /etc/modprobe.d/blacklist.conf
```

If it's not SIP ALG, check the From and To headers in your SIP trace. The domain portion needs to match what the registrar expects. On Asterisk:

```ini
; sip.conf — make sure fromdomain matches the carrier's expected domain
[trunk_carrier]
type=peer
host=sip.carrier.com
fromdomain=sip.carrier.com  ; This matters
fromuser=your_account_id
```

### 401 Unauthorized (After Second REGISTER)

**What the registrar is saying:** "You sent credentials, but they're wrong."

This is the classic authentication failure. The first 401 is normal (it's the challenge). A 401 after you've responded to the challenge means your credentials don't match.

**Common causes:**
- Wrong password (the #1 cause, by far)
- Wrong username (SIP auth username ≠ SIP URI username on many carriers)
- Wrong realm/domain in the digest calculation
- Password has special characters that aren't being escaped correctly
- Carrier changed credentials and didn't tell you (or it went to the spam folder)

**The fix:**

In Asterisk, verify your trunk credentials:

```bash
# Check what Asterisk thinks the credentials are
asterisk -rx "sip show peer trunk_name" | grep -i "secret\|username\|auth"
```

Cross-reference with your carrier portal. Pay attention to:
- **Auth username** vs **SIP username** — many carriers have two different usernames. The auth username goes in the REGISTER challenge response. The SIP username goes in the From header.
- **Realm** — some carriers require a specific realm in the digest. Check the 401 response to see what realm they're challenging with.

In the VICIdial admin GUI, go to **Carriers** and verify the trunk configuration. The username and password fields need to match exactly what your carrier issued.

```ini
; sip.conf
[trunk_carrier]
type=peer
host=sip.carrier.com
username=auth_username      ; Often different from the account ID
secret=yourpassword123
fromuser=sip_username       ; The From header username
```

### 403 Forbidden

**What the registrar is saying:** "I know who you are, but you're not allowed to register."

**Common causes:**
- IP address not whitelisted (carrier requires IP auth + registration)
- Account suspended or disabled
- Registration from a blocked region/country
- Exceeded maximum registration count (too many devices on one account)
- Anti-fraud system triggered (registrations from multiple IPs simultaneously)

**The fix:**

Log into your carrier's portal and check:
1. Account status — is it active?
2. IP whitelist — does it include your server's public IP?
3. Concurrent registration limit — have you exceeded it?

If your server is behind NAT, the carrier sees your public IP, not your private one. Make sure the public IP is whitelisted.

```bash
# Find your server's public IP
curl -s ifconfig.me
```

For VICIdial systems where the Asterisk server is behind a firewall, verify `externip` in sip.conf:

```ini
; sip.conf
externip=YOUR.PUBLIC.IP.ADDRESS
localnet=192.168.1.0/255.255.255.0
localnet=10.0.0.0/255.0.0.0
```

### 404 Not Found

**What the registrar is saying:** "The SIP address you're trying to register doesn't exist on my system."

**Common causes:**
- Wrong registrar domain/hostname
- Wrong SIP username (the user portion of the SIP URI)
- Account deleted on the carrier side
- Typo in the `register =>` line

**The fix:**

Check your registration string. In Asterisk chan_sip:

```ini
; sip.conf
register => username:password@sip.carrier.com/inbound_context
```

The hostname after `@` needs to resolve to the carrier's registrar. The username needs to be a valid account on that registrar. A 404 means one of these is wrong.

```bash
# Verify DNS resolution
dig sip.carrier.com

# Some carriers use SRV records
dig _sip._udp.carrier.com SRV
```

### 407 Proxy Authentication Required

**What the registrar is saying:** "You need to authenticate with my proxy, not just the registrar."

This is similar to 401 but for proxy authentication. Some SIP networks route REGISTER through a proxy that requires its own authentication step.

**Common causes:**
- Same as 401 (wrong credentials) but for the proxy auth layer
- Missing `outboundproxy` configuration
- Carrier uses an SBC (Session Border Controller) that requires proxy auth

**The fix:**

Add proxy auth credentials in Asterisk:

```ini
; sip.conf
[trunk_carrier]
type=peer
host=sip.carrier.com
outboundproxy=proxy.carrier.com
username=your_username
secret=your_password
```

### 408 Request Timeout

**What the registrar is saying:** Nothing. That's the problem. The registrar didn't respond at all, and your device gave up waiting.

**Common causes:**
- Firewall blocking UDP port 5060 outbound
- Carrier's registrar is down
- DNS not resolving (REGISTER sent to wrong IP or not sent at all)
- Network path issue (routing, ISP problem)
- SIP ALG or NAT breaking the return path so the response can't get back
- Wrong transport (sending UDP when carrier expects TCP, or vice versa)

This is the most frustrating error because it means your REGISTER went into a black hole. It could be a firewall issue, a DNS issue, or a network issue. The registrar may have never seen your request.

**The fix:**

Work from the bottom up:

```bash
# 1. Can you reach the registrar at all?
ping sip.carrier.com

# 2. Can you reach port 5060?
nc -zuv sip.carrier.com 5060

# 3. Does DNS resolve correctly?
dig sip.carrier.com

# 4. Is your firewall allowing outbound UDP 5060?
iptables -L -n | grep 5060

# 5. Can you see any SIP traffic at all?
tcpdump -i eth0 -n port 5060 -c 10
```

If tcpdump shows REGISTER going out but no response coming back, the problem is either the carrier's firewall (they need to whitelist your IP) or a NAT/ALG issue mangling the return path.

Check your transport. Some carriers require TCP instead of UDP:

```ini
; sip.conf
[trunk_carrier]
type=peer
host=sip.carrier.com
transport=tcp  ; Try this if UDP isn't working
```

### 423 Interval Too Brief

**What the registrar is saying:** "You're trying to register with an expiry time that's too short. I require a longer registration interval."

**Common causes:**
- Your device is set to register every 30 or 60 seconds
- The carrier requires a minimum registration interval (usually 120 or 300 seconds)

**The fix:**

Increase the registration expiry:

```ini
; sip.conf
[trunk_carrier]
type=peer
defaultexpiry=300      ; 5 minutes
```

Or in PJSIP:

```ini
; pjsip.conf
[trunk_carrier]
type=registration
expiration=300
```

### 480 Temporarily Unavailable

**What the registrar is saying:** "I exist and I heard you, but I can't handle your registration right now."

**Common causes:**
- Carrier is overloaded
- Maintenance window
- Rate limiting (you're sending too many REGISTER requests)

**The fix:**

Wait and retry. If it persists, contact your carrier. If you're sending rapid-fire registration attempts (because your system keeps retrying after each failure), you might be getting rate-limited. Increase your retry interval:

```ini
; sip.conf
registertimeout=60     ; Wait 60 seconds between retry attempts
registerattempts=0     ; Keep retrying forever (0 = infinite)
```

### 500 Internal Server Error

**What the registrar is saying:** "Something broke on my end."

**Common causes:**
- Bug in the carrier's SBC or registrar software
- Database issue on the carrier side
- Malformed data in your REGISTER that triggers a server bug

**The fix:**

This is the carrier's problem. Contact their support with your SIP trace showing the 500 response. Include the Call-ID header so they can track it in their logs.

If you're getting 500s intermittently, it might be a load issue on their side. Configure a failover trunk so your system can switch carriers when one is having problems.

### 503 Service Unavailable

**What the registrar is saying:** "I'm alive but not accepting registrations. Maybe I'm in maintenance, overloaded, or you're hitting a rate limit."

**Common causes:**
- Carrier at capacity
- Server maintenance
- Your account has been soft-disabled (carrier is rejecting your registrations but the account technically exists)
- Load balancer has no healthy backends

**The fix:**

Same as 500 — carrier-side issue. The difference is 503 is more likely temporary. Check your carrier's status page. Set up failover trunks.

In VICIdial, configure a secondary carrier through the admin GUI under **Carriers**. Set up dial patterns that try the primary trunk first and fall back to the secondary.

---

## SIP Registration Over TLS (SIPS)

If your carrier supports SIP over TLS (port 5061), use it. It encrypts the signaling path, prevents credential sniffing, and eliminates a class of NAT/ALG issues because TLS traffic is less likely to be mangled by middleboxes.

```ini
; sip.conf
[trunk_carrier]
type=peer
host=sip.carrier.com
transport=tls
port=5061
tlscafile=/etc/pki/tls/certs/ca-bundle.crt
tlsenable=yes
tlsverify=yes
```

For PJSIP (Asterisk 16+ / ViciBox 12.0.2):

```ini
; pjsip.conf
[transport-tls]
type=transport
protocol=tls
bind=0.0.0.0:5061
cert_file=/etc/asterisk/keys/asterisk.crt
priv_key_file=/etc/asterisk/keys/asterisk.key
ca_list_file=/etc/pki/tls/certs/ca-bundle.crt
method=tlsv1_2

[trunk_carrier]
type=registration
transport=transport-tls
outbound_auth=trunk_carrier_auth
server_uri=sips:sip.carrier.com
client_uri=sips:your_number@sip.carrier.com
```

Common TLS registration failures:
- **Certificate verification failure** — The carrier's cert is self-signed or your CA bundle is outdated. Either add their cert to your trust store or set `tlsverify=no` (less secure but functional).
- **Wrong TLS version** — Some carriers still require TLS 1.2. Others have upgraded to 1.3. Check with your carrier.
- **Port 5061 blocked** — Your firewall allows 5060 (UDP) but blocks 5061 (TCP/TLS).

---

## SIP Registration Behind NAT: The Eternal Problem

90% of SIP registration failures in the real world are NAT-related. Here's why.

When your Asterisk server sends a REGISTER from behind NAT, the SIP message contains the private IP (e.g., 192.168.1.100) in the Contact header. The carrier sends the 200 OK response to your public IP (which your router forwards correctly). But when the carrier later tries to send an INVITE to your registered contact address, it tries to reach 192.168.1.100 — which is unreachable from the internet.

The fix involves multiple layers:

```ini
; sip.conf
nat=force_rport,comedia   ; Force rport and symmetric RTP
externip=YOUR.PUBLIC.IP   ; Tell Asterisk what your public IP is
localnet=192.168.1.0/255.255.255.0  ; Define your private network
qualify=yes               ; Send OPTIONS keepalives to maintain NAT mapping
qualifyfreq=30            ; Every 30 seconds
```

The `qualify=yes` with `qualifyfreq=30` sends SIP OPTIONS messages every 30 seconds, which keeps the NAT mapping alive on your router. Without this, the NAT mapping expires (typically after 30-60 seconds), and inbound SIP messages can't reach your server.

For firewalls that do connection tracking, you also need to keep the UDP "connection" alive. The qualify setting handles this for SIP, but you also need to consider RTP (media) ports:

```bash
# Open RTP port range in firewall
iptables -A INPUT -p udp --dport 10000:20000 -j ACCEPT
```

---

## PJSIP vs. chan_sip Registration Differences

If you're on Asterisk 18+ (ViciBox 12.0.2), you have both chan_sip and PJSIP available. VICIdial traditionally uses chan_sip, but PJSIP is the future and handles some registration scenarios better.

Key differences for registration:

| Feature | chan_sip | PJSIP |
|---------|---------|-------|
| Config file | sip.conf | pjsip.conf |
| Registration syntax | `register =>` line | Separate `registration` object |
| Multiple registrations to same host | Awkward | Clean, separate objects |
| TLS support | Basic | Full, modern |
| DNS SRV | Limited | Full support |
| Outbound proxy | Basic | Full support |

If you're getting registration failures with chan_sip that you can't resolve, try the same trunk in PJSIP. We've seen cases where chan_sip's digest authentication implementation doesn't handle certain carrier challenge formats correctly, while PJSIP handles them fine.

---

## Automated Registration Monitoring

Don't wait for agents to tell you the phones aren't working. Monitor registration status automatically.

```bash
#!/bin/bash
# sip-reg-check.sh — Run from cron every 5 minutes
PEERS=$(asterisk -rx "sip show peers" | grep -c "UNREACHABLE\|UNKNOWN")
if [ "$PEERS" -gt 0 ]; then
    echo "WARNING: $PEERS SIP peers unreachable" | mail -s "SIP Registration Alert" admin@yourcompany.com
    # Log for trending
    echo "$(date): $PEERS peers down" >> /var/log/sip-registration-monitor.log
fi
```

For VICIdial specifically, you can also monitor from the Real-Time Report. If trunks show as unregistered, the **Server Stats** page in the admin will show carrier status.

---

## Quick Reference: Registration Failure Cheat Sheet

| Code | Meaning | First Thing to Check |
|------|---------|---------------------|
| 400 | Bad Request | Disable SIP ALG, check SIP URI format |
| 401 | Auth Failed | Password, auth username vs SIP username |
| 403 | Forbidden | IP whitelist, account status |
| 404 | Not Found | Username, registrar hostname |
| 407 | Proxy Auth | Outbound proxy credentials |
| 408 | Timeout | Firewall, DNS, network path |
| 423 | Interval Too Short | Increase registration expiry |
| 480 | Temporarily Unavailable | Carrier issue, retry later |
| 500 | Server Error | Carrier-side bug, contact support |
| 503 | Service Unavailable | Carrier overloaded, use failover |

---

## When to Stop Debugging and Call the Carrier

If you've verified:
- Credentials match the carrier portal exactly
- DNS resolves correctly
- Firewall allows SIP traffic bidirectionally
- SIP ALG is disabled
- NAT is configured properly
- sngrep shows your REGISTER going out and a non-401 error coming back

Then the problem is on the carrier side. Call them. Give them:
1. Your account ID
2. The source IP of your REGISTER attempts
3. The exact response code you're getting
4. A pcap file if possible

Good carriers will resolve this in minutes. Bad carriers will tell you to reboot your PBX.

---

**Tired of debugging SIP registration at 2 AM?** ViciStack builds and manages VICIdial infrastructure on bare metal with pre-configured, tested SIP trunks and automatic failover. Registration monitoring is built into our standard deployment. Our engagement starts at $5K ($1K deposit, $4K on completion) and includes full SIP trunk configuration with your carriers. [Talk to us](/contact/) before the next outage.

---

*Related: [VICIdial SIP Troubleshooting](/blog/vicidial-sip-troubleshooting/) | [VICIdial Asterisk Configuration](/blog/vicidial-asterisk-configuration/) | [VICIdial SIP Trunk Failover](/blog/vicidial-sip-trunk-failover/) | [Asterisk PJSIP TLS Guide](/blog/asterisk-pjsip-tls-openssl3-guide/)*

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/sip-registration-failed-fix).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
