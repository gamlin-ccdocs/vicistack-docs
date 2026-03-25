# VICIdial SIP Trunk Failover and Redundancy: Complete Setup Guide

A single SIP trunk failure during peak dialing hours is one of the most expensive things that can happen to a call center. If you are running 50 agents at 200 dials per hour each, that is 10,000 attempted calls per hour. When your primary carrier goes down and you have no failover, those agents sit idle. At a fully loaded cost of $25/hour per agent, you are burning $1,250 per hour in payroll alone -- not counting the revenue those agents would have generated.

This guide covers everything you need to implement production-grade SIP trunk failover in VICIdial: Asterisk trunk configuration, multi-carrier routing, Kamailio-based load balancing with health monitoring, and testing procedures that validate failover without disrupting live traffic.

## Why Default VICIdial Carrier Configuration Falls Short

Out of the box, VICIdial's carrier configuration in the admin interface supports multiple carriers. You can define Carrier A, Carrier B, and Carrier C, and assign them to campaigns. But the default behavior has critical limitations:

1. **No automatic failover.** If Carrier A returns a SIP 503 (Service Unavailable) or simply stops responding, VICIdial does not automatically retry the call through Carrier B. The call fails, gets logged as NA (No Answer) or error, and the lead goes back in the hopper.

2. **No health monitoring.** VICIdial does not continuously probe carrier health. You discover a carrier is down when agents start complaining about no calls, or when you notice the dial rate has cratered.

3. **Static routing.** Calls go to whichever carrier is assigned to the campaign. There is no dynamic routing based on carrier performance, latency, or current capacity.

To build real failover, you need to work at the Asterisk and (optionally) Kamailio layer.

## Asterisk Trunk Configuration for Failover

The foundation of SIP trunk failover is properly configured trunks in Asterisk with failover logic in the dial plan.

### Step 1: Define Your Carriers in sip.conf

For each carrier, define a trunk peer. Here is an example with three carriers:

```ini
; /etc/asterisk/sip.conf

; Primary carrier - best rates, highest quality
[carrier-primary]
type=peer
host=sip.primarycarrier.com
port=5060
username=your_account_id
secret=your_password
fromuser=your_account_id
fromdomain=sip.primarycarrier.com
insecure=invite,port
qualify=yes
qualifyfreq=30
dtmfmode=rfc2833
disallow=all
allow=ulaw
allow=g729
context=from-carrier-primary
nat=force_rport,comedia

; Secondary carrier - failover
[carrier-secondary]
type=peer
host=sip.secondarycarrier.com
port=5060
username=your_secondary_id
secret=your_secondary_pass
fromuser=your_secondary_id
fromdomain=sip.secondarycarrier.com
insecure=invite,port
qualify=yes
qualifyfreq=30
dtmfmode=rfc2833
disallow=all
allow=ulaw
allow=g729
context=from-carrier-secondary
nat=force_rport,comedia

; Tertiary carrier - emergency fallback
[carrier-tertiary]
type=peer
host=sip.tertiarycarrier.com
port=5060
username=your_tertiary_id
secret=your_tertiary_pass
fromuser=your_tertiary_id
fromdomain=sip.tertiarycarrier.com
insecure=invite,port
qualify=yes
qualifyfreq=30
dtmfmode=rfc2833
disallow=all
allow=ulaw
allow=g729
context=from-carrier-tertiary
nat=force_rport,comedia
```

Key settings to note:

- **qualify=yes** and **qualifyfreq=30** -- Asterisk sends SIP OPTIONS requests every 30 seconds to check if the carrier is reachable. If the carrier stops responding, Asterisk marks the peer as "UNREACHABLE."
- **nat=force_rport,comedia** -- Essential for most VoIP deployments behind NAT. Without this, one-way audio and registration failures are common.

### Step 2: Build the Failover Dial Plan

The dial plan is where the actual failover logic lives. VICIdial typically uses AGI scripts and the `extensions.conf` or `extensions_additional.conf` for outbound routing. You need to modify the outbound context to implement cascading failover.

