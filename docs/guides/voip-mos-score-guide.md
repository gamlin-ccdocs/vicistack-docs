# VoIP MOS Score: What It Means and How to Fix Bad Call Quality

**Last updated: March 2026 | Reading time: ~20 minutes**

You're getting complaints. Agents say calls sound "robotic." Leads say they can't understand the agent. Your QA team flags recordings where half the words are garbled. Someone says "we need to check the MOS score" and nobody in the room knows what that means.

MOS — Mean Opinion Score — is the standard metric for voice call quality. It runs from 1.0 (completely unintelligible) to 5.0 (perfect, indistinguishable from an in-person conversation). It was originally measured by having actual humans listen to calls and rate them. Now it's calculated algorithmically from network statistics.

In a call center running VICIdial, your MOS score directly affects conversion rates. An agent talking to a lead through a choppy, laggy connection loses trust in the first 10 seconds. The lead assumes you're a scammer calling from a boiler room overseas and hangs up.

Here's everything you need to know about MOS scores, how to measure them, and how to fix them when they drop.

---

## The MOS Scale: What the Numbers Mean

The MOS scale comes from ITU-T Recommendation P.800, which was originally a methodology for subjective voice quality testing. You'd get 20+ people in a room, play them voice samples, and have them rate quality from 1 to 5. Average the ratings and that's your MOS.

| MOS Score | Quality | What You Hear |
|-----------|---------|---------------|
| 4.3 - 5.0 | Excellent | Clear, natural conversation. Sounds like a landline call. |
| 4.0 - 4.3 | Good | Slightly less than perfect. Most people wouldn't notice. |
| 3.6 - 4.0 | Fair | Noticeable degradation. Some words might need repeating. |
| 3.1 - 3.6 | Poor | Clearly degraded. Conversation is difficult. Complaints start. |
| 2.6 - 3.1 | Bad | Nearly unintelligible. Most people will hang up. |
| 1.0 - 2.6 | Terrible | Unusable. Might as well be static. |

For a call center, you want to stay above 4.0 on every call. Below 3.6 and your agents start losing deals. Below 3.0 and your leads think something is wrong with your business.

### Why 5.0 Is Basically Impossible Over VoIP

The theoretical max for VoIP is about 4.4 with the G.711 codec and zero network impairment. That's because G.711 uses 64 kbps of bandwidth per call — it's essentially uncompressed audio — but even perfect G.711 loses some quality compared to the analog reference used in the original P.800 testing.

Lower-bitrate codecs (G.729 at 8 kbps, Opus at variable rates) start with a lower baseline MOS even before network effects:

| Codec | Bitrate | Best-Case MOS | Typical VoIP MOS |
|-------|---------|---------------|------------------|
| G.711 (ulaw/alaw) | 64 kbps | 4.4 | 4.0-4.3 |
| G.722 (wideband) | 64 kbps | 4.5 | 4.1-4.4 |
| G.729a | 8 kbps | 3.9 | 3.5-3.8 |
| Opus | 6-128 kbps | 4.5+ | 4.0-4.5 |
| GSM | 13 kbps | 3.5 | 3.0-3.4 |
| iLBC | 15 kbps | 3.7 | 3.3-3.6 |

G.711 is the default in most VICIdial deployments and is what we recommend. G.729 saves bandwidth but you're starting with a quality deficit. The bandwidth savings rarely matter — a call center with 100 concurrent calls uses about 6.4 Mbps with G.711. That's nothing in 2026.

---

## How MOS Is Calculated (The Short Version)

Modern MOS calculation is algorithmic, not subjective. Two common methods:

### PESQ (Perceptual Evaluation of Speech Quality) — ITU-T P.862

PESQ compares the original transmitted audio with the received audio and calculates a quality score. It's the gold standard for MOS measurement but requires access to both ends of the call, which makes it impractical for real-time monitoring.

### E-Model / R-Factor — ITU-T G.107

The E-Model calculates an "R-factor" from 0-100 based on network parameters. The R-factor then maps to a MOS score. This is what your VoIP equipment actually uses because it only needs network statistics, not the actual audio.

The calculation considers:
- **Packet loss** — Most destructive factor. Even 1% packet loss drops MOS by ~0.4
- **Latency (one-way)** — Above 150ms, noticeable. Above 300ms, conversation falls apart
- **Jitter** — Variation in latency. High jitter = choppy audio even if average latency is fine
- **Codec** — Each codec has a baseline impairment factor

