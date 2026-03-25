# VICIdial Multi-Tenant Setup for BPOs

**Last updated: March 2026 | Reading time: ~18 minutes**

You run a BPO. You have three clients. Client A needs 30 agents running insurance leads. Client B needs 15 agents doing appointment setting. Client C needs 50 agents on a political survey campaign. All three clients expect total isolation --- their leads are confidential, their recordings are privileged, their performance data is proprietary. And you need to run all of this on the same VICIdial infrastructure because running three separate VICIdial installations would triple your server costs and your management overhead.

This is the multi-tenant problem, and VICIdial solves it. Not perfectly --- the multi-tenancy is administrative rather than database-level --- but well enough that thousands of BPO operations worldwide run multiple clients on shared VICIdial instances. The key is understanding what VICIdial's tenancy model can and cannot do, and configuring it correctly so that Client A never sees Client B's data, Client C cannot hear Client A's recordings, and your billing system accurately tracks per-tenant usage.

This guide covers the complete multi-tenant VICIdial configuration: [user group](/glossary/user-group/) setup for access control, campaign isolation, recording restrictions, [DID](/glossary/did/) routing per tenant, [SIP trunk](/glossary/trunk/) allocation strategies, per-tenant billing via CDR analysis, the architectural limitations you need to understand, and the scaling decision of shared infrastructure versus dedicated servers per client.

**ViciStack builds multi-tenant VICIdial deployments for BPOs** with proper isolation, per-client reporting, and scalable infrastructure. [Talk to us about your BPO setup.](/pricing/)

---

## VICIdial's Multi-Tenancy Model: What You Are Working With

Before we get into configuration, understand the architecture. VICIdial was not designed from the ground up as a multi-tenant platform. It was designed as a single-organization dialer that gained multi-tenancy capabilities through its permission system. This distinction matters because it defines the boundaries of what is possible.

### What Multi-Tenancy Means in VICIdial

VICIdial's multi-tenancy is built on three pillars:

1. **[User Groups](/glossary/user-group/)** --- Every user (agent, manager, admin) belongs to a user group. User groups control which admin screens, reports, campaigns, and recordings each user can access.
2. **Campaign Assignment** --- Campaigns are associated with user groups via the `allowed_campaigns` mechanism. A user can only see and operate within campaigns their user group is allowed to access.
3. **Permission Granularity** --- Each user group has fine-grained permissions covering dozens of admin screens, report types, and operational capabilities.

### What Multi-Tenancy Does NOT Mean in VICIdial

- **There is no database-level isolation.** All tenants share the same MySQL `asterisk` database. Client A's leads sit in the same `vicidial_list` table as Client B's leads. They are differentiated by `list_id` and `campaign_id`, not by separate databases or schemas.
- **There is no [Asterisk](/glossary/asterisk/)-level isolation.** All tenants share the same Asterisk instance(s). SIP channels, conference bridges, and recording resources are shared.
- **There is no OS-level isolation.** No containers, no VMs, no process separation between tenants.

This means VICIdial's multi-tenancy is enforced at the **application layer** through the admin interface and reporting system. A user with direct MySQL access or SSH access to the server can see all tenants' data. The isolation is administrative, not technical.

For most BPO operations, this is perfectly adequate. Your clients do not have direct database access. They see VICIdial through the admin interface, the agent screen, and the reports you provide. As long as the user group permissions are configured correctly, each client sees only their own data.

For operations with contractual requirements for database-level isolation (common in healthcare, government, or high-security financial services), you may need separate VICIdial instances per client. We cover that decision in the scaling section below.

---

## User Group Configuration: The Foundation of Multi-Tenancy

User groups are the primary mechanism for tenant isolation. Every BPO client gets their own user group, and that group's permissions determine exactly what the client's users can see and do.

### Creating Client User Groups

Navigate to **Admin > User Groups** and create a group for each client. Here is the naming convention we recommend:

| User Group ID | Description | Purpose |
|--------------|-------------|---------|
| CLIENT_A_ADMIN | Client A - Manager | Client A managers who run reports and configure campaigns |
| CLIENT_A_AGENT | Client A - Agent | Client A agents who dial |
| CLIENT_B_ADMIN | Client B - Manager | Client B managers |
| CLIENT_B_AGENT | Client B - Agent | Client B agents |
| BPO_ADMIN | BPO Admin (Internal) | Your internal admin team with full access |