```ini
; /etc/asterisk/extensions_custom.conf
; Failover dial plan for outbound calls

[outbound-failover]
; Try primary carrier first
exten => _1NXXNXXXXXX,1,NoOp(Outbound call to ${EXTEN} - trying primary carrier)
exten => _1NXXNXXXXXX,n,Set(TRUNK_TRIED=primary)
exten => _1NXXNXXXXXX,n,Set(CALLERID(num)=${CALLERID(num)})
exten => _1NXXNXXXXXX,n,Dial(SIP/${EXTEN}@carrier-primary,60,tT)
exten => _1NXXNXXXXXX,n,NoOp(Primary result: ${DIALSTATUS})

; If primary fails with CHANUNAVAIL or CONGESTION, try secondary
exten => _1NXXNXXXXXX,n,GotoIf($["${DIALSTATUS}" = "CHANUNAVAIL"]?try_secondary)
exten => _1NXXNXXXXXX,n,GotoIf($["${DIALSTATUS}" = "CONGESTION"]?try_secondary)
exten => _1NXXNXXXXXX,n,Goto(done)

; Try secondary carrier
exten => _1NXXNXXXXXX,n(try_secondary),NoOp(Primary failed - trying secondary carrier)
exten => _1NXXNXXXXXX,n,Set(TRUNK_TRIED=${TRUNK_TRIED},secondary)
exten => _1NXXNXXXXXX,n,Dial(SIP/${EXTEN}@carrier-secondary,60,tT)
exten => _1NXXNXXXXXX,n,NoOp(Secondary result: ${DIALSTATUS})

; If secondary also fails, try tertiary
exten => _1NXXNXXXXXX,n,GotoIf($["${DIALSTATUS}" = "CHANUNAVAIL"]?try_tertiary)
exten => _1NXXNXXXXXX,n,GotoIf($["${DIALSTATUS}" = "CONGESTION"]?try_tertiary)
exten => _1NXXNXXXXXX,n,Goto(done)

; Try tertiary carrier
exten => _1NXXNXXXXXX,n(try_tertiary),NoOp(Secondary failed - trying tertiary carrier)
exten => _1NXXNXXXXXX,n,Set(TRUNK_TRIED=${TRUNK_TRIED},tertiary)
exten => _1NXXNXXXXXX,n,Dial(SIP/${EXTEN}@carrier-tertiary,60,tT)
exten => _1NXXNXXXXXX,n,NoOp(Tertiary result: ${DIALSTATUS})

exten => _1NXXNXXXXXX,n(done),NoOp(Final DIALSTATUS: ${DIALSTATUS} via ${TRUNK_TRIED})
exten => _1NXXNXXXXXX,n,Hangup()
```

**Important DIALSTATUS values for failover decisions:**

| DIALSTATUS | Meaning | Should Failover? |
|------------|---------|------------------|
| CHANUNAVAIL | Trunk unreachable | Yes |
| CONGESTION | Network congestion / SIP 503 | Yes |
| BUSY | Called party busy | No -- not a trunk problem |
| NOANSWER | Called party did not answer | No -- not a trunk problem |
| ANSWER | Call connected | No -- success |
| CANCEL | Caller hung up | No -- not a trunk problem |

Only fail over on CHANUNAVAIL and CONGESTION. Failing over on BUSY or NOANSWER would cause the same number to be called through multiple carriers simultaneously.

### Step 3: Integrate with VICIdial's Carrier Configuration

VICIdial manages outbound dialing through its own dial plan generation. To integrate failover without breaking VICIdial's call tracking, configure your VICIdial carrier to route through your custom failover context instead of directly to a SIP peer.

In VICIdial Admin > Carriers, set:

- **Dialplan Entry:** `exten => _1NXXNXXXXXX,1,Goto(outbound-failover,${EXTEN},1)`
- **Account Entry:** Leave the SIP trunk definition in sip.conf as shown above

This tells VICIdial to route all outbound calls through your failover context, which then handles the cascading carrier logic.

