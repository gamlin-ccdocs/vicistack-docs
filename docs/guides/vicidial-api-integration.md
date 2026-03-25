# VICIdial API Integration: Custom Workflows & Automation

*VICIdial has two APIs that most operators never touch. The Non-Agent API lets external systems inject leads, pull stats, control campaigns, and trigger calls without anyone logging into the admin GUI. The Agent API lets you build custom agent interfaces, programmatically disposition calls, and control screen behavior from outside the browser. Together, they turn VICIdial from a standalone dialer into a programmable call center engine that plugs into anything.*

*The documentation for these APIs exists — scattered across VICIdial's `docs/` directory, forum posts from 2014, and tribal knowledge locked in the heads of a few senior developers. This guide consolidates all of it into a single working reference with real code examples.*

VICIdial's API layer is HTTP-based. Every endpoint is a URL on your VICIdial web server that accepts GET or POST parameters and returns plain text, JSON, or pipe-delimited data. There are no WebSocket connections, no OAuth flows, no SDK libraries. You call a URL, pass credentials and parameters, and get a response. It's simple, which is both its greatest strength and most frustrating limitation.

This guide covers the Non-Agent API (for system-to-system automation), the Agent API (for controlling the agent interface programmatically), authentication and security, lead injection workflows, [disposition](/glossary/disposition/) webhooks, real-time stats pulling, click-to-call implementation, campaign control, rate limiting, error handling, and practical integration patterns you can deploy today.

---

## The Two APIs: Non-Agent vs. Agent

VICIdial exposes two distinct API endpoints that serve different purposes. Understanding which one to use for each task will save you hours of frustration.

### Non-Agent API

**Endpoint:** `https://your-server/vicidial/non_agent_api.php`

This is the workhorse API for backend automation. It does not require an active agent session. External systems — CRMs, web forms, marketing platforms, custom applications — call this API to interact with VICIdial's data and campaigns.

**What it can do:**
- Add leads to lists (`add_lead`)
- Update existing lead data (`update_lead`)
- Pull agent performance stats (`agent_stats`)
- Pull campaign stats (`campaign_stats`)
- Initiate outbound calls (`call_agent`)
- Start/stop/pause campaigns
- Check DNC lists (`dnc_check`)
- Manage callbacks (`add_callback`, `remove_callback`)
- Trigger recordings (`recording_lookup`)
- Look up lead data (`lead_search`)

### Agent API

**Endpoint:** `https://your-server/agc/api.php`

This API requires an active agent session. It controls agent-side behavior — what you'd normally do by clicking buttons on the [agent screen](/glossary/agent-screen/). Use this when you're building a custom agent interface, integrating a CRM that needs to control VICIdial call flow, or automating agent actions.

**What it can do:**
- Log agents in and out
- Pause/unpause agents
- Disposition calls
- Transfer calls
- Hang up calls
- Dial specific numbers
- Set callback schedules
- Update lead data from the agent side
- Switch agent campaigns
- Send DTMF tones
- Control recordings

The key distinction: the Non-Agent API is for *system-level* operations (data in, data out, campaign management). The Agent API is for *session-level* operations (things that happen during a live agent session).

---

## Authentication and Security

Both APIs use HTTP basic authentication via URL parameters. Every request must include a valid username and password.

### Non-Agent API Authentication

The Non-Agent API requires a user account with API access enabled:

1. Go to Admin -> Users -> [user] -> API Access
2. Set "API Access" to **1** (enabled)
3. Set "API Allowed Functions" to the specific functions this user can call (or leave blank for all)

```bash
# Basic authenticated request
curl "https://your-server/vicidial/non_agent_api.php?source=test&user=apiuser&pass=apipass&function=version"
```

Every request requires three base parameters:
- `source` — an identifier for the calling system (freeform text, used in logs)
- `user` — API username
- `pass` — API password

### Agent API Authentication

The Agent API authenticates against the active agent session:

```bash
curl "https://your-server/agc/api.php?source=test&user=agent001&pass=agentpass&function=external_status&agent_user=agent001&value=SALE"
```

The `agent_user` parameter identifies which agent session to control. The user/pass must match the agent's credentials.

### Security Hardening

VICIdial's API has no built-in rate limiting, no API keys, no OAuth, and no token-based authentication. Credentials are passed as URL parameters, which means they appear in web server access logs. Here's how to harden it:

**1. HTTPS is mandatory.** Without TLS, credentials travel in plaintext. Configure your Apache with a valid SSL certificate.

**2. IP whitelisting via .htaccess:**

```apache
# /var/www/html/vicidial/.htaccess
<Files "non_agent_api.php">
  Order Deny,Allow
  Deny from all
  Allow from 10.0.0.0/8
  Allow from 192.168.1.100
  Allow from 203.0.113.50
</Files>
```

**3. Create dedicated API users** with minimal permissions. Never use admin accounts for API access. Create a user with only the API functions it needs:

```sql
-- Create an API-only user directly in MySQL
INSERT INTO vicidial_users (user, pass, full_name, user_level, api_list_restrict, api_allowed_functions)
VALUES ('crm_api', 'strongpassword', 'CRM API User', 1, 1, 'add_lead,update_lead,lead_search');
```

**4. Monitor API usage.** VICIdial logs API calls in the `vicidial_api_log` table:

```sql
SELECT function, source, user, api_date, result
FROM vicidial_api_log
WHERE api_date > DATE_SUB(NOW(), INTERVAL 1 HOUR)
ORDER BY api_date DESC
LIMIT 50;
```

If you see unexpected sources, unknown IPs, or functions being called that shouldn't be, you have an access problem.

---

## Lead Injection via add_lead

This is the most commonly used API endpoint. Every web form, landing page, CRM, and lead vendor integration starts here. The `add_lead` function inserts a new lead record into a VICIdial list, making it immediately available for dialing.

### Basic Lead Injection

```bash
curl -G "https://your-server/vicidial/non_agent_api.php" \
  --data-urlencode "source=webform" \
  --data-urlencode "user=apiuser" \
  --data-urlencode "pass=apipass" \
  --data-urlencode "function=add_lead" \
  --data-urlencode "phone_number=5551234567" \
  --data-urlencode "first_name=John" \
  --data-urlencode "last_name=Smith" \
  --data-urlencode "list_id=10001" \
  --data-urlencode "phone_code=1"
```

**Response (success):**
```
SUCCESS: add_lead - leadass added - 5551234567|10001|12345678
```

The response includes the phone number, list ID, and the newly created `lead_id`. Parse and store that lead ID — you'll need it for updates, callbacks, and tracking.

### Full Lead Injection with All Fields

```bash
curl -G "https://your-server/vicidial/non_agent_api.php" \
  --data-urlencode "source=crm" \
  --data-urlencode "user=apiuser" \
  --data-urlencode "pass=apipass" \
  --data-urlencode "function=add_lead" \
  --data-urlencode "phone_number=5551234567" \
  --data-urlencode "phone_code=1" \
  --data-urlencode "list_id=10001" \
  --data-urlencode "first_name=John" \
  --data-urlencode "last_name=Smith" \
  --data-urlencode "address1=123 Main St" \
  --data-urlencode "city=Phoenix" \
  --data-urlencode "state=AZ" \
  --data-urlencode "postal_code=85001" \
  --data-urlencode "email=john.smith@example.com" \
  --data-urlencode "vendor_lead_code=CRM-2026-44921" \
  --data-urlencode "source_id=google_ads" \
  --data-urlencode "alt_phone=5559876543" \
  --data-urlencode "comments=Requested callback re: Medicare Advantage" \
  --data-urlencode "rank=99" \
  --data-urlencode "owner=team_a"
```

### Key Parameters for add_lead

| Parameter | Required | Description |
|-----------|----------|-------------|
| `phone_number` | Yes | 10-digit phone number (no formatting) |
| `phone_code` | Yes | Country code (1 for US/Canada) |
| `list_id` | Yes | Target list ID — must exist in VICIdial |
| `first_name` | No | Lead first name |
| `last_name` | No | Lead last name |
| `vendor_lead_code` | No | Your external ID — critical for CRM sync |
| `source_id` | No | Lead source tracking (ad campaign, vendor, etc.) |
| `rank` | No | Lead priority (-99 to 99, higher = dialed sooner) |
| `owner` | No | Lead owner — used for routing via owner-based dialing |
| `status` | No | Initial status (defaults to NEW) |
| `duplicate_check` | No | DUPLIST, DUPCAMP, DUPSYS, DUPLISTPHONE, etc. |
| `custom_fields` | No | Y to enable custom field data in the same request |