**Why separate ADMIN and AGENT groups per client?** Because managers and agents need different permission levels. Agents need to log in, take calls, and disposition. Managers need to view reports, monitor agents, and possibly adjust campaign settings. Having separate groups lets you give managers report access without giving agents the same visibility.

### User Group Permission Configuration

Navigate to **Admin > User Groups > [Group] > Modify**. Here is the complete permission configuration for a client manager group:

#### Admin Screen Permissions

These control which admin screens the user can access. For a client manager:

| Permission | Recommended Setting | Notes |
|-----------|-------------------|-------|
| Admin Access | 7 or 8 | 7 = manager, 8 = senior manager. Never 9 (superadmin) for clients |
| Campaigns | View only (or Modify for trusted clients) | Restricts what they can change |
| Lists | View and Modify | Clients need to upload their own leads |
| Users | View | Clients can see their agents but not modify them |
| Scripts | View | Let them see scripts but not change them |
| Carriers | NONE | Clients should never see trunk configuration |
| DIDs | View only | Clients can see their assigned DIDs |
| Servers | NONE | No infrastructure visibility |
| System Settings | NONE | Absolutely no system-level access |
| User Groups | NONE | Clients should not see or modify group permissions |

#### Allowed Campaigns

This is the most critical setting for tenant isolation. In the user group configuration, the `Allowed Campaigns` field lists which campaigns users in this group can access.

```
Allowed Campaigns: CLIENT_A_SALES CLIENT_A_SURVEY CLIENT_A_INBOUND
```

Users in this group will only see these campaigns in the admin interface, agent login, and reports. All other campaigns are invisible to them.

**Important:** The `Allowed Campaigns` list must be manually updated whenever you create a new campaign for a client. Forgetting to add a new campaign to the allowed list means the client cannot see it. Forgetting to remove a campaign when it is reassigned means the client can see data they should not have access to.

#### Report Permissions

Control which reports each user group can access:

| Report | Client Manager | Client Agent | BPO Admin |
|--------|---------------|-------------|-----------|
| Agent Performance | Yes (own campaigns only) | No | Yes (all) |
| Campaign Summary | Yes (own campaigns only) | No | Yes (all) |
| [Outbound Calling Report](/glossary/dial-status/) | Yes (own campaigns only) | No | Yes (all) |
| Inbound Report | Yes (own campaigns only) | No | Yes (all) |
| Recording Lookup | Yes (own campaigns only) | Own recordings | Yes (all) |
| Real-Time Report | Yes (own campaigns only) | No | Yes (all) |
| Agent Status Detail | Yes (own user group) | No | Yes (all) |
| DNC List | Yes (own campaigns) | No | Yes (all) |
| System Status | No | No | Yes |
| Server Stats | No | No | Yes |

The critical behavior: when a client manager runs a report, VICIdial automatically filters the results to show only data from campaigns in their `Allowed Campaigns` list. They do not see other clients' data even if they could theoretically construct a URL that references another campaign --- VICIdial enforces the filter server-side.

#### Recording Access Permissions

Control who can listen to [call recordings](/blog/vicidial-call-recording/):

| Setting | Purpose |
|---------|---------|
| `Recording Access` | ALL, OWN, or NONE --- controls whether the user can listen to recordings |
| `Allowed Recording Campaigns` | Which campaigns' recordings this group can access |

For client managers, set `Recording Access = ALL` and `Allowed Recording Campaigns` to only their campaigns. This lets them listen to any recording from their campaigns but prevents access to other clients' recordings.

For client agents, set `Recording Access = OWN` so they can only hear their own calls (useful for self-improvement) or `NONE` if you do not want agents accessing recordings at all.

> **Multi-Tenant Done Right, from Day One.**
> ViciStack configures user groups, campaign isolation, and recording restrictions as part of every BPO deployment. [Get Your Multi-Tenant Setup.](/pricing/)

---

## Campaign Isolation

User groups control who can see what. Campaign architecture controls how data is actually organized. Here is how to structure campaigns for clean multi-tenancy.

### Naming Convention

Use a consistent naming convention for campaigns that includes the client identifier:

```
CLIENT_A_SALES_OB     (Client A outbound sales)
CLIENT_A_INBOUND      (Client A inbound)
CLIENT_B_APPT_SET     (Client B appointment setting)
CLIENT_C_SURVEY_Q1    (Client C Q1 survey)
```

This makes administration easier, prevents confusion, and makes per-client billing queries straightforward.

### List Organization

Each client's leads should be in lists that are assigned only to that client's campaigns. Never share lists across clients.

Recommended list ID ranges per client to avoid collisions:

| Client | List ID Range | Example |
|--------|--------------|---------|
| Client A | 1000-1999 | 1001 = Client A Fresh Insurance Leads |
| Client B | 2000-2999 | 2001 = Client B Appointment Leads March |
| Client C | 3000-3999 | 3001 = Client C Voter File Q1 |
| Internal/Test | 9000-9999 | 9001 = Internal Test List |

This convention is not enforced by VICIdial --- it is an organizational practice. But it makes it visually obvious which lists belong to which client and simplifies auditing.

### Preventing Cross-Tenant Data Leakage in Reports

The primary risk in multi-tenant VICIdial is a misconfigured user group that lets one client see another client's data. Here are the common leakage vectors and how to prevent them:

**Vector 1: Campaigns in wrong user group.** Always double-check `Allowed Campaigns` when adding or removing campaigns. A quarterly audit of user group configurations is worth the 30 minutes it takes.

**Vector 2: Lists assigned to wrong campaigns.** If Client A's list is accidentally assigned to Client B's campaign, Client B's agents will dial Client A's leads. VICIdial does not prevent this at the list level --- it is a campaign configuration issue. Audit list-to-campaign assignments monthly.

**Vector 3: Shared lead table queries.** If a client has any direct database access (rare, but some BPOs provide it for custom reporting), they could query the `vicidial_list` table and see all leads, not just their own. Mitigate this with MySQL views:

```sql
-- Create a view that only shows Client A's leads
CREATE VIEW client_a_leads AS
SELECT * FROM vicidial_list
WHERE list_id BETWEEN 1000 AND 1999;

-- Grant the client's MySQL user access only to the view
GRANT SELECT ON asterisk.client_a_leads TO 'client_a_report'@'%';
```

**Vector 4: Real-time report showing all agents.** The VICIdial real-time report, by default, can show all agents across all campaigns. User group restrictions limit this to allowed campaigns only --- but verify this by logging in as a client user and confirming they only see their own agents on the real-time screen.

---

## DID Routing Per Tenant

Most BPO clients have their own inbound phone numbers ([DIDs](/glossary/did/)) that need to route to their specific campaigns and [inbound groups](/glossary/inbound-group/).

### Setting Up Client-Specific Inbound Groups

For each client that has inbound traffic:

1. Navigate to **Admin > Inbound Groups** and create a group:
   - Group ID: `CLIENT_A_INBOUND`
   - Group Name: "Client A - Main Inbound"
   - Campaign: `CLIENT_A_SALES_OB` (if blended) or standalone inbound campaign

2. Navigate to **Admin > DIDs** and create DID entries for the client's phone numbers:
   - DID Pattern: `18005551234`
   - DID Route: `IN_GROUP`
   - In-Group: `CLIENT_A_INBOUND`

3. In the inbound group configuration, set:
   - **Call Time**: The client's desired hours of operation
   - **[Queue Priority](/settings/queue-priority/)**: If clients share agents (not recommended but sometimes necessary)
   - **After Hours Route**: Voicemail, overflow group, or different IVR

### Per-Client IVR/Call Menus

Each client likely wants their own [IVR](/glossary/ivr/) greeting and menu structure. Create separate [call menus](/glossary/call-menu/) per client:

1. **Admin > Call Menus** > Create new menu
2. Upload the client's custom greeting audio
3. Configure menu options (press 1 for sales, press 2 for support, etc.)
4. Route each option to the appropriate client-specific inbound group

### DID Cost Tracking

Track which DIDs belong to which client for accurate billing:

```sql
-- DID inventory by assumed client (based on DID route)
SELECT vd.did_pattern, vd.did_description, vd.did_route,
       vig.group_id as inbound_group, vig.group_name
FROM vicidial_inbound_dids vd
LEFT JOIN vicidial_inbound_groups vig
    ON vd.group_id = vig.group_id
WHERE vd.did_active = 'Y'
ORDER BY vig.group_name, vd.did_pattern;
```

