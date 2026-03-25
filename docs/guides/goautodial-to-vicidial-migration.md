# Migrating from GoAutoDial to VICIdial: Step-by-Step

**Last updated: March 2026 | Reading time: ~16 minutes**

You are running GoAutoDial and something is not right. Maybe the admin interface keeps throwing PHP errors after an update. Maybe you need a VICIdial feature that GoAutoDial's custom GUI does not expose. Maybe you just realized that GoAutoDial CE 4.0 shipped with VICIdial 2.14-something from 2019, and you are running a contact center on software that is six years behind the current SVN trunk.

Whatever brought you here, the decision to migrate from GoAutoDial to VICIdial is almost always the correct one. GoAutoDial served a purpose --- it made VICIdial more approachable for people who were intimidated by the native admin interface. But that purpose has been overtaken by GoAutoDial's stagnating development, VICIdial's continuous improvement, and the operational reality that the extra GUI layer creates more problems than it solves.

This guide covers the complete migration path: what GoAutoDial is at a technical level, why migration makes sense in 2026, how to inventory your existing system, step-by-step data migration, the fresh install path, configuration mapping between GoAutoDial and VICIdial, testing procedures, and the rollback plan you should have but hopefully will not need.

**ViciStack migrates GoAutoDial operations to optimized VICIdial every month.** We handle the data migration, carrier reconfiguration, and agent retraining. [Let us handle your migration.](/free-audit/)

---

## What GoAutoDial Actually Is (Technically)

This is important to understand because it determines what your migration actually involves.

GoAutoDial is not a separate dialer. It is VICIdial with a custom PHP frontend bolted on top. Specifically:

- **GoAutoDial CE (Community Edition) 4.x** runs VICIdial 2.14 underneath, typically SVN revision 2900-3200 (compared to the current trunk at 3939+)
- **The GoAutoDial GUI** is a separate PHP application (usually at `/goautodial/` on the web server) that reads from and writes to the same MySQL `asterisk` database that VICIdial uses
- **The underlying [Asterisk](/glossary/asterisk/) instance**, cron jobs, screen processes, and telephony layer are all standard VICIdial
- **GoAutoDial-specific tables** in the database (prefixed with `go_` or `goautodial_`) store GUI preferences, user sessions, and configuration that the GoAutoDial interface needs but VICIdial does not

What this means for migration: **your core dialer data (campaigns, lists, leads, agents, carriers, recordings, CDR) is already in VICIdial-format tables.** The GoAutoDial GUI is a presentation layer on top of those tables. Migrating to VICIdial is primarily about (1) installing a current VICIdial build and (2) importing your existing data. You are not converting between two different systems --- you are upgrading the same system and removing a deprecated interface layer.

### GoAutoDial-Specific Components

These are the things GoAutoDial adds on top of VICIdial that you will leave behind:

| Component | GoAutoDial | VICIdial Equivalent |
|-----------|-----------|-------------------|
| Admin GUI | Custom PHP dashboard at `/goautodial/` | Native VICIdial admin at `/vicidial/admin.php` |
| Agent interface | GoAutoDial agent panel | VICIdial agent screen at `/agc/vicidial.php` |
| User management | GoAutoDial user management with roles | VICIdial [User Groups](/glossary/user-group/) with permission levels |
| Reports | GoAutoDial report dashboard | VICIdial built-in reports + custom SQL |
| API layer | GoAutoDial API wrapper | VICIdial non-agent API (more capable) |
| Branding | GoAutoDial logos and styling | VICIdial default or custom branding |

The GoAutoDial agent panel is worth discussing specifically. Many operations use GoAutoDial because agents find the GoAutoDial agent interface more visually appealing than VICIdial's default agent screen. This is a valid concern, but VICIdial's agent screen is highly customizable through CSS, custom scripts, and the [agent screen](/glossary/agent-screen/) configuration options. The visual gap between GoAutoDial's agent panel and a properly customized VICIdial agent screen is minimal.

---

## Why Migrate: The Case for Leaving GoAutoDial

### Development Has Stalled

GoAutoDial CE 4.0 was released in 2019. Since then:

- No major version releases
- Security patches lag months to years behind VICIdial SVN
- The community forums are largely inactive
- The development team has shifted focus to GoAutoDial Cloud (their hosted offering) and away from the self-hosted CE edition

Meanwhile, VICIdial SVN trunk has received continuous updates. Between 2019 and 2026, VICIdial added [ConfBridge](/glossary/confbridge/) support, [PHP 8 compatibility](/blog/vicidial-34-new-features/), two-factor authentication, ViciPhone v3.0 WebRTC, new API endpoints, AMD improvements, STIR/SHAKEN awareness, and dozens of bug fixes and performance improvements. None of these are available in GoAutoDial CE unless you manually update the VICIdial component underneath --- which risks breaking the GoAutoDial GUI layer.

### Security Exposure

Running GoAutoDial CE in 2026 means running:

- **An outdated VICIdial build** with known security vulnerabilities that have been patched in newer SVN revisions
- **An additional PHP application** (the GoAutoDial GUI) that increases your attack surface. The GoAutoDial GUI has its own authentication system, session management, and database access patterns that are separate from VICIdial's. More code means more potential vulnerabilities
- **CentOS 7** in most cases --- GoAutoDial CE 4.0's installer targets CentOS 7, which reached end-of-life in June 2024. No security patches, no kernel updates, no TLS library updates

If you are handling consumer financial data, healthcare information, or operating under any compliance framework that requires current security patches, GoAutoDial CE on CentOS 7 is a compliance violation waiting to happen.

### The GUI Layer Adds Complexity Without Proportional Benefit

The GoAutoDial GUI duplicates functionality that VICIdial already provides. This duplication creates confusion:

- Settings changed in the GoAutoDial GUI sometimes do not propagate correctly to the underlying VICIdial tables
- Settings changed directly in VICIdial's admin interface are not always reflected in the GoAutoDial GUI
- Troubleshooting requires checking two admin interfaces instead of one
- Documentation and community support for GoAutoDial is sparse compared to VICIdial, which has the vicidial.org forums, the EFLO wiki, and a global community of administrators

### VICIdial SVN Trunk is Actively Developed

Matt Florell and the VICIdial Group continue active development. The SVN trunk gets regular commits. The vicidial.org forums are active. The ViciBox distribution is maintained and updated. PoundTeam provides professional support. The ecosystem is healthy. Staying on GoAutoDial means opting out of this ecosystem.

---

## Pre-Migration Assessment

Before you touch anything, inventory your current GoAutoDial/VICIdial installation completely. This inventory determines your migration scope and identifies potential complications.

### Data Inventory Checklist

Run these queries on your GoAutoDial MySQL database to understand what you are working with:

```sql
-- Campaign count and configuration
SELECT campaign_id, campaign_name, active, dial_method, auto_dial_level
FROM vicidial_campaigns
ORDER BY active DESC, campaign_id;

-- List inventory with lead counts
SELECT vl.list_id, vl.list_name, vl.campaign_id, vl.active,
       COUNT(vll.lead_id) as lead_count
FROM vicidial_lists vl
LEFT JOIN vicidial_list vll ON vl.list_id = vll.list_id
GROUP BY vl.list_id
ORDER BY lead_count DESC;

-- Total leads by status
SELECT status, COUNT(*) as count
FROM vicidial_list
GROUP BY status
ORDER BY count DESC;

-- User/agent inventory
SELECT user_group, COUNT(*) as user_count, active
FROM vicidial_users
GROUP BY user_group, active;

-- Carrier/trunk configuration
SELECT carrier_id, carrier_name, active, protocol, server_ip
FROM vicidial_server_carriers
ORDER BY active DESC;

-- Inbound groups
SELECT group_id, group_name, active
FROM vicidial_inbound_groups
ORDER BY active DESC;

-- Recording volume
SELECT DATE_FORMAT(start_time, '%Y-%m') as month,
       COUNT(*) as recordings,
       ROUND(SUM(length_in_sec)/3600, 1) as total_hours
FROM recording_log
GROUP BY month
ORDER BY month DESC
LIMIT 12;

-- GoAutoDial-specific tables (to understand what's GoAutoDial-only)
SHOW TABLES LIKE 'go_%';
SHOW TABLES LIKE 'goautodial_%';
```