### Duplicate Handling

The `duplicate_check` parameter controls what happens when a lead with the same phone number already exists:

- `DUPLIST` — Check for duplicates within the same list
- `DUPCAMP` — Check across all lists in the same campaign
- `DUPSYS` — Check across the entire VICIdial system
- `DUPLISTPHONE` — Check for duplicate phone + list combination
- `DUPLISTPHONECAMPAIGN` — Check phone across all lists in campaign

```bash
# Reject duplicates within the campaign
curl -G "https://your-server/vicidial/non_agent_api.php" \
  --data-urlencode "source=webform" \
  --data-urlencode "user=apiuser" \
  --data-urlencode "pass=apipass" \
  --data-urlencode "function=add_lead" \
  --data-urlencode "phone_number=5551234567" \
  --data-urlencode "list_id=10001" \
  --data-urlencode "phone_code=1" \
  --data-urlencode "duplicate_check=DUPCAMP"
```

If a duplicate is found, the response will be:
```
NOTICE: add_lead DUPLICATE -  5551234567|10001|DUPCAMP
```

### Python Lead Injection Script

Here's a production-ready Python script for injecting leads from a CSV file or API webhook:

```python
import requests
import csv
import time
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

VICIDIAL_API = "https://your-server/vicidial/non_agent_api.php"
API_USER = "apiuser"
API_PASS = "apipass"
DEFAULT_LIST = "10001"

def inject_lead(lead_data):
    """Inject a single lead into VICIdial. Returns lead_id on success."""
    params = {
        "source": "lead_injector",
        "user": API_USER,
        "pass": API_PASS,
        "function": "add_lead",
        "phone_code": "1",
        "list_id": lead_data.get("list_id", DEFAULT_LIST),
        "phone_number": lead_data["phone"],
        "first_name": lead_data.get("first_name", ""),
        "last_name": lead_data.get("last_name", ""),
        "address1": lead_data.get("address", ""),
        "city": lead_data.get("city", ""),
        "state": lead_data.get("state", ""),
        "postal_code": lead_data.get("zip", ""),
        "email": lead_data.get("email", ""),
        "vendor_lead_code": lead_data.get("external_id", ""),
        "source_id": lead_data.get("source", ""),
        "duplicate_check": "DUPCAMP",
    }

    try:
        resp = requests.get(VICIDIAL_API, params=params, timeout=10, verify=True)
        result = resp.text.strip()

        if "SUCCESS" in result:
            # Parse lead_id from response: "SUCCESS: add_lead - ... |lead_id"
            lead_id = result.split("|")[-1]
            logger.info(f"Injected lead {lead_data['phone']} -> lead_id {lead_id}")
            return lead_id
        elif "DUPLICATE" in result:
            logger.warning(f"Duplicate lead: {lead_data['phone']}")
            return None
        else:
            logger.error(f"API error: {result}")
            return None

    except requests.RequestException as e:
        logger.error(f"Request failed: {e}")
        return None


def inject_from_csv(filepath):
    """Bulk inject leads from a CSV file."""
    success = 0
    failed = 0
    duplicates = 0

    with open(filepath, "r") as f:
        reader = csv.DictReader(f)
        for row in reader:
            result = inject_lead(row)
            if result:
                success += 1
            elif result is None:
                # Could be duplicate or error — check logs
                duplicates += 1
            else:
                failed += 1
            # Throttle to avoid hammering the API
            time.sleep(0.1)

    logger.info(f"Import complete: {success} added, {duplicates} duplicates, {failed} failed")


# Webhook handler example (Flask)
from flask import Flask, request, jsonify

app = Flask(__name__)

@app.route("/webhook/lead", methods=["POST"])
def receive_lead():
    """Receive leads from external sources and inject into VICIdial."""
    data = request.json
    if not data or "phone" not in data:
        return jsonify({"error": "phone is required"}), 400

    lead_id = inject_lead(data)
    if lead_id:
        return jsonify({"status": "success", "lead_id": lead_id}), 200
    else:
        return jsonify({"status": "duplicate_or_error"}), 409


if __name__ == "__main__":
    # Run as webhook server
    app.run(host="0.0.0.0", port=5000)
```

### PHP Lead Injection

For operations running PHP (which most VICIdial servers already have):

```php
<?php
function vicidial_add_lead($lead) {
    $api_url = "https://your-server/vicidial/non_agent_api.php";

    $params = array(
        'source'          => 'php_injector',
        'user'            => 'apiuser',
        'pass'            => 'apipass',
        'function'        => 'add_lead',
        'phone_code'      => '1',
        'phone_number'    => preg_replace('/[^0-9]/', '', $lead['phone']),
        'list_id'         => $lead['list_id'] ?? '10001',
        'first_name'      => $lead['first_name'] ?? '',
        'last_name'       => $lead['last_name'] ?? '',
        'address1'        => $lead['address'] ?? '',
        'city'            => $lead['city'] ?? '',
        'state'           => $lead['state'] ?? '',
        'postal_code'     => $lead['zip'] ?? '',
        'email'           => $lead['email'] ?? '',
        'vendor_lead_code'=> $lead['external_id'] ?? '',
        'source_id'       => $lead['source'] ?? '',
        'duplicate_check' => 'DUPCAMP',
    );

    $url = $api_url . '?' . http_build_query($params);

    $ch = curl_init();
    curl_setopt($ch, CURLOPT_URL, $url);
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_TIMEOUT, 10);
    curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, true);
    $response = curl_exec($ch);
    $http_code = curl_getinfo($ch, CURLINFO_HTTP_CODE);
    curl_close($ch);

    if ($http_code !== 200) {
        error_log("VICIdial API HTTP error: $http_code");
        return false;
    }

    if (strpos($response, 'SUCCESS') !== false) {
        // Extract lead_id
        $parts = explode('|', $response);
        return end($parts); // lead_id
    }

    if (strpos($response, 'DUPLICATE') !== false) {
        return 'DUPLICATE';
    }

    error_log("VICIdial API error: $response");
    return false;
}

// Usage
$lead_id = vicidial_add_lead([
    'phone'      => '5551234567',
    'first_name' => 'John',
    'last_name'  => 'Smith',
    'city'       => 'Phoenix',
    'state'      => 'AZ',
    'zip'        => '85001',
    'external_id'=> 'CRM-44921',
    'source'     => 'google_ads',
    'list_id'    => '10001',
]);

if ($lead_id && $lead_id !== 'DUPLICATE') {
    echo "Lead injected with ID: $lead_id\n";
}
?>
```

### Custom Fields in add_lead

If your VICIdial lists have custom fields defined (Admin -> Custom Fields), you can populate them during lead injection:

```bash
curl -G "https://your-server/vicidial/non_agent_api.php" \
  --data-urlencode "source=crm" \
  --data-urlencode "user=apiuser" \
  --data-urlencode "pass=apipass" \
  --data-urlencode "function=add_lead" \
  --data-urlencode "phone_number=5551234567" \
  --data-urlencode "phone_code=1" \
  --data-urlencode "list_id=10001" \
  --data-urlencode "first_name=John" \
  --data-urlencode "last_name=Smith" \
  --data-urlencode "custom_fields=Y" \
  --data-urlencode "policy_number=POL-2026-8812" \
  --data-urlencode "insurance_type=medicare_advantage" \
  --data-urlencode "monthly_budget=100"
```

The `custom_fields=Y` flag tells the API to look for additional parameters matching your custom field names. The field names must match exactly what's configured in VICIdial's Custom Fields setup.

> **Stop Hand-Loading Leads.** ViciStack builds automated lead injection pipelines that connect your web forms, CRM, and lead vendors directly to VICIdial — leads flow in real-time, deduplicated and routed to the right campaign. [Get Your Free Integration Audit →](/free-audit/)

---

## Updating Leads with update_lead

Once a lead exists in VICIdial, you can update any field via the API. This is essential for keeping VICIdial data in sync with your CRM, updating [lead scoring](/glossary/lead-scoring/) values, or changing lead status based on external events.

### Basic Lead Update

```bash
curl -G "https://your-server/vicidial/non_agent_api.php" \
  --data-urlencode "source=crm_sync" \
  --data-urlencode "user=apiuser" \
  --data-urlencode "pass=apipass" \
  --data-urlencode "function=update_lead" \
  --data-urlencode "lead_id=12345678" \
  --data-urlencode "first_name=Jonathan" \
  --data-urlencode "email=john.updated@example.com" \
  --data-urlencode "rank=50"
```