---

## SIP Trunk Allocation Strategies

How you allocate [SIP trunks](/glossary/trunk/) across tenants is one of the most impactful decisions in multi-tenant VICIdial. There are three strategies, each with trade-offs.

### Strategy 1: Shared Trunk Pool

All clients share the same SIP trunks. The carrier does not know (or care) which client a call belongs to.

**Advantages:**
- Simplest to configure --- one carrier, one set of trunks
- Highest trunk utilization --- idle capacity from one client is automatically available to another
- Lowest cost --- bulk trunk pricing, no per-client minimums

**Disadvantages:**
- No per-client trunk isolation --- a Client A spike can starve Client B
- [Caller ID](/glossary/cid/) management is more complex --- all clients' outbound CID must be registered on the same carrier
- If the carrier goes down, all clients are affected simultaneously
- Harder to attribute per-client carrier costs

**Configuration:** Use a single carrier entry in **Admin > Carriers**. Assign it to all outbound campaigns. Set per-campaign [dial prefixes](/settings/dial-prefix/) if clients need different routing.

### Strategy 2: Dedicated Trunks Per Client

Each client gets their own SIP carrier entry (possibly the same upstream carrier but different credentials/accounts).

**Advantages:**
- Complete trunk isolation --- Client A's traffic cannot affect Client B's capacity
- Separate [caller ID](/glossary/cid/) management per client
- Clean per-client carrier cost allocation
- If one client's trunk goes down, others are unaffected

**Disadvantages:**
- Higher cost --- each client must pay for their trunk capacity even during idle periods
- More complex configuration --- multiple carrier entries, per-campaign carrier assignments
- You may exceed your server's concurrent call capacity if all clients spike simultaneously

**Configuration:** Create separate carrier entries per client in **Admin > Carriers**. In each client's campaign settings, set the carrier to their dedicated carrier entry. VICIdial supports per-campaign carrier assignment in **Admin > Campaigns > [Campaign] > Detail > Account Code** and carrier configuration.

### Strategy 3: Hybrid (Dedicated Primary + Shared Overflow)

Each client has a dedicated primary trunk, with a shared overflow trunk pool for peak traffic.

**Advantages:**
- Guaranteed capacity for each client
- Overflow prevents busy signals during spikes
- Cost-effective balance between isolation and utilization

**Disadvantages:**
- Most complex to configure and monitor
- Overflow calls may use different caller IDs

**Configuration:** In **Admin > Carriers**, create per-client carriers plus a shared overflow carrier. In each campaign, configure the primary carrier first, with the overflow carrier as a fallback. VICIdial's carrier weight and priority system handles the failover automatically.

### Our Recommendation for BPOs

For most BPOs with 3-10 clients:

- **Small clients (<10 agents):** Shared trunk pool. The cost savings outweigh the isolation benefits.
- **Medium clients (10-50 agents):** Dedicated trunks. These clients generate enough volume to justify their own trunk allocation, and they typically have caller ID and compliance requirements that demand isolation.
- **Large clients (50+ agents):** Dedicated trunks, possibly on a separate [carrier](/glossary/carrier/) entirely. Consider whether this client should have their own VICIdial instance.

---

## Per-Tenant Billing and Usage Tracking

Most BPOs bill clients based on agent hours, call minutes, or a combination. VICIdial provides the raw data to calculate any billing model.

### Agent Hours Per Client

```sql
-- Agent hours by user group (proxy for client) for the current month
SELECT
    vu.user_group,
    COUNT(DISTINCT va.user) as agents,
    ROUND(SUM(va.pause_sec + va.wait_sec + va.talk_sec + va.dispo_sec + va.dead_sec) / 3600, 2) as total_hours
FROM vicidial_agent_log va
JOIN vicidial_users vu ON va.user = vu.user
WHERE va.event_time >= DATE_FORMAT(NOW(), '%Y-%m-01')
GROUP BY vu.user_group
ORDER BY total_hours DESC;
```

### Call Minutes Per Client

