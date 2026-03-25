# VICIdial CNAM Lookup Integration for Inbound Routing

When an inbound call arrives at your VICIdial system, you typically see a 10-digit number and nothing else. The agent answers blind -- no idea who is calling, what company they represent, or whether this is a hot prospect returning your call or a wrong number. CNAM (Caller Name) lookup changes that equation. By dipping the caller ID against a national database before the call reaches the agent, you can display the caller's name, route calls intelligently based on caller data, and give your agents a critical few seconds of context before they say hello.

This guide covers everything you need to implement CNAM lookup in a VICIdial environment: what CNAM is and how it works, configuring the dip in Asterisk, choosing providers, using CNAM data for intelligent routing, customizing your inbound IVR, and mapping DIDs to campaigns effectively.

## What CNAM Is and Why It Matters

CNAM is a database system that maps phone numbers to subscriber names. When you see "JOHN SMITH" on your caller ID display instead of just a phone number, that is CNAM at work. The calling party's carrier stores the name in the CNAM database when they provision the number, and the receiving carrier queries (dips) the database when the call arrives.

### How CNAM Works Technically

1. An inbound call arrives at your carrier with a Caller ID number (CID) in the SIP `From` header or SS7 Calling Party Number field
2. Your carrier (or your Asterisk server, if you do the dip yourself) sends a CNAM query to a CNAM database provider
3. The CNAM provider responds with the registered name for that number (up to 15 characters)
4. The name is delivered to Asterisk in the SIP `From` display name field or via an AGI script response

### Why CNAM Matters for Call Centers

For VICIdial inbound operations, CNAM provides:

- **Agent preparation**: Seeing "ACME CORP" vs. a bare number lets agents tailor their greeting
- **Routing intelligence**: Route calls from known businesses to your commercial team, residential callers to your consumer team
- **Lead matching**: Cross-reference CNAM data against your `vicidial_list` to identify returning leads
- **Fraud detection**: Calls showing "WIRELESS CALLER" or blank CNAM from numbers that should be landlines can indicate spoofing
- **Reporting**: Track inbound call sources by caller type (business, residential, wireless, VOIP)

## Configuring CNAM Dip in Asterisk

There are two approaches to CNAM lookup in a VICIdial/Asterisk environment: letting your SIP trunk provider do the dip, or doing it yourself via AGI.

### Option 1: Carrier-Side CNAM

Most SIP trunk providers offer CNAM as an add-on service. When enabled, the carrier performs the dip before delivering the call to your server. The caller name arrives in the SIP INVITE's `From` header:

```
From: "JOHN SMITH" <sip:5551234567@carrier.com>
```

Asterisk reads this automatically and populates the `CALLERID(name)` channel variable. No configuration needed on your end beyond enabling the feature with your carrier.

**Pros**: Zero configuration, carrier handles all caching and failover.
**Cons**: Per-dip charges from carrier (typically $0.004-$0.01 per dip), no control over caching, and you pay even for calls you do not answer.

### Option 2: Self-Service CNAM via AGI

For more control and lower costs at scale, perform the CNAM dip yourself using an AGI (Asterisk Gateway Interface) script. This lets you cache results, skip dips for known numbers, and use any CNAM provider's API.

Create an AGI script that queries a CNAM API:

