# VICIdial Caller ID Reputation Monitoring and Recovery Guide

Your VICIdial dialer can have perfect AMD tuning, optimized agent routing, and the best scripts in the industry -- but none of it matters if your calls are not being answered. And the number one reason calls go unanswered in 2026 is caller ID reputation.

When your outbound DIDs get flagged as "Spam Likely," "Scam Probable," or "Telemarketer" by carrier analytics systems, your answer rates collapse. We have seen centers go from 45% answer rates to 12% in a matter of days because their caller IDs got flagged. At 200 dials per agent per hour, that is the difference between 90 live connections per hour and 24. Your agents sit idle, your cost per acquisition triples, and your campaign bleeds money.

This guide covers how caller IDs get flagged, how to monitor your reputation before it tanks, how to implement DID rotation and cool-down strategies in VICIdial, and how to build automated reputation management that keeps your answer rates high.

## How Caller ID Gets Flagged

Understanding the flagging ecosystem is essential to managing it. There are three major pathways by which your outbound DIDs get labeled as spam.

### Pathway 1: Carrier Analytics Engines

The major US carriers (AT&T, Verizon, T-Mobile) run analytics engines that score every inbound call:

- **AT&T:** Uses Hiya (formerly TrustID) to analyze call patterns and flag suspicious numbers
- **Verizon:** Uses TNS (Transaction Network Services) Call Guardian
- **T-Mobile:** Uses their own Scam Shield system with multiple data sources

These engines analyze:

- **Call volume patterns.** A DID that originates 500+ calls per day raises flags. Normal consumer phone usage is 5-20 calls per day.
- **Short duration calls.** High volumes of calls lasting under 10 seconds (hangups, voicemails) signal robocalling.
- **Low answer rates.** If most people do not answer your calls, that is a negative signal.
- **Complaint velocity.** When recipients press "Report Spam" on their phone or register complaints through carrier apps, each complaint accelerates flagging.
- **Geographic patterns.** Calling patterns that are geographically inconsistent (a Miami number calling thousands of people in Oregon) trigger scrutiny.
- **Number age.** Newly provisioned DIDs that immediately start high-volume calling are suspicious.

### Pathway 2: Consumer Reporting

Individual consumers can report numbers as spam through:

- **Carrier apps:** T-Mobile Scam Shield, Verizon Call Filter, AT&T Call Protect
- **Third-party apps:** Truecaller, Hiya, Nomorobo, RoboKiller
- **FTC complaints:** Consumers filing complaints at donotcall.gov
- **Community databases:** Online forums and spam reporting websites

Each report feeds back into the carrier analytics engines, creating a negative feedback loop. It takes surprisingly few reports to trigger flagging -- as few as 3-5 reports within a 24-hour period can flag a number on some platforms.

### Pathway 3: STIR/SHAKEN Attestation Failures

STIR/SHAKEN (Secure Telephone Identity Revisited / Signature-based Handling of Asserted information using toKENs) is a framework that cryptographically signs calls to verify caller identity. Every call now carries an attestation level:

- **Full Attestation (A):** The carrier has verified the calling party's identity and their right to use the calling number. These calls are rarely flagged.
- **Partial Attestation (B):** The carrier has verified the calling party but cannot confirm they have the right to use the specific calling number. These calls receive moderate scrutiny.
- **Gateway Attestation (C):** The carrier received the call from an international gateway or cannot verify the origin. These calls receive maximum scrutiny.

If your SIP carrier provides only B or C attestation for your outbound DIDs, you start with a disadvantage. Carriers are increasingly using attestation level as a primary input to their spam scoring.

```
; Example STIR/SHAKEN header in a SIP INVITE
Identity: eyJhbGciOiJFUzI1NiIsInBwdCI6InNoYWtlbiIsInR5cCI6InBhc3Nwb3J0...
;info=<https://cert.example.com/cert.pem>;alg=ES256;ppt=shaken
; The 'attest' field in the decoded token shows A, B, or C
```

To check what attestation your carrier provides:

