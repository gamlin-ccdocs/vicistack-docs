# VICIdial to Hosted Migration Checklist: Step-by-Step Guide

Migrating a live call center is one of the highest-risk operations you'll undertake. Every minute of downtime costs real money -- agents sitting idle, leads going stale, clients asking questions you don't want to answer. Whether you're moving from self-hosted VICIdial to a managed provider, migrating from another dialer platform to VICIdial, or consolidating multiple VICIdial instances, the process demands careful planning.

This guide provides a comprehensive, field-tested migration checklist covering every phase from initial assessment through post-migration verification. It's based on our experience migrating over 100 call centers to managed VICIdial environments. Every item on this list exists because someone, somewhere, learned the hard way what happens when you skip it.

## Phase 1: Pre-Migration Assessment

Before you touch a single configuration file, you need a complete picture of what you're working with. The assessment phase typically takes 3-5 business days.

### 1.1 Current Setup Audit

Document every aspect of your current environment. This becomes your migration blueprint.

**Server inventory:**

```bash
# On your current VICIdial server(s), run:
# System specs
cat /proc/cpuinfo | grep "model name" | head -1
free -h
df -h
cat /etc/os-release

# VICIdial version
cat /usr/src/astguiclient/trunk/bin/ADMIN_keepalive_ALL.pl | grep "build"

# Asterisk version
asterisk -rx "core show version"

# MySQL version and database size
mysql -e "SELECT VERSION();"
mysql -e "SELECT table_schema, ROUND(SUM(data_length + index_length) / 1024 / 1024, 1) AS 'Size (MB)' FROM information_schema.TABLES GROUP BY table_schema;"
```

**Configuration snapshot:**

```bash
# Export critical configs
mkdir -p /tmp/migration-audit

# Asterisk configuration
cp -r /etc/asterisk/ /tmp/migration-audit/asterisk-conf/

# VICIdial settings
mysqldump asterisk system_settings servers conferences phones \
  vicidial_campaigns vicidial_inbound_groups vicidial_call_times \
  vicidial_campaign_cid_areacodes vicidial_server_carriers \
  > /tmp/migration-audit/vicidial-settings.sql

# Crontab entries
crontab -l > /tmp/migration-audit/crontab-root.txt
su - asterisk -c "crontab -l" > /tmp/migration-audit/crontab-asterisk.txt
```

### 1.2 Campaign Configuration Documentation

For each active campaign, record:

| Setting | Where to Find It |
|---------|-----------------|
| Dial method | Campaign > Dialing > Dial Method |
| Auto dial level | Campaign > Dialing > Auto Dial Level |
| Adaptive settings | Campaign > Dialing > Adaptive section |
| AMD settings | Campaign > AMD/CPD section |
| Local caller ID | Campaign > Caller ID |
| Transfer settings | Campaign > Transfers |
| Call times | Campaign > Call Times |
| DNC settings | Campaign > DNC section |
| Drop action | Campaign > Drop Call settings |
| Hopper level/priority | Campaign > Hopper section |

Export all campaigns via the admin API:

```bash
# Export campaign list
mysql asterisk -e "
SELECT campaign_id, campaign_name, dial_method, auto_dial_level,
       adaptive_maximum_level, adaptive_dropped_percentage,
       active, campaign_cid, amd_type
FROM vicidial_campaigns
ORDER BY campaign_id;" > /tmp/migration-audit/campaigns.tsv
```

### 1.3 Integration Inventory

Document every system that talks to VICIdial:

- **CRM integrations:** Web form URLs, API connections, webhook endpoints
- **Lead sources:** How leads get into VICIdial (API, FTP upload, manual, CRM push)
- **Reporting feeds:** Any external systems pulling VICIdial data
- **Recording access:** Who accesses recordings and how (web UI, API, direct file access)
- **IVR/routing:** Custom AGI scripts, dialplan modifications
- **Monitoring:** Any external monitoring (Nagios, Zabbix, Datadog)

```bash
# Find custom AGI scripts
find /var/lib/asterisk/agi-bin/ -name "*.php" -o -name "*.pl" -o -name "*.py" | sort

# Find custom VICIdial modifications
find /usr/share/astguiclient/www/ -newer /usr/share/astguiclient/www/vicidial/welcome.php -type f | sort

# Find web form URLs in campaigns
mysql asterisk -e "
SELECT campaign_id, web_form_address, web_form_address_two
FROM vicidial_campaigns
WHERE web_form_address != '' OR web_form_address_two != '';"
```