```sql
-- Outbound call minutes by campaign for billing period
SELECT
    campaign_id,
    COUNT(*) as total_calls,
    SUM(CASE WHEN status IN ('SALE','XFER','CALLBK','A','B') THEN 1 ELSE 0 END) as connected_calls,
    ROUND(SUM(length_in_sec) / 60, 2) as total_minutes,
    ROUND(SUM(CASE WHEN length_in_sec > 0 THEN length_in_sec ELSE 0 END) / 60, 2) as billed_minutes
FROM vicidial_log
WHERE call_date >= '2026-03-01'
  AND call_date < '2026-04-01'
GROUP BY campaign_id
ORDER BY campaign_id;
```

### Inbound Minutes Per Client

```sql
-- Inbound call minutes by inbound group
SELECT
    campaign_id as inbound_group,
    COUNT(*) as total_calls,
    ROUND(SUM(length_in_sec) / 60, 2) as total_minutes,
    ROUND(AVG(queue_seconds), 1) as avg_queue_time
FROM vicidial_closer_log
WHERE call_date >= '2026-03-01'
  AND call_date < '2026-04-01'
GROUP BY campaign_id
ORDER BY campaign_id;
```

### Recording Storage Per Client

```sql
-- Recording storage consumption by campaign (proxy for client)
SELECT
    vl.campaign_id,
    COUNT(rl.recording_id) as recording_count,
    ROUND(SUM(rl.length_in_sec) / 3600, 1) as total_hours,
    -- Estimate storage: ~1 MB/min for WAV, ~0.12 MB/min for MP3
    ROUND(SUM(rl.length_in_sec) / 60 * 1, 0) as est_storage_mb_wav,
    ROUND(SUM(rl.length_in_sec) / 60 * 0.12, 0) as est_storage_mb_mp3
FROM recording_log rl
JOIN vicidial_log vl ON rl.vicidial_id = vl.uniqueid
WHERE rl.start_time >= '2026-03-01'
GROUP BY vl.campaign_id
ORDER BY total_hours DESC;
```

### Automated Billing Reports

Create a monthly billing script that runs on the 1st of each month:

```bash
#!/bin/bash
# monthly_billing.sh - Generate per-client billing reports
MONTH=$(date -d "last month" +%Y-%m)
REPORT_DIR="/reports/billing/${MONTH}"
mkdir -p ${REPORT_DIR}

# Define clients and their campaigns
declare -A CLIENTS
CLIENTS[CLIENT_A]="CLIENT_A_SALES_OB,CLIENT_A_INBOUND"
CLIENTS[CLIENT_B]="CLIENT_B_APPT_SET"
CLIENTS[CLIENT_C]="CLIENT_C_SURVEY_Q1"

for CLIENT in "${!CLIENTS[@]}"; do
    CAMPAIGNS=${CLIENTS[$CLIENT]}
    CAMPAIGN_LIST=$(echo ${CAMPAIGNS} | sed "s/,/','/g")

    mysql -u cron -p'password' asterisk -e "
    SELECT
        '${CLIENT}' as client,
        campaign_id,
        COUNT(*) as total_calls,
        SUM(CASE WHEN length_in_sec > 0 THEN 1 ELSE 0 END) as connected,
        ROUND(SUM(length_in_sec)/60, 2) as minutes,
        ROUND(SUM(length_in_sec)/3600, 2) as hours
    FROM vicidial_log
    WHERE call_date >= '${MONTH}-01'
      AND call_date < DATE_ADD('${MONTH}-01', INTERVAL 1 MONTH)
      AND campaign_id IN ('${CAMPAIGN_LIST}')
    GROUP BY campaign_id;
    " > ${REPORT_DIR}/${CLIENT}_billing.tsv

    echo "Generated billing report for ${CLIENT}"
done
```

> **Per-Client Billing, Automated.**
> ViciStack BPO deployments include billing infrastructure with per-tenant usage tracking and automated monthly reports. [Learn More.](/pricing/)

---

## Resource Allocation and Capacity Planning

Running multiple clients on shared infrastructure requires careful capacity planning to ensure one client's traffic does not degrade another client's experience.

### Server Sizing for Multi-Tenant

Use the single-tenant sizing guidelines from the [setup guide](/blog/vicidial-setup-guide/) as a starting point, then add a 20-30% buffer for multi-tenant overhead:

| Total Agents (All Clients) | Server Configuration | Notes |
|---------------------------|---------------------|-------|
| Under 25 | Single server, 8 cores, 16 GB RAM | All roles on one box |
| 25-75 | Web + DB server, separate telephony | Standard [cluster split](/blog/vicidial-cluster-guide/) |
| 75-150 | Web, DB, 2-3 telephony servers | Multiple dialers for capacity |
| 150-300 | Full cluster with redundancy | Consider per-client server isolation |
| 300+ | Strongly consider separate instances | Shared infrastructure becomes a risk |

### Trunk Capacity Per Client

A critical capacity planning metric for multi-tenant VICIdial is concurrent call capacity. Each outbound agent with [predictive dialing](/glossary/predictive-dialing/) at a 3:1 [dial ratio](/settings/auto-dial-level/) generates 3 concurrent SIP channels. Add inbound calls and you need:

```
Total concurrent channels = (Outbound agents x Dial ratio) + Inbound calls
```

Example for three clients:
- Client A: 30 agents x 3 ratio = 90 channels + 10 inbound = 100
- Client B: 15 agents x 2 ratio = 30 channels + 5 inbound = 35
- Client C: 50 agents x 4 ratio = 200 channels + 0 inbound = 200
- **Total: 335 concurrent channels**

Each Asterisk server on modern hardware (8+ cores, 16 GB RAM) handles approximately 250-350 concurrent channels with [recording](/glossary/recording/) active. This three-client scenario needs at least 2 telephony servers.

### Preventing "Noisy Neighbor" Problems

When one client's traffic spike degrades other clients' performance:

**CPU contention:** One client running aggressive [AMD](/glossary/amd/) processing can consume CPU that other clients need for real-time audio. Monitor per-campaign CPU usage via Asterisk:

```bash
# Check active channels per campaign
asterisk -rx "core show channels" | grep -c "campaign_id"
```

**Hopper contention:** The [hopper](/glossary/hopper/) fill process runs per campaign, but it shares MySQL resources. A client with 500,000 active leads fills the hopper slower than a client with 10,000 leads, and during the fill process, other campaigns' hopper fills may be delayed. Mitigate this by staggering [hopper level](/settings/hopper-level/) settings --- high-volume campaigns get higher hopper levels to reduce fill frequency.

**Recording I/O:** [Recording](/blog/vicidial-call-recording/) 200 concurrent calls writes roughly 200 MB/minute to disk. If multiple clients spike simultaneously, disk I/O becomes the bottleneck. Use separate disks or partitions for recordings versus the database.

---

## Limitations of VICIdial Multi-Tenancy

Be honest with your clients about what VICIdial's multi-tenancy cannot do. Setting expectations correctly prevents contractual disputes.

### No Database-Level Isolation

All tenants share the same MySQL instance and the same tables. A user with `SELECT` access to `vicidial_list` can see all leads from all clients. Application-level permissions prevent this through the admin interface, but the data is not physically separated.

**Implication:** If a client requires SOC 2, HIPAA BAA, or contractual language guaranteeing data isolation at the database level, shared VICIdial cannot meet that requirement. You need separate instances.

### No Per-Tenant Resource Guarantees

You cannot guarantee Client A gets 50% of the server's CPU or 100 dedicated SIP channels. VICIdial does not have resource quotas or QoS (Quality of Service) at the tenant level. Resource allocation is first-come-first-served.

**Implication:** During peak hours, one client's traffic can impact another's [contact rate](/glossary/contact-rate/) and [agent utilization](/glossary/agent-utilization/). Overprovisioning (running at 60-70% capacity instead of 90%) is the practical mitigation.

### Shared System Settings

VICIdial's System Settings (Admin > System Settings) apply globally. This includes:

- Maximum [hopper](/glossary/hopper/) size
- Recording format (WAV/MP3)
- Session timeout values
- API access controls
- Time zone settings

You cannot have Client A on WAV recordings and Client B on MP3. You cannot have Client A with 30-minute session timeouts and Client B with 2-hour timeouts. These are system-wide settings.

### Admin User Visibility

VICIdial admin users at level 9 (the highest) can see all campaigns, all users, all data regardless of user group assignments. Your internal admin team necessarily has full cross-tenant visibility. This is operationally required but may concern clients who want absolute data isolation.