```python
#!/usr/bin/env python3
# /var/lib/asterisk/agi-bin/cnam_lookup.py
"""
AGI script for CNAM lookup with caching.
Called from Asterisk dialplan before routing inbound calls.
"""

import sys
import os
import json
import time
import sqlite3
import urllib.request
import urllib.error

# AGI communication
def agi_read():
    """Read AGI environment variables."""
    env = {}
    while True:
        line = sys.stdin.readline().strip()
        if line == '':
            break
        key, _, value = line.partition(':')
        env[key.strip()] = value.strip()
    return env

def agi_command(cmd):
    """Send AGI command and read response."""
    sys.stdout.write(cmd + '\n')
    sys.stdout.flush()
    return sys.stdin.readline().strip()

# Configuration
CNAM_API_URL = "https://api.cnamprovidier.com/v1/lookup"
CNAM_API_KEY = os.environ.get('CNAM_API_KEY', 'your-api-key-here')
CACHE_DB = "/var/lib/asterisk/cnam_cache.db"
CACHE_TTL = 86400 * 30  # 30 days - CNAM data rarely changes

def init_cache():
    """Initialize SQLite cache database."""
    conn = sqlite3.connect(CACHE_DB)
    conn.execute("""
        CREATE TABLE IF NOT EXISTS cnam_cache (
            phone_number TEXT PRIMARY KEY,
            caller_name TEXT,
            lookup_time INTEGER,
            provider TEXT
        )
    """)
    conn.commit()
    return conn

def cache_lookup(conn, phone):
    """Check cache for existing CNAM data."""
    cursor = conn.execute(
        "SELECT caller_name, lookup_time FROM cnam_cache WHERE phone_number = ?",
        (phone,)
    )
    row = cursor.fetchone()
    if row and (time.time() - row[1]) < CACHE_TTL:
        return row[0]
    return None

def cache_store(conn, phone, name):
    """Store CNAM result in cache."""
    conn.execute(
        "INSERT OR REPLACE INTO cnam_cache (phone_number, caller_name, lookup_time, provider) "
        "VALUES (?, ?, ?, ?)",
        (phone, name, int(time.time()), 'cnam_provider')
    )
    conn.commit()

def api_lookup(phone):
    """Query CNAM API for caller name."""
    try:
        url = f"{CNAM_API_URL}?number={phone}&api_key={CNAM_API_KEY}"
        req = urllib.request.Request(url, headers={'Accept': 'application/json'})
        with urllib.request.urlopen(req, timeout=2) as resp:
            data = json.loads(resp.read().decode())
            return data.get('name', '')
    except (urllib.error.URLError, json.JSONDecodeError, KeyError):
        return ''

def main():
    env = agi_read()

    # Get caller ID number
    callerid_num = env.get('agi_callerid', '')
    if not callerid_num or len(callerid_num) < 10:
        agi_command('SET VARIABLE CNAM_RESULT "UNKNOWN"')
        return

    # Normalize to 10 digits
    phone = callerid_num[-10:]

    # Check cache first
    conn = init_cache()
    cached_name = cache_lookup(conn, phone)

    if cached_name:
        name = cached_name
    else:
        # API lookup
        name = api_lookup(phone)
        if name:
            cache_store(conn, phone, name)

    conn.close()

    if name:
        # Set the caller ID name in Asterisk
        agi_command(f'SET CALLERID "{name}" <{callerid_num}>')
        agi_command(f'SET VARIABLE CNAM_RESULT "{name}"')
    else:
        agi_command('SET VARIABLE CNAM_RESULT "UNKNOWN"')

if __name__ == '__main__':
    main()
```

Make it executable:

```bash
chmod +x /var/lib/asterisk/agi-bin/cnam_lookup.py
chown asterisk:asterisk /var/lib/asterisk/agi-bin/cnam_lookup.py
```

### Adding the AGI to Your Dialplan

In the inbound context of your Asterisk dialplan (`/etc/asterisk/extensions.conf` or the relevant custom context):

```ini
[inbound-cnam]
; CNAM lookup before routing to VICIdial
exten => _X.,1,AGI(cnam_lookup.py)
exten => _X.,n,NoOp(CNAM Result: ${CNAM_RESULT})
exten => _X.,n,Goto(trunkinbound,${EXTEN},1)
```

Then route your inbound DIDs through this context before they hit VICIdial's normal inbound handling.

### Timeout Handling

The AGI script must complete quickly. If the CNAM API is slow or unreachable, you do not want callers waiting. The `timeout=2` in the API call limits the HTTP request to 2 seconds. If it times out, the call proceeds without CNAM data.

You can also set an Asterisk-level timeout on the AGI:

```ini
exten => _X.,1,Set(AGITIMEOUT=3)
exten => _X.,n,AGI(cnam_lookup.py)
```

## CNAM Providers and Pricing

### Major CNAM Providers

| Provider | Per-Dip Cost | Caching Allowed | API Type | Notes |
|----------|-------------|-----------------|----------|-------|
| Telnyx | $0.004 | Yes | REST | Good for VoIP-native shops |
| Twilio | $0.01 | No (ToS) | REST | Most common, higher cost |
| OpenCNAM | $0.004 | Yes | REST | CNAM-specific, bulk discounts |
| Neustar/TransUnion | $0.003 | Varies | REST/ENUM | Enterprise, volume discounts |
| Bandwidth | $0.005 | Yes | REST | Bundled with SIP trunking |
| BulkCNAM | $0.002 | Yes | REST | Cheapest, bulk-focused |