### 1.4 Volume Baseline

Record current performance metrics to validate post-migration:

```sql
-- Daily call volumes for the past 30 days
SELECT DATE(call_date) AS day,
       COUNT(*) AS total_calls,
       SUM(CASE WHEN status NOT IN ('A','AA','AM','AL','N','NI','NP','DC','DNC','B','NA','ADC','DROP') THEN 1 ELSE 0 END) AS contacts,
       ROUND(AVG(length_in_sec), 1) AS avg_duration,
       MAX(HOUR(call_date)) AS last_call_hour
FROM vicidial_log
WHERE call_date >= DATE_SUB(CURDATE(), INTERVAL 30 DAY)
GROUP BY DATE(call_date)
ORDER BY day;

-- Agent count by day
SELECT DATE(event_time) AS day,
       COUNT(DISTINCT user) AS unique_agents
FROM vicidial_agent_log
WHERE event_time >= DATE_SUB(CURDATE(), INTERVAL 30 DAY)
GROUP BY DATE(event_time)
ORDER BY day;
```

Save these numbers. You'll compare against them after migration to confirm everything is working correctly.

## Phase 2: Data Export

### 2.1 Lead Data Export

Export all lead data, not just active leads. Historical data is valuable for analytics and compliance.

```bash
# Full lead export
mysql asterisk -e "
SELECT * FROM vicidial_list
" > /tmp/migration-export/leads_full.tsv

# Custom fields export (if using custom fields)
for table in $(mysql asterisk -N -e "SHOW TABLES LIKE 'custom_%'"); do
    mysql asterisk -e "SELECT * FROM $table" > "/tmp/migration-export/${table}.tsv"
done

# Lead count verification
mysql asterisk -e "
SELECT list_id, COUNT(*) AS lead_count,
       SUM(CASE WHEN status = 'NEW' THEN 1 ELSE 0 END) AS new_leads,
       SUM(CASE WHEN status IN ('SALE','XFER') THEN 1 ELSE 0 END) AS converted
FROM vicidial_list
GROUP BY list_id
ORDER BY list_id;"
```

**Verification step:** Count rows in the export file and compare to the database count. They must match exactly.

```bash
# Count exported rows (subtract 1 for header)
wc -l /tmp/migration-export/leads_full.tsv
mysql asterisk -e "SELECT COUNT(*) FROM vicidial_list;"
```

### 2.2 Campaign and Configuration Export

```bash
# Full campaign configuration dump
mysqldump asterisk \
  vicidial_campaigns \
  vicidial_campaign_agents \
  vicidial_campaign_statuses \
  vicidial_campaign_hotkeys \
  vicidial_campaign_cid_areacodes \
  vicidial_inbound_groups \
  vicidial_inbound_group_agents \
  vicidial_call_times \
  vicidial_state_call_times \
  vicidial_lists \
  vicidial_statuses \
  vicidial_pause_codes \
  vicidial_user_groups \
  vicidial_users \
  vicidial_scripts \
  vicidial_filters \
  vicidial_lead_filters \
  vicidial_lead_recycle \
  phones \
  > /tmp/migration-export/config-dump.sql
```

### 2.3 Recording Export

Call recordings are often the largest data set and the most commonly forgotten during migration.

```bash
# Find recording storage location
mysql asterisk -e "SELECT recording_path FROM system_settings;"

# Calculate total recording storage
du -sh /var/spool/asterisk/monitor/
du -sh /var/spool/asterisk/VDmonitor/

# Export recording database index
mysql asterisk -e "
SELECT recording_id, channel, server_ip, extension,
       start_time, end_time, length_in_sec, filename, location,
       lead_id, user, vicidial_id
FROM recording_log
ORDER BY start_time;" > /tmp/migration-export/recording_index.tsv
```

**Recording transfer options:**

1. **rsync** (best for large archives):
```bash
rsync -avz --progress /var/spool/asterisk/monitor/ \
  user@new-server:/var/spool/asterisk/monitor/
```