### What to Document

Create a migration document (spreadsheet or document) with:

1. **Campaigns:** Name, ID, dial method, active/inactive, custom settings
2. **Lists:** List IDs, campaign assignments, lead counts, active status
3. **Users:** User IDs, user groups, permission levels, active agents vs. managers
4. **Carriers:** Carrier names, [SIP](/glossary/sip/) credentials, IPs, [codec](/glossary/codec/) settings, trunk count
5. **DIDs:** [DID](/glossary/did/) numbers, routing destinations (inbound groups, agents, IVRs)
6. **Custom scripts:** Agent screen scripts, web form URLs, custom JavaScript
7. **Recordings:** Total volume, retention policy, storage location
8. **Cron jobs:** Any custom cron jobs beyond the default VICIdial set
9. **Integrations:** CRM connections, API consumers, webhook destinations, [Start Call URLs](/settings/start-call-url/)
10. **Custom code:** Any modifications to VICIdial or GoAutoDial PHP files

### Carrier Coordination

Contact your [SIP carrier(s)](/glossary/carrier/) before migration day:

- Inform them you will be migrating to a new server
- Get the new server's IP whitelisted in advance (if you are moving to new hardware)
- Confirm SIP credentials are documented (username, password, registration server, codec preferences)
- Verify any IP-based authentication will work with your new server IP
- Ask about their SIP registration cooldown --- some carriers lock out rapid re-registrations, which can cause a 5-15 minute outage if you switch servers too quickly

---

## Migration Path 1: Fresh ViciBox Install + Data Import (Recommended)

This is the clean path. It gives you a modern OS, current VICIdial, and a fresh start. It requires a new server (or a reformatted existing server) and a maintenance window.

### Step 1: Build the New Server

Follow the [VICIdial setup guide](/blog/vicidial-setup-guide/) to install ViciBox 12.0.2 on your new server. If you are deploying in the cloud, see the [cloud deployment guide](/blog/vicidial-cloud-deployment/).

Key points:
- ViciBox 12.0.2 runs on OpenSuSE Leap 15.6 with Asterisk 18 and MariaDB 10.11
- Set a static IP and configure DNS
- Run `vicibox-express` for a single-server setup
- Verify the base installation works (login to admin with 6666/1234)

### Step 2: Export Data from GoAutoDial

On your GoAutoDial server, export the core VICIdial tables. These are the same tables on both systems:

```bash
#!/bin/bash
# export_vicidial_data.sh - Run on GoAutoDial server
EXPORT_DIR="/root/migration_export"
MYSQL_PASS="your_mysql_root_password"
DB="asterisk"

mkdir -p ${EXPORT_DIR}

# Core campaign and configuration tables
TABLES=(
    "vicidial_campaigns"
    "vicidial_campaign_stats"
    "vicidial_lists"
    "vicidial_list"
    "vicidial_users"
    "vicidial_user_groups"
    "vicidial_inbound_groups"
    "vicidial_server_carriers"
    "vicidial_call_menu"
    "vicidial_remote_agents"
    "vicidial_scripts"
    "vicidial_filters"
    "vicidial_lead_filters"
    "vicidial_statuses"
    "vicidial_campaign_statuses"
    "vicidial_phone_codes"
    "vicidial_dnc"
    "vicidial_campaign_dnc"
    "vicidial_inbound_dids"
    "vicidial_did_log"
    "vicidial_callbacks"
    "vicidial_closer_log"
    "vicidial_log"
    "recording_log"
    "vicidial_list_alt_phones"
)

for TABLE in "${TABLES[@]}"; do
    echo "Exporting ${TABLE}..."
    mysqldump -u root -p"${MYSQL_PASS}" --single-transaction \
        --no-create-info --complete-insert \
        ${DB} ${TABLE} > ${EXPORT_DIR}/${TABLE}.sql
    echo "  Rows: $(wc -l < ${EXPORT_DIR}/${TABLE}.sql)"
done

# Also export table structures for reference
mysqldump -u root -p"${MYSQL_PASS}" --no-data \
    ${DB} > ${EXPORT_DIR}/schema_only.sql

echo "Export complete. Files in ${EXPORT_DIR}/"
ls -lh ${EXPORT_DIR}/
```