**Mitigation:** Limit level 9 access to a small number of trusted administrators. Use level 7-8 for day-to-day administration. Log admin access activities.

---

## When to Split: Separate Instances vs. Shared Infrastructure

The shared multi-tenant model works well up to a point. Beyond that point, the risks and limitations outweigh the cost savings.

### Signals That You Need Separate Instances

- **Client requires database-level isolation** (contractual, regulatory, or compliance-driven)
- **Client has 100+ agents** and their traffic can impact other tenants
- **Client needs custom VICIdial modifications** (custom PHP code, agent screen changes) that would affect other tenants
- **Client needs a different Asterisk version** or custom Asterisk modules
- **Client is in a heavily regulated industry** (healthcare, financial services, government) with strict data handling requirements
- **Client demands dedicated infrastructure** as a contractual term

### The Cost Trade-Off

| Model | Server Cost (50 agents) | Admin Overhead | Client Isolation | Flexibility |
|-------|------------------------|---------------|-----------------|-------------|
| Shared VICIdial (3 clients on 1 instance) | $200-500/month total | Low | Administrative only | Limited |
| Dedicated VICIdial per client | $200-500/month **per client** | High (3x management) | Database + OS level | Full |
| Hybrid (large clients dedicated, small shared) | Varies | Medium | Best of both | Good |

The [true cost of running VICIdial](/blog/vicidial-cost-2026/) includes not just server hardware but also administration time. Each separate VICIdial instance needs its own patching, monitoring, backup, and maintenance. Three instances means three times the admin work --- or a managed service provider like ViciStack handling it for you.

### The Hybrid Approach

The most common BPO architecture we see:

- **Large anchor clients (50+ agents):** Dedicated VICIdial instance on dedicated hardware
- **Medium clients (15-50 agents):** Shared VICIdial instance with proper user group isolation, grouped with 2-3 similar-sized clients
- **Small clients (<15 agents):** Shared multi-tenant instance with 5-10 other small clients

This balances cost, isolation, and administrative overhead. It also lets you move clients between tiers as they grow --- a small client that scales to 50 agents graduates to a dedicated instance.

---

## Example: Complete Multi-Tenant Configuration for a 3-Client BPO

Let us walk through a concrete example. You are a BPO with three clients:

- **Apex Insurance** --- 25 outbound agents, 5 inbound agents
- **GreenSolar** --- 15 outbound agents, appointment setting
- **VoteReach** --- 40 outbound agents, survey campaign (seasonal)

### User Group Setup

| Group ID | Users | Admin Level | Allowed Campaigns |
|----------|-------|------------|-------------------|
| BPO_SUPER | 2 super admins | 9 | All |
| BPO_MANAGER | 4 managers | 8 | All |
| APEX_MANAGER | 2 Apex managers | 7 | APEX_OB_SALES, APEX_INBOUND |
| APEX_AGENT | 30 Apex agents | 1 | APEX_OB_SALES, APEX_INBOUND |
| GREEN_MANAGER | 1 GreenSolar manager | 7 | GREEN_APPTSET |
| GREEN_AGENT | 15 GreenSolar agents | 1 | GREEN_APPTSET |
| VOTE_MANAGER | 2 VoteReach managers | 7 | VOTE_SURVEY_Q1 |
| VOTE_AGENT | 40 VoteReach agents | 1 | VOTE_SURVEY_Q1 |

### Campaign Configuration

| Campaign ID | Client | Type | Lists | Carrier |
|------------|--------|------|-------|---------|
| APEX_OB_SALES | Apex Insurance | Outbound | 1001-1050 | APEX_SIP (dedicated) |
| APEX_INBOUND | Apex Insurance | Inbound | N/A | APEX_SIP |
| GREEN_APPTSET | GreenSolar | Outbound | 2001-2020 | SHARED_POOL |
| VOTE_SURVEY_Q1 | VoteReach | Outbound | 3001-3100 | SHARED_POOL |

### Recording Permissions