### Cost Analysis for a 25-Agent Center

Assume 500 inbound calls per day:

| Scenario | Daily Cost | Monthly Cost |
|----------|-----------|--------------|
| No caching, $0.005/dip | $2.50 | $75 |
| With 60% cache hit, $0.005/dip | $1.00 | $30 |
| BulkCNAM with caching, $0.002/dip | $0.40 | $12 |

With the AGI caching approach above (30-day TTL), repeat callers are served from cache at zero cost. For centers with high repeat-caller rates, this reduces costs by 50-70%.

### Choosing a Provider

For VICIdial environments, prioritize:

1. **Low latency**: The dip must complete in under 2 seconds. Test from your server's location.
2. **Caching terms**: Some providers (Twilio) prohibit caching in their ToS. If you want to cache, use a provider that allows it.
3. **Accuracy**: Not all CNAM databases are equal. Wireless numbers and VoIP numbers often return "WIRELESS CALLER" or "CITY STATE" instead of a name. Test with 50 known numbers before committing.
4. **Volume pricing**: If you do 10,000+ dips per month, negotiate bulk pricing.

## Using CNAM Data for Skill-Based Routing

Once you have CNAM data available on the call, you can use it to make routing decisions before the call reaches an agent.

### Route by Caller Type

CNAM results often indicate the caller type:

- **Business name** (e.g., "ACME CORP"): Route to commercial agents
- **Personal name** (e.g., "JOHN SMITH"): Route to consumer agents
- **"WIRELESS CALLER"**: Mobile callback, likely a lead returning your call
- **"UNKNOWN" or blank**: Could be spoofed, VoIP, or international -- handle with care

In the Asterisk dialplan:

```ini
[inbound-routing]
exten => _X.,1,AGI(cnam_lookup.py)
exten => _X.,n,GotoIf($["${CNAM_RESULT}" = "UNKNOWN"]?unknown)
exten => _X.,n,GotoIf($["${CNAM_RESULT}" = "WIRELESS CALLER"]?wireless)

; Check if CNAM contains common business indicators
exten => _X.,n,Set(IS_BUSINESS=${REGEX("(CORP|LLC|INC|LTD|COMPANY|ENTERPRISES)" ${CNAM_RESULT})})
exten => _X.,n,GotoIf($[${IS_BUSINESS} = 1]?business)

; Default: route to consumer in-group
exten => _X.,n,Goto(default)

; Business routing
exten => _X.,n(business),NoOp(Routing business call: ${CNAM_RESULT})
exten => _X.,n,Set(__CAMPAIGN=COMMERCIAL_INBOUND)
exten => _X.,n,Goto(trunkinbound,${EXTEN},1)

; Wireless/callback routing
exten => _X.,n(wireless),NoOp(Wireless callback: ${CALLERID(num)})
exten => _X.,n,Set(__CAMPAIGN=CALLBACK_INBOUND)
exten => _X.,n,Goto(trunkinbound,${EXTEN},1)

; Unknown caller routing
exten => _X.,n(unknown),NoOp(Unknown caller)
exten => _X.,n,Set(__CAMPAIGN=GENERAL_INBOUND)
exten => _X.,n,Goto(trunkinbound,${EXTEN},1)

; Default consumer routing
exten => _X.,n(default),NoOp(Consumer call: ${CNAM_RESULT})
exten => _X.,n,Set(__CAMPAIGN=CONSUMER_INBOUND)
exten => _X.,n,Goto(trunkinbound,${EXTEN},1)
```

### Cross-Reference with VICIdial Lead Data

For even smarter routing, check if the inbound caller exists in your `vicidial_list`:

```python
# Addition to the AGI script: check vicidial_list for the caller
import mysql.connector

def vicidial_lead_lookup(phone):
    """Check if caller exists in vicidial_list."""
    try:
        conn = mysql.connector.connect(
            host='localhost',
            user='cron',
            password='your_db_password',
            database='asterisk'
        )
        cursor = conn.cursor(dictionary=True)
        cursor.execute("""
            SELECT lead_id, first_name, last_name, status, owner,
                   campaign_id, list_id
            FROM vicidial_list
            WHERE phone_number = %s
            ORDER BY modify_date DESC
            LIMIT 1
        """, (phone,))
        result = cursor.fetchone()
        conn.close()
        return result
    except Exception:
        return None
```