**Important:** We use `--no-create-info` because the new ViciBox installation already has the correct table schemas for the current VICIdial version. We only want the data rows. The schema on the new server may have additional columns that the old server does not have --- that is expected and the import will populate those new columns with defaults.

### Step 3: Transfer Data to New Server

```bash
# From the GoAutoDial server
scp -r /root/migration_export root@NEW_SERVER_IP:/root/migration_import/

# Also transfer recordings if you want them on the new server
rsync -avz --progress /var/spool/asterisk/monitor/ \
    root@NEW_SERVER_IP:/var/spool/asterisk/monitor/
```

Recording transfer can take hours or days depending on volume. Start this well before your migration window. You can run rsync incrementally --- it will only transfer new/changed files on subsequent runs.

### Step 4: Prepare the New Server for Import

Before importing data, you need to handle potential conflicts with the default ViciBox data:

```bash
#!/bin/bash
# prepare_import.sh - Run on NEW ViciBox server
MYSQL_PASS="your_new_mysql_root_password"
DB="asterisk"

# Back up the default ViciBox data (just in case)
mysqldump -u root -p"${MYSQL_PASS}" ${DB} > /root/vicibox_default_backup.sql

# Clear default sample data that would conflict with imports
mysql -u root -p"${MYSQL_PASS}" ${DB} << 'EOF'
-- Remove default campaigns (keep the schema, remove rows)
DELETE FROM vicidial_campaigns WHERE campaign_id IN ('TESTCAMP', 'SAMPLE');

-- Remove default users EXCEPT admin (6666) and API users you'll need
DELETE FROM vicidial_users WHERE user NOT IN ('6666', 'VDAD', 'VDCL');

-- Remove default user groups EXCEPT ADMIN
DELETE FROM vicidial_user_groups WHERE user_group != 'ADMIN';

-- Clear default lists
DELETE FROM vicidial_lists WHERE list_id < 200;

-- Clear default inbound groups
DELETE FROM vicidial_inbound_groups WHERE group_id IN ('AGENTDIRECT', 'SALESLINE');

-- Note: Do NOT delete from system-level tables like servers, conferences, etc.
EOF

echo "Default data cleared. Ready for import."
```

### Step 5: Import Data

```bash
#!/bin/bash
# import_data.sh - Run on NEW ViciBox server
IMPORT_DIR="/root/migration_import"
MYSQL_PASS="your_new_mysql_root_password"
DB="asterisk"

# Import order matters - reference tables first, then data tables
IMPORT_ORDER=(
    "vicidial_user_groups"
    "vicidial_users"
    "vicidial_statuses"
    "vicidial_campaigns"
    "vicidial_campaign_statuses"
    "vicidial_lists"
    "vicidial_list"
    "vicidial_server_carriers"
    "vicidial_inbound_groups"
    "vicidial_call_menu"
    "vicidial_inbound_dids"
    "vicidial_scripts"
    "vicidial_filters"
    "vicidial_lead_filters"
    "vicidial_remote_agents"
    "vicidial_phone_codes"
    "vicidial_dnc"
    "vicidial_campaign_dnc"
    "vicidial_callbacks"
    "vicidial_list_alt_phones"
    "vicidial_log"
    "vicidial_closer_log"
    "vicidial_did_log"
    "recording_log"
)

for TABLE in "${IMPORT_ORDER[@]}"; do
    if [ -f "${IMPORT_DIR}/${TABLE}.sql" ]; then
        echo "Importing ${TABLE}..."
        mysql -u root -p"${MYSQL_PASS}" ${DB} < ${IMPORT_DIR}/${TABLE}.sql
        if [ $? -eq 0 ]; then
            echo "  Success"
        else
            echo "  WARNING: Errors during import of ${TABLE}"
        fi
    else
        echo "  SKIP: ${IMPORT_DIR}/${TABLE}.sql not found"
    fi
done

echo "Import complete."
```