2. **Compressed archive** for smaller sets:
```bash
# Compress by month for manageable file sizes
for month in $(seq -w 1 12); do
    tar czf "recordings-2025-${month}.tar.gz" \
        /var/spool/asterisk/monitor/2025/${month}/
done
```

3. **S3/object storage** for long-term archive:
```bash
aws s3 sync /var/spool/asterisk/monitor/ \
  s3://your-bucket/vicidial-recordings/ \
  --storage-class STANDARD_IA
```

### 2.4 DNC List Export

```bash
mysql asterisk -e "
SELECT phone_number FROM vicidial_dnc
ORDER BY phone_number;" > /tmp/migration-export/dnc_internal.tsv

mysql asterisk -e "
SELECT phone_number FROM vicidial_campaign_dnc
ORDER BY campaign_id, phone_number;" > /tmp/migration-export/dnc_campaign.tsv

# Count for verification
echo "Internal DNC count:"
mysql asterisk -e "SELECT COUNT(*) FROM vicidial_dnc;"
echo "Campaign DNC count:"
mysql asterisk -e "SELECT COUNT(*) FROM vicidial_campaign_dnc;"
```

### 2.5 Disposition Export

```bash
# System dispositions
mysql asterisk -e "SELECT * FROM vicidial_statuses ORDER BY status;" \
  > /tmp/migration-export/system_dispos.tsv

# Campaign-specific dispositions
mysql asterisk -e "SELECT * FROM vicidial_campaign_statuses ORDER BY campaign_id, status;" \
  > /tmp/migration-export/campaign_dispos.tsv
```

## Phase 3: Carrier and DID Migration Planning

Carrier migration is often the longest lead-time item. Start this process early.

### 3.1 Carrier Inventory

```bash
# Current carrier configuration
mysql asterisk -e "
SELECT carrier_id, carrier_name, registration_string,
       account_entry, protocol, globals_string
FROM vicidial_server_carriers
ORDER BY carrier_id;"
```

Document for each carrier:
- Carrier name and account number
- Number of concurrent channels
- Per-minute rates
- DID inventory
- Contract end date and porting authorization requirements

### 3.2 DID Inventory and Porting Plan

```bash
# All DIDs configured in VICIdial
mysql asterisk -e "
SELECT did_id, did_pattern, did_description, did_route,
       group_id, did_carrier_description
FROM vicidial_inbound_dids
ORDER BY did_pattern;"

# CID numbers used by campaigns
mysql asterisk -e "
SELECT DISTINCT campaign_cid, campaign_id
FROM vicidial_campaigns
WHERE campaign_cid != ''
UNION
SELECT outbound_cid, campaign_id
FROM vicidial_campaign_cid_areacodes
ORDER BY 1;"
```

**Porting timeline:**

| Porting Type | Typical Duration |
|-------------|-----------------|
| Between major carriers | 5-10 business days |
| Toll-free numbers | 5-15 business days |
| Wireless to VOIP | 7-14 business days |
| Between VOIP providers | 3-7 business days |

### 3.3 SIP Trunk Configuration

Prepare new SIP trunk configurations for the target environment:

```
; Example SIP trunk for new carrier
[carrier-new]
type=peer
host=sip.newcarrier.com
fromuser=your_account
secret=your_password
insecure=port,invite
qualify=yes
context=trunkinbound
dtmfmode=rfc2833
disallow=all
allow=ulaw
allow=alaw
```

Test trunk connectivity before the migration day:

```bash
# Register and test from the new server
asterisk -rx "sip show peers" | grep carrier-new
asterisk -rx "sip show registry"

# Place a test call
asterisk -rx "channel originate SIP/carrier-new/12125551234 application Playback hello-world"
```

## Phase 4: Agent Training

### 4.1 Training Plan

Even if agents are moving between VICIdial versions, interface differences can cause confusion. Build a training plan:

**For VICIdial-to-VICIdial migrations:**
- 30-minute session covering any UI differences
- New login URLs and credentials
- Changes in softphone configuration (if any)
- Updated processes for recording access
- Emergency procedures (who to call if the dialer is down)

**For migrations from other platforms to VICIdial:**
- 2-4 hour initial training session
- Cover: login, agent screen layout, call handling, disposition, transfers
- Hands-on practice with test campaigns
- Print quick-reference cards for the first week
- Designate floor walkers for the first 2 days post-migration