The simplified formula (leaving out the truly painful math):

```
R = 93.2 - Id - Ie + A

Where:
  93.2 = base R-factor (perfect conditions)
  Id   = delay impairment (from latency)
  Ie   = equipment impairment (codec + packet loss)
  A    = advantage factor (mobile users tolerate more)
```

Then R maps to MOS:

| R-Factor | MOS | Quality |
|----------|-----|---------|
| 90-100   | 4.3-4.5 | Excellent |
| 80-90    | 4.0-4.3 | Good |
| 70-80    | 3.6-4.0 | Fair |
| 60-70    | 3.1-3.6 | Poor |
| Below 60 | Below 3.1 | Bad |

---

## Measuring MOS on Your System

### Asterisk RTCP Statistics

Asterisk tracks RTP/RTCP statistics per channel. At the end of each call, these stats are available in the CDR and can be queried during the call.

```bash
# Show RTP statistics for active channels
asterisk -rx "rtp set debug on"

# Check a specific channel's RTP stats
asterisk -rx "core show channel SIP/carrier-00000042"
```

The channel output includes `rxjitter` (receive jitter), `txjitter` (transmit jitter), `rxcount` (received packets), and `rxploss` (received packet loss). From these you can estimate MOS.

### VICIdial CDR Analysis

VICIdial logs call quality data in `vicidial_log` and Asterisk CDRs. To check quality trends:

```bash
# Average call duration — short calls can indicate quality issues
mysql -e "
SELECT DATE(call_date) as day,
       AVG(length_in_sec) as avg_duration,
       COUNT(*) as calls
FROM vicidial_log
WHERE call_date > DATE_SUB(NOW(), INTERVAL 7 DAY)
GROUP BY DATE(call_date)
ORDER BY day;
" asterisk
```

If average call duration is dropping across the board, quality degradation is a likely cause. Leads hang up on choppy calls.

### Network-Level Testing

Before blaming VoIP, test your network fundamentals:

```bash
# Latency to your SIP carrier
ping -c 100 sip.carrier.com

# Check for packet loss over time
mtr -r -c 200 sip.carrier.com

# Measure jitter (variance in ping times)
ping -c 100 sip.carrier.com | tail -1
# The mdev value is your jitter

# Example output:
# rtt min/avg/max/mdev = 12.3/14.7/89.2/8.4 ms
# mdev of 8.4ms = your jitter. Under 10ms is decent. Over 30ms is a problem.
```

For more thorough VoIP quality testing:

```bash
# iperf3 — test bandwidth and jitter to a remote endpoint
iperf3 -c remote_server -u -b 200K -t 60 -i 5

# The output shows jitter and packet loss for the test stream
```

### Wireshark RTP Analysis

If you need detailed per-call MOS measurement, capture traffic with tcpdump and analyze in Wireshark:

```bash
# Capture RTP traffic (usually ports 10000-20000 on Asterisk)
tcpdump -i eth0 -s 0 -w /tmp/rtp-capture.pcap portrange 10000-20000
```

In Wireshark: Telephony → RTP → RTP Streams. Select a stream and click "Analyze." Wireshark calculates jitter, packet loss, and estimated MOS for each RTP stream.

---

## What Kills Your MOS Score

### 1. Packet Loss

The single biggest MOS killer. Voice codecs send audio in 20ms packets. When a packet is lost, the decoder has to guess what was in it (packet loss concealment). A few lost packets and you get clicks or brief gaps. Sustained loss and the audio becomes unintelligible.

| Packet Loss | Effect on MOS (G.711) |
|-------------|----------------------|
| 0% | 4.4 (baseline) |
| 0.5% | 4.2 |
| 1% | 4.0 |
| 2% | 3.6 |
| 3% | 3.2 |
| 5% | 2.6 |
| 10% | 1.8 |

Look at those numbers. Just 2% packet loss drops you from "good" to "fair." At 5% your calls are unusable.

**How to find packet loss:**

```bash
# Quick test to your carrier
ping -c 1000 -i 0.02 sip.carrier.com | tail -3

# Check Asterisk RTP stats for active calls
asterisk -rx "rtp set debug on"
# Look for "Lost packets" in the output
```