### Step 6: Post-Import Adjustments

After importing data, several things need updating:

```sql
-- Update server_ip references to point to the new server
-- GoAutoDial had the old server's IP; VICIdial needs the new one
UPDATE vicidial_campaigns SET server_ip = 'NEW_SERVER_IP'
    WHERE server_ip = 'OLD_SERVER_IP';

UPDATE vicidial_server_carriers SET server_ip = 'NEW_SERVER_IP'
    WHERE server_ip = 'OLD_SERVER_IP';

UPDATE vicidial_servers SET server_ip = 'NEW_SERVER_IP'
    WHERE server_ip = 'OLD_SERVER_IP';

-- Update any recording_log entries to reflect new file paths
-- (only needed if recording directory structure changed)
UPDATE recording_log
SET location = REPLACE(location, '/old/path/', '/var/spool/asterisk/monitor/MIXmonitor/')
WHERE location LIKE '/old/path/%';

-- Reset conference extensions (they're server-specific)
-- The new ViciBox install already has these configured correctly
-- Don't import vicidial_conferences from the old server

-- Verify admin user still works
SELECT user, pass, full_name, user_level, active
FROM vicidial_users
WHERE user_level >= 8;
```

### Step 7: Configure Carriers

Carriers (SIP trunks) need manual reconfiguration because they involve both database entries and Asterisk configuration files:

1. Navigate to **Admin > Carriers** on the new ViciBox server
2. For each carrier from your inventory document:
   - If the carrier was imported correctly, verify the settings match your documentation
   - Update the `Server IP` field to the new server's IP
   - If using IP-based authentication, ensure your carrier has whitelisted the new server's IP
   - If using registration-based authentication, verify credentials are correct
3. Reload Asterisk: `asterisk -rx "sip reload"` or `asterisk -rx "pjsip reload"`
4. Verify registration: `asterisk -rx "sip show peers"` or `asterisk -rx "pjsip show endpoints"`

For [PJSIP](/glossary/pjsip/) configurations (recommended for new installs), you may need to convert old `sip.conf` carrier entries to `pjsip.conf` format. This is a one-time effort but important for forward compatibility with [Asterisk 18](/glossary/asterisk/).

### Step 8: Test Everything

Before switching production traffic:

- [ ] Admin login works (with your existing admin credentials, not just 6666)
- [ ] Agent login works for at least 2-3 agents
- [ ] Outbound test call connects with two-way audio
- [ ] Inbound test call routes correctly to the right [inbound group](/glossary/inbound-group/)
- [ ] [Call recording](/blog/vicidial-call-recording/) creates files and recording_log entries
- [ ] Lead data displays correctly on the agent screen
- [ ] [Callbacks](/glossary/callback/) fire at scheduled times
- [ ] Reports generate correctly with historical data
- [ ] [DNC](/glossary/dnc/) list is active (test by trying to dial a known DNC number)
- [ ] [Hopper](/glossary/hopper/) fills correctly (`screen -r VDhopper` to watch)
- [ ] All custom web forms and scripts load from the agent screen

> **Migration Without the Guesswork.**
> ViciStack has migrated dozens of GoAutoDial operations to current VICIdial builds. We handle the data, the carriers, and the testing. [Schedule Your Migration.](/free-audit/)

---

## Migration Path 2: In-Place Upgrade (Not Recommended)

It is technically possible to update VICIdial underneath GoAutoDial without a fresh install. We do not recommend this for several reasons, but documenting it for completeness.

### The Process

1. SSH into your GoAutoDial server
2. Navigate to the VICIdial installation directory
3. Run `svn update` to pull the latest VICIdial code from the SVN repository
4. Run the database upgrade scripts to update the schema

```bash
# This is what the process looks like (DO NOT run this blindly)
cd /usr/share/astguiclient/
svn update svn://svn.eflo.net:3690/agc_2-X/trunk
perl install.pl --run-db-upgrade
```

### Why This Is Risky

