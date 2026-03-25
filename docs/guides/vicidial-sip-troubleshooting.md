# VICIdial SIP Troubleshooting: Fix One-Way Audio, 503s, and Registration Failures

**You're staring at the Asterisk CLI. Calls connect but nobody can hear anything. Or worse — they hear you but you can't hear them. Welcome to SIP troubleshooting.**

This isn't a theoretical guide. Every problem here came from a real VICIdial system we fixed. Packet captures included.

## The Debugging Toolkit

Before touching any config, get your tools ready:

```bash
# Live SIP packet capture — the single most useful command
sngrep -d eth0 port 5060

# If sngrep isn't installed
tcpdump -i eth0 -n -s 0 port 5060 -w /tmp/sip-capture.pcap

# Check current trunk registration
asterisk -rx "sip show peers"

# Active channels with codec info
asterisk -rx "core show channels verbose"

# RTP debug — shows media flow in real time
asterisk -rx "rtp set debug on"
```

`sngrep` is the real MVP here. It shows the full SIP ladder diagram — INVITE, 100 Trying, 180 Ringing, 200 OK, ACK, BYE — in a visual flow. Install it: `yum install sngrep` or `apt install sngrep`.

## Problem 1: One-Way Audio

This is the most common SIP issue in VICIdial deployments, and it's almost always NAT.

**Symptoms:**
- Agent hears the customer, customer hears silence (or vice versa)
- Works fine on internal calls, breaks on outbound through the carrier
- Intermittent — some calls fine, others one-way

**Root cause:** The SDP body in the INVITE contains a private IP (10.x, 172.16.x, 192.168.x) instead of the public IP. The carrier sends RTP media to the private IP, which is unreachable.

**Check it:**

```bash
# Look at the SDP in sngrep — find the line starting with "c="
# BAD: c=IN IP4 10.0.0.50
# GOOD: c=IN IP4 203.0.113.50

# Also check the "o=" line for the same issue
```

**Fix in sip.conf:**

```ini
[general]
externaddr=203.0.113.50       ; Your public IP
localnet=10.0.0.0/8           ; Your private network
localnet=172.16.0.0/12
localnet=192.168.0.0/16
nat=force_rport,comedia        ; Force symmetric RTP
```

If your public IP is dynamic, use `externhost` instead:

```ini
externhost=sip.yourdomain.com
externrefresh=60
```

**The other cause:** Firewall blocking RTP ports. VICIdial uses UDP ports 10000-20000 for RTP media by default.

```bash
# Verify RTP ports are open
iptables -L -n | grep -E "100[0-9]{2}|1[0-9]{4}|20000"

# If using firewalld
firewall-cmd --list-ports | grep udp
```

Open them:

```bash
iptables -A INPUT -p udp --dport 10000:20000 -j ACCEPT
```

## Problem 2: 503 Service Unavailable

**Symptoms:**
- Calls fail immediately with 503
- `sip show peers` shows the trunk as "UNREACHABLE" or "UNKNOWN"
- Happens suddenly after working fine

**Common causes:**

1. **Carrier is down or overloaded.** Check their status page. This is more common than you'd think.

2. **You hit the channel limit.** Most SIP trunks have a concurrent call limit. Check:

```bash
# Count active channels on a specific trunk
asterisk -rx "core show channels" | grep -c "SIP/your-trunk"
```

3. **Registration expired.** If your trunk requires registration:

```bash
# Check registration status
asterisk -rx "sip show registry"

# Force re-register
asterisk -rx "sip reload"
```

4. **DNS failure.** If the trunk uses a hostname, DNS resolution might have failed:

```bash
# Test DNS resolution
dig sip.carrier.com

# Check if Asterisk cached a stale IP
asterisk -rx "sip show peer your-trunk"
```

**Quick fix — implement failover trunks in your VICIdial campaign dial plan:**

```
# In Admin > System Settings > Dialplan
# Primary trunk
exten => _NXXNXXXXXX,1,Dial(SIP/${EXTEN}@primary-trunk,60,tT)
# Failover if primary returns 503
exten => _NXXNXXXXXX,2,Dial(SIP/${EXTEN}@backup-trunk,60,tT)
```