**Causes:**
- Congested network links (not enough bandwidth)
- Faulty network equipment (bad switch port, failing NIC)
- ISP issues (especially with "best effort" internet connections)
- WiFi interference (if anyone on the voice path is on WiFi, stop that)
- Server CPU overload (Asterisk drops packets when the CPU can't keep up)

**Fixes:**
- QoS / DSCP markings (more on this below)
- Dedicated VLAN for voice traffic
- Bandwidth reservation or dedicated internet circuit for voice
- Replace consumer-grade equipment with business-grade networking
- Move to bare metal if you're on a VM and seeing packet loss from hypervisor contention

### 2. Jitter

Jitter is the variation in packet arrival time. If packets arrive at perfectly regular 20ms intervals, jitter is zero. If one arrives at 15ms, the next at 25ms, another at 10ms, and then one at 40ms, you have high jitter.

The jitter buffer on the receiving end absorbs some of this variation by holding packets for a moment before playing them. But if jitter exceeds the buffer size, packets arrive "too late" and are treated as lost.

| Jitter | Effect |
|--------|--------|
| < 10ms | Excellent. No audible impact. |
| 10-30ms | Acceptable. Jitter buffer handles it. |
| 30-50ms | Noticeable. May cause audio gaps depending on buffer config. |
| > 50ms | Severe. Audio will be choppy regardless of buffer settings. |

**How to measure jitter:**

```bash
# Ping-based jitter estimate
ping -c 500 sip.carrier.com
# The mdev (mean deviation) value approximates jitter

# Asterisk channel jitter
asterisk -rx "core show channel SIP/carrier-00000042" | grep jitter
```

**Configuring jitter buffers in Asterisk:**

```ini
; sip.conf — enable adaptive jitter buffer
[general]
jbenable = yes        ; Enable jitter buffer
jbforce = yes         ; Force jitter buffer even when Asterisk is bridging
jbmaxsize = 200       ; Max buffer size in ms
jbresyncthreshold = 1000  ; Resync if jitter exceeds this
jbimpl = adaptive     ; Use adaptive (vs fixed) jitter buffer
```

The adaptive jitter buffer adjusts its size based on observed jitter. Start with these settings and monitor. If you're still getting choppy audio, increase `jbmaxsize` to 300ms, but know that larger buffers add latency.

### 3. Latency (Delay)

One-way latency is how long it takes audio to travel from speaker to listener. In a normal phone call, you don't notice anything under 150ms. Above 150ms, you start getting that "satellite call" feeling where both parties talk over each other. Above 300ms, normal conversation becomes really hard.

| One-Way Latency | Effect |
|-----------------|--------|
| < 100ms | Excellent. No perceivable delay. |
| 100-150ms | Good. Barely noticeable. |
| 150-200ms | Fair. Noticeable. Double-talk becomes awkward. |
| 200-300ms | Poor. Conversation is difficult. |
| > 300ms | Bad. Like talking on a 1990s satellite phone. |

In a VICIdial deployment, latency comes from:
- Network path between your server and the carrier (typically 10-50ms)
- Codec encoding/decoding (5-30ms depending on codec)
- Jitter buffer delay (20-200ms depending on settings)
- If using WebRTC for remote agents: additional browser/network hops

**Reducing latency:**
- Use a SIP carrier with a PoP (point of presence) near your server
- Use G.711 instead of G.729 (lower encoding delay)
- Keep jitter buffer size minimal (trade-off with jitter tolerance)
- Avoid double-NAT configurations
- Use wired connections, not WiFi

### 4. Codec Mismatch and Transcoding

When two endpoints negotiate different codecs, Asterisk has to transcode — convert audio from one codec to another in real time. Transcoding adds latency, uses CPU, and degrades quality (you lose information in each conversion).

```bash
# Check if transcoding is happening on active calls
asterisk -rx "core show channels verbose" | grep -i "transcode\|Codec"
```

If you see calls where one side is G.711 and the other is G.729, Asterisk is transcoding. Fix it by ensuring both sides negotiate the same codec:

```ini
; sip.conf — force consistent codec
[general]
disallow = all
allow = ulaw         ; G.711 ulaw first
allow = alaw         ; G.711 alaw second (international)
; Only add g729 if you really need it for bandwidth
```

In the VICIdial admin, set the codec preferences in your Carrier configuration to match what the carrier supports. Ideally, everything in the call path uses G.711 and transcoding never happens.

---

## Network Configuration for Voice Quality

### QoS and DSCP Marking

Quality of Service (QoS) prioritizes voice packets over data packets when network links are congested. It works by tagging voice packets with a DSCP (Differentiated Services Code Point) value that routers and switches recognize as high-priority.

For SIP signaling, use DSCP CS3 (24). For RTP media, use DSCP EF (46) — Expedited Forwarding, the highest standard priority.

On Linux:

```bash
# Mark SIP packets with CS3
iptables -t mangle -A OUTPUT -p udp --dport 5060 -j DSCP --set-dscp-class CS3

# Mark RTP packets with EF (Asterisk default RTP range: 10000-20000)
iptables -t mangle -A OUTPUT -p udp --dport 10000:20000 -j DSCP --set-dscp-class EF
```

In Asterisk, you can also set the TOS (Type of Service) value directly:

```ini
; sip.conf
[general]
tos_sip = cs3
tos_audio = ef
tos_video = af41

; rtp.conf
[general]
rtpstart = 10000
rtpend = 20000
```

QoS marking only matters if your network equipment (switches, routers, firewalls) is configured to honor DSCP markings. On a shared internet connection without QoS-aware equipment, the markings do nothing. On a properly configured enterprise network or MPLS circuit, they make a big difference.

### Bandwidth Planning

Each G.711 call uses about 85 kbps bidirectional (64 kbps audio + IP/UDP/RTP overhead). For a 50-agent call center with concurrent calls, you need:

```
50 calls × 85 kbps × 2 (send + receive) = 8.5 Mbps
```

That's the voice traffic alone. Add overhead for SIP signaling, web traffic, CRM, VPN, and other business traffic. A 100 Mbps dedicated connection is usually more than enough. But if you're sharing a 50 Mbps connection with 200 employees streaming YouTube and downloading files, voice quality will suffer during peak hours.

**Dedicated voice VLAN** — Separate voice and data traffic at the switch level. Voice VLAN gets priority queuing.

**Bandwidth reservation** — On your router/firewall, reserve a minimum guaranteed bandwidth for voice traffic. Even if the data side is saturated, voice always gets its allocation.

---

## Monitoring MOS in Production

### Continuous Monitoring Script

```bash
#!/bin/bash
# voip-quality-check.sh — Run every 5 minutes via cron

CARRIER_IP="sip.carrier.com"
LOGFILE="/var/log/voip-quality.log"
ALERT_THRESHOLD_LOSS=2    # Alert if packet loss exceeds 2%
ALERT_THRESHOLD_JITTER=30 # Alert if jitter exceeds 30ms

# Run 200 pings at voice packet intervals
RESULT=$(ping -c 200 -i 0.02 -q $CARRIER_IP 2>/dev/null | tail -2)

# Extract packet loss percentage
LOSS=$(echo "$RESULT" | grep -oP '\d+(?=% packet loss)')
# Extract jitter (mdev)
JITTER=$(echo "$RESULT" | grep -oP 'mdev = [\d.]+/[\d.]+/[\d.]+/([\d.]+)' | grep -oP '[\d.]+$')

TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')
echo "$TIMESTAMP loss=${LOSS}% jitter=${JITTER}ms" >> $LOGFILE

if [ "$LOSS" -gt "$ALERT_THRESHOLD_LOSS" ] || [ "$(echo "$JITTER > $ALERT_THRESHOLD_JITTER" | bc)" -eq 1 ]; then
    echo "VoIP quality alert: loss=${LOSS}%, jitter=${JITTER}ms" | \
        mail -s "VoIP Quality Degradation" admin@yourcompany.com
fi
```

### Grafana Dashboard

If you're running [Grafana for VICIdial monitoring](/blog/vicidial-grafana-dashboards/), add a VoIP quality panel that graphs:
- Packet loss percentage over time
- Jitter (mdev) over time
- Active channel count (to correlate quality drops with load)
- Average call duration (a proxy for quality — short calls = people hanging up)

Feed the data from the monitoring script into InfluxDB or Prometheus and build dashboards that show quality trends over days and weeks.

---

## Troubleshooting Decision Tree

**Complaint: "Calls sound robotic/choppy"**

1. Check packet loss: `ping -c 500 -i 0.02 carrier_ip`
   - Loss > 1%? → Network issue. Check ISP, switches, cabling.
   - Loss < 1%? → Continue to step 2.

2. Check jitter: Look at mdev from ping
   - Jitter > 30ms? → Enable/tune jitter buffer in sip.conf
   - Jitter < 30ms? → Continue to step 3.

3. Check codec: `asterisk -rx "core show channels verbose"`
   - Transcoding happening? → Align codecs across all trunks and endpoints
   - No transcoding? → Continue to step 4.

4. Check server load: `top` and `vmstat 1 10`
   - CPU > 80%? → Asterisk can't keep up. Reduce load or add server capacity.
   - CPU fine? → Continue to step 5.

5. Check carrier quality:
   - Run the same tests to a different carrier endpoint
   - If quality is fine to other destinations, the problem is your current carrier's network
   - Contact carrier support with your test data

**Complaint: "There's a delay when I talk / people talk over each other"**

1. Check one-way latency: `ping carrier_ip` (one-way ≈ round-trip / 2)
   - RTT > 200ms? → Carrier PoP is too far. Switch to a regional carrier.
   - RTT < 200ms? → Continue to step 2.

2. Check jitter buffer size:
   - Buffer too large adds latency. Try reducing `jbmaxsize` from 200 to 100.
   - Trade-off: lower buffer = less jitter tolerance but less delay.

3. Check for double NAT:
   - `traceroute carrier_ip` — do you see two 192.168.x.x or 10.x.x.x hops?
   - Double NAT adds processing delay at each translation point.

**Complaint: "Audio cuts out for a few seconds then comes back"**

This is usually bursty packet loss — a momentary network interruption. Causes:
- Switch port flapping
- ISP route change
- WiFi interference (if any hop is wireless)
- QoS policy kicking in and throttling voice during a bandwidth spike

Check for packet loss bursts with a long-running ping:

```bash
ping -c 3600 -i 1 sip.carrier.com | while read line; do
    echo "$(date '+%H:%M:%S') $line"
done > /tmp/ping-test-24h.log
```

Run this for 24 hours and look for clusters of lost packets. Correlate the timing with known network events (backups running, large file transfers, etc.).

---

## Real-World MOS Benchmarks

From our deployments running on bare-metal VICIdial infrastructure:

| Configuration | Typical MOS | Notes |
|--------------|-------------|-------|
| Bare metal, G.711, dedicated circuit | 4.2-4.4 | Best possible for VoIP |
| Bare metal, G.711, business internet | 4.0-4.3 | Good, occasional dips during congestion |
| VM (VMware/KVM), G.711 | 3.8-4.2 | Hypervisor scheduling adds jitter |
| Cloud (AWS/GCP), G.711 | 3.6-4.1 | Variable. Noisy neighbors affect quality. |
| Any platform, G.729 | 3.5-3.9 | Codec ceiling is lower |
| WebRTC agents, remote | 3.4-4.0 | Depends entirely on agent's home internet |

The pattern is clear: bare metal + G.711 + dedicated network = best quality. VMs and cloud add unpredictable jitter from shared resources. G.729 starts with a quality penalty. Remote agents on home internet are the wildcard.

---

## The Hardware Factor

Network isn't the only thing that affects MOS. Your server hardware matters too.

**CPU:** Asterisk processes RTP on the CPU. On bare metal with Intel NICs, this is efficient. On VMs, the hypervisor adds context-switch overhead. If Asterisk can't process RTP fast enough, packets get delayed or dropped.

**NIC:** Intel server NICs with hardware offloading handle VoIP traffic better than Realtek consumer NICs. The interrupt handling is more predictable, which reduces jitter.

**Disk I/O:** Doesn't directly affect MOS, but if you're recording calls (which VICIdial does), disk I/O contention can steal CPU cycles from RTP processing. Use separate drives for recordings and OS.

---

**Poor call quality is costing you conversions right now.** ViciStack deploys VICIdial on bare-metal infrastructure optimized for voice quality — dedicated NICs, G.711 codec alignment, QoS configuration, and continuous MOS monitoring included in every deployment. We consistently deliver MOS scores above 4.2 in production. Our standard engagement is $5K ($1K deposit, $4K on completion). [Find out what your calls should sound like](/contact/).

---

*Related: [VICIdial SIP Troubleshooting](/blog/vicidial-sip-troubleshooting/) | [VICIdial Carrier Selection](/blog/vicidial-carrier-selection/) | [VICIdial Performance Tuning](/blog/vicidial-performance-tuning/) | [VICIdial Asterisk Configuration](/blog/vicidial-asterisk-configuration/)*

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/voip-mos-score-guide).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