1. **GoAutoDial's PHP files may conflict with updated VICIdial code.** GoAutoDial modifies some VICIdial PHP files. An SVN update will overwrite those modifications.
2. **CentOS 7 is end-of-life.** Updating VICIdial does not fix the underlying OS problem. You are still running on an unpatched OS.
3. **Asterisk version mismatch.** GoAutoDial CE 4.0 typically runs Asterisk 13 or 16. Current VICIdial SVN expects Asterisk 18. Updating VICIdial without updating Asterisk will break things.
4. **No clean rollback path.** If the update breaks something, you cannot easily undo an SVN update + database schema change. You would need to restore from backup.
5. **You are still left with GoAutoDial's GUI layer** which serves no purpose if you are running current VICIdial.

If you absolutely cannot do a fresh install (e.g., you have 500,000 leads and cannot afford the downtime), the in-place path exists. But plan for 2-3x the troubleshooting time compared to a fresh install, and have a full backup you have tested restoring from.

---

## Configuration Mapping: GoAutoDial GUI to VICIdial Admin

For administrators who learned VICIdial through GoAutoDial's interface, here is where to find things in the native VICIdial admin.

### Campaign Management

| GoAutoDial Path | VICIdial Admin Path |
|-----------------|-------------------|
| Telephony > Campaigns > Campaign List | Admin > Campaigns |
| Campaign > General Settings | Admin > Campaigns > [Campaign] > Detail |
| Campaign > Dial Settings | Admin > Campaigns > [Campaign] > Detail (same page, scroll down) |
| Campaign > Dispositions | Admin > Campaigns > [Campaign] > Statuses |
| Campaign > Recycle Rules | Admin > Campaigns > [Campaign] > [Lead Recycling](/settings/lead-recycling/) |
| Campaign > Callback Settings | Admin > Campaigns > [Campaign] > Detail > [Scheduled Callbacks](/settings/scheduled-callbacks/) section |
| Campaign > Caller ID | Admin > Campaigns > [Campaign] > Detail > [Outbound CID](/settings/outbound-cid/) |

### List and Lead Management

| GoAutoDial Path | VICIdial Admin Path |
|-----------------|-------------------|
| Telephony > Lists | Admin > Lists |
| Lists > Upload Leads | Admin > Lists > [List] > Load Leads |
| Lists > Lead Management | Admin > Lists > [List] > Leads (or direct SQL) |
| DNC Management | Admin > DNC |
| Lead Filters | Admin > Filters |

For a complete guide to list management in VICIdial, see [VICIdial List Management Best Practices](/blog/vicidial-list-management/).

### User and Agent Management

| GoAutoDial Path | VICIdial Admin Path |
|-----------------|-------------------|
| Users > User List | Admin > Users |
| Users > User Groups | Admin > User Groups |
| Users > Agent Phones | Admin > Phones |
| User Permissions | Admin > User Groups > [Group] > Permissions |

### Carrier and Trunk Configuration

| GoAutoDial Path | VICIdial Admin Path |
|-----------------|-------------------|
| Telephony > Carriers | Admin > Carriers |
| Telephony > DIDs | Admin > DIDs |
| DID Routing | Admin > [Inbound Groups](/glossary/inbound-group/) + Admin > DIDs |

### Reports

| GoAutoDial Report | VICIdial Admin Report |
|-------------------|---------------------|
| Agent Performance | Reports > Agent Performance Detail |
| Campaign Summary | Reports > Campaign Summary |
| Call Detail Records | Reports > Outbound Calling Report (or CDR search) |
| Inbound Report | Reports > Inbound Report |
| Recording Search | Reports > Recording Lookup |

**The biggest difference:** GoAutoDial's reports had a visual dashboard with graphs. VICIdial's built-in reports are table-based. If you need graphical reporting, integrate with Grafana (free) or a commercial BI tool. The underlying data is all accessible via MySQL queries.

---

## Agent Transition: What Changes for Your Team

The most disruptive part of a GoAutoDial-to-VICIdial migration is not the technical work --- it is agent confusion. If your agents have only ever used GoAutoDial's agent panel, the VICIdial agent screen looks different. Here is how to manage the transition.