### Update by Vendor Lead Code

If you don't have the VICIdial lead_id but you stored your external ID as the `vendor_lead_code`, you can look up by that:

```bash
curl -G "https://your-server/vicidial/non_agent_api.php" \
  --data-urlencode "source=crm_sync" \
  --data-urlencode "user=apiuser" \
  --data-urlencode "pass=apipass" \
  --data-urlencode "function=update_lead" \
  --data-urlencode "vendor_lead_code=CRM-2026-44921" \
  --data-urlencode "search_method=VENDOR_LEAD_CODE" \
  --data-urlencode "search_location=LIST" \
  --data-urlencode "list_id=10001" \
  --data-urlencode "status=CALLBK" \
  --data-urlencode "comments=Customer called back, wants quote"
```

### Changing Lead Status

Status updates are one of the most common API operations. Use them to:

- Mark leads as DNC from your compliance system
- Reset leads to NEW for re-dialing after a cooling period
- Set leads to a callback status from your CRM
- Move leads between statuses based on external workflow triggers

```bash
# Mark a lead as Do Not Call
curl -G "https://your-server/vicidial/non_agent_api.php" \
  --data-urlencode "source=compliance" \
  --data-urlencode "user=apiuser" \
  --data-urlencode "pass=apipass" \
  --data-urlencode "function=update_lead" \
  --data-urlencode "lead_id=12345678" \
  --data-urlencode "status=DNCL"
```

### Bulk Lead Updates with Python

```python
import requests
import time

VICIDIAL_API = "https://your-server/vicidial/non_agent_api.php"
API_USER = "apiuser"
API_PASS = "apipass"

def update_lead(lead_id, updates):
    """Update a lead in VICIdial by lead_id."""
    params = {
        "source": "bulk_updater",
        "user": API_USER,
        "pass": API_PASS,
        "function": "update_lead",
        "lead_id": str(lead_id),
    }
    params.update(updates)

    resp = requests.get(VICIDIAL_API, params=params, timeout=10)
    result = resp.text.strip()

    if "SUCCESS" in result:
        return True
    else:
        print(f"Update failed for lead {lead_id}: {result}")
        return False


# Example: Reset all leads in a list that were called 30+ days ago
def reset_aged_leads(list_id, days=30):
    """Reset old called leads back to NEW status for re-dialing."""
    import mysql.connector

    db = mysql.connector.connect(
        host="localhost", user="cron", password="cronpass", database="asterisk"
    )
    cursor = db.cursor()

    cursor.execute("""
        SELECT lead_id FROM vicidial_list
        WHERE list_id = %s
          AND status IN ('NA', 'NI', 'B', 'DC', 'N')
          AND last_local_call_time < DATE_SUB(NOW(), INTERVAL %s DAY)
    """, (list_id, days))

    leads = cursor.fetchall()
    print(f"Found {len(leads)} aged leads to reset")

    for (lead_id,) in leads:
        update_lead(lead_id, {"status": "NEW"})
        time.sleep(0.05)  # Throttle

    cursor.close()
    db.close()
```

---

## Real-Time Stats: agent_stats and campaign_stats

Pulling live operational data from VICIdial is essential for custom dashboards, alerting systems, and integration with business intelligence tools. For a deep dive on building dashboards with this data, see our [reporting and monitoring guide](/blog/vicidial-reporting-monitoring/).

### Campaign Stats

```bash
curl -G "https://your-server/vicidial/non_agent_api.php" \
  --data-urlencode "source=dashboard" \
  --data-urlencode "user=apiuser" \
  --data-urlencode "pass=apipass" \
  --data-urlencode "function=campaign_stats" \
  --data-urlencode "campaign_id=SALESCAMP" \
  --data-urlencode "stage=csv"
```

The `stage` parameter controls the output format:
- `csv` — CSV formatted output
- `json` — JSON output (available in newer VICIdial builds)

**Response includes:**
- Total calls today
- Total agents logged in
- Agents in each status (INCALL, READY, PAUSED, DEAD)
- Calls waiting in queue
- Average wait time
- Current dial level
- Drop rate
- Hopper count

### Agent Stats

```bash
curl -G "https://your-server/vicidial/non_agent_api.php" \
  --data-urlencode "source=dashboard" \
  --data-urlencode "user=apiuser" \
  --data-urlencode "pass=apipass" \
  --data-urlencode "function=agent_stats" \
  --data-urlencode "agent_user=agent001" \
  --data-urlencode "stage=csv"
```

Returns per-agent metrics: calls taken, talk time, pause time, wait time, dispositions, login/logout times.

### Python Stats Polling for Custom Dashboards

```python
import requests
import json
import time

VICIDIAL_API = "https://your-server/vicidial/non_agent_api.php"
API_USER = "apiuser"
API_PASS = "apipass"

def get_campaign_stats(campaign_id):
    """Pull real-time campaign statistics."""
    params = {
        "source": "dashboard",
        "user": API_USER,
        "pass": API_PASS,
        "function": "campaign_stats",
        "campaign_id": campaign_id,
        "stage": "csv",
    }

    resp = requests.get(VICIDIAL_API, params=params, timeout=10)
    lines = resp.text.strip().split("\n")

    # Parse CSV response into a dict
    stats = {}
    for line in lines:
        if "|" in line:
            parts = line.split("|")
            if len(parts) >= 2:
                stats[parts[0].strip()] = parts[1].strip()

    return stats


def monitor_campaigns(campaign_ids, interval=30):
    """Continuously poll campaign stats and push to your monitoring system."""
    while True:
        for cid in campaign_ids:
            stats = get_campaign_stats(cid)
            # Push to your alerting system, database, or dashboard
            print(f"[{cid}] Agents: {stats.get('agents_logged_in', 'N/A')}, "
                  f"Calls Waiting: {stats.get('calls_waiting', 'N/A')}, "
                  f"Drop Rate: {stats.get('drop_rate', 'N/A')}%")

            # Alert if drop rate exceeds threshold
            drop_rate = float(stats.get("drop_rate", 0))
            if drop_rate > 2.5:
                send_alert(f"WARNING: Campaign {cid} drop rate at {drop_rate}%")

        time.sleep(interval)


def send_alert(message):
    """Send alert via Slack, email, PagerDuty, etc."""
    # Example: Slack webhook
    requests.post("https://hooks.slack.com/services/YOUR/WEBHOOK/URL", json={
        "text": message
    })


if __name__ == "__main__":
    monitor_campaigns(["SALESCAMP", "MEDICARE", "SOLAR"], interval=30)
```

### Direct Database Queries for Stats

For higher-resolution stats or metrics the API doesn't expose, query the MySQL database directly. The key tables:

```sql
-- Real-time agent status
SELECT user, status, campaign_id, calls_today,
       TIMESTAMPDIFF(SECOND, last_state_change, NOW()) as seconds_in_state
FROM vicidial_live_agents
WHERE campaign_id = 'SALESCAMP'
ORDER BY status, seconds_in_state DESC;

-- Campaign performance today
SELECT campaign_id,
       COUNT(*) as total_calls,
       SUM(IF(status = 'SALE', 1, 0)) as sales,
       AVG(length_in_sec) as avg_talk_time,
       SUM(IF(status = 'DROP', 1, 0)) as drops,
       ROUND(SUM(IF(status = 'DROP', 1, 0)) / COUNT(*) * 100, 2) as drop_pct
FROM vicidial_log
WHERE call_date >= CURDATE()
  AND campaign_id = 'SALESCAMP'
GROUP BY campaign_id;

-- Hourly breakdown
SELECT HOUR(call_date) as hour,
       COUNT(*) as calls,
       SUM(IF(status = 'SALE', 1, 0)) as sales,
       ROUND(AVG(length_in_sec), 0) as avg_seconds
FROM vicidial_log
WHERE call_date >= CURDATE()
  AND campaign_id = 'SALESCAMP'
GROUP BY HOUR(call_date)
ORDER BY hour;
```

---

## Disposition Webhooks and Callbacks

VICIdial doesn't have a native webhook system that pushes data to external URLs when events happen. But it has two mechanisms that achieve the same result: the [Start Call URL](/settings/start-call-url/) (and related URL triggers) and disposition-triggered URL calls.

### URL Trigger Settings

VICIdial can call an external URL at specific points in the call lifecycle. These are configured at the campaign level:

| Setting | Trigger Point | Use Case |
|---------|--------------|----------|
| Start Call URL | When a call connects to an agent | Push call data to CRM in real-time |
| Dispo Call URL | When the agent dispositions the call | Sync disposition to CRM, trigger workflows |
| No Agent Call URL | When a call connects but no agent is available | Log abandoned calls, trigger callback |
| Extension Append Dispo Call URL | On disposition with extension data | Advanced routing workflows |

### Configuring the Dispo Call URL (Disposition Webhook)

This is the most useful webhook for CRM integration. When an agent dispositions a call, VICIdial fires an HTTP request to your URL with full call and lead data.

**Campaign Detail -> Dispo Call URL:**

```
https://your-crm.example.com/api/vicidial/disposition?lead_id=--A--lead_id--B--&phone=--A--phone_number--B--&status=--A--dispo--B--&agent=--A--user--B--&talk_time=--A--talk_time--B--&first_name=--A--first_name--B--&last_name=--A--last_name--B--&vendor_lead_code=--A--vendor_lead_code--B--&campaign=--A--campaign--B--&call_id=--A--call_id--B--&uniqueid=--A--uniqueid--B--&recording=--A--recording_filename--B--
```

VICIdial replaces the `--A--field--B--` variables with actual values at disposition time, then makes an HTTP GET request to the resulting URL.

### Building a Disposition Webhook Receiver

Here's a complete webhook receiver that processes VICIdial dispositions and syncs them to external systems:

**Python (Flask):**

```python
from flask import Flask, request, jsonify
import logging
import requests
from datetime import datetime

app = Flask(__name__)
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

@app.route("/api/vicidial/disposition", methods=["GET", "POST"])
def handle_disposition():
    """Receive disposition webhooks from VICIdial's Dispo Call URL."""
    # VICIdial sends as GET parameters
    data = {
        "lead_id": request.args.get("lead_id"),
        "phone": request.args.get("phone"),
        "status": request.args.get("status"),
        "agent": request.args.get("agent"),
        "talk_time": request.args.get("talk_time"),
        "first_name": request.args.get("first_name"),
        "last_name": request.args.get("last_name"),
        "vendor_lead_code": request.args.get("vendor_lead_code"),
        "campaign": request.args.get("campaign"),
        "call_id": request.args.get("call_id"),
        "uniqueid": request.args.get("uniqueid"),
        "recording": request.args.get("recording"),
        "timestamp": datetime.utcnow().isoformat(),
    }

    logger.info(f"Disposition received: {data['phone']} -> {data['status']} by {data['agent']}")

    # Route based on disposition
    status = data["status"]

    if status == "SALE":
        handle_sale(data)
    elif status == "CALLBK":
        handle_callback_request(data)
    elif status == "DNCL":
        handle_dnc(data)
    elif status in ("NI", "NA", "B", "DC"):
        handle_no_contact(data)

    # Always sync to CRM
    sync_to_crm(data)

    return jsonify({"status": "received"}), 200


def handle_sale(data):
    """Process a sale disposition — trigger fulfillment, notify manager."""
    logger.info(f"SALE: {data['first_name']} {data['last_name']} by {data['agent']}")
    # Send Slack notification
    requests.post("https://hooks.slack.com/services/YOUR/WEBHOOK/URL", json={
        "text": f"New sale by {data['agent']}: {data['first_name']} {data['last_name']} "
                f"({data['phone']}) - Talk time: {data['talk_time']}s"
    })


def handle_callback_request(data):
    """Process a callback disposition — schedule in your system."""
    logger.info(f"Callback requested for {data['phone']}")


def handle_dnc(data):
    """Process a DNC disposition — add to suppression lists."""
    logger.info(f"DNC: {data['phone']}")


def handle_no_contact(data):
    """Process no-contact dispositions — update lead scoring."""
    logger.info(f"No contact ({data['status']}): {data['phone']}")


def sync_to_crm(data):
    """Push disposition data to your CRM."""
    # Example: HubSpot API
    # Example: Salesforce API
    # Example: Custom database
    pass


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5001)
```

**PHP:**

```php
<?php
// webhook_handler.php — receives VICIdial disposition webhooks

$data = array(
    'lead_id'          => $_GET['lead_id'] ?? '',
    'phone'            => $_GET['phone'] ?? '',
    'status'           => $_GET['status'] ?? '',
    'agent'            => $_GET['agent'] ?? '',
    'talk_time'        => $_GET['talk_time'] ?? '',
    'first_name'       => $_GET['first_name'] ?? '',
    'last_name'        => $_GET['last_name'] ?? '',
    'vendor_lead_code' => $_GET['vendor_lead_code'] ?? '',
    'campaign'         => $_GET['campaign'] ?? '',
    'call_id'          => $_GET['call_id'] ?? '',
    'uniqueid'         => $_GET['uniqueid'] ?? '',
    'recording'        => $_GET['recording'] ?? '',
    'timestamp'        => date('Y-m-d H:i:s'),
);

// Log the disposition
error_log("DISPO: {$data['phone']} -> {$data['status']} by {$data['agent']}");

// Insert into your tracking database
$pdo = new PDO('mysql:host=localhost;dbname=your_crm', 'dbuser', 'dbpass');
$stmt = $pdo->prepare("
    INSERT INTO call_dispositions
    (lead_id, phone, status, agent, talk_time, first_name, last_name,
     vendor_lead_code, campaign, call_id, uniqueid, recording, created_at)
    VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, NOW())
");
$stmt->execute([
    $data['lead_id'], $data['phone'], $data['status'], $data['agent'],
    $data['talk_time'], $data['first_name'], $data['last_name'],
    $data['vendor_lead_code'], $data['campaign'], $data['call_id'],
    $data['uniqueid'], $data['recording']
]);

// Route based on disposition status
switch ($data['status']) {
    case 'SALE':
        // Trigger order fulfillment
        trigger_fulfillment($data);
        break;
    case 'CALLBK':
        // Schedule callback in external system
        schedule_external_callback($data);
        break;
    case 'DNCL':
        // Add to suppression list
        add_to_suppression($data);
        break;
}

// Return 200 OK
http_response_code(200);
echo "OK";

function trigger_fulfillment($data) {
    // Your fulfillment logic
}

function schedule_external_callback($data) {
    // Your callback scheduling logic
}

function add_to_suppression($data) {
    // Your DNC/suppression logic
}
?>
```

### Start Call URL: Real-Time Screen Pops

The Start Call URL fires when a call connects to an agent — before the agent says a word. This is how you build real-time screen pops in external CRMs. When the call connects, VICIdial hits your URL, your system receives the lead data, and you can pop the relevant record in the CRM or web application.

Configure it at Campaign Detail -> [Start Call URL](/settings/start-call-url/):

```
https://your-crm.example.com/api/screen-pop?agent=--A--user--B--&lead_id=--A--lead_id--B--&phone=--A--phone_number--B--&first_name=--A--first_name--B--&last_name=--A--last_name--B--&vendor_lead_code=--A--vendor_lead_code--B--
```

---

## Click-to-Call via the Agent API

Click-to-call lets external applications — CRMs, web pages, custom dashboards — initiate an outbound call through VICIdial. The agent doesn't need to type a number or navigate the dialer. They click a button in your application, and VICIdial dials the number through their active session.

### How Click-to-Call Works

1. Agent is logged into VICIdial and in READY or PAUSED state
2. External application calls the Agent API's `external_dial` function
3. VICIdial places the call through the agent's session
4. The call connects and appears in the agent's screen as a normal call

### Implementation

```bash
# Dial a number through an agent's session
curl -G "https://your-server/agc/api.php" \
  --data-urlencode "source=crm" \
  --data-urlencode "user=agent001" \
  --data-urlencode "pass=agentpass" \
  --data-urlencode "function=external_dial" \
  --data-urlencode "agent_user=agent001" \
  --data-urlencode "value=5551234567" \
  --data-urlencode "phone_code=1" \
  --data-urlencode "search=YES" \
  --data-urlencode "preview=NO" \
  --data-urlencode "focus=YES"
```

**Parameters:**

| Parameter | Description |
|-----------|-------------|
| `value` | Phone number to dial |
| `phone_code` | Country code |
| `search` | YES = look up the number in VICIdial's leads and load data if found |
| `preview` | YES = show lead preview before dialing, NO = dial immediately |
| `focus` | YES = bring the VICIdial agent screen to focus |
| `lead_id` | Optional — specify a lead_id to associate with this call |
| `vendor_lead_code` | Optional — look up by vendor code instead of phone |