### 4.2 Documentation Updates

Prepare before migration day:

- [ ] New login URL documented and distributed
- [ ] New softphone configuration instructions (if applicable)
- [ ] Updated agent quick-reference guide
- [ ] Updated supervisor/manager guide
- [ ] Emergency contact list (new server admin, carrier support, ViciStack support)
- [ ] FAQ document addressing common questions

## Phase 5: Parallel Running Period

Never cut over cold. Run both systems simultaneously for at least one week.

### 5.1 Setup Parallel Environment

```
Week 1: Build and configure new environment
Week 2: Load data, configure campaigns, test internally
Week 3: Run 5-10 agents on new system alongside old system
Week 4: Expand to 50% of agents on new system
Week 5: Full cutover
```

### 5.2 Parallel Running Checklist

During the parallel period, validate daily:

- [ ] Call volume on new system matches expected proportion
- [ ] Connect rates are within 10% of old system
- [ ] AMD accuracy is comparable or better
- [ ] Drop rates are within compliance limits
- [ ] Agent talk time and utilization are comparable
- [ ] All dispositions are recording correctly
- [ ] Recordings are being saved and accessible
- [ ] CRM integration is functioning
- [ ] Lead loading is working
- [ ] Callbacks are firing correctly
- [ ] DNC checks are working
- [ ] Caller ID is displaying correctly
- [ ] Transfer destinations are connecting

### 5.3 Performance Comparison Query

Run this daily during parallel operation to compare old vs new:

```sql
-- Run on each system
SELECT
    'NEW_SYSTEM' AS system,
    DATE(call_date) AS day,
    COUNT(*) AS total_calls,
    SUM(CASE WHEN status NOT IN ('A','AA','AM','AL','N','NI','NP','DC','DNC','B','NA','ADC','DROP')
        THEN 1 ELSE 0 END) AS contacts,
    ROUND(SUM(CASE WHEN status NOT IN ('A','AA','AM','AL','N','NI','NP','DC','DNC','B','NA','ADC','DROP')
        THEN 1 ELSE 0 END) / NULLIF(COUNT(*), 0) * 100, 2) AS contact_rate,
    ROUND(SUM(CASE WHEN status = 'DROP' THEN 1 ELSE 0 END) / NULLIF(COUNT(*), 0) * 100, 2) AS drop_rate,
    ROUND(AVG(length_in_sec), 1) AS avg_duration
FROM vicidial_log
WHERE call_date >= CURDATE()
GROUP BY DATE(call_date);
```

## Phase 6: Cutover Checklist (30 Items)

This is the full cutover checklist. Execute items in order on migration day.

### Pre-Cutover (Evening Before)

- [ ] **1.** Confirm all DIDs have been ported to new carrier and are routing correctly
- [ ] **2.** Verify all SIP trunks are registered on new server (`asterisk -rx "sip show registry"`)
- [ ] **3.** Run final lead data sync from old to new system
- [ ] **4.** Verify lead counts match between old and new (`SELECT COUNT(*) FROM vicidial_list`)
- [ ] **5.** Confirm all campaigns are configured and paused on new system
- [ ] **6.** Test outbound calling on each carrier with test numbers
- [ ] **7.** Test inbound routing on each DID with test calls
- [ ] **8.** Verify recording is working (place test call, confirm file exists)
- [ ] **9.** Confirm CRM integration URLs are updated for new server
- [ ] **10.** Distribute new login credentials to all agents

### Cutover Morning

- [ ] **11.** Stop dialing on old system (pause all campaigns)
- [ ] **12.** Wait for all active calls to complete on old system
- [ ] **13.** Run final disposition sync (any calls completed after last sync)
- [ ] **14.** Export callback records from old system and import to new
- [ ] **15.** Verify callback import (`SELECT COUNT(*) FROM vicidial_callbacks WHERE status = 'ACTIVE'`)
- [ ] **16.** Start agent logins on new system
- [ ] **17.** Activate campaigns on new system one at a time
- [ ] **18.** Monitor first 10 calls per campaign for correct behavior
- [ ] **19.** Verify agents are connecting to live humans (not all AMD)
- [ ] **20.** Check drop rate is under 3% after first 100 calls