If the caller is a known lead, you can:
- Route them to the agent who last spoke with them (owner-based routing)
- Set the lead ID as a channel variable so the agent screen auto-populates
- Route based on their current status (hot lead vs. cold callback)

```ini
; In dialplan, after AGI returns lead data
exten => _X.,n,GotoIf($["${LEAD_ID}" != ""]?known_lead)
exten => _X.,n,Goto(new_caller)

exten => _X.,n(known_lead),NoOp(Known lead: ${LEAD_ID} - ${LEAD_NAME})
; Route to the agent who owns this lead
exten => _X.,n,Set(__owner=${LEAD_OWNER})
exten => _X.,n,Goto(trunkinbound,${EXTEN},1)
```

## Inbound IVR Customization Based on Caller Data

VICIdial's built-in IVR (Call Menu) system can be enhanced with CNAM data to create personalized caller experiences.

### Dynamic Greeting

```ini
; Play a personalized greeting if we know the caller
exten => _X.,1,AGI(cnam_lookup.py)
exten => _X.,n,GotoIf($["${CNAM_RESULT}" = "UNKNOWN"]?generic_greeting)

; Known caller - personalized greeting
exten => _X.,n,Playback(thank-you-for-calling)
exten => _X.,n,SayAlpha(${CNAM_RESULT:0:15})
exten => _X.,n,Playback(please-hold)
exten => _X.,n,Goto(route_call,${EXTEN},1)

; Unknown caller - generic greeting
exten => _X.,n(generic_greeting),Playback(welcome-thank-you-for-calling)
exten => _X.,n,Goto(ivr_menu,${EXTEN},1)
```

### Priority Queuing

Route high-value callers (identified by CNAM or lead status) to the front of the queue:

In VICIdial, configure multiple In-Groups with different priority levels:

```
In-Group: HIGH_PRIORITY_INBOUND
  Queue Priority: 99

In-Group: NORMAL_INBOUND
  Queue Priority: 50
```

Route based on CNAM data in the dialplan:

```ini
exten => _X.,n,GotoIf($[${IS_BUSINESS} = 1]?high_priority)
exten => _X.,n,Goto(normal_priority)

exten => _X.,n(high_priority),Set(__CAMPAIGN=HIGH_PRIORITY_INBOUND)
exten => _X.,n,Goto(trunkinbound,${EXTEN},1)

exten => _X.,n(normal_priority),Set(__CAMPAIGN=NORMAL_INBOUND)
exten => _X.,n,Goto(trunkinbound,${EXTEN},1)
```

## DID-to-Campaign Mapping

Proper DID mapping ensures inbound calls land in the right VICIdial campaign and in-group. This is foundational for CNAM-enhanced routing.

### VICIdial DID Configuration

In VICIdial Admin, navigate to **Inbound > DIDs** to configure each DID:

| Setting | Purpose | Example |
|---------|---------|---------|
| DID Extension | The DID number | 8005551234 |
| DID Route | Where to send the call | IN_GROUP |
| In-Group | The VICIdial in-group | SALES_INBOUND |
| Filter In-Group | Secondary routing based on filters | COMMERCIAL_INBOUND |
| Exten Context | Custom Asterisk context for pre-processing | inbound-cnam |

### Using DID Filters with CNAM

VICIdial's DID Filter system can route calls based on caller ID patterns. While it does not natively read CNAM data, you can set a custom channel variable in your AGI script and use VICIdial's custom dialplan routing:

```ini
; Set a custom variable that VICIdial can read
exten => _X.,n,Set(__VICIcnam=${CNAM_RESULT})
exten => _X.,n,Set(__VICIcallertype=${CALLER_TYPE})
```

### Multi-DID Strategy

For centers with multiple marketing campaigns, assign different DIDs to different traffic sources:

| DID | Source | In-Group | Notes |
|-----|--------|----------|-------|
| 800-555-1234 | Google Ads | GOOGLE_INBOUND | Highest priority |
| 800-555-1235 | TV commercial | TV_INBOUND | High volume, lower intent |
| 800-555-1236 | Existing customer line | SUPPORT_INBOUND | Route to senior agents |
| 800-555-1237 | Callback number (outbound CID) | CALLBACK_INBOUND | These are leads returning calls |

The callback DID (1237) is particularly important. When your outbound campaigns use a specific caller ID, the return calls to that number are warm leads. CNAM lookup on these calls, combined with a `vicidial_list` cross-reference, lets you identify the lead and route them to the original agent.