### Click-to-Call from a CRM (JavaScript)

```javascript
/**
 * VICIdial click-to-call integration for CRM web applications.
 * Embed this in your CRM and call vicidialDial() with a phone number.
 */
const ViciDialer = {
    apiUrl: "https://your-vicidial-server/agc/api.php",
    agentUser: null,
    agentPass: null,

    init(agentUser, agentPass) {
        this.agentUser = agentUser;
        this.agentPass = agentPass;
    },

    async dial(phoneNumber, options = {}) {
        const params = new URLSearchParams({
            source: "crm_click2call",
            user: this.agentUser,
            pass: this.agentPass,
            function: "external_dial",
            agent_user: this.agentUser,
            value: phoneNumber.replace(/\D/g, ""),
            phone_code: options.phoneCode || "1",
            search: options.search || "YES",
            preview: options.preview || "NO",
            focus: options.focus || "YES",
        });

        if (options.leadId) {
            params.set("lead_id", options.leadId);
        }

        try {
            const resp = await fetch(`${this.apiUrl}?${params}`);
            const result = await resp.text();

            if (result.includes("SUCCESS")) {
                return { success: true, message: "Call initiated" };
            } else {
                return { success: false, message: result.trim() };
            }
        } catch (error) {
            return { success: false, message: error.message };
        }
    },

    async hangup() {
        const params = new URLSearchParams({
            source: "crm_click2call",
            user: this.agentUser,
            pass: this.agentPass,
            function: "external_hangup",
            agent_user: this.agentUser,
            value: "1",
        });

        const resp = await fetch(`${this.apiUrl}?${params}`);
        return resp.text();
    },

    async pause(pauseCode = "BREAK") {
        const params = new URLSearchParams({
            source: "crm_click2call",
            user: this.agentUser,
            pass: this.agentPass,
            function: "external_pause",
            agent_user: this.agentUser,
            value: "PAUSE",
            pause_code: pauseCode,
        });

        const resp = await fetch(`${this.apiUrl}?${params}`);
        return resp.text();
    },

    async resume() {
        const params = new URLSearchParams({
            source: "crm_click2call",
            user: this.agentUser,
            pass: this.agentPass,
            function: "external_pause",
            agent_user: this.agentUser,
            value: "RESUME",
        });

        const resp = await fetch(`${this.apiUrl}?${params}`);
        return resp.text();
    }
};

// Usage in your CRM
ViciDialer.init("agent001", "agentpass");

// Attach to CRM buttons
document.querySelectorAll(".dial-button").forEach(btn => {
    btn.addEventListener("click", async function() {
        const phone = this.dataset.phone;
        const result = await ViciDialer.dial(phone);
        if (result.success) {
            showNotification("Call initiated to " + phone);
        } else {
            showNotification("Dial failed: " + result.message, "error");
        }
    });
});
```

### Click-to-Call with Lead ID Lookup

When your CRM knows the VICIdial lead ID (because you stored it during injection), you can dial by lead_id to automatically load all lead data into the agent screen:

```bash
curl -G "https://your-server/agc/api.php" \
  --data-urlencode "source=crm" \
  --data-urlencode "user=agent001" \
  --data-urlencode "pass=agentpass" \
  --data-urlencode "function=external_dial" \
  --data-urlencode "agent_user=agent001" \
  --data-urlencode "value=5551234567" \
  --data-urlencode "lead_id=12345678" \
  --data-urlencode "phone_code=1" \
  --data-urlencode "search=YES" \
  --data-urlencode "preview=NO" \
  --data-urlencode "focus=YES"
```

This is significantly better than dialing by phone number alone. The agent screen populates with the full lead record — name, address, notes, custom fields, call history — instead of just a phone number.

---

## Campaign Control via API

The Non-Agent API lets you start, stop, and modify campaigns programmatically. This enables scheduled campaign activation, automated responses to performance metrics, and integration with workforce management systems.

### Starting and Stopping Campaigns

```bash
# Activate a campaign
curl -G "https://your-server/vicidial/non_agent_api.php" \
  --data-urlencode "source=scheduler" \
  --data-urlencode "user=apiuser" \
  --data-urlencode "pass=apipass" \
  --data-urlencode "function=update_campaign" \
  --data-urlencode "campaign_id=SALESCAMP" \
  --data-urlencode "active=Y"

# Deactivate a campaign
curl -G "https://your-server/vicidial/non_agent_api.php" \
  --data-urlencode "source=scheduler" \
  --data-urlencode "user=apiuser" \
  --data-urlencode "pass=apipass" \
  --data-urlencode "function=update_campaign" \
  --data-urlencode "campaign_id=SALESCAMP" \
  --data-urlencode "active=N"
```

### Adjusting Dial Level

```bash
# Set the dial level to a specific value
curl -G "https://your-server/vicidial/non_agent_api.php" \
  --data-urlencode "source=optimizer" \
  --data-urlencode "user=apiuser" \
  --data-urlencode "pass=apipass" \
  --data-urlencode "function=update_campaign" \
  --data-urlencode "campaign_id=SALESCAMP" \
  --data-urlencode "dial_level=2.5"
```

### Automated Campaign Management Script

This Python script implements basic automated campaign management — starting campaigns on schedule, adjusting dial levels based on agent availability, and shutting down when leads run dry:

```python
import requests
import time
from datetime import datetime

VICIDIAL_API = "https://your-server/vicidial/non_agent_api.php"
API_USER = "apiuser"
API_PASS = "apipass"

def api_call(function, **kwargs):
    """Make a VICIdial API call."""
    params = {
        "source": "campaign_manager",
        "user": API_USER,
        "pass": API_PASS,
        "function": function,
    }
    params.update(kwargs)
    resp = requests.get(VICIDIAL_API, params=params, timeout=10)
    return resp.text.strip()


def update_campaign(campaign_id, **settings):
    """Update campaign settings."""
    return api_call("update_campaign", campaign_id=campaign_id, **settings)


def get_campaign_stats(campaign_id):
    """Get campaign stats as a dict."""
    result = api_call("campaign_stats", campaign_id=campaign_id, stage="csv")
    stats = {}
    for line in result.split("\n"):
        if "|" in line:
            parts = line.split("|")
            if len(parts) >= 2:
                stats[parts[0].strip()] = parts[1].strip()
    return stats


def campaign_scheduler():
    """
    Automated campaign management loop.
    - Start campaigns at configured times
    - Monitor hopper levels and drop rates
    - Alert on problems
    - Stop campaigns at end of shift
    """
    campaigns = {
        "SALESCAMP": {"start": "09:00", "stop": "17:00", "min_agents": 5},
        "MEDICARE":  {"start": "08:00", "stop": "20:00", "min_agents": 3},
    }

    while True:
        now = datetime.now()
        current_time = now.strftime("%H:%M")

        for camp_id, config in campaigns.items():
            stats = get_campaign_stats(camp_id)
            agents = int(stats.get("agents_logged_in", 0))
            hopper = int(stats.get("hopper_count", 0))

            # Start campaign if it's time and enough agents
            if current_time >= config["start"] and current_time < config["stop"]:
                if agents >= config["min_agents"]:
                    update_campaign(camp_id, active="Y")

                # Alert if hopper is running low
                if hopper < 50 and agents > 0:
                    send_alert(f"Campaign {camp_id} hopper low: {hopper} leads remaining")

            # Stop campaign at end of shift
            if current_time >= config["stop"]:
                update_campaign(camp_id, active="N")

        time.sleep(60)


def send_alert(message):
    print(f"ALERT: {message}")
    # Add your Slack/email/PagerDuty integration here
```

---

## Agent API: Session Control

The Agent API goes beyond click-to-call. It provides full control over the agent session — pause, resume, disposition, transfer, hangup, and more. This is what you use when building a custom agent interface or integrating VICIdial controls into a CRM.

### Pausing and Resuming Agents

```bash
# Pause an agent with a pause code
curl -G "https://your-server/agc/api.php" \
  --data-urlencode "source=crm" \
  --data-urlencode "user=agent001" \
  --data-urlencode "pass=agentpass" \
  --data-urlencode "function=external_pause" \
  --data-urlencode "agent_user=agent001" \
  --data-urlencode "value=PAUSE" \
  --data-urlencode "pause_code=LUNCH"

# Resume an agent
curl -G "https://your-server/agc/api.php" \
  --data-urlencode "source=crm" \
  --data-urlencode "user=agent001" \
  --data-urlencode "pass=agentpass" \
  --data-urlencode "function=external_pause" \
  --data-urlencode "agent_user=agent001" \
  --data-urlencode "value=RESUME"
```