### Post-Cutover Verification (First 2 Hours)

- [ ] **21.** Verify real-time reporting is populating on new system
- [ ] **22.** Confirm recordings are being saved (`ls -lt /var/spool/asterisk/monitor/$(date +%Y/%m/%d)/`)
- [ ] **23.** Test transfer functionality (blind and warm)
- [ ] **24.** Verify caller ID display is correct on test callbacks
- [ ] **25.** Confirm DNC checking is working (attempt to dial a DNC number, verify block)
- [ ] **26.** Test lead API loading (push a test lead, verify it appears in hopper)
- [ ] **27.** Verify cron jobs are running (`grep -i vicidial /var/log/cron`)
- [ ] **28.** Check server load (`top`, `iostat`, `asterisk -rx "core show channels"`)
- [ ] **29.** Confirm old system is fully stopped (no stray calls)
- [ ] **30.** Send migration-complete notification to all stakeholders

## Phase 7: Post-Migration Verification

### 7.1 Day 1 Verification

At end of first full day, compare to baseline:

```sql
-- Compare to pre-migration baseline
SELECT
    COUNT(*) AS total_calls,
    SUM(CASE WHEN status NOT IN ('A','AA','AM','AL','N','NI','NP','DC','DNC','B','NA','ADC','DROP')
        THEN 1 ELSE 0 END) AS contacts,
    ROUND(SUM(CASE WHEN status NOT IN ('A','AA','AM','AL','N','NI','NP','DC','DNC','B','NA','ADC','DROP')
        THEN 1 ELSE 0 END) / NULLIF(COUNT(*), 0) * 100, 2) AS contact_rate,
    ROUND(SUM(CASE WHEN status = 'DROP' THEN 1 ELSE 0 END) / NULLIF(COUNT(*), 0) * 100, 2) AS drop_rate,
    COUNT(DISTINCT user) AS agents_active
FROM vicidial_log
WHERE call_date >= CURDATE();
```

**Acceptable variances on Day 1:**
- Total calls: within 15% of baseline (agents are adjusting)
- Contact rate: within 20% of baseline
- Drop rate: must be under 3% (compliance requirement)
- Agent count: should match expected staffing

### 7.2 Week 1 Daily Checks

Run daily for the first week:

- [ ] Call volumes trending toward baseline
- [ ] Connect rates stabilizing
- [ ] AMD accuracy verified (check for short-duration non-machine dispositions)
- [ ] Recording storage growing at expected rate
- [ ] No carrier issues (check `vicidial_carrier_log` for error patterns)
- [ ] Agent feedback collected and addressed
- [ ] CRM integration confirmed working end-to-end

### 7.3 Day 30 Review

After one month, conduct a formal review:

```sql
-- 30-day post-migration summary
SELECT
    DATE(call_date) AS day,
    COUNT(*) AS calls,
    ROUND(SUM(CASE WHEN status NOT IN ('A','AA','AM','AL','N','NI','NP','DC','DNC','B','NA','ADC','DROP')
        THEN 1 ELSE 0 END) / NULLIF(COUNT(*), 0) * 100, 2) AS contact_rate,
    ROUND(SUM(CASE WHEN status = 'DROP' THEN 1 ELSE 0 END) / NULLIF(COUNT(*), 0) * 100, 2) AS drop_rate,
    ROUND(AVG(length_in_sec), 1) AS avg_duration
FROM vicidial_log
WHERE call_date >= DATE_SUB(CURDATE(), INTERVAL 30 DAY)
GROUP BY DATE(call_date)
ORDER BY day;
```

Compare this to the pre-migration baseline captured in Phase 1. Post-migration performance should meet or exceed pre-migration levels.

## Phase 8: Common Migration Pitfalls

### Pitfall 1: Forgetting Callback Records

Agents schedule callbacks on the old system. If you don't migrate callback records, those leads get abandoned. Always export and import active callbacks as part of cutover.

```sql
-- Export active callbacks from old system
SELECT * FROM vicidial_callbacks
WHERE status = 'ACTIVE'
  AND callback_time > NOW();
```

### Pitfall 2: Time Zone Mismatches