## Health Monitoring with qualify and SIP OPTIONS

Asterisk's `qualify` mechanism is your first line of defense for detecting carrier problems.

### How qualify Works

When `qualify=yes` is set on a SIP peer, Asterisk periodically sends a SIP OPTIONS request to the carrier. If the carrier responds within the qualify timeout (default 2000ms), the peer is marked as "OK." If it does not respond, the peer is marked as "UNREACHABLE" after a configurable number of failed attempts.

### Tuning qualify Parameters

```ini
; In sip.conf, under each carrier peer
qualify=yes           ; Enable health checking
qualifyfreq=30        ; Check every 30 seconds
qualifysmoothing=yes  ; Use smoothed RTT for latency tracking
qualifypeers=yes      ; Also qualify registered peers
```

For more aggressive monitoring, reduce `qualifyfreq` to 15 seconds. But be aware that some carriers rate-limit OPTIONS requests and may temporarily block your IP if you probe too frequently.

### Monitoring qualify Status

Check carrier health from the Asterisk CLI:

```bash
# Show all SIP peer status
asterisk -rx "sip show peers" | grep carrier

# Output example:
# carrier-primary    192.168.1.100    5060  Yes  Yes  25ms   OK (42 ms)
# carrier-secondary  10.0.0.50        5060  Yes  Yes  45ms   OK (67 ms)
# carrier-tertiary   172.16.0.1       5060  Yes  Yes  0ms    UNREACHABLE
```

### Building an External Health Monitor

Asterisk's qualify is useful but limited. It only checks if the carrier responds to OPTIONS -- it does not verify that the carrier can actually route calls. Build an external monitor that validates end-to-end call routing:

```bash
#!/bin/bash
# carrier_health_monitor.sh
# Run via cron every 2 minutes

LOG_FILE="/var/log/carrier_health.log"
ALERT_EMAIL="ops@yourdomain.com"
TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')

check_carrier() {
    local CARRIER_NAME=$1
    local CARRIER_HOST=$2
    local CARRIER_PORT=${3:-5060}

    # Check 1: SIP OPTIONS response
    OPTIONS_RESULT=$(asterisk -rx "sip show peer $CARRIER_NAME" | grep "Status")

    if echo "$OPTIONS_RESULT" | grep -q "UNREACHABLE"; then
        echo "$TIMESTAMP CRITICAL: $CARRIER_NAME is UNREACHABLE" >> $LOG_FILE
        return 2
    fi

    # Check 2: Latency check
    LATENCY=$(echo "$OPTIONS_RESULT" | grep -oP '\d+(?= ms)')
    if [ -n "$LATENCY" ] && [ "$LATENCY" -gt 500 ]; then
        echo "$TIMESTAMP WARNING: $CARRIER_NAME latency ${LATENCY}ms exceeds 500ms threshold" >> $LOG_FILE
        return 1
    fi

    # Check 3: Recent call success rate (last 10 minutes)
    SUCCESS_RATE=$(mysql -u monitor -pmonitorpass -D asterisk -N -e "
        SELECT ROUND(
            SUM(CASE WHEN status NOT IN ('NA','DROP','DNCL') THEN 1 ELSE 0 END) /
            NULLIF(COUNT(*), 0) * 100, 1
        ) FROM vicidial_log
        WHERE call_date > DATE_SUB(NOW(), INTERVAL 10 MINUTE);
    ")

    if [ -n "$SUCCESS_RATE" ] && (( $(echo "$SUCCESS_RATE < 30" | bc -l) )); then
        echo "$TIMESTAMP WARNING: $CARRIER_NAME success rate ${SUCCESS_RATE}% below 30% threshold" >> $LOG_FILE
        return 1
    fi

    echo "$TIMESTAMP OK: $CARRIER_NAME healthy (${LATENCY}ms, ${SUCCESS_RATE}% success)" >> $LOG_FILE
    return 0
}

# Check each carrier
check_carrier "carrier-primary" "sip.primarycarrier.com"
PRIMARY_STATUS=$?

check_carrier "carrier-secondary" "sip.secondarycarrier.com"
SECONDARY_STATUS=$?

check_carrier "carrier-tertiary" "sip.tertiarycarrier.com"
TERTIARY_STATUS=$?

# Alert if primary is down
if [ $PRIMARY_STATUS -eq 2 ]; then
    echo "ALERT: Primary carrier is UNREACHABLE. Failover to secondary is active." | \
        mail -s "CRITICAL: Primary SIP carrier down" $ALERT_EMAIL
fi

# Alert if ALL carriers have issues
if [ $PRIMARY_STATUS -ge 1 ] && [ $SECONDARY_STATUS -ge 1 ] && [ $TERTIARY_STATUS -ge 1 ]; then
    echo "CRITICAL: ALL carriers degraded or unreachable. Immediate attention required." | \
        mail -s "CRITICAL: All SIP carriers degraded" $ALERT_EMAIL
fi
```