### What Looks Different

- **Layout:** VICIdial's agent screen uses a different layout than GoAutoDial's agent panel. The call controls, disposition buttons, and script area are in different positions.
- **Styling:** VICIdial's default styling is more utilitarian. This is fixable with CSS customization.
- **Phone registration:** If using softphones, agents may need to register to a new SIP server IP.

### What Stays the Same

- **Core workflow:** Login > Wait for call > Talk > Disposition > Repeat. This does not change.
- **Disposition codes:** All your custom dispositions were imported with the data migration. Same codes, same meanings.
- **Callback scheduling:** Same CALLBK/CBHOLD workflow.
- **Scripts:** Agent scripts were imported and display the same content.
- **Transfer process:** Same [transfer](/glossary/transfer/) workflow with blind and consultative options.

### Training Approach

1. **Before migration day:** Send agents a 1-page visual guide showing the VICIdial agent screen with annotations pointing out where familiar controls are located
2. **Migration day:** Have a team lead or admin available on a video call to walk agents through their first login
3. **First hour:** Run a practice campaign with test data so agents can click through dispositions, schedule callbacks, and transfer calls without affecting live leads
4. **First day:** Monitor agent performance metrics closely. Expect a 10-20% productivity dip on day one as agents adjust. This recovers by day 2-3.

### Customizing the VICIdial Agent Screen

If the default VICIdial agent screen is a problem for your team, customize it:

1. **CSS overrides:** Add a custom CSS file to restyle the agent screen. Colors, fonts, button sizes, layout adjustments.
2. **Custom scripts with HTML/CSS:** VICIdial's script system supports full HTML. Use it to create a custom layout within the script panel.
3. **Custom web forms:** The agent screen can load external web pages in an iframe. If you want a completely custom agent experience, build it as a web application that communicates with VICIdial via the agent API.

---

## Rollback Plan

Every migration needs a rollback plan. Here is yours.

### Before Migration

1. **Full backup of GoAutoDial server:**
```bash
# Complete database dump
mysqldump -u root -p --all-databases > /root/goautodial_full_backup_$(date +%Y%m%d).sql

# Full system backup (if you have space)
tar czf /root/goautodial_server_backup.tar.gz \
    /etc/asterisk/ \
    /var/spool/asterisk/ \
    /usr/share/astguiclient/ \
    /var/www/html/ \
    /etc/crontab
```

2. **Do not decommission the GoAutoDial server** until you have verified the new VICIdial server is stable for at least one full business week

3. **Keep the GoAutoDial server's SIP registrations intact** (just inactive). If you need to roll back, you can re-register with your carrier immediately

### Rollback Procedure

If the new VICIdial server fails critically on migration day:

1. Re-register your SIP trunks on the old GoAutoDial server: `asterisk -rx "sip reload"`
2. If your carrier needs to re-whitelist the old server's IP, call them immediately
3. Point agents back to the GoAutoDial server URL
4. Export any new leads or CDR from the VICIdial server that were created during the failed migration window
5. Import those records into GoAutoDial to avoid data loss

**The rollback window is practical for about 7 days** after migration. After that, new data (leads, CDR, recordings, dispositions) accumulates on the VICIdial server and becomes increasingly difficult to merge back into GoAutoDial.

---

## Post-Migration Optimization

Once you are running on current VICIdial, take advantage of the features GoAutoDial never had access to:

### Immediate Wins

- **[Predictive dialer optimization](/blog/vicidial-predictive-dialer-settings/):** Current VICIdial's [adaptive dialing](/glossary/adaptive-dialing/) algorithm is significantly improved over the 2019-era code in GoAutoDial. Retune your [dial levels](/settings/auto-dial-level/) and [abandon rate targets](/settings/abandon-rate-target/).
- **[WebRTC with ViciPhone 3.0](/glossary/webrtc/):** Current VICIdial includes ViciPhone v3.0, a much more capable WebRTC softphone than what GoAutoDial offered. Remote agents can connect through the browser without external softphones.
- **Two-factor authentication:** Secure your admin accounts with 2FA, a feature not available in GoAutoDial CE.
- **[ConfBridge](/glossary/confbridge/):** Replace the legacy MeetMe conferencing engine with ConfBridge for better performance and no DAHDI dependency for conference rooms.