### Dispositioning Calls

```bash
# Set the disposition for the current call
curl -G "https://your-server/agc/api.php" \
  --data-urlencode "source=crm" \
  --data-urlencode "user=agent001" \
  --data-urlencode "pass=agentpass" \
  --data-urlencode "function=external_status" \
  --data-urlencode "agent_user=agent001" \
  --data-urlencode "value=SALE"
```

This programmatically dispositions the active call with the specified status code. The status must be a valid [disposition](/glossary/disposition/) code for the agent's current campaign. After this call, the agent is moved to the next state (typically back to READY or to wrap-up, depending on campaign settings).

### Hanging Up Calls

```bash
curl -G "https://your-server/agc/api.php" \
  --data-urlencode "source=crm" \
  --data-urlencode "user=agent001" \
  --data-urlencode "pass=agentpass" \
  --data-urlencode "function=external_hangup" \
  --data-urlencode "agent_user=agent001" \
  --data-urlencode "value=1"
```

### Transferring Calls

```bash
# Blind transfer to an extension or In-Group
curl -G "https://your-server/agc/api.php" \
  --data-urlencode "source=crm" \
  --data-urlencode "user=agent001" \
  --data-urlencode "pass=agentpass" \
  --data-urlencode "function=transfer_conference" \
  --data-urlencode "agent_user=agent001" \
  --data-urlencode "value=BLIND_TRANSFER" \
  --data-urlencode "ingroup_choices=CLOSERS"
```

### Updating Lead Data from Agent Side

During a live call, you can update the lead's data fields via the Agent API:

```bash
curl -G "https://your-server/agc/api.php" \
  --data-urlencode "source=crm" \
  --data-urlencode "user=agent001" \
  --data-urlencode "pass=agentpass" \
  --data-urlencode "function=update_fields" \
  --data-urlencode "agent_user=agent001" \
  --data-urlencode "first_name=Jonathan" \
  --data-urlencode "email=j.smith@example.com" \
  --data-urlencode "comments=Interested in Plan G supplement"
```

This updates the lead record in real-time while the call is active. The agent screen reflects the changes immediately.

### Getting Agent Session Status

```bash
curl -G "https://your-server/agc/api.php" \
  --data-urlencode "source=crm" \
  --data-urlencode "user=agent001" \
  --data-urlencode "pass=agentpass" \
  --data-urlencode "function=external_status" \
  --data-urlencode "agent_user=agent001" \
  --data-urlencode "value=---"
```

Passing `---` as the value returns the agent's current status without changing it. Use this to check agent state before issuing commands.

---

## DNC Checking and Management

The Non-Agent API provides DNC (Do Not Call) list checking, which is critical for compliance workflows and lead validation.

### Checking a Number Against DNC Lists

```bash
curl -G "https://your-server/vicidial/non_agent_api.php" \
  --data-urlencode "source=compliance" \
  --data-urlencode "user=apiuser" \
  --data-urlencode "pass=apipass" \
  --data-urlencode "function=dnc_check" \
  --data-urlencode "phone_number=5551234567"
```

**Response:**
```
# If on DNC list:
MAIN DNC - 5551234567
# If clean:
NOT ON DNC - 5551234567
```

### Bulk DNC Validation (Python)

```python
import requests
import csv

VICIDIAL_API = "https://your-server/vicidial/non_agent_api.php"
API_USER = "apiuser"
API_PASS = "apipass"

def check_dnc(phone):
    """Check a phone number against VICIdial's DNC list."""
    params = {
        "source": "dnc_checker",
        "user": API_USER,
        "pass": API_PASS,
        "function": "dnc_check",
        "phone_number": phone,
    }
    resp = requests.get(VICIDIAL_API, params=params, timeout=10)
    result = resp.text.strip()
    return "DNC" in result and "NOT ON DNC" not in result


def validate_lead_file(input_csv, output_csv):
    """Filter a lead file against VICIdial's DNC list."""
    clean = 0
    blocked = 0

    with open(input_csv, "r") as fin, open(output_csv, "w", newline="") as fout:
        reader = csv.DictReader(fin)
        writer = csv.DictWriter(fout, fieldnames=reader.fieldnames)
        writer.writeheader()

        for row in reader:
            phone = row.get("phone", "").strip()
            if not phone:
                continue

            if check_dnc(phone):
                blocked += 1
                print(f"DNC: {phone}")
            else:
                writer.writerow(row)
                clean += 1

    print(f"Results: {clean} clean, {blocked} blocked by DNC")
```

---

## Lead Search and Lookup

The `lead_search` function lets you find existing leads by phone number, lead ID, or other criteria. This is essential for deduplication checks, CRM lookups, and building search interfaces.

### Search by Phone Number

```bash
curl -G "https://your-server/vicidial/non_agent_api.php" \
  --data-urlencode "source=crm" \
  --data-urlencode "user=apiuser" \
  --data-urlencode "pass=apipass" \
  --data-urlencode "function=lead_search" \
  --data-urlencode "phone_number=5551234567" \
  --data-urlencode "stage=csv"
```

### Search by Lead ID

```bash
curl -G "https://your-server/vicidial/non_agent_api.php" \
  --data-urlencode "source=crm" \
  --data-urlencode "user=apiuser" \
  --data-urlencode "pass=apipass" \
  --data-urlencode "function=lead_field_info" \
  --data-urlencode "lead_id=12345678" \
  --data-urlencode "stage=csv"
```

This returns all fields for the specified lead, including custom fields. Use `lead_field_info` when you know the lead ID and need full data, and `lead_search` when you're searching by phone or other criteria.

---

## Callback Management

VICIdial supports both agent-specific and campaign-wide callbacks via the API. This lets external systems schedule callbacks that VICIdial will automatically dial at the specified time.

### Adding a Callback

```bash
curl -G "https://your-server/vicidial/non_agent_api.php" \
  --data-urlencode "source=crm" \
  --data-urlencode "user=apiuser" \
  --data-urlencode "pass=apipass" \
  --data-urlencode "function=add_callback" \
  --data-urlencode "lead_id=12345678" \
  --data-urlencode "callback_datetime=2026-03-19 14:30:00" \
  --data-urlencode "callback_type=USERONLY" \
  --data-urlencode "callback_user=agent001" \
  --data-urlencode "callback_comments=Customer wants to discuss Plan G options"
```

**Callback types:**
- `USERONLY` — Only the specified agent will receive the callback
- `ANYONE` — Any available agent in the campaign can handle the callback

### Removing a Callback

```bash
curl -G "https://your-server/vicidial/non_agent_api.php" \
  --data-urlencode "source=crm" \
  --data-urlencode "user=apiuser" \
  --data-urlencode "pass=apipass" \
  --data-urlencode "function=remove_callback" \
  --data-urlencode "lead_id=12345678" \
  --data-urlencode "callback_user=agent001"
```

---

## Recording Lookup

Pull call recording metadata via the API. This is useful for quality assurance integrations, compliance audits, and CRM call history features.

```bash
curl -G "https://your-server/vicidial/non_agent_api.php" \
  --data-urlencode "source=qa_system" \
  --data-urlencode "user=apiuser" \
  --data-urlencode "pass=apipass" \
  --data-urlencode "function=recording_lookup" \
  --data-urlencode "lead_id=12345678" \
  --data-urlencode "stage=csv"
```

Returns recording filenames, timestamps, and agent information for all recordings associated with a lead. You can then construct the download URL:

```
https://your-server/RECORDINGS/MP3/20260318-140532_5551234567_agent001-all.mp3
```

The exact path depends on your recording configuration. Check your `recording_exten` and recording path settings in System Settings.

---

## Error Handling and Rate Limiting

### Common API Errors