Add this to cron:

```bash
# Run carrier health check every 2 minutes
*/2 * * * * /usr/local/bin/carrier_health_monitor.sh
```

## Automatic Failover Triggers

Beyond simple reachability, you want to trigger failover based on quality degradation -- before the carrier goes completely down.

### SIP Response Code-Based Triggers

Certain SIP response codes indicate carrier-side problems that warrant failover:

| SIP Code | Meaning | Action |
|----------|---------|--------|
| 403 | Forbidden | Check credentials, may need failover |
| 480 | Temporarily Unavailable | Retry, then failover |
| 486 | Busy Here | Do NOT failover (called party busy) |
| 487 | Request Terminated | Do NOT failover (caller cancelled) |
| 500 | Server Internal Error | Failover immediately |
| 502 | Bad Gateway | Failover immediately |
| 503 | Service Unavailable | Failover immediately |

Modify the dial plan to handle specific SIP codes:

```ini
[outbound-failover]
exten => _1NXXNXXXXXX,1,NoOp(Dialing ${EXTEN})
exten => _1NXXNXXXXXX,n,Dial(SIP/${EXTEN}@carrier-primary,60,tT)
exten => _1NXXNXXXXXX,n,Set(SIP_CODE=${HANGUPCAUSE})
exten => _1NXXNXXXXXX,n,NoOp(SIP hangup cause: ${SIP_CODE})

; Failover on server errors (500-599 map to cause codes location)
exten => _1NXXNXXXXXX,n,GotoIf($["${DIALSTATUS}" = "CHANUNAVAIL"]?failover)
exten => _1NXXNXXXXXX,n,GotoIf($["${DIALSTATUS}" = "CONGESTION"]?failover)
exten => _1NXXNXXXXXX,n,GotoIf($["${SIP_CODE}" = "location"]?failover)
exten => _1NXXNXXXXXX,n,Goto(done)

exten => _1NXXNXXXXXX,n(failover),NoOp(Failover triggered - SIP code: ${SIP_CODE})
exten => _1NXXNXXXXXX,n,Dial(SIP/${EXTEN}@carrier-secondary,60,tT)
; ... continue cascade

exten => _1NXXNXXXXXX,n(done),Hangup()
```

### Rate-Based Automatic Failover

If you see a sudden spike in failed calls (say, failure rate jumps from 5% to 40% within 5 minutes), you want to proactively shift all traffic to the secondary carrier rather than failing over call by call.

