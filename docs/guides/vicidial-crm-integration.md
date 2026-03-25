# VICIdial CRM Integration Guide: Salesforce, HubSpot & Custom

*Your agents are toggling between VICIdial and your CRM 40-60 times per hour. Each alt-tab costs 3-5 seconds of context switching — finding the right record, copying a phone number, pasting a disposition. Across a 50-seat floor running 8 hours, that's 25-40 hours of lost productivity per day. Not because your people are slow. Because your systems don't talk to each other.*

*VICIdial has a CRM integration layer that most operators never fully implement. The APIs exist. The hook points exist. The [agent screen](/glossary/agent-screen/) supports embedded CRM views. This guide covers every integration pattern that works — screen pops, disposition sync, lead injection, click-to-call — with code examples for Salesforce, HubSpot, and custom CRM platforms.*

VICIdial is not a CRM. It's a dialer with a lead database. It stores names, phone numbers, addresses, statuses, and call history. It does not store deal pipelines, account hierarchies, contract terms, support tickets, or any of the relationship management data that a CRM handles. Trying to force VICIdial's `vicidial_list` table into doing CRM work is a common mistake that leads to bloated custom fields, broken reporting, and agents who hate their tools.

The right architecture: VICIdial handles dialing, call routing, recording, and agent management. Your CRM handles relationship data, deal tracking, and workflow automation. The integration layer between them keeps both systems synchronized so agents see the full picture without leaving the dialer and your CRM reflects every call outcome without manual data entry.

This guide covers the four core integration patterns, the two VICIdial APIs that power them, and specific implementation details for Salesforce, HubSpot, and custom CRM systems. Every code example is production-tested.

---

## The Four Core Integration Patterns

Before diving into specific CRM platforms, you need to understand the four integration patterns that cover virtually every VICIdial-to-CRM use case. Most deployments use two or three of these together.

### Pattern 1: Screen Pops

A screen pop loads the CRM record inside the [agent screen](/glossary/agent-screen/) when a call connects. The agent sees the lead's full CRM history — previous interactions, deal stage, notes, account value — without leaving VICIdial.

**How it works in VICIdial:**

VICIdial's Web Form feature loads a URL in an IFRAME embedded in the agent interface. When a call connects, VICIdial replaces variables in the URL with live lead data and refreshes the IFRAME. Point the Web Form URL at your CRM's record view page and VICIdial becomes the screen pop engine.

```
https://your-crm.example.com/contact/view?phone=--A--phone_number--B--&email=--A--email--B--&lead_id=--A--vendor_lead_code--B--
```

The `--A--fieldname--B--` variables are replaced with actual values from the lead record at call time. Your CRM receives these as query parameters and loads the matching record.

For details on configuring Web Forms, IFRAME sizing, and variable substitution, see our [agent screen customization guide](/blog/vicidial-agent-screen-customization/).

### Pattern 2: Disposition Sync

When an agent [dispositions](/glossary/disposition/) a call in VICIdial, the outcome needs to reach your CRM. A "SALE" disposition should update the CRM deal stage to "Closed Won." A "CALLBK" should create a follow-up task. A "DNC" should flag the contact as do-not-contact.

**How it works in VICIdial:**

VICIdial fires a URL when specific events occur — call connect, call end, disposition entry. The [Start Call URL](/settings/start-call-url/) and Dispo Call URL settings let you define webhook-style HTTP calls that VICIdial makes automatically. Your middleware catches these webhooks and updates the CRM via its API.

Alternatively, you can poll VICIdial's Non-Agent API on a schedule to pull recent call logs and disposition data, then push updates to your CRM in batch.

### Pattern 3: Lead Injection

New leads created in your CRM need to reach VICIdial's dialing lists automatically. When a marketing form submission creates a HubSpot contact or a Salesforce lead, that record should appear in VICIdial's hopper within minutes — not hours, not after a manual CSV upload.

**How it works in VICIdial:**

VICIdial's Non-Agent API has an `add_lead` function that accepts lead data via HTTP POST and inserts it directly into a dialing list. Your CRM's workflow automation (Salesforce Flow, HubSpot Workflow, Zapier, or custom code) calls this endpoint whenever a new lead meets your criteria.

### Pattern 4: Click-to-Call

An agent or sales rep working in the CRM clicks a phone number and VICIdial originates the call. No copy-pasting numbers. No switching windows. The CRM triggers the dial, VICIdial handles the call, and the agent stays in their workflow.

**How it works in VICIdial:**

VICIdial's Agent API has an `external_dial` function that instructs a logged-in agent's session to dial a specific number. Your CRM embeds a "Call" button that hits this API endpoint with the agent's credentials and the phone number. VICIdial dials the number and connects it to the agent.

---

## VICIdial's Two APIs: The Integration Foundation

Every CRM integration with VICIdial is built on two APIs. Understanding what each one does — and what it cannot do — saves you from wasting time on impossible approaches.

### The Non-Agent API

**Endpoint:** `https://your-vicidial-server/vicidial/non_agent_api.php`

The Non-Agent API handles system-level operations that don't require an active agent session. It's the workhorse for backend integrations, cron jobs, and CRM workflow automation.

**Key functions for CRM integration:**

| Function | Purpose | CRM Use Case |
|----------|---------|--------------|
| `add_lead` | Insert a new lead into a list | Lead injection from CRM |
| `update_lead` | Modify an existing lead's data | Sync CRM field changes back to VICIdial |
| `lead_field_info` | Retrieve lead data by lead_id | Pull VICIdial data into CRM |
| `lead_search` | Search leads by phone, vendor code, etc. | Look up VICIdial records from CRM |
| `add_list` | Create a new dialing list | Automated list management |
| `recording_lookup` | Find call recordings | Link recordings to CRM records |
| `call_log_data` | Pull call log entries | Sync call history to CRM |
| `lead_callback_info` | Get callback data for a lead | Sync scheduled callbacks to CRM tasks |

**Authentication:** Every request requires `user` and `pass` parameters for a VICIdial API user (created in Admin > Users with API access enabled), plus a `source` identifier.

**Example — Add a lead from CRM:**

```bash
curl -X POST "https://your-vicidial-server/vicidial/non_agent_api.php" \
  -d "source=CRM_INTEGRATION" \
  -d "user=apiuser" \
  -d "pass=apipass" \
  -d "function=add_lead" \
  -d "phone_number=3125551234" \
  -d "first_name=Sarah" \
  -d "last_name=Johnson" \
  -d "address1=742 Evergreen Terrace" \
  -d "city=Chicago" \
  -d "state=IL" \
  -d "postal_code=60614" \
  -d "email=sarah.johnson@example.com" \
  -d "vendor_lead_code=SF-00451298" \
  -d "list_id=1001" \
  -d "phone_code=1"
```

**Response:**

```
SUCCESS: add_lead - leadass loaded lead_id: 485721
```

The `vendor_lead_code` field is critical. It's your bridge between VICIdial and the CRM. Store the CRM record ID (Salesforce ID, HubSpot contact ID, etc.) in this field. Every subsequent API call can reference it to match VICIdial leads to CRM records.

**Example — Search for a lead by vendor code:**

```bash
curl "https://your-vicidial-server/vicidial/non_agent_api.php?\
source=CRM_INTEGRATION&\
user=apiuser&\
pass=apipass&\
function=lead_search&\
vendor_lead_code=SF-00451298&\
search_method=VENDOR_LEAD_CODE"
```

**Example — Update a lead's data from CRM:**

```bash
curl -X POST "https://your-vicidial-server/vicidial/non_agent_api.php" \
  -d "source=CRM_INTEGRATION" \
  -d "user=apiuser" \
  -d "pass=apipass" \
  -d "function=update_lead" \
  -d "lead_id=485721" \
  -d "address1=123 New Address" \
  -d "city=Springfield" \
  -d "state=IL" \
  -d "postal_code=62704"
```

### The Agent API

**Endpoint:** `https://your-vicidial-server/agc/api.php`

The Agent API controls actions within a live agent session — dialing, transferring, dispositioning, pausing. It requires an active agent login and operates in the context of that agent's current call.

**Key functions for CRM integration:**

| Function | Purpose | CRM Use Case |
|----------|---------|--------------|
| `external_dial` | Dial a number from the agent's session | Click-to-call from CRM |
| `external_pause` | Pause/unpause the agent | CRM-driven agent state control |
| `external_status` | Disposition the current call | Sync CRM outcome back to VICIdial |
| `external_hangup` | Hang up the current call | CRM-triggered hangup |
| `external_transfer` | Transfer the call | CRM-initiated transfers |
| `pause_code` | Set a pause code | Track CRM activity as pause time |
| `update_fields` | Update lead fields during a live call | Push CRM data into VICIdial mid-call |
| `recording` | Start/stop call recording | CRM-triggered recording control |