If the new server is in a different time zone, call times, callbacks, and scheduled reports all shift. Set the server timezone to match your operation:

```bash
timedatectl set-timezone America/New_York

# Verify MySQL timezone
mysql -e "SELECT @@global.time_zone, @@session.time_zone, NOW();"
```

### Pitfall 3: DNS Propagation Delays

If agents access the dialer via a hostname (e.g., `dialer.yourcenter.com`), updating DNS to point to the new server takes time to propagate. Options:

- Update DNS 48 hours before cutover with a low TTL (60 seconds)
- Have agents use the new IP directly during transition
- Use `/etc/hosts` overrides on agent workstations

### Pitfall 4: Recording Codec Mismatch

If the new server uses a different audio codec than the old one, recordings may not play back correctly. Ensure codec consistency:

```bash
# Check recording format
grep "MIXMON_FORMAT" /etc/astguiclient.conf
# Should output: MIXMON_FORMAT=wav (or whatever you're using)
```

### Pitfall 5: Firewall Rules

VICIdial requires specific ports to be open. On the new server, verify:

```bash
# Required ports
# 80/443 - Web interface
# 5060 - SIP signaling
# 10000-20000 - RTP media (Asterisk)
# 8089 - WebSocket (if using WebRTC)
# 3306 - MySQL (if remote access needed)

# Check firewall
firewall-cmd --list-ports
# Or: iptables -L -n
```

### Pitfall 6: Cron Job Differences

VICIdial relies heavily on cron jobs for keepalive processes, call logging, and cleanup. Verify all cron entries are present on the new server:

```bash
# Critical VICIdial cron entries that must exist
crontab -l | grep -E "(AST_manager_send|ADMIN_keepalive|AST_cleanup|AST_DB_dead)"

# Missing any? Check the official VICIdial cron template:
cat /usr/src/astguiclient/trunk/extras/cron/crontab-asterisk
```

### Pitfall 7: Not Testing Under Load

A system that works with 5 test agents may crumble with 50 production agents. Before cutover, simulate production load:

```bash
# Check Asterisk channel capacity
asterisk -rx "core show channels count"

# Monitor during peak
watch -n 5 'asterisk -rx "core show channels count" && echo "---" && uptime'
```

### Pitfall 8: Skipping the Rollback Plan

Always have a way to roll back to the old system within 30 minutes. This means:

- Keep the old system running (paused) for at least 48 hours post-cutover
- Don't delete old server data for at least 30 days
- Document the rollback procedure: re-point DNS, activate old campaigns, re-sync leads

## How ViciStack Helps

Migration is where ViciStack's managed service delivers the most obvious value. We've handled over 100 VICIdial migrations and have the process refined to minimize risk and downtime.

**What ViciStack migration includes:**

- **Full pre-migration audit** of your current environment
- **Data migration** with verified lead counts, recordings, callbacks, DNC lists, and custom configurations
- **Carrier coordination** including DID porting management and SIP trunk setup
- **Parallel running support** with daily performance comparisons
- **Cutover management** by experienced engineers who've done this 100+ times
- **Post-migration optimization** that typically produces connect rates 50-100% higher than your pre-migration baseline
- **Zero-downtime guarantee** -- we schedule cutover during off-hours and maintain rollback capability throughout

Most migrations are completed in 2-3 weeks with zero production downtime. Agent training is typically a single 1-hour session because the VICIdial interface they know isn't changing -- just getting faster and better optimized.

**Planning a migration?** Start with a free analysis of your current setup. We'll assess your environment, identify optimization opportunities, and provide a detailed migration timeline specific to your operation.

[Get Your Free Migration Assessment at vicistack.com/proof/](https://vicistack.com/proof/) -- we respond within 5 minutes during business hours.

## Related Articles

- [VICIdial vs CallTools: Detailed Comparison](/blog/vicidial-vs-calltools)
- [VICIdial vs XenCall: Which Dialer Is Right?](/blog/vicidial-vs-xencall)
- [VICIdial Performance Tuning Guide](/blog/vicidial-performance-tuning)
- [VICIdial Server Sizing and Architecture](/blog/vicidial-server-sizing)
- [VICIdial Custom MySQL Reports Guide](/blog/vicidial-custom-mysql-reports)

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-hosted-migration-checklist).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