```bash
#!/bin/bash
# auto_failover.sh - Proactive carrier failover based on failure rate
# Run every minute via cron

FAILURE_THRESHOLD=30  # Percentage
LOOKBACK_MINUTES=5
ACTIVE_CARRIER_FILE="/tmp/active_carrier"

# Default to primary
if [ ! -f "$ACTIVE_CARRIER_FILE" ]; then
    echo "primary" > $ACTIVE_CARRIER_FILE
fi

CURRENT_CARRIER=$(cat $ACTIVE_CARRIER_FILE)

# Check failure rate on current carrier
FAILURE_RATE=$(mysql -u cron -pcronpass -D asterisk -N -e "
    SELECT ROUND(
        SUM(CASE WHEN status IN ('NA','DROP') AND term_reason = 'SYSTEM' THEN 1 ELSE 0 END) /
        NULLIF(COUNT(*), 0) * 100, 1
    ) FROM vicidial_log
    WHERE call_date > DATE_SUB(NOW(), INTERVAL $LOOKBACK_MINUTES MINUTE);
")

if [ -z "$FAILURE_RATE" ]; then
    exit 0
fi

if (( $(echo "$FAILURE_RATE > $FAILURE_THRESHOLD" | bc -l) )); then
    if [ "$CURRENT_CARRIER" = "primary" ]; then
        # Deactivate primary carrier, activate secondary
        mysql -u cron -pcronpass -D asterisk -e "
            UPDATE vicidial_server_carriers SET active = 'N' WHERE carrier_id = 'carrier-primary';
            UPDATE vicidial_server_carriers SET active = 'Y' WHERE carrier_id = 'carrier-secondary';
        "
        echo "secondary" > $ACTIVE_CARRIER_FILE
        logger "AUTO-FAILOVER: Switched from primary to secondary carrier (failure rate: ${FAILURE_RATE}%)"
    elif [ "$CURRENT_CARRIER" = "secondary" ]; then
        mysql -u cron -pcronpass -D asterisk -e "
            UPDATE vicidial_server_carriers SET active = 'N' WHERE carrier_id = 'carrier-secondary';
            UPDATE vicidial_server_carriers SET active = 'Y' WHERE carrier_id = 'carrier-tertiary';
        "
        echo "tertiary" > $ACTIVE_CARRIER_FILE
        logger "AUTO-FAILOVER: Switched from secondary to tertiary carrier (failure rate: ${FAILURE_RATE}%)"
    fi
fi
```

## Using Kamailio for Advanced Load Balancing

For call centers running 100+ agents, Asterisk-level failover is not enough. You need Kamailio sitting in front of Asterisk as a SIP proxy, handling load balancing across multiple carriers and multiple Asterisk servers.

For a detailed Kamailio configuration guide, see our companion article: [VICIdial Kamailio Load Balancing for 100+ Agent Call Centers](/blog/vicidial-kamailio-load-balancing).

The key advantages of adding Kamailio to your failover architecture:

- **Sub-second failover** -- Kamailio detects carrier failure and reroutes within milliseconds, versus Asterisk which must wait for the Dial() timeout
- **Weighted distribution** -- Send 70% of traffic to your cheapest carrier, 20% to the second, 10% to the third
- **Transaction-level failover** -- If the INVITE to Carrier A gets a 503, Kamailio immediately tries Carrier B within the same SIP transaction, transparent to Asterisk
- **Health probing independent of Asterisk** -- Kamailio runs its own OPTIONS probes with configurable intervals and failure thresholds

## Testing Failover Without Dropping Calls

Never test failover during production hours by actually killing a carrier. Instead, use these approaches.

### Method 1: Simulate Carrier Failure with iptables

Block traffic to a specific carrier at the firewall level. This simulates a network-level carrier outage without affecting other traffic:

```bash
# Block outbound SIP to primary carrier
sudo iptables -A OUTPUT -d sip.primarycarrier.com -p udp --dport 5060 -j DROP

# Monitor: watch Asterisk detect the carrier as unreachable
watch -n 5 'asterisk -rx "sip show peer carrier-primary" | grep Status'

# Verify: calls should now route through secondary carrier
# Check vicidial_log for carrier_id on new calls

# Restore: remove the block
sudo iptables -D OUTPUT -d sip.primarycarrier.com -p udp --dport 5060 -j DROP
```

### Method 2: SIP Peer Manipulation

Temporarily modify the carrier peer to point to a non-existent host:

```bash
# Save current config
cp /etc/asterisk/sip.conf /etc/asterisk/sip.conf.bak

# Change primary carrier host to an unreachable IP
sed -i 's/host=sip.primarycarrier.com/host=192.0.2.1/' /etc/asterisk/sip.conf

# Reload SIP configuration (does NOT drop active calls)
asterisk -rx "sip reload"

# Test failover behavior
# ...

# Restore
cp /etc/asterisk/sip.conf.bak /etc/asterisk/sip.conf
asterisk -rx "sip reload"
```

### Method 3: Low-Volume Test Campaign

Create a dedicated test campaign in VICIdial with a small lead list (your own test numbers). Run the test campaign through your failover dial plan and manually trigger carrier failures while monitoring call flow.

```sql
-- Create a test campaign (or use existing test campaign)
-- Route it through the failover dial plan
-- Use test DIDs that you own as the lead list

INSERT INTO vicidial_list (list_id, status, phone_number, first_name, last_name)
VALUES (99999, 'NEW', '5551234567', 'Test', 'Failover');
```

### Validating Failover Metrics

After testing, query the database to confirm calls actually failed over:

```sql
-- Check call results during test window via carrier log
SELECT
    DATE_FORMAT(cl.call_date, '%H:%i') as minute,
    cl.channel,
    COUNT(*) as calls,
    SUM(CASE WHEN cl.dialstatus = 'ANSWER' THEN 1 ELSE 0 END) as successful
FROM vicidial_carrier_log cl
WHERE cl.call_date BETWEEN '2026-03-19 02:00:00' AND '2026-03-19 02:30:00'
GROUP BY minute, cl.channel
ORDER BY minute;
```

## Capacity Planning for Redundancy

Failover only works if your backup carriers have enough capacity to absorb the traffic. Plan for these scenarios:

| Scenario | Required Backup Capacity |
|----------|--------------------------|
| Primary carrier down | Secondary must handle 100% of traffic |
| Primary degraded (50% capacity) | Secondary handles overflow (50%) |
| Primary + secondary down | Tertiary handles 100% |
| All carriers degraded | Reduce dial rate to match available capacity |

**Rule of thumb:** Your secondary carrier should have contracts and capacity for at least 80% of your primary carrier's concurrent call limit. Your tertiary should handle at least 50%.

For a 100-agent center doing 200 dials/agent/hour, you need approximately 150-200 concurrent channels at peak. Ensure each carrier can handle at least that many channels -- and that your Asterisk server's `maxcalls` setting in `asterisk.conf` is set high enough.

```ini
; /etc/asterisk/asterisk.conf
[options]
maxcalls = 500    ; Ensure this exceeds your total concurrent call capacity
maxload = 5.0     ; Limit based on system load
```

## How ViciStack Helps

Implementing SIP trunk failover correctly requires deep knowledge of Asterisk dial plan programming, SIP protocol behavior, and VICIdial's internal routing. Most call centers configure a single carrier and hope for the best.

ViciStack deploys production-grade failover as part of every managed VICIdial optimization:

- **Multi-carrier failover** configured and tested before go-live
- **Real-time carrier health monitoring** with sub-minute detection
- **Automatic traffic shifting** based on carrier performance metrics
- **Monthly capacity reviews** to ensure backup carriers can absorb failover traffic
- **All included in the flat $150/agent/month** -- no per-minute charges, no surprise fees

We have handled carrier outages for centers running 200+ agents without a single dropped call reaching the agents.

**[Get your free VICIdial infrastructure analysis -- we will review your current carrier setup and identify single points of failure.](https://vicistack.com/proof/)**

5-minute response time. No commitment required.

## Further Reading

- [VICIdial Kamailio Load Balancing for 100+ Agent Call Centers](/blog/vicidial-kamailio-load-balancing) -- the next step after basic failover
- [VICIdial Caller ID Reputation Monitoring and Recovery Guide](/blog/vicidial-caller-id-reputation) -- carrier failover can affect caller ID presentation
- [How to Reduce VICIdial AMD False Positives from 20% to Under 5%](/blog/vicidial-amd-false-positive-reduction) -- carrier changes affect AMD tuning

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-sip-trunk-failover).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