| Group | Recording Access | Allowed Recording Campaigns |
|-------|-----------------|---------------------------|
| APEX_MANAGER | ALL | APEX_OB_SALES, APEX_INBOUND |
| APEX_AGENT | OWN | APEX_OB_SALES, APEX_INBOUND |
| GREEN_MANAGER | ALL | GREEN_APPTSET |
| GREEN_AGENT | NONE | N/A |
| VOTE_MANAGER | ALL | VOTE_SURVEY_Q1 |
| VOTE_AGENT | NONE | N/A |

### Infrastructure

| Component | Specification | Role |
|-----------|--------------|------|
| Server 1 | 8-core, 32 GB RAM, 2 TB SSD | DB + Web |
| Server 2 | 8-core, 16 GB RAM, 1 TB SSD | Telephony (Apex + GreenSolar) |
| Server 3 | 8-core, 16 GB RAM, 1 TB SSD | Telephony (VoteReach, seasonal) |

Server 3 can be decommissioned when VoteReach's seasonal campaign ends, reducing costs during off-peak months.

---

## Frequently Asked Questions

### Can a single agent work across multiple clients' campaigns?

Yes, but this creates a data isolation concern. If an agent is in a user group that has access to both Client A and Client B campaigns, they could theoretically see data from both clients (in reports or callback lists). If you need agents to float between clients, create a separate user group with access to multiple campaigns and understand that those agents are not isolated to a single tenant.

### How do I handle DNC lists for multiple clients?

VICIdial supports both a system-wide [DNC](/glossary/dnc/) list and per-campaign DNC lists. For multi-tenant operations, use per-campaign DNC lists (Admin > Campaigns > [Campaign] > DNC settings). This way, if a number is DNC for Client A, it is not automatically DNC for Client B unless it is also on Client B's list. The system-wide DNC list applies to all campaigns --- use it for numbers that should be blocked globally (e.g., your own company numbers, known litigators).

### Can I give clients direct admin access to manage their own campaigns?

Yes, with careful user group configuration. Set the client's manager user group to admin level 7-8 with campaign modification permissions limited to their allowed campaigns. They can then adjust [dial levels](/settings/auto-dial-level/), [lead recycling](/settings/lead-recycling/), dispositions, and other campaign settings without seeing or affecting other clients. We recommend starting with read-only access and upgrading to modify access as the client demonstrates competence.

### How do I prevent one client from using too many SIP trunks?

VICIdial does not have per-campaign trunk limits built in. The practical solution is to limit the [max dial level](/settings/adaptive-dl-level/) on a client's campaign, which indirectly limits their concurrent trunk usage. If a campaign's dial level is capped at 3.0 with 20 agents, the maximum concurrent outbound channels is 60. For dedicated trunk strategies, configure separate carriers with limited channel counts at the carrier level.

### Is there a way to brand the agent interface per client?

Not natively. VICIdial's agent screen is the same for all user groups. However, you can use campaign-level scripts (Admin > Campaigns > Script) to display client-specific branding, logos, and content within the script panel of the agent screen. For fully separate branded interfaces, you would need to build custom agent screens using VICIdial's agent API.

### Can I run different VICIdial versions for different clients on the same server?

No. A VICIdial installation is a single codebase. All campaigns run on the same VICIdial revision and PHP version. If different clients need different VICIdial versions (e.g., one client needs a feature only available in the latest SVN, while another needs to stay on an older version for compatibility), you need separate VICIdial instances.

### How do I handle client onboarding and offboarding?

**Onboarding:** Create user groups, campaigns, lists, carrier entries (if dedicated), and DID routing. Assign list ID ranges. Create manager and agent user accounts. Test with a small pilot list before going live.

**Offboarding:** Export the client's data (leads, CDR, recordings) and provide it to them. Deactivate their campaigns, users, and user groups (do not delete --- you may need the data for billing disputes or compliance). Archive their recordings to long-term storage. Release their DID numbers after a reasonable holding period (30-90 days in case the client returns).

### What about multi-tenant VICIdial in the cloud?

All of the multi-tenancy principles in this guide apply equally to [cloud-deployed VICIdial](/blog/vicidial-cloud-deployment/). Cloud adds the option of easily spinning up dedicated instances per client when needed, since provisioning a new server takes minutes instead of days. For BPOs, the hybrid cloud approach works well: shared infrastructure for small clients, dedicated cloud instances for large clients, all managed centrally.

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-multi-tenant).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