### Configuring Multiple DIDs in VICIdial

```sql
-- Verify your DID configuration
SELECT did_id, did_pattern, did_description, did_route,
       group_id, menu_id, exten_context
FROM vicidial_inbound_dids
ORDER BY did_pattern;
```

Ensure each DID has the correct in-group and that the exten context points to your CNAM AGI. In the VICIdial admin GUI, go to **Admin > Inbound DID** and for each DID you want CNAM on, set the **Extension Context** field to `inbound-cnam`. This routes calls through the custom context where the AGI script runs before handing off to the in-group.

## Performance Considerations

### CNAM Lookup Latency

Every millisecond of CNAM lookup time adds to the caller's wait time. Benchmark your provider:

```bash
# Test CNAM API latency from your server
for i in {1..10}; do
  START=$(date +%s%N)
  curl -s "https://api.cnam-provider.com/v1/lookup?number=5551234567&api_key=KEY" > /dev/null
  END=$(date +%s%N)
  echo "Dip $i: $(( (END - START) / 1000000 ))ms"
done
```

Target: under 500ms average. If your provider consistently exceeds 1 second, switch providers or enable caching more aggressively.

### Cache Hit Rate Monitoring

Track your cache performance to ensure you are getting value from the caching layer:

```sql
-- If using the SQLite cache from the AGI script
-- Run this from a monitoring script
SELECT
  COUNT(*) AS total_cached,
  SUM(CASE WHEN lookup_time > strftime('%s','now') - 86400 THEN 1 ELSE 0 END) AS dips_today,
  SUM(CASE WHEN lookup_time > strftime('%s','now') - 2592000 THEN 1 ELSE 0 END) AS dips_30d
FROM cnam_cache;
```

### Failover Strategy

Never let a CNAM provider outage block your inbound calls. The AGI script above handles this with the 2-second timeout, but also consider:

```python
# Multi-provider failover in the AGI script
PROVIDERS = [
    {'url': 'https://api.primary-provider.com/lookup', 'key': 'KEY1'},
    {'url': 'https://api.backup-provider.com/lookup', 'key': 'KEY2'},
]

def api_lookup_with_failover(phone):
    for provider in PROVIDERS:
        try:
            url = f"{provider['url']}?number={phone}&api_key={provider['key']}"
            req = urllib.request.Request(url, headers={'Accept': 'application/json'})
            with urllib.request.urlopen(req, timeout=1.5) as resp:
                data = json.loads(resp.read().decode())
                name = data.get('name', '')
                if name:
                    return name
        except Exception:
            continue
    return ''
```

## How ViciStack Helps

CNAM integration touches multiple layers of your VICIdial stack: Asterisk dialplan, AGI scripting, database queries, caching infrastructure, and in-group routing configuration. Getting it wrong means missed routing opportunities at best and blocked inbound calls at worst.

ViciStack implements and manages CNAM integration for VICIdial call centers:

- **CNAM provider selection** based on your call volume, budget, and accuracy requirements
- **AGI script deployment** with caching, failover, and monitoring built in
- **Intelligent routing configuration** that uses CNAM data alongside lead database lookups
- **DID-to-campaign mapping** optimized for your specific inbound traffic sources
- **Performance monitoring** ensuring CNAM dips do not add perceptible delay for callers
- **Cost optimization** through caching strategy and provider negotiation

We have integrated CNAM into VICIdial deployments handling 1,000+ inbound calls per day. The combination of caller identification, lead matching, and intelligent routing typically improves inbound conversion rates by 10-20% because agents are better prepared and calls are better routed.

**Get a free analysis of your VICIdial inbound routing.** We will review your DID configuration, in-group routing, and show you exactly where CNAM integration would improve your connect-to-conversion rate.

[Request your free ViciStack analysis](https://vicistack.com/proof/) -- response in under 5 minutes.

## Related Articles

- [VICIdial Remote Agent Setup: NAT Traversal, WebRTC, and SIP Configuration](/blog/vicidial-remote-agent-setup) -- SIP configuration for the trunks carrying your inbound calls
- [VICIdial Quality Assurance Scoring with Call Recordings](/blog/vicidial-qa-scoring) -- QA your inbound call handling
- [VICIdial Timezone-Aware Dialing and TCPA Compliance](/blog/vicidial-timezone-dialing-tcpa) -- compliance for your outbound calls that generate inbound callbacks

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-cnam-lookup-inbound).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