### Settings to Review

After migration, walk through these settings that may have been configured for GoAutoDial's defaults rather than optimal VICIdial values:

- [Hopper Level](/settings/hopper-level/) --- GoAutoDial often set this low. Increase to 2-4x your agent count.
- [Available Only Ratio Tally](/settings/available-only-ratio-tally/) --- Set to Y for more accurate [predictive dialing](/glossary/predictive-dialing/).
- [Wait Between Calls](/settings/wait-between-calls/) --- Ensure this matches your desired agent workflow.
- [Campaign Recording](/settings/campaign-recording/) --- Verify recording settings migrated correctly.
- [AMD Type](/settings/amd-type/) --- If you are using [AMD](/glossary/amd/), current VICIdial supports CPD (Call Progress Detection) which is more accurate than the older AMD method. See our [AMD tuning guide](/blog/vicidial-amd-guide/).

---

## Frequently Asked Questions

### How long does the migration take?

For a typical operation (1-5 campaigns, 10-50 agents, under 1 million leads), the technical migration takes 4-8 hours including data export, fresh install, data import, carrier configuration, and testing. The total project including planning, agent training, and stabilization is 1-2 weeks. ViciStack can complete the technical migration overnight for most operations.

### Will I lose my historical CDR and reporting data?

No. The `vicidial_log`, `vicidial_closer_log`, and `recording_log` tables are all migrated. Your historical call data, recordings, and agent performance history will be available on the new VICIdial server. Reports will show the same historical data.

### Can I migrate leads that are in active callback status?

Yes. The `vicidial_callbacks` table is included in the standard migration. All LIVE callbacks will continue to fire on the new server at their scheduled times. Verify this by checking `SELECT * FROM vicidial_callbacks WHERE status = 'LIVE'` after migration.

### Do I need to re-train agents on VICIdial?

Minimally. The core agent workflow (login, take calls, disposition, callback) is functionally identical. The visual layout is different but navigable within a few calls. Plan for a 1-hour walkthrough and expect full productivity by day 2. The biggest adjustment is for managers who used GoAutoDial's admin dashboard --- VICIdial's admin interface has a steeper learning curve but is more powerful.

### What about GoAutoDial Cloud --- is that different from GoAutoDial CE?

Yes. GoAutoDial Cloud is the company's hosted SaaS offering. It may be running newer VICIdial code than CE 4.0, but you have no control over the underlying infrastructure, no SSH access, and limited customization. Migrating from GoAutoDial Cloud to self-hosted VICIdial requires requesting a data export from GoAutoDial (which they may or may not provide promptly) and then following the same import process described in this guide.

### Can I run GoAutoDial's GUI on top of current VICIdial?

Technically possible by installing the GoAutoDial PHP files on a current VICIdial server. Practically, this is a bad idea. GoAutoDial's PHP code was written for PHP 5.x/7.x and VICIdial 2.14 schema circa 2019. Running it against a PHP 8.2 environment with VICIdial 2.14b0.5 schema 1729 will produce errors, broken pages, and potentially data corruption. The whole point of migration is to leave the GoAutoDial layer behind.

### Is there a commercial migration tool?

No dedicated GoAutoDial-to-VICIdial migration tool exists because the data is fundamentally the same format --- it is all MySQL tables in the `asterisk` database. The migration is a standard MySQL export/import with some post-import cleanup. Tools like ViciStack's migration service handle this as part of a managed deployment.

### What if I have custom PHP modifications in GoAutoDial?

Document them before migration. If the customizations are in GoAutoDial-specific files (under `/goautodial/`), they will not carry over and you will need to reimplement them in VICIdial's framework. If the customizations are in VICIdial's core files (under `/agc/` or `/vicidial/`), they may work on the new server but should be reviewed for compatibility with the current VICIdial codebase. Any customizations that reference GoAutoDial-specific database tables (`go_*` tables) will need rewriting.

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/goautodial-to-vicidial-migration).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