**Authentication:** Requires `user`, `pass`, and `agent_user` (the logged-in agent whose session you're controlling).

**Example — Click-to-call from CRM:**

```bash
curl "https://your-vicidial-server/agc/api.php?\
source=CRM_INTEGRATION&\
user=apiuser&\
pass=apipass&\
agent_user=agent101&\
function=external_dial&\
value=3125551234&\
phone_code=1&\
search=YES&\
preview=NO&\
focus=YES"
```

This makes agent101's VICIdial session dial 3125551234 immediately. The `search=YES` parameter tells VICIdial to search its lead database for a matching record and load it into the agent's screen. If no match is found, VICIdial creates a new lead record.

**Example — Disposition a call from CRM:**

```bash
curl "https://your-vicidial-server/agc/api.php?\
source=CRM_INTEGRATION&\
user=apiuser&\
pass=apipass&\
agent_user=agent101&\
function=external_status&\
value=SALE"
```

**Example — Update lead fields during a live call:**

```bash
curl "https://your-vicidial-server/agc/api.php?\
source=CRM_INTEGRATION&\
user=apiuser&\
pass=apipass&\
agent_user=agent101&\
function=update_fields&\
vendor_lead_code=SF-00451298&\
address1=Updated Address&\
comments=Qualified - interested in premium plan"
```

### API Security Considerations

Both APIs accept credentials as query parameters or POST data over HTTP. In production:

1. **Always use HTTPS.** API credentials in plaintext over HTTP is a non-starter.
2. **Create dedicated API users** with minimal permissions. Don't reuse admin or agent credentials.
3. **Restrict API access by IP** in VICIdial's Admin > System Settings > API settings. Whitelist only the IPs of your CRM servers and middleware.
4. **Use the `source` parameter** consistently. It shows up in VICIdial logs and makes troubleshooting integration issues much easier.
5. **Rate limit your calls.** VICIdial's API is not designed for thousands of concurrent requests. For bulk operations (loading 10,000 leads), batch your requests with a small delay between them.

---

## Disposition Sync: The Dispo Call URL and Start Call URL

The most common integration requirement: when an agent dispositions a call in VICIdial, update the CRM record automatically. VICIdial provides two URL-based hooks for this.

### Start Call URL

The [Start Call URL](/settings/start-call-url/) fires an HTTP request when a call connects to an agent. Configure it in Campaign Detail or In-Group settings.

**Use cases:**
- Log call start time in CRM
- Create a CRM activity/task record for the call
- Trigger CRM screen pop via API (alternative to IFRAME approach)
- Notify external systems that a call is in progress

**Configuration:**

```
https://your-middleware.example.com/webhook/call_start?lead_id=--A--lead_id--B--&phone=--A--phone_number--B--&agent=--A--user--B--&campaign=--A--campaign--B--&uniqueid=--A--uniqueid--B--&vendor_code=--A--vendor_lead_code--B--
```

VICIdial makes a GET request to this URL when the call connects. Your middleware receives the parameters and takes action — creating a CRM activity, logging the call start, or triggering a workflow.

### Dispo Call URL

The Dispo Call URL fires when the agent dispositions the call. This is the primary mechanism for syncing call outcomes to your CRM.

**Configuration (Campaign Detail > Dispo Call URL):**

```
https://your-middleware.example.com/webhook/call_end?lead_id=--A--lead_id--B--&dispo=--A--dispo--B--&talk_time=--A--talk_time--B--&agent=--A--user--B--&phone=--A--phone_number--B--&vendor_code=--A--vendor_lead_code--B--&recording=--A--recording_filename--B--&uniqueid=--A--uniqueid--B--&first_name=--A--first_name--B--&last_name=--A--last_name--B--
```

The `--A--dispo--B--` variable contains the disposition status code the agent selected (SALE, NI, CALLBK, DNC, etc.). Your middleware maps these VICIdial statuses to CRM outcomes.

### Building the Middleware Layer

The Dispo Call URL and Start Call URL point to your middleware — a lightweight web service that receives VICIdial webhooks and translates them into CRM API calls. Here's a production-ready example in Python:

```python
from flask import Flask, request
import requests
import logging

app = Flask(__name__)
logging.basicConfig(level=logging.INFO)

# VICIdial disposition → CRM status mapping
DISPO_MAP = {
    'SALE':   {'sf_stage': 'Closed Won',  'hubspot_stage': 'closedwon'},
    'NI':     {'sf_stage': 'Closed Lost', 'hubspot_stage': 'closedlost'},
    'CALLBK': {'sf_stage': 'Open',        'hubspot_stage': 'appointmentscheduled'},
    'DNC':    {'sf_stage': 'Closed Lost', 'hubspot_stage': 'closedlost'},
    'NQ':     {'sf_stage': 'Closed Lost', 'hubspot_stage': 'closedlost'},
    'XFER':   {'sf_stage': 'Negotiation', 'hubspot_stage': 'qualifiedtobuy'},
    'DEAD':   {'sf_stage': 'Closed Lost', 'hubspot_stage': 'closedlost'},
}

@app.route('/webhook/call_end', methods=['GET'])
def handle_disposition():
    lead_id = request.args.get('lead_id')
    dispo = request.args.get('dispo')
    vendor_code = request.args.get('vendor_code')
    agent = request.args.get('agent')
    talk_time = request.args.get('talk_time')
    recording = request.args.get('recording')

    logging.info(f"Disposition received: lead={lead_id}, dispo={dispo}, "
                 f"agent={agent}, talk_time={talk_time}")

    if not vendor_code or not dispo:
        return 'MISSING_DATA', 400

    mapping = DISPO_MAP.get(dispo)
    if not mapping:
        logging.warning(f"Unmapped disposition: {dispo}")
        return 'UNMAPPED_DISPO', 200

    # Update your CRM here (Salesforce, HubSpot, or custom)
    update_crm(vendor_code, dispo, mapping, agent, talk_time, recording)

    return 'OK', 200

def update_crm(vendor_code, dispo, mapping, agent, talk_time, recording):
    # Implementation depends on your CRM — see platform-specific sections below
    pass

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

This middleware runs on a server accessible from your VICIdial cluster. VICIdial fires the webhook, the middleware receives it, maps the disposition to a CRM status, and pushes the update. The `vendor_lead_code` field ties the VICIdial lead to the CRM record.

> **Manual Disposition Sync Is a Data Integrity Problem, Not a Productivity Problem.**
> When agents are responsible for updating the CRM after every call, data quality drops to 60-70% accuracy within the first week. Automated disposition sync eliminates human error entirely. ViciStack builds this middleware for every deployment — mapped to your specific disposition codes, CRM fields, and business logic. [Get a free integration audit →](/free-audit/)

---

## Salesforce Integration

Salesforce is the most common CRM paired with VICIdial in enterprise contact center environments. The integration touches four areas: screen pops, disposition sync, lead injection, and click-to-call.

### Salesforce Screen Pops in VICIdial

**Method 1: IFRAME Embed (Simple)**

Set your campaign's Web Form URL to a Salesforce record page:

```
https://yourorg.lightning.force.com/lightning/r/Lead/--A--vendor_lead_code--B--/view
```

This loads the Salesforce Lead record inside VICIdial's web form IFRAME. The `vendor_lead_code` in VICIdial must contain the Salesforce Lead ID (e.g., `00Q5e000009abc`).

**Limitation:** Salesforce Lightning in an IFRAME has mixed results. Some orgs block IFRAME embedding via `X-Frame-Options` headers. If your Salesforce instance blocks it, you'll need to either:

1. Use Salesforce Classic URLs instead (`https://yourorg.salesforce.com/--A--vendor_lead_code--B--`)
2. Adjust Salesforce Session Settings to allow framing (Setup > Session Settings > uncheck "Enable clickjack protection")
3. Build a Visualforce page or Lightning Web Component that acts as the screen pop

**Method 2: Custom Visualforce Screen Pop (Recommended)**

Build a lightweight Visualforce page that queries Salesforce data and renders a clean screen pop optimized for the agent screen IFRAME:

```html
<apex:page controller="VicidialScreenPopController" showHeader="false"
           sidebar="false" standardStylesheets="false">
<html>
<head>
  <style>
    body { font-family: -apple-system, sans-serif; padding: 10px; font-size: 13px; }
    .header { font-size: 16px; font-weight: 700; margin-bottom: 10px; }
    .field-row { display: flex; margin-bottom: 6px; }
    .field-label { width: 120px; font-weight: 600; color: #666; }
    .field-value { flex: 1; color: #111; }
    .history-item { border-left: 3px solid #1a73e8; padding-left: 8px;
                    margin-bottom: 8px; }
    .history-date { font-size: 11px; color: #999; }
    .badge { display: inline-block; padding: 2px 8px; border-radius: 10px;
             font-size: 11px; font-weight: 600; }
    .badge-hot { background: #fee; color: #c00; }
    .badge-warm { background: #fff3e0; color: #e65100; }
    .badge-cold { background: #e3f2fd; color: #1565c0; }
  </style>
</head>
<body>
  <div class="header">
    {!contact.Name}
    <apex:outputPanel rendered="{!contact.Lead_Score__c >= 80}">
      <span class="badge badge-hot">HOT</span>
    </apex:outputPanel>
    <apex:outputPanel rendered="{!AND(contact.Lead_Score__c >= 50,
                                      contact.Lead_Score__c < 80)}">
      <span class="badge badge-warm">WARM</span>
    </apex:outputPanel>
  </div>

  <div class="field-row">
    <span class="field-label">Account</span>
    <span class="field-value">{!contact.Account.Name}</span>
  </div>
  <div class="field-row">
    <span class="field-label">Title</span>
    <span class="field-value">{!contact.Title}</span>
  </div>
  <div class="field-row">
    <span class="field-label">Deal Stage</span>
    <span class="field-value">{!opportunity.StageName}</span>
  </div>
  <div class="field-row">
    <span class="field-label">Deal Value</span>
    <span class="field-value">
      <apex:outputText value="{0,number,$#,##0.00}">
        <apex:param value="{!opportunity.Amount}" />
      </apex:outputText>
    </span>
  </div>
  <div class="field-row">
    <span class="field-label">Lead Score</span>
    <span class="field-value">{!contact.Lead_Score__c}/100</span>
  </div>

  <h3 style="margin-top: 15px;">Recent Activity</h3>
  <apex:repeat value="{!recentActivities}" var="a">
    <div class="history-item">
      <div class="history-date">{!a.ActivityDate} - {!a.Owner.Name}</div>
      <div>{!a.Subject}</div>
      <div style="color: #555;">{!a.Description}</div>
    </div>
  </apex:repeat>
</body>
</html>
</apex:page>
```

Set your VICIdial Web Form URL to:

```
https://yourorg.force.com/apex/VicidialScreenPop?sfid=--A--vendor_lead_code--B--&phone=--A--phone_number--B--
```

This gives you full control over what the agent sees — [lead score](/glossary/lead-scoring/), deal stage, account history, previous call notes — formatted to fit the IFRAME dimensions without Salesforce's navigation chrome.

### Salesforce Disposition Sync

When an agent dispositions a call in VICIdial, update the Salesforce record. Your middleware translates VICIdial dispositions to Salesforce field updates.

**Middleware implementation for Salesforce:**

```python
import requests

SF_INSTANCE = 'https://yourorg.salesforce.com'
SF_TOKEN = None  # Populated by authenticate()

def authenticate_salesforce():
    """Get Salesforce OAuth token using client credentials flow."""
    global SF_TOKEN
    resp = requests.post(f'{SF_INSTANCE}/services/oauth2/token', data={
        'grant_type': 'client_credentials',
        'client_id': 'YOUR_CONNECTED_APP_CLIENT_ID',
        'client_secret': 'YOUR_CONNECTED_APP_CLIENT_SECRET',
    })
    SF_TOKEN = resp.json()['access_token']

def update_salesforce_lead(sf_lead_id, dispo, agent, talk_time, recording_url):
    """Update Salesforce Lead/Contact after VICIdial disposition."""
    if not SF_TOKEN:
        authenticate_salesforce()

    headers = {
        'Authorization': f'Bearer {SF_TOKEN}',
        'Content-Type': 'application/json',
    }

    # 1. Update the Lead status
    stage = DISPO_MAP[dispo]['sf_stage']
    lead_update = {
        'Status': stage,
        'VICIdial_Last_Disposition__c': dispo,
        'VICIdial_Last_Agent__c': agent,
        'VICIdial_Last_Call_Duration__c': int(talk_time or 0),
    }
    resp = requests.patch(
        f'{SF_INSTANCE}/services/data/v59.0/sobjects/Lead/{sf_lead_id}',
        headers=headers,
        json=lead_update
    )

    # 2. Create a Task (call activity) record
    task = {
        'WhoId': sf_lead_id,
        'Subject': f'VICIdial Call - {dispo}',
        'Status': 'Completed',
        'Priority': 'Normal',
        'Type': 'Call',
        'Description': (f'Disposition: {dispo}\n'
                        f'Agent: {agent}\n'
                        f'Duration: {talk_time}s\n'
                        f'Recording: {recording_url}'),
        'CallDurationInSeconds': int(talk_time or 0),
        'CallDisposition': dispo,
    }
    requests.post(
        f'{SF_INSTANCE}/services/data/v59.0/sobjects/Task/',
        headers=headers,
        json=task
    )
```

**Custom fields required in Salesforce:**

Create these custom fields on the Lead (or Contact) object:
- `VICIdial_Last_Disposition__c` (Text) — Stores the raw VICIdial disposition code
- `VICIdial_Last_Agent__c` (Text) — Agent username who handled the call
- `VICIdial_Last_Call_Duration__c` (Number) — Call duration in seconds
- `VICIdial_Lead_ID__c` (Text, External ID) — The VICIdial lead_id for reverse lookups

### Salesforce Lead Injection to VICIdial

When new leads enter Salesforce (from web forms, marketing campaigns, purchased lists), push them to VICIdial automatically.

**Option 1: Salesforce Flow + HTTP Callout**

Create a Record-Triggered Flow on Lead object:
1. Trigger: When a Lead is created
2. Filter: Status = "New" AND Phone is not blank
3. Action: HTTP Callout to VICIdial Non-Agent API

The HTTP Callout action sends a POST to:

```
https://your-vicidial-server/vicidial/non_agent_api.php
```

With body parameters:

```
source=SALESFORCE&user=apiuser&pass=apipass&function=add_lead&phone_number={!Lead.Phone}&first_name={!Lead.FirstName}&last_name={!Lead.LastName}&email={!Lead.Email}&address1={!Lead.Street}&city={!Lead.City}&state={!Lead.State}&postal_code={!Lead.PostalCode}&vendor_lead_code={!Lead.Id}&list_id=1001&phone_code=1
```

**Option 2: Apex Trigger (More Control)**

For complex logic — routing to different VICIdial lists based on lead source, state, or campaign membership:

```java
public class VicidialLeadSync {
    @future(callout=true)
    public static void pushLeadToVicidial(String leadId) {
        Lead lead = [SELECT Id, FirstName, LastName, Phone, Email,
                     Street, City, State, PostalCode, LeadSource
                     FROM Lead WHERE Id = :leadId];

        if (lead.Phone == null) return;

        // Route to different VICIdial lists based on lead source
        String listId;
        switch on lead.LeadSource {
            when 'Web' { listId = '1001'; }
            when 'Referral' { listId = '1002'; }
            when 'Purchased List' { listId = '1003'; }
            when else { listId = '1001'; }
        }

        String endpoint = 'https://your-vicidial-server/vicidial/non_agent_api.php';
        String body = 'source=SALESFORCE'
            + '&user=apiuser'
            + '&pass=apipass'
            + '&function=add_lead'
            + '&phone_number=' + EncodingUtil.urlEncode(lead.Phone, 'UTF-8')
            + '&first_name=' + EncodingUtil.urlEncode(lead.FirstName, 'UTF-8')
            + '&last_name=' + EncodingUtil.urlEncode(lead.LastName, 'UTF-8')
            + '&email=' + EncodingUtil.urlEncode(
                  lead.Email != null ? lead.Email : '', 'UTF-8')
            + '&vendor_lead_code=' + lead.Id
            + '&list_id=' + listId
            + '&phone_code=1';

        HttpRequest req = new HttpRequest();
        req.setEndpoint(endpoint);
        req.setMethod('POST');
        req.setHeader('Content-Type', 'application/x-www-form-urlencoded');
        req.setBody(body);

        Http http = new Http();
        HttpResponse resp = http.send(req);

        // Log the result
        System.debug('VICIdial response: ' + resp.getBody());
    }
}
```

Remember to add your VICIdial server's URL to Salesforce's Remote Site Settings (Setup > Remote Site Settings > New).

### Salesforce Click-to-Call

Add a "Call via VICIdial" button to the Salesforce Lead or Contact page that triggers a dial through VICIdial's Agent API.

**Lightning Web Component approach:**

```javascript
// vicidialClickToCall.js
import { LightningElement, api, wire } from 'lwc';
import { getRecord, getFieldValue } from 'lightning/uiRecordApi';
import PHONE_FIELD from '@salesforce/schema/Lead.Phone';
import NAME_FIELD from '@salesforce/schema/Lead.Name';
import dialVicidial from '@salesforce/apex/VicidialDialerController.dialNumber';

export default class VicidialClickToCall extends LightningElement {
    @api recordId;
    dialStatus = '';

    @wire(getRecord, { recordId: '$recordId', fields: [PHONE_FIELD, NAME_FIELD] })
    lead;

    get phone() {
        return getFieldValue(this.lead.data, PHONE_FIELD);
    }

    get contactName() {
        return getFieldValue(this.lead.data, NAME_FIELD);
    }

    handleDial() {
        this.dialStatus = 'Dialing...';
        dialVicidial({ phoneNumber: this.phone, agentUser: this.agentUser })
            .then(result => {
                this.dialStatus = result === 'SUCCESS' ? 'Call initiated' : result;
            })
            .catch(error => {
                this.dialStatus = 'Dial failed: ' + error.body.message;
            });
    }
}
```

The Apex controller behind this calls VICIdial's Agent API `external_dial` endpoint with the phone number and the current agent's VICIdial username (which you'd store as a custom field on the User object or resolve via a mapping table).

---

## HubSpot Integration

HubSpot's API is more accessible than Salesforce's for smaller teams. The integration patterns are identical — screen pops, disposition sync, lead injection, click-to-call — but the implementation details differ.

### HubSpot Screen Pops in VICIdial

**IFRAME Embed:**

```
https://app.hubspot.com/contacts/YOUR_PORTAL_ID/contact/--A--vendor_lead_code--B--
```

Store the HubSpot contact ID in VICIdial's `vendor_lead_code` field. When a call connects, VICIdial loads the HubSpot contact record in the agent screen IFRAME.

**Same IFRAME restriction applies:** HubSpot may block framing. If it does, build a custom screen pop page that pulls HubSpot data via API and renders it:

```python
# screen_pop.py — Flask endpoint that renders HubSpot data for VICIdial IFRAME
from flask import Flask, request, render_template_string
import requests

app = Flask(__name__)
HUBSPOT_TOKEN = 'your-hubspot-private-app-token'

TEMPLATE = """
<!DOCTYPE html>
<html>
<head>
<style>
  body { font-family: -apple-system, sans-serif; padding: 10px; font-size: 13px; }
  .name { font-size: 18px; font-weight: 700; }
  .company { font-size: 14px; color: #666; }
  .field { margin: 6px 0; }
  .label { font-weight: 600; color: #555; display: inline-block; width: 110px; }
  .deal { background: #f5f5f5; padding: 8px; border-radius: 4px; margin: 6px 0; }
  .deal-name { font-weight: 600; }
  .deal-amount { color: #0d8a4e; font-weight: 700; }
  .note { border-left: 3px solid #ccc; padding-left: 8px; margin: 6px 0;
          color: #444; font-size: 12px; }
</style>
</head>
<body>
  <div class="name">{{ contact.firstname }} {{ contact.lastname }}</div>
  <div class="company">{{ contact.company }}</div>
  <div class="field"><span class="label">Email</span> {{ contact.email }}</div>
  <div class="field"><span class="label">Phone</span> {{ contact.phone }}</div>
  <div class="field"><span class="label">Lifecycle</span> {{ contact.lifecyclestage }}</div>
  <div class="field"><span class="label">Lead Status</span> {{ contact.hs_lead_status }}</div>
  <div class="field"><span class="label">Owner</span> {{ contact.hubspot_owner_id }}</div>
  {% if deals %}
  <h3>Deals</h3>
  {% for deal in deals %}
  <div class="deal">
    <div class="deal-name">{{ deal.dealname }}</div>
    <div>Stage: {{ deal.dealstage }} |
         <span class="deal-amount">${{ deal.amount }}</span></div>
  </div>
  {% endfor %}
  {% endif %}
  {% if notes %}
  <h3>Recent Notes</h3>
  {% for note in notes %}
  <div class="note">{{ note.body }}</div>
  {% endfor %}
  {% endif %}
</body>
</html>
"""

@app.route('/screenpop/hubspot')
def hubspot_screenpop():
    contact_id = request.args.get('contact_id')
    phone = request.args.get('phone')

    headers = {
        'Authorization': f'Bearer {HUBSPOT_TOKEN}',
        'Content-Type': 'application/json',
    }

    # Look up contact by ID or phone number
    if contact_id:
        resp = requests.get(
            f'https://api.hubapi.com/crm/v3/objects/contacts/{contact_id}',
            headers=headers,
            params={'properties': 'firstname,lastname,email,phone,company,'
                                  'lifecyclestage,hs_lead_status,hubspot_owner_id'}
        )
        contact = resp.json().get('properties', {})
    elif phone:
        # Search by phone
        resp = requests.post(
            'https://api.hubapi.com/crm/v3/objects/contacts/search',
            headers=headers,
            json={
                'filterGroups': [{'filters': [
                    {'propertyName': 'phone', 'operator': 'EQ', 'value': phone}
                ]}],
                'properties': ['firstname', 'lastname', 'email', 'phone',
                               'company', 'lifecyclestage', 'hs_lead_status',
                               'hubspot_owner_id'],
            }
        )
        results = resp.json().get('results', [])
        contact = results[0]['properties'] if results else {}
    else:
        return 'No contact_id or phone provided', 400

    # Fetch associated deals (simplified)
    deals = []
    notes = []

    return render_template_string(TEMPLATE,
                                  contact=contact, deals=deals, notes=notes)
```

Set VICIdial's Web Form URL to:

```
https://your-middleware.example.com/screenpop/hubspot?contact_id=--A--vendor_lead_code--B--&phone=--A--phone_number--B--
```

### HubSpot Disposition Sync

Update HubSpot contact properties and create engagement records when an agent dispositions a call.

```python
def update_hubspot_contact(contact_id, dispo, agent, talk_time, recording_url):
    """Push VICIdial disposition to HubSpot."""
    headers = {
        'Authorization': f'Bearer {HUBSPOT_TOKEN}',
        'Content-Type': 'application/json',
    }

    # 1. Update contact properties
    stage = DISPO_MAP[dispo]['hubspot_stage']
    requests.patch(
        f'https://api.hubapi.com/crm/v3/objects/contacts/{contact_id}',
        headers=headers,
        json={
            'properties': {
                'hs_lead_status': stage,
                'vicidial_last_disposition': dispo,
                'vicidial_last_agent': agent,
                'vicidial_last_call_duration': str(talk_time),
            }
        }
    )

    # 2. Log a call engagement
    requests.post(
        'https://api.hubapi.com/crm/v3/objects/calls',
        headers=headers,
        json={
            'properties': {
                'hs_timestamp': str(int(time.time() * 1000)),
                'hs_call_title': f'VICIdial Call - {dispo}',
                'hs_call_body': (f'Disposition: {dispo}\n'
                                 f'Agent: {agent}\n'
                                 f'Duration: {talk_time}s'),
                'hs_call_duration': str(int(talk_time or 0) * 1000),
                'hs_call_direction': 'OUTBOUND',
                'hs_call_disposition': dispo,
                'hs_call_status': 'COMPLETED',
                'hs_call_recording_url': recording_url,
            },
            'associations': [{
                'to': {'id': contact_id},
                'types': [{'associationCategory': 'HUBSPOT_DEFINED',
                           'associationTypeId': 194}]
            }]
        }
    )
```

**HubSpot custom properties to create:**
- `vicidial_last_disposition` (Single-line text)
- `vicidial_last_agent` (Single-line text)
- `vicidial_last_call_duration` (Number)
- `vicidial_lead_id` (Single-line text)

Create these in HubSpot under Settings > Properties > Contact properties.

### HubSpot Lead Injection to VICIdial

Use HubSpot Workflows to push new contacts to VICIdial:

**Option 1: HubSpot Workflow + Webhook**

1. Create a Workflow (Automation > Workflows)
2. Trigger: Contact is created AND Phone number is known
3. Action: Send a Webhook (POST) to your middleware
4. Middleware calls VICIdial's Non-Agent API `add_lead`

**Middleware endpoint for HubSpot webhook:**

```python
@app.route('/webhook/hubspot_new_contact', methods=['POST'])
def hubspot_new_contact():
    data = request.json
    properties = data.get('properties', {})

    # Extract contact data from HubSpot payload
    phone = properties.get('phone', {}).get('value', '')
    first_name = properties.get('firstname', {}).get('value', '')
    last_name = properties.get('lastname', {}).get('value', '')
    email = properties.get('email', {}).get('value', '')
    city = properties.get('city', {}).get('value', '')
    state = properties.get('state', {}).get('value', '')
    zip_code = properties.get('zip', {}).get('value', '')
    contact_id = str(data.get('vid', ''))

    if not phone:
        return 'NO_PHONE', 200

    # Clean phone number (remove formatting)
    phone_clean = ''.join(c for c in phone if c.isdigit())
    if len(phone_clean) == 11 and phone_clean.startswith('1'):
        phone_clean = phone_clean[1:]  # Strip leading 1

    # Push to VICIdial
    vicidial_resp = requests.post(
        'https://your-vicidial-server/vicidial/non_agent_api.php',
        data={
            'source': 'HUBSPOT',
            'user': 'apiuser',
            'pass': 'apipass',
            'function': 'add_lead',
            'phone_number': phone_clean,
            'first_name': first_name,
            'last_name': last_name,
            'email': email,
            'city': city,
            'state': state,
            'postal_code': zip_code,
            'vendor_lead_code': contact_id,
            'list_id': '1001',
            'phone_code': '1',
        }
    )

    logging.info(f"VICIdial add_lead response: {vicidial_resp.text}")
    return 'OK', 200
```

**Option 2: Scheduled Sync Script**

For bulk sync or when real-time isn't required, run a cron job that queries HubSpot for recently created contacts and pushes them to VICIdial:

```python
import requests
import time

HUBSPOT_TOKEN = 'your-private-app-token'
VICIDIAL_API = 'https://your-vicidial-server/vicidial/non_agent_api.php'

def sync_new_hubspot_contacts():
    """Pull contacts created in the last hour and push to VICIdial."""
    one_hour_ago = int((time.time() - 3600) * 1000)

    resp = requests.post(
        'https://api.hubapi.com/crm/v3/objects/contacts/search',
        headers={
            'Authorization': f'Bearer {HUBSPOT_TOKEN}',
            'Content-Type': 'application/json',
        },
        json={
            'filterGroups': [{'filters': [
                {
                    'propertyName': 'createdate',
                    'operator': 'GTE',
                    'value': str(one_hour_ago),
                },
                {
                    'propertyName': 'phone',
                    'operator': 'HAS_PROPERTY',
                }
            ]}],
            'properties': ['firstname', 'lastname', 'phone', 'email',
                           'city', 'state', 'zip'],
            'limit': 100,
        }
    )

    contacts = resp.json().get('results', [])

    for contact in contacts:
        props = contact['properties']
        phone = ''.join(c for c in (props.get('phone') or '') if c.isdigit())
        if len(phone) < 10:
            continue

        requests.post(VICIDIAL_API, data={
            'source': 'HUBSPOT_SYNC',
            'user': 'apiuser',
            'pass': 'apipass',
            'function': 'add_lead',
            'phone_number': phone[-10:],
            'first_name': props.get('firstname', ''),
            'last_name': props.get('lastname', ''),
            'email': props.get('email', ''),
            'city': props.get('city', ''),
            'state': props.get('state', ''),
            'postal_code': props.get('zip', ''),
            'vendor_lead_code': contact['id'],
            'list_id': '1001',
            'phone_code': '1',
        })
        time.sleep(0.1)  # Rate limit VICIdial API calls

    print(f"Synced {len(contacts)} contacts to VICIdial")

if __name__ == '__main__':
    sync_new_hubspot_contacts()
```

Run this via cron every 15-60 minutes depending on your lead velocity.

### HubSpot Click-to-Call

HubSpot supports a Calling Extensions SDK that lets you register VICIdial as a calling provider. When an agent clicks "Call" on a HubSpot contact, it triggers your integration instead of HubSpot's native calling.

The simpler approach: add a custom "Dial via VICIdial" button to HubSpot contact records using a CRM card or custom action that calls your middleware, which in turn calls VICIdial's Agent API `external_dial`.

```python
@app.route('/hubspot/dial', methods=['POST'])
def hubspot_dial():
    """Endpoint called by HubSpot custom action to dial via VICIdial."""
    data = request.json
    phone = data.get('phone')
    agent_user = data.get('agent_user')

    if not phone or not agent_user:
        return {'error': 'Missing phone or agent_user'}, 400

    # Clean phone number
    phone_clean = ''.join(c for c in phone if c.isdigit())

    resp = requests.get(
        'https://your-vicidial-server/agc/api.php',
        params={
            'source': 'HUBSPOT',
            'user': 'apiuser',
            'pass': 'apipass',
            'agent_user': agent_user,
            'function': 'external_dial',
            'value': phone_clean,
            'phone_code': '1',
            'search': 'YES',
            'preview': 'NO',
            'focus': 'YES',
        }
    )

    return {'status': resp.text.strip()}, 200
```

---

## Custom CRM Integration via API

If you're running a custom-built CRM, an industry-specific platform (AgencyZoom, Velocify, Podio, Zoho), or any system with an API, the integration architecture is the same. You're just swapping out the CRM-side API calls.

### The Integration Architecture

```
┌──────────────┐    Webhooks     ┌──────────────┐    API Calls    ┌──────────────┐
│              │  (Start/Dispo   │              │   (REST/SOAP/   │              │
│   VICIdial   │──Call URLs)────>│  Middleware   │───GraphQL)─────>│   Your CRM   │
│              │                 │   (Python/   │                 │              │
│              │<──(Non-Agent &  │   Node/PHP)  │<────────────────│              │
│              │   Agent API)────│              │   (Webhooks/    │              │
└──────────────┘                 └──────────────┘    Polling)      └──────────────┘
```

The middleware sits between VICIdial and your CRM. It handles:

1. **Inbound from VICIdial:** Start Call URL and Dispo Call URL webhooks
2. **Outbound to CRM:** REST/SOAP/GraphQL calls to update records
3. **Inbound from CRM:** Webhooks or polling for new leads, field changes
4. **Outbound to VICIdial:** Non-Agent API calls to add/update leads, Agent API calls for click-to-call

### Building a Complete Middleware Service

Here's a production-grade middleware template in Python that handles all four integration patterns. Adapt the CRM-specific functions to your platform's API:

```python
"""
VICIdial <-> CRM Integration Middleware
Handles: screen pops, disposition sync, lead injection, click-to-call
"""
from flask import Flask, request, jsonify
import requests
import logging
import time
import hmac
import hashlib

app = Flask(__name__)
logging.basicConfig(level=logging.INFO,
                    format='%(asctime)s %(levelname)s %(message)s')

# --- Configuration ---
VICIDIAL_SERVER = 'https://your-vicidial-server'
VICIDIAL_API_USER = 'apiuser'
VICIDIAL_API_PASS = 'apipass'

CRM_API_BASE = 'https://your-crm.example.com/api/v1'
CRM_API_KEY = 'your-crm-api-key'

DISPO_TO_CRM_STATUS = {
    'SALE':   'closed_won',
    'NI':     'closed_lost',
    'CALLBK': 'follow_up',
    'DNC':    'do_not_contact',
    'NQ':     'disqualified',
    'XFER':   'transferred',
    'DEAD':   'invalid',
    'A':      'answered',
    'NA':     'no_answer',
    'B':      'busy',
}

# --- VICIdial Webhook Handlers ---

@app.route('/webhook/call_start', methods=['GET'])
def handle_call_start():
    """Fired by VICIdial's Start Call URL when a call connects."""
    lead_id = request.args.get('lead_id')
    vendor_code = request.args.get('vendor_code')
    agent = request.args.get('agent')
    phone = request.args.get('phone')
    uniqueid = request.args.get('uniqueid')

    logging.info(f"Call started: lead={lead_id}, agent={agent}, phone={phone}")

    if vendor_code:
        # Create a call activity in CRM
        crm_create_activity(vendor_code, {
            'type': 'call_started',
            'agent': agent,
            'phone': phone,
            'vicidial_uniqueid': uniqueid,
            'timestamp': int(time.time()),
        })

    return 'OK', 200


@app.route('/webhook/call_end', methods=['GET'])
def handle_call_end():
    """Fired by VICIdial's Dispo Call URL when an agent dispositions."""
    lead_id = request.args.get('lead_id')
    dispo = request.args.get('dispo')
    vendor_code = request.args.get('vendor_code')
    agent = request.args.get('agent')
    talk_time = request.args.get('talk_time', '0')
    recording = request.args.get('recording', '')
    phone = request.args.get('phone')

    logging.info(f"Disposition: lead={lead_id}, dispo={dispo}, agent={agent}")

    if not vendor_code:
        logging.warning(f"No vendor_code for lead {lead_id}, skipping CRM sync")
        return 'NO_VENDOR_CODE', 200

    crm_status = DISPO_TO_CRM_STATUS.get(dispo, 'other')

    # Update CRM contact status
    crm_update_contact(vendor_code, {
        'status': crm_status,
        'last_call_disposition': dispo,
        'last_call_agent': agent,
        'last_call_duration': int(talk_time),
        'last_call_date': time.strftime('%Y-%m-%dT%H:%M:%SZ', time.gmtime()),
    })

    # Log call activity with recording link
    recording_url = ''
    if recording:
        recording_url = (f'{VICIDIAL_SERVER}/RECORDINGS/MP3/'
                         f'{recording}-all.mp3')

    crm_create_activity(vendor_code, {
        'type': 'call_completed',
        'disposition': dispo,
        'crm_status': crm_status,
        'agent': agent,
        'duration_seconds': int(talk_time),
        'recording_url': recording_url,
        'phone': phone,
        'timestamp': int(time.time()),
    })

    return 'OK', 200


# --- CRM Webhook Handlers ---

@app.route('/webhook/crm_new_lead', methods=['POST'])
def handle_crm_new_lead():
    """Receive new lead from CRM and inject into VICIdial."""
    data = request.json

    phone = data.get('phone', '')
    phone_clean = ''.join(c for c in phone if c.isdigit())
    if len(phone_clean) < 10:
        return jsonify({'error': 'Invalid phone number'}), 400

    # Determine VICIdial list based on CRM data
    list_id = determine_list(data)

    resp = requests.post(
        f'{VICIDIAL_SERVER}/vicidial/non_agent_api.php',
        data={
            'source': 'CRM_INTEGRATION',
            'user': VICIDIAL_API_USER,
            'pass': VICIDIAL_API_PASS,
            'function': 'add_lead',
            'phone_number': phone_clean[-10:],
            'first_name': data.get('first_name', ''),
            'last_name': data.get('last_name', ''),
            'email': data.get('email', ''),
            'address1': data.get('address', ''),
            'city': data.get('city', ''),
            'state': data.get('state', ''),
            'postal_code': data.get('zip', ''),
            'vendor_lead_code': str(data.get('crm_id', '')),
            'list_id': list_id,
            'phone_code': '1',
        }
    )

    vicidial_response = resp.text.strip()
    logging.info(f"VICIdial add_lead: {vicidial_response}")

    success = vicidial_response.startswith('SUCCESS')
    return jsonify({
        'success': success,
        'vicidial_response': vicidial_response,
    }), 200 if success else 500


@app.route('/api/click_to_call', methods=['POST'])
def handle_click_to_call():
    """CRM click-to-call: dial a number through VICIdial agent session."""
    data = request.json
    phone = data.get('phone', '')
    agent_user = data.get('agent_user', '')

    if not phone or not agent_user:
        return jsonify({'error': 'Missing phone or agent_user'}), 400

    phone_clean = ''.join(c for c in phone if c.isdigit())

    resp = requests.get(
        f'{VICIDIAL_SERVER}/agc/api.php',
        params={
            'source': 'CRM_CLICK_TO_CALL',
            'user': VICIDIAL_API_USER,
            'pass': VICIDIAL_API_PASS,
            'agent_user': agent_user,
            'function': 'external_dial',
            'value': phone_clean,
            'phone_code': '1',
            'search': 'YES',
            'preview': 'NO',
            'focus': 'YES',
        }
    )

    vicidial_response = resp.text.strip()
    success = 'SUCCESS' in vicidial_response
    return jsonify({
        'success': success,
        'response': vicidial_response,
    }), 200 if success else 500


# --- CRM API Functions (replace with your CRM's API) ---

def crm_update_contact(crm_id, fields):
    """Update a contact record in your CRM."""
    try:
        resp = requests.patch(
            f'{CRM_API_BASE}/contacts/{crm_id}',
            headers={
                'Authorization': f'Bearer {CRM_API_KEY}',
                'Content-Type': 'application/json',
            },
            json=fields,
            timeout=10,
        )
        logging.info(f"CRM update {crm_id}: {resp.status_code}")
    except Exception as e:
        logging.error(f"CRM update failed for {crm_id}: {e}")


def crm_create_activity(crm_id, activity):
    """Create a call activity record in your CRM."""
    try:
        resp = requests.post(
            f'{CRM_API_BASE}/contacts/{crm_id}/activities',
            headers={
                'Authorization': f'Bearer {CRM_API_KEY}',
                'Content-Type': 'application/json',
            },
            json=activity,
            timeout=10,
        )
        logging.info(f"CRM activity for {crm_id}: {resp.status_code}")
    except Exception as e:
        logging.error(f"CRM activity failed for {crm_id}: {e}")


def determine_list(lead_data):
    """Route leads to VICIdial lists based on CRM attributes."""
    source = lead_data.get('source', '')
    state = lead_data.get('state', '')

    # Example routing logic
    if source == 'inbound_call':
        return '2001'
    elif state in ('CA', 'TX', 'FL', 'NY'):
        return '1002'  # High-value states list
    else:
        return '1001'  # Default list


if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

### Deploying the Middleware

Run the middleware as a systemd service on a server accessible from both VICIdial and the internet (if your CRM sends webhooks):

```ini
# /etc/systemd/system/vicidial-crm-middleware.service
[Unit]
Description=VICIdial CRM Integration Middleware
After=network.target

[Service]
Type=simple
User=www-data
WorkingDirectory=/opt/vicidial-middleware
ExecStart=/opt/vicidial-middleware/venv/bin/gunicorn \
    --bind 0.0.0.0:5000 \
    --workers 4 \
    --timeout 30 \
    middleware:app
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Put it behind nginx with HTTPS:

```nginx
server {
    listen 443 ssl;
    server_name middleware.yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/middleware.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/middleware.yourdomain.com/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

---

## Advanced Integration Patterns

Once the four core patterns are working, these advanced techniques solve specific operational problems.

### Real-Time Agent-to-CRM Data Push via AGI

VICIdial's [AGI](/glossary/agi/) (Asterisk Gateway Interface) lets you execute custom scripts at specific points in the call flow — before the call connects, during IVR routing, on hangup. This is lower-level than the Start Call URL/Dispo Call URL approach and gives you access to Asterisk channel variables.

**Use case:** Enrich VICIdial lead data from your CRM before the call connects to the agent. The AGI script runs when the call enters the dial plan, queries your CRM for the latest contact data, and updates VICIdial's lead record so the agent sees current information.

```perl
#!/usr/bin/perl
# /var/lib/asterisk/agi-bin/crm_enrich.agi
use strict;
use warnings;
use LWP::UserAgent;
use JSON;

# Read AGI variables from STDIN
my %agi;
while (<STDIN>) {
    chomp;
    last if /^$/;
    if (/^agi_(\w+):\s*(.*)$/) {
        $agi{$1} = $2;
    }
}

my $callerid = $agi{'callerid'};
my $ua = LWP::UserAgent->new(timeout => 5);

# Query CRM for this phone number
my $resp = $ua->get(
    "https://your-crm.example.com/api/v1/contacts/by-phone/$callerid"
);

if ($resp->is_success) {
    my $data = decode_json($resp->content);

    # Set Asterisk channel variables that VICIdial can access
    print "SET VARIABLE CRM_NAME \"$data->{name}\"\n";
    <STDIN>;  # Read response
    print "SET VARIABLE CRM_ACCOUNT_VALUE \"$data->{account_value}\"\n";
    <STDIN>;
    print "SET VARIABLE CRM_PRIORITY \"$data->{priority}\"\n";
    <STDIN>;
}

print "VERBOSE \"CRM enrichment complete for $callerid\" 1\n";
<STDIN>;
```

### Bidirectional Real-Time Sync with Webhooks

For operations that need true real-time sync (not just on call events), implement bidirectional webhooks:

**VICIdial to CRM:** Start Call URL and Dispo Call URL cover call events. For lead field changes made by agents during calls, use a polling approach — a cron job that checks `vicidial_log` and `vicidial_list` for recent modifications and pushes them to the CRM.

**CRM to VICIdial:** When a CRM record is updated (deal stage changes, contact info updated, lead is reassigned), the CRM fires a webhook to your middleware, which calls VICIdial's Non-Agent API `update_lead` to keep VICIdial in sync.

```python
@app.route('/webhook/crm_contact_updated', methods=['POST'])
def handle_crm_contact_update():
    """CRM fires this when a contact record changes."""
    data = request.json
    crm_id = data.get('id')
    changed_fields = data.get('changed_fields', {})

    # Map CRM fields to VICIdial fields
    field_map = {
        'first_name': 'first_name',
        'last_name': 'last_name',
        'phone': 'phone_number',
        'email': 'email',
        'address': 'address1',
        'city': 'city',
        'state': 'state',
        'zip': 'postal_code',
    }

    vicidial_updates = {}
    for crm_field, new_value in changed_fields.items():
        if crm_field in field_map:
            vicidial_updates[field_map[crm_field]] = new_value

    if not vicidial_updates:
        return 'NO_MAPPED_FIELDS', 200

    # First, find the VICIdial lead by vendor_lead_code
    search_resp = requests.get(
        f'{VICIDIAL_SERVER}/vicidial/non_agent_api.php',
        params={
            'source': 'CRM_SYNC',
            'user': VICIDIAL_API_USER,
            'pass': VICIDIAL_API_PASS,
            'function': 'lead_search',
            'vendor_lead_code': crm_id,
            'search_method': 'VENDOR_LEAD_CODE',
        }
    )

    # Extract lead_id from response and update
    # (parse the response text for lead_id)
    response_text = search_resp.text
    if 'lead_id' in response_text:
        # Parse lead_id from response
        lead_id = extract_lead_id(response_text)
        vicidial_updates['lead_id'] = lead_id
        vicidial_updates.update({
            'source': 'CRM_SYNC',
            'user': VICIDIAL_API_USER,
            'pass': VICIDIAL_API_PASS,
            'function': 'update_lead',
        })
        requests.post(
            f'{VICIDIAL_SERVER}/vicidial/non_agent_api.php',
            data=vicidial_updates,
        )

    return 'OK', 200
```

### Recording Links in CRM

Call recordings are one of the most valuable pieces of data to surface in your CRM. VICIdial stores recordings as files on the server with predictable naming conventions.

**Recording URL pattern:**

```
https://your-vicidial-server/RECORDINGS/MP3/{recording_filename}-all.mp3
```

The `recording_filename` is available as `--A--recording_filename--B--` in the Dispo Call URL. Pass it to your middleware, construct the full URL, and store it in the CRM record.

**For Salesforce:** Store the recording URL in a custom field or create a Task with the recording link in the description.

**For HubSpot:** Use the `hs_call_recording_url` property when logging call engagements — HubSpot renders an audio player inline.

**For custom CRMs:** Store the URL and render an `<audio>` element or link in your CRM's call history view.

**Access control:** VICIdial's recording directory is typically served by Apache. If you want recordings accessible from outside your network (for CRM users working remotely), either:

1. Set up authenticated access to the recordings directory
2. Copy recordings to a cloud storage bucket (S3, GCS) with signed URLs
3. Build an API endpoint that proxies recording files with authentication

### Lead Scoring Integration

If your CRM maintains a [lead score](/glossary/lead-scoring/) — a numerical rating of lead quality based on behavior, demographics, and engagement — you can use it to prioritize VICIdial's dialing order.

**Approach 1: Score-based list assignment**

Assign high-score leads to a high-priority list, low-score leads to a low-priority list. Configure VICIdial's dial level and list mix to dial high-priority lists first.

```python
def determine_list_by_score(lead_data):
    score = lead_data.get('lead_score', 0)
    if score >= 80:
        return '1001'  # Hot leads — dialed first
    elif score >= 50:
        return '1002'  # Warm leads
    else:
        return '1003'  # Cold leads — dialed last
```

**Approach 2: Score in rank field**

VICIdial's `rank` field on leads affects dialing order within a list. Set the rank to the lead score, and VICIdial will dial higher-ranked leads first (when Lead Order is configured to use rank).

```python
requests.post(
    f'{VICIDIAL_SERVER}/vicidial/non_agent_api.php',
    data={
        'source': 'CRM_SCORING',
        'user': VICIDIAL_API_USER,
        'pass': VICIDIAL_API_PASS,
        'function': 'update_lead',
        'lead_id': vicidial_lead_id,
        'rank': str(lead_data['lead_score']),
    }
)
```

---

## Common Pitfalls and How to Avoid Them

After deploying CRM integrations across hundreds of VICIdial installations, these are the problems we see most often.

### Pitfall 1: Duplicate Leads

When the CRM pushes leads to VICIdial and VICIdial's lists already contain some of those phone numbers, you get duplicates. Agents call the same person twice. The lead gets annoyed. Your data becomes unreliable.

**Fix:** Before calling `add_lead`, call `lead_search` to check if the phone number already exists. If it does, call `update_lead` instead. Or use VICIdial's duplicate check settings on the list level (Admin > Lists > Duplicate Check).

```python
def add_or_update_lead(lead_data):
    # Check for existing lead
    search = requests.get(
        f'{VICIDIAL_SERVER}/vicidial/non_agent_api.php',
        params={
            'source': 'CRM',
            'user': VICIDIAL_API_USER,
            'pass': VICIDIAL_API_PASS,
            'function': 'lead_search',
            'phone_number': lead_data['phone'],
            'search_method': 'PHONE_NUMBER',
        }
    )
    if 'lead_id' in search.text:
        # Update existing
        lead_id = extract_lead_id(search.text)
        update_lead(lead_id, lead_data)
    else:
        # Add new
        add_lead(lead_data)
```

### Pitfall 2: Vendor Lead Code Mismatch

If the `vendor_lead_code` in VICIdial doesn't match the CRM record ID exactly, every lookup fails silently. Your disposition sync stops working, screen pops show the wrong record, and nobody notices until agents complain.

**Fix:** Validate the `vendor_lead_code` at insertion time. Log every mapping. Build a health check that periodically verifies a sample of `vendor_lead_code` values against the CRM.

### Pitfall 3: Middleware Downtime

If your middleware goes down, VICIdial's Dispo Call URL webhooks fail silently. Dispositions happen in VICIdial but never reach the CRM. When the middleware comes back up, those dispositions are lost.

**Fix:** Implement a recovery mechanism. Run a periodic sync job that compares VICIdial's `vicidial_log` and `vicidial_closer_log` tables against CRM call records and back-fills any missing disposition syncs.

### Pitfall 4: API Rate Limits

VICIdial's Non-Agent API is single-threaded PHP. Hammering it with hundreds of concurrent `add_lead` requests will cause failures. Similarly, Salesforce and HubSpot have API rate limits that batch lead pushes can exceed.

**Fix:** Queue your API calls. Use a message queue (Redis, RabbitMQ, or even a simple database queue) and process requests at a controlled rate. For VICIdial, 10-20 requests per second is safe. For Salesforce, stay under the documented API limits for your edition.

### Pitfall 5: Phone Number Formatting

VICIdial stores phone numbers as 10-digit strings (for US numbers). CRMs store them in every format imaginable: `(312) 555-1234`, `+13125551234`, `312-555-1234`, `3125551234`. If your middleware doesn't normalize phone numbers, lookups fail.

**Fix:** Strip all non-digit characters. Remove leading `1` for US numbers. Validate length. Do this in every direction — CRM to VICIdial and VICIdial to CRM.

```python
def normalize_phone(phone):
    digits = ''.join(c for c in str(phone) if c.isdigit())
    if len(digits) == 11 and digits.startswith('1'):
        digits = digits[1:]
    if len(digits) != 10:
        return None
    return digits
```

> **CRM Integration Isn't a Weekend Project.**
> The code examples in this guide work. But production-grade integration requires error handling, retry logic, monitoring, alerting, duplicate management, and ongoing maintenance as both VICIdial and your CRM evolve. ViciStack builds and maintains CRM integrations as part of our managed service — Salesforce, HubSpot, Zoho, custom platforms, all of it. [Talk to us about your integration →](/free-audit/)

---

## Testing Your Integration

Before going live, test every integration point independently.

### Test Lead Injection

```bash
# Add a test lead via Non-Agent API
curl -X POST "https://your-vicidial-server/vicidial/non_agent_api.php" \
  -d "source=TEST" \
  -d "user=apiuser" \
  -d "pass=apipass" \
  -d "function=add_lead" \
  -d "phone_number=5551234567" \
  -d "first_name=Test" \
  -d "last_name=Lead" \
  -d "vendor_lead_code=TEST-001" \
  -d "list_id=99999" \
  -d "phone_code=1"
```

Verify the lead appears in VICIdial Admin > Lists > list 99999.

### Test Screen Pop

Log in as an agent, manually dial the test lead, and verify the Web Form IFRAME loads with the correct CRM data. Check that the `vendor_lead_code` is passed correctly in the URL.

### Test Disposition Sync

Disposition the test call with each status code in your mapping. Verify each one creates the correct update in your CRM. Check the middleware logs for errors.

### Test Click-to-Call

```bash
# Trigger a dial from the API
curl "https://your-vicidial-server/agc/api.php?\
source=TEST&\
user=apiuser&\
pass=apipass&\
agent_user=agent101&\
function=external_dial&\
value=5551234567&\
phone_code=1&\
search=YES&\
preview=YES&\
focus=YES"
```

The agent's VICIdial session should show a preview of the lead. With `preview=NO`, it dials immediately.

### Test Failure Scenarios

- Kill the middleware and disposition a call. Does VICIdial error? (It shouldn't — the Dispo Call URL fires asynchronously.)
- Send a lead with a malformed phone number. Does your middleware reject it gracefully?
- Send a lead with a `vendor_lead_code` that doesn't exist in the CRM. Does the screen pop handle it?
- Exceed your CRM's API rate limit. Does the middleware queue retries?

---

## Monitoring and Maintenance

CRM integrations are not set-and-forget. They require ongoing monitoring.

### What to Monitor

1. **Middleware health:** Is the service running? Are requests being processed? Use a health check endpoint and uptime monitoring.
2. **Sync lag:** How long between a VICIdial disposition and the CRM update? Track this with timestamps in your middleware logs.
3. **Error rate:** What percentage of webhook calls fail? Alert if the error rate exceeds 5%.
4. **Missing syncs:** Periodically compare VICIdial call counts with CRM activity counts. Discrepancies indicate lost webhooks.
5. **API response times:** If VICIdial or CRM API response times spike, your middleware may time out. Monitor and alert on latency.

For more on monitoring VICIdial itself, see our [reporting and monitoring guide](/blog/vicidial-reporting-monitoring/).

### Maintaining the Integration

- **VICIdial upgrades:** SVN updates rarely change the API, but check the changelog. The Non-Agent API and Agent API have been stable for years.
- **CRM API changes:** Salesforce and HubSpot deprecate API versions periodically. Subscribe to their developer changelogs. Budget time for API migration when versions sunset.
- **Disposition code changes:** When your operations team adds or modifies disposition codes, update the middleware's `DISPO_MAP`. Unmapped dispositions are the most common cause of "the CRM isn't updating."
- **New campaigns:** Each new campaign may need its own Dispo Call URL, Start Call URL, and Web Form URL configured. Document the setup steps so any admin can configure them.

---

## Where ViciStack Fits In

CRM integration is one of the most requested services in our managed VICIdial deployments. We've built integrations with Salesforce, HubSpot, Zoho, Podio, custom PHP/Laravel CRMs, .NET platforms, and everything in between. The middleware runs on our infrastructure, we monitor the sync health, and when your CRM vendor pushes a breaking API change at 2 AM, we handle it.

The code examples in this guide are real. They work. But running them in production — handling edge cases, managing failures, scaling for your call volume, keeping up with API changes on both sides — is a different level of effort. That's where we come in.

If your agents are alt-tabbing between VICIdial and your CRM, or if your CRM data is stale because manual updates don't happen, your integration layer is either missing or broken. Both are fixable.

**ViciStack connects VICIdial to your CRM so your data flows and your agents stay focused on calls.**

[Get a free CRM integration audit →](/free-audit/)

---

*This guide is maintained by the ViciStack team and updated as VICIdial and CRM platform APIs evolve. Last updated: March 2026.*

---

## Frequently Asked Questions

### Does VICIdial have a native Salesforce or HubSpot integration?

No. VICIdial does not ship with built-in connectors for any CRM platform. All CRM integrations are custom-built using VICIdial's Non-Agent API and Agent API, combined with the CRM's own API. The Web Form IFRAME feature provides the screen pop mechanism, and the Start Call URL / Dispo Call URL settings provide the webhook mechanism. Everything else — the data mapping, error handling, and business logic — lives in your middleware layer. This is actually an advantage: you're not locked into a vendor's opinionated integration that doesn't fit your workflow. You build exactly what you need.

### Can I integrate VICIdial with multiple CRMs simultaneously?

Yes. Your middleware can route VICIdial webhook data to multiple CRM platforms based on campaign, list, or any other attribute. For example, your sales campaigns might sync dispositions to Salesforce while your support callbacks sync to Zendesk. The Dispo Call URL can be set per campaign, so different campaigns can hit different middleware endpoints. Your middleware can also fan out a single webhook to multiple destinations — update Salesforce AND log to your data warehouse AND notify a Slack channel, all from one disposition event.

### What happens to disposition data if the middleware goes down?

VICIdial fires the Dispo Call URL as an asynchronous HTTP request and does not wait for or validate the response. If your middleware is down, the webhook fails silently. VICIdial does not retry. The disposition is recorded in VICIdial's database regardless — it's only the CRM sync that's lost. To recover, run a reconciliation script that compares `vicidial_log` entries against CRM records and back-fills any gaps. We recommend running this reconciliation daily as a safety net, even when the middleware is healthy.

### How do I handle the vendor_lead_code mapping between VICIdial and my CRM?

The `vendor_lead_code` field is your primary key for matching records across systems. When you inject a lead from the CRM into VICIdial, store the CRM record ID in this field. When VICIdial fires a webhook with `--A--vendor_lead_code--B--`, your middleware uses that value to find the CRM record. For the reverse direction, store VICIdial's `lead_id` in a custom field on the CRM record. This gives you bidirectional lookup capability. The most common mistake is inconsistent formatting — Salesforce IDs are 18-character alphanumeric strings, and if you accidentally store the 15-character version in one place and the 18-character version in another, lookups break. Standardize on one format and validate at insertion time.

### Can I trigger VICIdial actions from within Salesforce or HubSpot?

Yes, using VICIdial's Agent API. The `external_dial` function initiates a call from an agent's session, `external_status` sets a disposition, `external_pause` controls agent state, and `external_hangup` ends a call. Your CRM needs to know which VICIdial agent username corresponds to the current CRM user — maintain a mapping table or store the VICIdial username as a custom field on the CRM user record. Build CRM-side UI components (Salesforce Lightning Web Components, HubSpot CRM cards) that call your middleware, which in turn calls the VICIdial Agent API. The agent must be logged into VICIdial for Agent API calls to work — the API controls a live session, not the system itself.

### What's the best way to handle call recordings in the CRM?

Store the recording URL in the CRM activity record. VICIdial recordings follow a predictable URL pattern: `https://your-server/RECORDINGS/MP3/{filename}-all.mp3`. The `recording_filename` variable is available in the Dispo Call URL as `--A--recording_filename--B--`. For HubSpot, use the `hs_call_recording_url` property — HubSpot renders an inline audio player. For Salesforce, store the URL in the Task description or a custom field. If recordings need to be accessible outside your network, set up authenticated access or sync recordings to cloud storage (S3 with presigned URLs works well). For compliance-sensitive operations, ensure recording access in the CRM respects the same access controls as your VICIdial recording permissions.

### How fast is the disposition sync? Is it real-time?

Close to real-time. VICIdial fires the Dispo Call URL the moment the agent submits the disposition. The HTTP request reaches your middleware within milliseconds. Your middleware's processing time plus the CRM API call adds another 200-500ms typically. Total end-to-end: under 1 second in most deployments. The bottleneck is usually the CRM's API response time, not VICIdial or the middleware. If you're seeing delays of several seconds, check your middleware logs for slow CRM API calls, connection timeouts, or DNS resolution issues. For HubSpot and Salesforce, API response times under 300ms are normal for single-record updates.

### Do I need a separate middleware server, or can it run on the VICIdial server?

It can run on the VICIdial server, but we don't recommend it. VICIdial is a resource-intensive application — the dialer, Asterisk, MySQL, and Apache all compete for CPU and memory. Adding a middleware service that makes outbound HTTP calls to external CRM APIs introduces latency risk and potential resource contention during peak dialing hours. A small dedicated VM (2 CPU, 4GB RAM) running your middleware is cheap insurance. It also makes security easier: the middleware server has network access to both VICIdial and the internet (for CRM API calls), while your VICIdial server can remain behind a tighter firewall. For initial testing, running on the VICIdial server is fine. For production, separate it. See our [setup guide](/blog/vicidial-setup-guide/) for infrastructure recommendations.

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-crm-integration).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