| Error Response | Cause | Fix |
|---------------|-------|-----|
| `ERROR: NO FUNCTION SPECIFIED` | Missing `function` parameter | Add `function=add_lead` (or whatever function) |
| `ERROR: Invalid Source` | `source` parameter missing or too long | Add `source=your_app_name` (keep it short) |
| `ERROR: Invalid Username` | User doesn't exist or API access disabled | Check user exists and has API access enabled |
| `ERROR: Invalid Password` | Wrong password | Check credentials |
| `ERROR: user_not_found` | Agent user doesn't exist (Agent API) | Verify agent username |
| `ERROR: agent_not_logged_in` | Agent isn't in an active session (Agent API) | Agent must be logged into the dialer |
| `ERROR: Function Not Allowed` | API user doesn't have permission for this function | Update `api_allowed_functions` for the user |
| `ERROR: add_lead INVALID PHONE NUMBER LENGTH` | Phone number isn't 6-16 digits | Strip formatting, ensure correct length |
| `ERROR: add_lead INVALID LIST ID` | Specified list_id doesn't exist | Verify list exists in VICIdial |
| `NOTICE: add_lead DUPLICATE` | Duplicate detected per `duplicate_check` setting | Not an error — handle as expected behavior |

### Implementing Retry Logic

VICIdial's API can be slow under load — it's PHP hitting a MySQL database with no caching layer. Build retry logic into your integration:

```python
import requests
import time
import logging

logger = logging.getLogger(__name__)

def vicidial_api_call(url, params, max_retries=3, backoff=2):
    """Make a VICIdial API call with retry logic."""
    for attempt in range(max_retries):
        try:
            resp = requests.get(url, params=params, timeout=15)

            if resp.status_code == 200:
                result = resp.text.strip()

                # Check for transient errors
                if "ERROR: server_busy" in result or "ERROR: timeout" in result:
                    logger.warning(f"Transient error (attempt {attempt + 1}): {result}")
                    time.sleep(backoff ** attempt)
                    continue

                return result

            elif resp.status_code in (502, 503, 504):
                logger.warning(f"HTTP {resp.status_code} (attempt {attempt + 1})")
                time.sleep(backoff ** attempt)
                continue

            else:
                logger.error(f"HTTP {resp.status_code}: {resp.text}")
                return None

        except requests.Timeout:
            logger.warning(f"Timeout (attempt {attempt + 1})")
            time.sleep(backoff ** attempt)
            continue

        except requests.ConnectionError as e:
            logger.error(f"Connection error: {e}")
            time.sleep(backoff ** attempt)
            continue

    logger.error(f"All {max_retries} retries failed")
    return None
```

### Rate Limiting Considerations

VICIdial has no built-in API rate limiting. Every API call generates a PHP process and a MySQL query. On a busy dialer server handling 100+ concurrent agents, flooding the API with requests can degrade dialer performance.

**Guidelines:**
- **Lead injection:** Throttle to 5-10 leads per second. For bulk imports, use a queue with a rate limiter.
- **Stats polling:** Poll no more than once every 15-30 seconds per campaign. VICIdial's own real-time reports poll every 4 seconds, so you have headroom, but don't go below 10-second intervals.
- **Agent API calls:** These are typically low-volume (one call per agent action), so rate limiting isn't usually a concern.
- **Batch operations:** For bulk lead updates or DNC checks, add a 50-100ms delay between requests.

If you need high-throughput data access (thousands of queries per minute), bypass the API entirely and query the MySQL database directly. The API is a convenience layer over the database — for analytics and bulk operations, direct SQL is faster and puts less load on the web server.

> **Your API Integration Shouldn't Be a Weekend Project.** ViciStack builds production-grade API integrations that connect VICIdial to your CRM, marketing stack, and custom workflows — with proper error handling, retry logic, monitoring, and documentation. [Talk to Our Engineering Team →](/free-audit/)

---

## Building Custom AGI Integrations

For call-flow-level automation — things that happen during the call itself rather than before or after — VICIdial uses Asterisk's [AGI](/glossary/agi/) (Asterisk Gateway Interface). AGI scripts execute during call processing and can make routing decisions, play prompts, collect DTMF input, and interact with external APIs in real-time.

### When to Use AGI vs. the API

| Scenario | Use API | Use AGI |
|----------|---------|---------|
| Inject leads before dialing | Yes | No |
| Route calls based on IVR input | No | Yes |
| Update lead data after disposition | Yes | No |
| Play dynamic prompts during a call | No | Yes |
| Trigger a screen pop | Yes | No |
| Check a database mid-call for routing | No | Yes |
| Schedule a callback | Yes | No |
| Collect DTMF and route accordingly | No | Yes |

AGI scripting is beyond the scope of this guide, but it's worth knowing where it fits. VICIdial's call flow uses AGI scripts extensively (the `agi-VDADtransfer.agi` and `agi-VDAD_ALL_inbound.agi` scripts handle most call routing). Custom AGI scripts can be added to the dialplan for advanced routing logic that the API and admin GUI can't handle.

---

## Integration Patterns: Putting It All Together

Here are the most common integration architectures we deploy, and when to use each.

### Pattern 1: Web Form to Dialer (Speed-to-Lead)

A prospect fills out a web form. The lead is immediately injected into VICIdial and dialed within seconds. This is the highest-converting lead handling pattern for inbound marketing.

**Flow:**
1. Web form submits to your server
2. Your server calls VICIdial's `add_lead` API
3. Lead is inserted with high `rank` value (e.g., 99)
4. VICIdial's hopper picks up the high-priority lead
5. The lead is dialed within the next hopper cycle (typically 15-30 seconds)

```python
@app.route("/submit-form", methods=["POST"])
def handle_form():
    """Receive a web form submission and inject into VICIdial immediately."""
    lead = {
        "phone": request.form["phone"],
        "first_name": request.form["first_name"],
        "last_name": request.form["last_name"],
        "email": request.form["email"],
        "source": "website_form",
        "external_id": f"WEB-{int(time.time())}",
    }

    # Inject with high priority for immediate dialing
    params = {
        "source": "website",
        "user": API_USER,
        "pass": API_PASS,
        "function": "add_lead",
        "phone_number": lead["phone"],
        "phone_code": "1",
        "list_id": "10001",
        "first_name": lead["first_name"],
        "last_name": lead["last_name"],
        "email": lead["email"],
        "vendor_lead_code": lead["external_id"],
        "source_id": lead["source"],
        "rank": "99",           # Highest priority
        "duplicate_check": "DUPCAMP",
    }

    resp = requests.get(VICIDIAL_API, params=params, timeout=10)

    if "SUCCESS" in resp.text:
        return redirect("/thank-you")
    else:
        return redirect("/form-error")
```

### Pattern 2: CRM Bidirectional Sync

VICIdial and your CRM stay in sync. Leads flow from the CRM to VICIdial for dialing, and dispositions flow back from VICIdial to the CRM.

**Flow:**
1. CRM creates/updates a lead -> webhook triggers `add_lead` or `update_lead`
2. VICIdial dials the lead
3. Agent dispositions the call -> Dispo Call URL triggers a webhook to your CRM
4. CRM receives the disposition and updates the contact record

This is the pattern described in detail in our [CRM integration guide](/blog/vicidial-crm-integration/).

### Pattern 3: Scheduled Campaign Automation

Campaigns start and stop on schedule, dial levels adjust automatically based on agent count and performance, and alerts fire when metrics go out of bounds.

**Components:**
- Cron job or long-running daemon polling `campaign_stats`
- Logic to `update_campaign` based on time of day and metrics
- Alert integration (Slack, email, PagerDuty) for threshold breaches
- The campaign management script from the Campaign Control section above

### Pattern 4: Custom Agent Desktop

Replace the VICIdial agent screen with a custom web application that communicates entirely via the Agent API. The agent logs into your application instead of `vicidial.php`. Your application handles all call controls, lead display, disposition, and data entry — calling the Agent API for each action.

**This is the nuclear option.** It gives you full control over the agent experience but requires significant development effort. You need to implement:

- Agent login/logout via API
- Call state polling (checking if a call is active, who it's to, etc.)
- Dial controls (manual dial, hangup, transfer)
- Disposition interface
- Lead data display (pulling from VICIdial's database or your CRM)
- Recording controls
- Callback scheduling

This pattern is most common in operations that already have a sophisticated CRM and want agents to live in a single application. For more on customizing the agent screen within VICIdial's native interface, see our [agent screen customization guide](/blog/vicidial-agent-screen-customization/).

---

## Testing Your Integration

### The Test Environment Approach

Never develop against your production VICIdial server. Set up a test environment:

1. **Clone your production server** to a VM or cloud instance
2. **Create a test campaign** with a test list containing internal phone numbers
3. **Create test API users** with access limited to the test campaign's lists
4. **Use a test webhook receiver** (ngrok or a staging server) for URL triggers

### Verifying API Responses

Build a simple test script that exercises every API endpoint you plan to use:

```bash
#!/bin/bash
# test_vicidial_api.sh — verify API connectivity and functions

SERVER="https://your-server"
USER="apiuser"
PASS="apipass"

echo "=== Testing Non-Agent API ==="

# Test 1: Version check
echo -n "Version: "
curl -s "$SERVER/vicidial/non_agent_api.php?source=test&user=$USER&pass=$PASS&function=version"
echo ""

# Test 2: Add a test lead
echo -n "Add lead: "
curl -s "$SERVER/vicidial/non_agent_api.php?source=test&user=$USER&pass=$PASS&function=add_lead&phone_number=5550000001&phone_code=1&list_id=99999&first_name=Test&last_name=Lead"
echo ""

# Test 3: Search for the test lead
echo -n "Search lead: "
curl -s "$SERVER/vicidial/non_agent_api.php?source=test&user=$USER&pass=$PASS&function=lead_search&phone_number=5550000001"
echo ""

# Test 4: Update the test lead
echo -n "Update lead: "
curl -s "$SERVER/vicidial/non_agent_api.php?source=test&user=$USER&pass=$PASS&function=update_lead&phone_number=5550000001&search_method=PHONE_NUMBER&first_name=Updated"
echo ""

# Test 5: DNC check
echo -n "DNC check: "
curl -s "$SERVER/vicidial/non_agent_api.php?source=test&user=$USER&pass=$PASS&function=dnc_check&phone_number=5550000001"
echo ""

# Test 6: Campaign stats
echo -n "Campaign stats: "
curl -s "$SERVER/vicidial/non_agent_api.php?source=test&user=$USER&pass=$PASS&function=campaign_stats&campaign_id=SALESCAMP"
echo ""

echo "=== Tests Complete ==="
```

### Monitoring API Health in Production

Add API health monitoring to your existing monitoring stack:

```python
def check_api_health():
    """Verify VICIdial API is responding correctly."""
    try:
        resp = requests.get(VICIDIAL_API, params={
            "source": "healthcheck",
            "user": API_USER,
            "pass": API_PASS,
            "function": "version",
        }, timeout=5)

        if resp.status_code == 200 and "VERSION" in resp.text:
            return True
        else:
            return False
    except Exception:
        return False
```

---

## Where ViciStack Fits In

Most VICIdial operators use the admin GUI for everything. They manually upload CSV files, manually adjust dial levels, manually check reports, and manually log into their CRM to update records after calls. That works at 10 agents. At 50, it's a bottleneck. At 200, it's a crisis.

The API is what turns VICIdial from a standalone dialer into the engine of an integrated call center operation. But building reliable API integrations takes time, domain expertise, and an understanding of VICIdial's quirks — the undocumented parameters, the error responses that don't match the documentation, the edge cases where the API returns success but the lead didn't actually insert because a list was inactive.

ViciStack builds and maintains API integrations for every deployment. Lead injection from web forms, CRM bidirectional sync, real-time dashboards pulling stats every 15 seconds, disposition webhooks triggering fulfillment workflows, click-to-call from Salesforce and HubSpot, and automated campaign management that starts and stops campaigns on schedule without anyone touching the GUI.

**ViciStack handles the plumbing so you can focus on selling.**

[Get a free API integration audit ->](/free-audit/)

---

*This guide is maintained by the ViciStack team and updated as VICIdial releases add new API capabilities. Last updated: March 2026.*

---

## Frequently Asked Questions

### Does VICIdial have a REST API?

Not in the modern sense. VICIdial's API is HTTP-based — you make GET or POST requests to PHP endpoints and receive plain text or CSV responses. There's no RESTful resource model, no JSON by default (though some functions support a JSON stage), no HATEOAS, no versioning. It's a functional API: you call a function with parameters and get a result. This makes it simple to integrate with but requires wrapper code if you want to work with it like a modern REST API. The Python and PHP examples in this guide show how to build that wrapper layer.

### Can I use the API to build a completely custom agent interface?

Yes, using the Agent API (`agc/api.php`). You can build a web application that handles login, call controls, disposition, transfers, and lead display — calling the Agent API for every action. The agent never sees `vicidial.php`. However, this is a significant development project. You need to replicate all the state management that VICIdial's agent screen handles natively: call state polling, timer management, recording controls, transfer workflows, callback scheduling, and more. Most operations get better ROI from customizing the existing agent screen via web forms and JavaScript injection. See our [agent screen customization guide](/blog/vicidial-agent-screen-customization/) for that approach.

### How do I handle API authentication securely?

VICIdial's API passes credentials as URL parameters, which is inherently less secure than header-based authentication. Mitigate this by: (1) always using HTTPS, (2) creating dedicated API users with minimal permissions, (3) IP-whitelisting API access via `.htaccess` or firewall rules, (4) monitoring the `vicidial_api_log` table for unauthorized access, and (5) rotating API passwords regularly. Never use admin accounts for API access. Never hardcode credentials in client-side JavaScript — use a server-side proxy that adds credentials to API calls.

### What's the maximum rate I can call the API?

VICIdial has no built-in rate limiting, which means you can hammer it until the web server or MySQL falls over. In practice, the Non-Agent API handles 5-10 requests per second comfortably on a standard VICIdial server. The Agent API handles similar throughput. Beyond that, you risk degrading dialer performance — especially during peak calling hours when the server is already under load from the dialer processes, real-time reporting, and agent sessions. For bulk operations (importing thousands of leads), throttle to 5-10 per second and run during off-peak hours. For real-time operations (webhook processing, stats polling), you're unlikely to hit limits unless something is misconfigured.

### How do I trigger an action in my CRM when a call is dispositioned?

Use the Dispo Call URL setting on your campaign. Set it to a URL on your CRM or middleware server, with `--A--field--B--` variables for lead data, agent info, and the [disposition](/glossary/disposition/) code. VICIdial fires a GET request to that URL every time an agent dispositions a call. Your receiving endpoint processes the data and updates the CRM. See the "Disposition Webhooks and Callbacks" section above for complete code examples in Python and PHP.

### Can I inject leads and have them dialed within seconds?

Yes. Set the lead's `rank` to 99 (maximum priority) when injecting via `add_lead`. VICIdial's hopper process loads high-rank leads first. The hopper runs on a cycle (configurable, typically every 15-30 seconds), so your lead will be queued for dialing within one hopper cycle after injection. For true sub-10-second speed-to-lead, you can also use the Agent API's `external_dial` function to immediately dial the number through a specific agent's session — bypassing the hopper entirely. This requires an agent to be in READY state.

### How do I sync VICIdial data with Salesforce or HubSpot?

Use a combination of `add_lead` (to push leads from CRM to VICIdial), the Dispo Call URL (to push dispositions from VICIdial to CRM), and `update_lead` (to sync field changes bidirectionally). For Salesforce, use Salesforce's REST API on the receiving end of your disposition webhook. For HubSpot, use HubSpot's contact API. Most operations use a lightweight middleware layer (a Python or Node.js service) that sits between VICIdial and the CRM, handling data transformation, deduplication, and error recovery. Our [CRM integration guide](/blog/vicidial-crm-integration/) covers this architecture in detail.

### Is there a way to get webhook-style push notifications from VICIdial without polling?

VICIdial doesn't have a native event streaming or webhook system. The URL trigger settings (Start Call URL, Dispo Call URL, No Agent Call URL) cover the most important events — call connect, disposition, and dropped calls. For other events (agent login/logout, campaign state changes, hopper alerts), you need to poll either the API or the MySQL database directly. A common pattern is running a daemon that queries `vicidial_live_agents` and `vicidial_auto_calls` every 10-15 seconds and fires events to your message queue (Redis, RabbitMQ, or Kafka) when state changes are detected. From there, downstream systems consume events in real-time without polling VICIdial directly.

### What happens if my webhook receiver is down when VICIdial fires a Dispo Call URL?

VICIdial makes a single HTTP request to the Dispo Call URL. If your server doesn't respond (timeout, connection refused, 500 error), the request is lost. VICIdial does not retry, does not queue, and does not log the failure in an easily accessible way. The disposition itself still happens — the call is dispositioned in VICIdial regardless of whether the URL call succeeds. To protect against this, put a highly available reverse proxy or message queue in front of your webhook receiver. Alternatively, run a reconciliation job that compares VICIdial's `vicidial_log` table against your CRM's records and catches any missed disposition syncs.

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-api-integration).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