## Problem 3: Registration Failures

**Symptoms:**
- `sip show registry` shows "Rejected" or "Request Sent" that never resolves
- New trunk just won't connect

**Debug it step by step:**

```bash
# Enable SIP debug
asterisk -rx "sip set debug on"

# Watch the CLI — you'll see the full REGISTER exchange
# Look for:
# - 401 Unauthorized (credentials wrong)
# - 403 Forbidden (IP not whitelisted)
# - 407 Proxy Authentication Required (wrong auth type)
```

**401 Unauthorized fix:**

Triple-check your credentials. The most common mistake is whitespace in the password field in sip.conf:

```ini
; WRONG — trailing space after password
secret=MyP@ssw0rd

; RIGHT
secret=MyP@ssw0rd
```

Also check `username` vs `defaultuser`. Some carriers want the account number in `defaultuser`, not `username`:

```ini
[carrier-trunk]
type=peer
host=sip.carrier.com
defaultuser=12345          ; Account number goes here
secret=MyP@ssw0rd
fromuser=12345             ; Some carriers need this too
fromdomain=sip.carrier.com ; And this
```

**403 Forbidden fix:**

Your carrier needs to whitelist your IP. If you recently changed servers or IPs, contact them. Some carriers auto-whitelist on first successful registration, others require manual provisioning.

## Problem 4: Oualysis Timeouts

If your carrier is returning audio but VICIdial's answering machine detection or call routing takes too long, you'll see calls drop during the analysis phase.

```bash
# Check oualysis timing in the Asterisk CLI
# Look for "AMD" entries with timing data
asterisk -rx "core show channels verbose" | grep AMD
```

This one is deep enough to warrant its own post. See our [AMD configuration guide](https://vicistack.com/blog/vicidial-amd-guide) for the full breakdown.

## Problem 5: Codec Mismatch

**Symptoms:**
- Calls connect but audio sounds garbled, robotic, or has artifacts
- "No compatible codecs" in Asterisk CLI

**Check what codecs your carrier supports:**

```bash
asterisk -rx "sip show peer your-trunk" | grep -A5 "Codecs"
```

**Fix in sip.conf:**

```ini
[carrier-trunk]
; Allow the codecs your carrier supports, in order of preference
allow=!all
allow=ulaw        ; G.711 ulaw — best quality, 87 kbps
allow=alaw        ; G.711 alaw — same quality, EU standard
allow=g729        ; G.729 — compressed, 8 kbps, needs license
```

G.711 (ulaw) is the safe default for domestic US/Canada calling. Only use g729 if bandwidth is a real constraint — it adds latency and requires purchasing codec licenses for Asterisk.

## The Nuclear Option: Start From Scratch

If nothing above fixes it, sometimes the fastest path is a clean SIP trace from scratch:

```bash
# 1. Clear all state
asterisk -rx "sip reload"

# 2. Start fresh capture
sngrep -d eth0 port 5060

# 3. Make one test call and watch the entire flow

# 4. Look for the FIRST thing that goes wrong
#    - INVITE sent but no response? Firewall/routing.
#    - 100 Trying received but no 180? Carrier routing issue.
#    - 200 OK received but no audio? NAT/RTP issue.
#    - Audio works for 30 seconds then dies? Session timer.
```

## When to Call for Help

If you've been staring at packet captures for more than 2 hours and the problem isn't obvious, it's usually something environmental — an ISP middlebox doing SIP ALG, a firewall with stateful UDP tracking, or a carrier-side misconfiguration.

We've fixed hundreds of these at [ViciStack](https://vicistack.com). The pattern is usually the same: 80% of SIP problems are NAT or firewall, 15% are credentials/registration, and 5% are actually weird. The weird ones are where things get interesting.

**Need help debugging SIP issues on your VICIdial system?** That's literally what we do — [vicistack.com](https://vicistack.com) can usually pinpoint the problem in under 30 minutes.

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-sip-troubleshooting).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