```bash
# Capture a SIP INVITE from your carrier and decode the Identity header
# Or ask your carrier directly what attestation level they provide for your DIDs
sngrep -c -d eth0 port 5060
# Look for the Identity header in outgoing INVITEs
```

## Monitoring Your Caller ID Reputation

You cannot manage what you do not measure. Set up monitoring before problems occur.

### Free Caller Registry (FCR)

The Free Caller Registry (https://freecallerregistry.com) is run by the major analytics providers. You can register your business and your DIDs, which signals to the analytics engines that you are a legitimate caller. This does not guarantee you will not be flagged, but it establishes a baseline identity.

Steps to register:
1. Create an account at freecallerregistry.com
2. Verify your business identity (requires EIN, business address)
3. Register all your outbound DIDs
4. Set your business name and calling purpose
5. Monitor the dashboard for reputation changes

### STIR/SHAKEN Attestation Verification

Check what attestation level your calls receive:

```bash
#!/bin/bash
# check_attestation.sh - Verify STIR/SHAKEN attestation on outbound calls
# Requires sngrep or a SIP capture tool

# Capture an outbound INVITE
tshark -i eth0 -f "udp port 5060" -Y "sip.Method == INVITE" \
    -T fields -e sip.Identity \
    -c 1 2>/dev/null

# Decode the JWT token (base64 decode the middle section)
# The 'attest' field shows your attestation level
```

Alternatively, use your SIP carrier's portal. Most carriers now show attestation information in their dashboards.

### Carrier-Specific Reputation Checks

Each major carrier offers tools to check your number's status:

**T-Mobile / Sprint:**
- Scam Shield app: Call your own number from a T-Mobile phone
- Business Portal: https://www.t-mobile.com/business/solutions/scam-shield

**AT&T:**
- Call Protect app: Call your own number from an AT&T phone
- Business request form for removing incorrect labels

**Verizon:**
- Call Filter app: Call your own number from a Verizon phone
- Business portal: https://voicefilter.verizon.com

### Third-Party Monitoring Tools

For automated, ongoing monitoring across all carriers:

- **Numeracle:** Entity Identity Management (EIM) platform that monitors your DID reputation across all major carriers and analytics engines
- **Quality Voice & Data (QVD):** Monitors caller ID display across 200+ carriers
- **CallerID Reputation:** Automated monitoring and alerting service

### Building Your Own Monitor

For a quick and dirty internal monitoring system, track your answer rates per DID in VICIdial:

```sql
-- Monitor answer rate by caller ID (DID)
-- Run daily to catch reputation drops early
SELECT
    dl.outbound_cid as caller_id,
    COUNT(*) as total_dials,
    SUM(CASE WHEN vl.status NOT IN ('NA','B','DC','N') THEN 1 ELSE 0 END) as answered,
    ROUND(
        SUM(CASE WHEN vl.status NOT IN ('NA','B','DC','N') THEN 1 ELSE 0 END) /
        COUNT(*) * 100, 1
    ) as answer_rate_pct,
    SUM(CASE WHEN vl.status = 'AA' THEN 1 ELSE 0 END) as amd_machine,
    SUM(CASE WHEN vl.status IN ('A','SALE','XFER') THEN 1 ELSE 0 END) as live_connects
FROM vicidial_dial_log dl
JOIN vicidial_log vl ON dl.uniqueid = vl.uniqueid
WHERE vl.call_date > DATE_SUB(NOW(), INTERVAL 24 HOUR)
GROUP BY dl.outbound_cid
HAVING total_dials > 50
ORDER BY answer_rate_pct ASC;
```

**Red flag:** If a DID's answer rate drops below 25% when your campaign average is 40%+, that DID is likely flagged.

```bash
#!/bin/bash
# did_reputation_alert.sh - Alert on sudden answer rate drops
# Run hourly via cron

MYSQL="mysql -u monitor -pmonitorpass -D asterisk -N -e"
ALERT_EMAIL="ops@yourdomain.com"
THRESHOLD=25  # Answer rate below this triggers alert

FLAGGED_DIDS=$($MYSQL "
    SELECT dl.outbound_cid,
           ROUND(SUM(CASE WHEN vl.status NOT IN ('NA','B','DC','N') THEN 1 ELSE 0 END) / COUNT(*) * 100, 1) as ar
    FROM vicidial_dial_log dl
    JOIN vicidial_log vl ON dl.uniqueid = vl.uniqueid
    WHERE vl.call_date > DATE_SUB(NOW(), INTERVAL 4 HOUR)
    GROUP BY dl.outbound_cid
    HAVING COUNT(*) > 30 AND ar < $THRESHOLD;
")

if [ -n "$FLAGGED_DIDS" ]; then
    echo "The following DIDs have answer rates below ${THRESHOLD}% in the last 4 hours:" > /tmp/did_alert.txt
    echo "" >> /tmp/did_alert.txt
    echo "$FLAGGED_DIDS" >> /tmp/did_alert.txt
    echo "" >> /tmp/did_alert.txt
    echo "These DIDs may be flagged as spam. Consider pulling them from rotation." >> /tmp/did_alert.txt

    mail -s "WARNING: Low answer rate DIDs detected" $ALERT_EMAIL < /tmp/did_alert.txt
fi
```

## DID Rotation Strategies

DID rotation is the primary tactical response to caller ID degradation. The concept is simple: spread your call volume across many DIDs so that no single DID exceeds the volume thresholds that trigger flagging.

### How Many DIDs Do You Need?

The rule of thumb is **1 DID per 50-100 daily calls**. If your center makes 10,000 dials per day, you need 100-200 DIDs in rotation. This keeps each DID under the radar of carrier analytics engines.

More conservative targets for premium campaigns (insurance, financial services):
- **1 DID per 30-50 daily calls** for maximum protection
- **Local presence DIDs** matching the area code of the called number

### Configuring DID Rotation in VICIdial

VICIdial supports CID Group rotation natively. Here is how to set it up.

**Step 1: Create a CID Group in VICIdial Admin.**

Navigate to Admin > CID Groups > Add New CID Group:

```
CID Group Name: main_rotation_pool
CID Group Description: Primary outbound DID rotation pool
```

**Step 2: Add DIDs to the CID Group.**

Add each DID with its associated area codes. VICIdial can automatically select a DID matching the area code of the called number (local presence):

```sql
-- Add DIDs to the CID group
INSERT INTO vicidial_campaign_cid_areacodes
    (campaign_id, areacode, outbound_cid, active, cid_description)
VALUES
    ('main_rotation_pool', '212', '2125551001', 'Y', 'NYC DID 1'),
    ('main_rotation_pool', '212', '2125551002', 'Y', 'NYC DID 2'),
    ('main_rotation_pool', '212', '2125551003', 'Y', 'NYC DID 3'),
    ('main_rotation_pool', '213', '2135551001', 'Y', 'LA DID 1'),
    ('main_rotation_pool', '213', '2135551002', 'Y', 'LA DID 2'),
    ('main_rotation_pool', '312', '3125551001', 'Y', 'Chicago DID 1'),
    ('main_rotation_pool', '312', '3125551002', 'Y', 'Chicago DID 2');
```

**Step 3: Assign the CID Group to your campaign.**

In Campaign Settings, set:
- **Use CID Group:** `main_rotation_pool`
- **CID Group Rotation:** `ROUND_ROBIN` or `RANDOM`

### Advanced Rotation: Weighted by Reputation

Standard round-robin rotation treats all DIDs equally. A smarter approach weights rotation by each DID's current reputation score:

```sql
-- Add a reputation score column to track DID health
ALTER TABLE vicidial_campaign_cid_areacodes
    ADD COLUMN reputation_score DECIMAL(3,1) DEFAULT 100.0,
    ADD COLUMN last_reputation_check DATETIME,
    ADD COLUMN daily_call_count INT DEFAULT 0;

-- Update reputation scores based on answer rates
-- Run this hourly
UPDATE vicidial_campaign_cid_areacodes cid
JOIN (
    SELECT
        dl.outbound_cid,
        ROUND(
            SUM(CASE WHEN vl.status NOT IN ('NA','B','DC','N') THEN 1 ELSE 0 END) /
            NULLIF(COUNT(*), 0) * 100, 1
        ) as answer_rate
    FROM vicidial_dial_log dl
    JOIN vicidial_log vl ON dl.uniqueid = vl.uniqueid
    WHERE vl.call_date > DATE_SUB(NOW(), INTERVAL 24 HOUR)
    GROUP BY dl.outbound_cid
    HAVING COUNT(*) > 20
) stats ON cid.outbound_cid = stats.outbound_cid
SET cid.reputation_score = stats.answer_rate,
    cid.last_reputation_check = NOW();
```

## Cool-Down Periods and Recovery

When a DID gets flagged, pulling it from rotation is only half the battle. You need a structured cool-down and recovery process.

### Cool-Down Timeline

Based on our experience managing DIDs across 100+ call centers:

| Flagging Severity | Cool-Down Period | Recovery Likelihood |
|-------------------|-----------------|---------------------|
| Soft flag (1-2 carriers) | 7-14 days | 85-90% |
| Hard flag (all major carriers) | 30-60 days | 60-70% |
| Persistent flag (> 60 days active) | 90+ days or retire | 30-40% |

During the cool-down period, the DID should make **zero outbound calls**. Any activity resets the clock.

### Active Recovery Steps

While a DID is cooling down, take these steps to accelerate recovery:

**1. Register with Free Caller Registry.** If you have not already, register the DID and your business. This signals legitimacy to analytics providers.

**2. Request carrier reviews.** Each carrier has a process for reviewing flagged numbers:

- **T-Mobile:** Submit a request through their business portal or email ip.support@t-mobile.com
- **AT&T/Hiya:** Submit through the Hiya portal (hiya.com) for reputation review
- **Verizon/TNS:** Submit through the TNS portal for Call Guardian review

**3. Verify STIR/SHAKEN attestation.** Ensure the DID receives Full Attestation (A) from your carrier. If it is getting B or C attestation, work with your carrier to resolve it.

**4. Send legitimate traffic first.** When re-introducing a recovered DID, start with low-volume, high-answer-rate traffic. Make 20-30 calls per day to known-good numbers (existing customers, warm leads) for 1-2 weeks before adding it back to cold outbound campaigns.

### Automating Cool-Down Management

```bash
#!/bin/bash
# did_cooldown_manager.sh
# Run daily via cron to manage DID rotation and cool-down

MYSQL="mysql -u cron -pcronpass -D asterisk -N -e"
MIN_ANSWER_RATE=25
COOLDOWN_DAYS=14

# Step 1: Identify DIDs that need cool-down
FLAGGED=$($MYSQL "
    SELECT outbound_cid
    FROM vicidial_campaign_cid_areacodes
    WHERE reputation_score < $MIN_ANSWER_RATE
    AND active = 'Y'
    AND daily_call_count > 50;
")

# Step 2: Deactivate flagged DIDs
for DID in $FLAGGED; do
    $MYSQL "UPDATE vicidial_campaign_cid_areacodes
            SET active = 'N',
                cooldown_start = NOW()
            WHERE outbound_cid = '$DID';"
    logger "DID COOLDOWN: Deactivated $DID (low answer rate)"
done

# Step 3: Check if any DIDs have completed cool-down
RECOVERED=$($MYSQL "
    SELECT outbound_cid
    FROM vicidial_campaign_cid_areacodes
    WHERE active = 'N'
    AND cooldown_start IS NOT NULL
    AND cooldown_start < DATE_SUB(NOW(), INTERVAL $COOLDOWN_DAYS DAY);
")

# Step 4: Re-activate recovered DIDs with low volume limit
for DID in $RECOVERED; do
    $MYSQL "UPDATE vicidial_campaign_cid_areacodes
            SET active = 'Y',
                daily_call_count = 0,
                reputation_score = 50.0,
                cooldown_start = NULL
            WHERE outbound_cid = '$DID';"
    logger "DID RECOVERY: Re-activated $DID after ${COOLDOWN_DAYS}-day cool-down"
done

# Step 5: Report
ACTIVE_COUNT=$($MYSQL "SELECT COUNT(*) FROM vicidial_campaign_cid_areacodes WHERE active = 'Y';")
COOLDOWN_COUNT=$($MYSQL "SELECT COUNT(*) FROM vicidial_campaign_cid_areacodes WHERE active = 'N' AND cooldown_start IS NOT NULL;")

echo "DID Pool Status: $ACTIVE_COUNT active, $COOLDOWN_COUNT in cool-down"
```

## Local Presence Dialing

Local presence dialing -- showing a caller ID with the same area code as the called party -- consistently produces 15-30% higher answer rates compared to a non-local number. VICIdial supports this natively through CID Groups with area code matching.

### Setting Up Local Presence in VICIdial

**Step 1: Acquire DIDs in target area codes.** You need at least 2-3 DIDs per target area code for rotation. If you dial into 50 area codes, that is 100-150 DIDs minimum.

**Step 2: Build the CID Group with area code mappings.** Each DID is mapped to its corresponding area code:

```sql
-- Local presence CID group
INSERT INTO vicidial_campaign_cid_areacodes
    (campaign_id, areacode, outbound_cid, active, cid_description)
VALUES
    -- New York metro
    ('local_presence', '212', '2125559001', 'Y', 'NYC Manhattan 1'),
    ('local_presence', '212', '2125559002', 'Y', 'NYC Manhattan 2'),
    ('local_presence', '718', '7185559001', 'Y', 'NYC Brooklyn 1'),
    ('local_presence', '718', '7185559002', 'Y', 'NYC Brooklyn 2'),
    ('local_presence', '917', '9175559001', 'Y', 'NYC Mobile 1'),

    -- Los Angeles metro
    ('local_presence', '213', '2135559001', 'Y', 'LA Downtown 1'),
    ('local_presence', '213', '2135559002', 'Y', 'LA Downtown 2'),
    ('local_presence', '310', '3105559001', 'Y', 'LA West 1'),
    ('local_presence', '323', '3235559001', 'Y', 'LA Central 1'),

    -- Chicago metro
    ('local_presence', '312', '3125559001', 'Y', 'Chicago Loop 1'),
    ('local_presence', '312', '3125559002', 'Y', 'Chicago Loop 2'),
    ('local_presence', '773', '7735559001', 'Y', 'Chicago North 1');

-- For area codes without local DIDs, use a nearby regional number
-- or a toll-free as fallback
```

**Step 3: Configure campaign CID selection.**

In VICIdial Campaign Settings:
- **Use CID Group:** `local_presence`
- **CID Group Rotation:** `AREACODE` (matches called party's area code)
- **Default CID:** Your toll-free number (fallback for area codes without matching DIDs)

### Local Presence Compliance Notes

Local presence dialing is legal under current FCC regulations, but there are important compliance considerations:

1. **The DID must be able to receive calls.** If a called party calls back the displayed number, they must reach your business. Configure inbound routing for all local presence DIDs.
2. **CNAM registration.** Register your business name with the CNAM database for each DID so that recipients see your business name, not "Unknown Caller."
3. **State regulations.** Some states have additional requirements for outbound caller ID display. Consult your compliance team.

### Inbound Routing for Local Presence DIDs

Every DID you use for outbound local presence must have inbound routing configured. Prospects will call back these numbers.

```ini
; Asterisk dial plan for handling inbound calls to local presence DIDs
[from-local-presence]
; Route all local presence DID callbacks to the same IVR/queue
exten => _NXXNXXXXXX,1,NoOp(Callback to local presence DID ${EXTEN})
exten => _NXXNXXXXXX,n,Set(CDR(did)=${EXTEN})
exten => _NXXNXXXXXX,n,Goto(callback-ivr,s,1)

[callback-ivr]
exten => s,1,Answer()
exten => s,n,Playback(thank-you-for-calling)
exten => s,n,Queue(callback_queue,t,,,120)
exten => s,n,Voicemail(callback@default)
exten => s,n,Hangup()
```

In VICIdial, configure each local presence DID as an inbound DID routing to your callback queue or IVR.

## Automated Reputation Management

Putting it all together, here is a complete automated reputation management system for VICIdial.

### Architecture

```
+------------------+     +-------------------+     +------------------+
| VICIdial Dialer  |---->| Reputation Engine |---->| DID Pool Manager |
| (vicidial_log)   |     | (hourly cron)     |     | (daily cron)     |
+------------------+     +-------------------+     +------------------+
                                |                          |
                                v                          v
                         +-------------+            +-------------+
                         | Alert System|            | Cool-Down   |
                         | (email/SMS) |            | Tracker     |
                         +-------------+            +-------------+
```

### Complete Management Script

```bash
#!/bin/bash
# did_reputation_manager.sh
# Complete DID reputation management for VICIdial
# Run hourly via cron

set -e
MYSQL="mysql -u cron -pcronpass -D asterisk -N -e"
LOG="/var/log/did_reputation.log"
ALERT_EMAIL="ops@yourdomain.com"
TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')

log() {
    echo "$TIMESTAMP $1" >> $LOG
}

# === PHASE 1: Collect Metrics ===
log "Starting reputation check"

# Get per-DID metrics for last 4 hours
$MYSQL "
    CREATE TEMPORARY TABLE tmp_did_metrics AS
    SELECT
        dl.outbound_cid,
        COUNT(*) as calls_4h,
        SUM(CASE WHEN vl.status NOT IN ('NA','B','DC','N') THEN 1 ELSE 0 END) as answered_4h,
        ROUND(
            SUM(CASE WHEN vl.status NOT IN ('NA','B','DC','N') THEN 1 ELSE 0 END) /
            NULLIF(COUNT(*), 0) * 100, 1
        ) as answer_rate_4h,
        AVG(vl.length_in_sec) as avg_duration
    FROM vicidial_dial_log dl
    JOIN vicidial_log vl ON dl.uniqueid = vl.uniqueid
    WHERE vl.call_date > DATE_SUB(NOW(), INTERVAL 4 HOUR)
    AND dl.outbound_cid IS NOT NULL
    AND dl.outbound_cid != ''
    GROUP BY dl.outbound_cid
    HAVING calls_4h > 20;
"

# === PHASE 2: Score and Flag ===

# Update reputation scores
$MYSQL "
    UPDATE vicidial_campaign_cid_areacodes cid
    JOIN tmp_did_metrics m ON cid.outbound_cid = m.outbound_cid
    SET cid.reputation_score = CASE
        WHEN m.answer_rate_4h >= 40 THEN LEAST(cid.reputation_score + 5, 100)
        WHEN m.answer_rate_4h >= 30 THEN cid.reputation_score
        WHEN m.answer_rate_4h >= 20 THEN GREATEST(cid.reputation_score - 10, 0)
        ELSE GREATEST(cid.reputation_score - 25, 0)
    END,
    cid.last_reputation_check = NOW(),
    cid.daily_call_count = cid.daily_call_count + m.calls_4h;
"

# === PHASE 3: Take Action ===

# Deactivate critically low reputation DIDs
DEACTIVATED=$($MYSQL "
    SELECT outbound_cid, reputation_score
    FROM vicidial_campaign_cid_areacodes
    WHERE reputation_score < 20
    AND active = 'Y';
")

if [ -n "$DEACTIVATED" ]; then
    $MYSQL "
        UPDATE vicidial_campaign_cid_areacodes
        SET active = 'N', cooldown_start = NOW()
        WHERE reputation_score < 20 AND active = 'Y';
    "
    log "DEACTIVATED DIDs: $DEACTIVATED"

    # Alert
    echo "The following DIDs have been deactivated due to low reputation:" > /tmp/rep_alert.txt
    echo "$DEACTIVATED" >> /tmp/rep_alert.txt
    mail -s "DID Reputation Alert: DIDs Deactivated" $ALERT_EMAIL < /tmp/rep_alert.txt
fi

# === PHASE 4: Daily Volume Reset ===
HOUR=$(date +%H)
if [ "$HOUR" = "00" ]; then
    $MYSQL "UPDATE vicidial_campaign_cid_areacodes SET daily_call_count = 0;"
    log "Reset daily call counts"
fi

# === PHASE 5: Pool Health Report ===
ACTIVE=$($MYSQL "SELECT COUNT(*) FROM vicidial_campaign_cid_areacodes WHERE active = 'Y';")
COOLDOWN=$($MYSQL "SELECT COUNT(*) FROM vicidial_campaign_cid_areacodes WHERE active = 'N' AND cooldown_start IS NOT NULL;")
AVG_REP=$($MYSQL "SELECT ROUND(AVG(reputation_score),1) FROM vicidial_campaign_cid_areacodes WHERE active = 'Y';")

log "Pool status: $ACTIVE active, $COOLDOWN cooling down, avg reputation: $AVG_REP"

# Alert if active pool is getting thin
TOTAL=$(($ACTIVE + $COOLDOWN))
if [ "$TOTAL" -gt 0 ]; then
    ACTIVE_PCT=$(echo "scale=0; $ACTIVE * 100 / $TOTAL" | bc)
    if [ "$ACTIVE_PCT" -lt 60 ]; then
        echo "WARNING: Only ${ACTIVE_PCT}% of DID pool is active ($ACTIVE of $TOTAL). Order more DIDs." | \
            mail -s "WARNING: DID Pool Running Low" $ALERT_EMAIL
        log "WARNING: Active DID pool at ${ACTIVE_PCT}%"
    fi
fi

log "Reputation check complete"
```

Add to cron:

```bash
# Run reputation manager hourly
0 * * * * /usr/local/bin/did_reputation_manager.sh
```

## Preventive Best Practices

Prevention is far cheaper than recovery. Follow these practices to keep your DIDs clean:

1. **Start low, ramp slow.** When provisioning new DIDs, start with 20-30 calls per day and ramp up over 2 weeks to your target volume.

2. **Maintain a DID-to-call ratio of at least 1:100.** For every 100 daily dials, have at least 1 DID in rotation.

3. **Register every DID with Free Caller Registry** before using it for outbound dialing.

4. **Register CNAM** for every outbound DID. A business name appearing on caller ID significantly reduces spam reports.

5. **Configure inbound routing** for every outbound DID. Unroutable numbers get flagged faster.

6. **Monitor daily.** Do not wait for answer rates to crater. Catch reputation drops within hours, not days.

7. **Separate DID pools by campaign type.** Do not use the same DIDs for cold outbound and warm callback campaigns. If cold outbound flags a DID, it drags down your warm callback answer rates too.

8. **Ensure Full STIR/SHAKEN attestation.** Work with your SIP carrier to verify you receive A-level attestation on all outbound calls. Switch carriers if necessary.

## How ViciStack Helps

Caller ID reputation management is one of the highest-ROI optimizations we perform for VICIdial call centers. Most centers are losing 15-30% of their potential live connections to spam labeling without realizing it.

ViciStack's managed optimization includes:

- **Full DID reputation audit** across all major carriers and analytics platforms
- **Automated rotation and cool-down management** deployed and monitored 24/7
- **Local presence DID provisioning** with area code coverage for your target markets
- **STIR/SHAKEN attestation verification** and carrier escalation for attestation issues
- **CNAM registration** for all outbound DIDs
- **Monthly reputation reporting** with trend analysis and proactive recommendations

All included in the flat $150/agent/month. No per-DID surcharges, no per-minute fees.

**[Get your free caller ID reputation analysis -- we will check every one of your outbound DIDs across all major carriers and show you exactly which ones are flagged.](https://vicistack.com/proof/)**

We respond within 5 minutes. No commitment, no credit card. Just a clear picture of your caller ID health.

## Further Reading

- [How to Reduce VICIdial AMD False Positives from 20% to Under 5%](/blog/vicidial-amd-false-positive-reduction) -- AMD accuracy is meaningless if calls are not being answered in the first place
- [VICIdial SIP Trunk Failover and Redundancy: Complete Setup Guide](/blog/vicidial-sip-trunk-failover) -- carrier selection affects STIR/SHAKEN attestation levels
- [VICIdial Answering Machine Detection vs AI-Based AMD: Which Is Better?](/blog/vicidial-amd-vs-ai-amd) -- optimizing what happens after the call is answered

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-caller-id-reputation).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
