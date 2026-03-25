# VICIdial Disaster Recovery & High Availability Guide

*It's 2:14 AM. Your phone is buzzing. The database server just died — a failed RAID controller, a bad firmware update, doesn't matter. What matters is that 200 agents across three time zones are logging in at 8 AM, your dialer manager is already texting you, and every second of downtime is burning revenue. You open your laptop and realize you don't have a plan.*

*This guide is that plan.*

We've been in that 2 AM scenario more times than we'd like to admit — both on our own infrastructure and rescuing other operators who called us in a panic. We've watched a $40 RAID battery destroy an entire day's production. We've seen a single `DROP TABLE` command typed into the wrong terminal wipe out a campaign mid-shift. And we've seen operators who had proper DR in place recover from a total server loss in under 15 minutes while their agents never even noticed.

The difference between a minor hiccup and a catastrophic outage isn't luck. It's preparation. This guide covers everything: [MySQL replication](/glossary/database-replication/) topologies, Asterisk failover with keepalived, recording backup strategies, RTO/RPO planning for call center operations, automated and manual failover procedures, and the DR runbook template we use in production.

No theory. No "consult your vendor." Real configurations, real procedures, real timelines.

---

## Why VICIdial Disaster Recovery Is Different From Everything Else

VICIdial isn't a stateless web app you can just redeploy from a container image. It's a real-time telephony platform with a centralized MySQL database, multiple Asterisk processes maintaining live call state, recordings actively being written to disk, and agents whose sessions are tracked by timestamp updates every second. Standard IT disaster recovery playbooks don't account for any of this.

Here's what makes VICIdial DR uniquely challenging:

**The database is a single point of failure by design.** As we covered in the [VICIdial Cluster Guide](/blog/vicidial-cluster-guide/), every server in a [VICIdial cluster](/glossary/vicidial-cluster/) connects to one MySQL instance. There's no built-in clustering, no automatic failover, no Galera-style multi-master. If the database goes down, every agent screen freezes, every dial stops, and every call in progress gets orphaned.

**Live call state is ephemeral.** When Asterisk crashes or a telephony server reboots, every active call on that server drops instantly. There's no way to "resume" a call — the RTP streams, the channel state, the conference bridges are all gone. Your DR plan needs to account for this reality rather than pretending you can achieve zero-call-drop failover.

**Recordings are write-once, read-forever.** Call recordings have legal retention requirements (often 3-7 years depending on your industry and jurisdiction). Losing recordings isn't just an operational problem — it's a compliance liability that can result in fines.

**Time sensitivity is extreme.** A SaaS app can tolerate 30 minutes of downtime and nobody dies. A 200-agent call center losing 30 minutes of production at peak hours loses thousands of dollars in revenue and potentially hundreds of leads that will never be contactable again. Your recovery targets need to reflect this.

---

## RTO and RPO: Setting Recovery Targets That Actually Make Sense

Before you configure anything, you need two numbers. Everything else flows from them.

**Recovery Time Objective (RTO):** How long can your call center be completely down before the business impact becomes unacceptable? This isn't a technical question — it's a business question. Talk to your operations manager, not your sysadmin.

**Recovery Point Objective (RPO):** How much data can you afford to lose? If the database dies right now, what's the oldest acceptable backup? Losing the last 5 minutes of dispositions? The last hour? The last 24 hours?

Here's the framework we use with our clients:

| Scenario | Typical RTO | Typical RPO | DR Strategy |
|----------|-------------|-------------|-------------|
| Small operation (< 50 agents), single shift | 2-4 hours | 1 hour | Nightly backups + manual restore |
| Mid-size (50-200 agents), multi-shift | 15-30 minutes | 5 minutes | MySQL replication + keepalived failover |
| Large (200-500 agents), 16+ hours/day | < 5 minutes | Near-zero | Master-master replication + automated failover + hot standby |
| Enterprise / 24x7 (500+ agents) | < 1 minute | Zero | Multi-cluster with CCC + geographic redundancy |

**The money math that gets executives to fund DR:** A 200-agent outbound operation running predictive at $15/hour/agent (blended cost) loses $3,000 for every hour of downtime. A 4-hour outage costs $12,000 in direct labor waste alone — not counting lost leads, missed SLAs, or the agents who quit because they sat around doing nothing. A proper DR setup costs $100-300/month in additional infrastructure. The payback period is literally one incident.

---

## MySQL Replication: The Foundation of Everything

Your database is the single most important component to protect. If you lose a telephony server, 20 agents have a bad day. If you lose the database without a replica, your entire operation stops and you're restoring from backup — assuming you have one.

### Master-Slave Replication: The Starting Point

Master-slave (or primary-replica, if you prefer) replication is the minimum viable DR strategy for any VICIdial deployment above 30 agents. One MySQL instance (the master) processes all writes and streams its binary log to one or more slave instances that replay those writes in near-real-time.

**On the master (`/etc/my.cnf`):**

```ini
[mysqld]
server-id = 1
log-bin = /var/log/mysql/mysql-bin
binlog-format = MIXED
expire_logs_days = 7
max_binlog_size = 256M
sync_binlog = 1

# Include all VICIdial databases
binlog-do-db = asterisk
```

**On the slave (`/etc/my.cnf`):**

```ini
[mysqld]
server-id = 2
relay-log = /var/log/mysql/mysql-relay
read_only = 1
log-slave-updates = 1

# Replicate all VICIdial tables
replicate-do-db = asterisk
```

**Setting up replication from scratch:**

```bash
# On the master — lock tables and grab position
mysql -e "FLUSH TABLES WITH READ LOCK;"
mysql -e "SHOW MASTER STATUS\G"
# Note the File and Position values

# Dump the database (with master data for position)
mysqldump -u root --master-data=2 --single-transaction asterisk > /tmp/vicidial_dump.sql

mysql -e "UNLOCK TABLES;"

# Transfer dump to slave
rsync -avz /tmp/vicidial_dump.sql slave_ip:/tmp/

# On the slave — import and configure
mysql -e "CREATE DATABASE IF NOT EXISTS asterisk;"
mysql asterisk < /tmp/vicidial_dump.sql

mysql -e "CHANGE MASTER TO
  MASTER_HOST='master_ip',
  MASTER_USER='repl_user',
  MASTER_PASSWORD='strong_password_here',
  MASTER_LOG_FILE='mysql-bin.000001',
  MASTER_LOG_POS=12345;"

mysql -e "START SLAVE;"

# Verify it's working
mysql -e "SHOW SLAVE STATUS\G" | grep -E "Slave_IO_Running|Slave_SQL_Running|Seconds_Behind_Master"
```

You want to see `Slave_IO_Running: Yes`, `Slave_SQL_Running: Yes`, and `Seconds_Behind_Master: 0` (or close to it). If `Seconds_Behind_Master` is climbing, your slave hardware can't keep up with write volume — usually a disk I/O bottleneck.

**The replication user setup (do this on the master):**

```sql
CREATE USER 'repl_user'@'slave_ip' IDENTIFIED BY 'strong_password_here';
GRANT REPLICATION SLAVE ON *.* TO 'repl_user'@'slave_ip';
FLUSH PRIVILEGES;
```

**What master-slave gives you:** A warm standby database that's seconds behind production. If the master dies, you promote the slave to master, repoint your cluster's `$VARDB_server` in every server's `astguiclient.conf`, and restart services. Total downtime: 5-15 minutes with a practiced procedure, 30-60 minutes if you're fumbling through it for the first time.

**What master-slave doesn't give you:** Automatic failover. Someone (or something) has to detect the failure, promote the slave, and reconfigure every cluster node. During that window, everything is dead.

### Master-Master Replication: Active-Active for the Brave

Master-master replication means both MySQL instances accept writes simultaneously. Each server is both a master and a slave of the other. This enables automated failover because either server can serve as the primary at any time.

**This is significantly more dangerous than master-slave.** Write conflicts, auto-increment collisions, and split-brain scenarios are all real risks. VICIdial's MyISAM tables don't support transactions, which means there's no rollback safety net if replication breaks.

That said, master-master with proper configuration is the standard approach for production VICIdial HA deployments. The key is ensuring only one master receives writes at any given time — the second master is a hot standby that can accept writes instantly if the primary fails.

**Master 1 (`/etc/my.cnf`):**

```ini
[mysqld]
server-id = 1
log-bin = /var/log/mysql/mysql-bin
binlog-format = MIXED
relay-log = /var/log/mysql/mysql-relay
log-slave-updates = 1
expire_logs_days = 7

# Prevent auto-increment collisions
auto_increment_increment = 2
auto_increment_offset = 1

binlog-do-db = asterisk
replicate-do-db = asterisk
```

**Master 2 (`/etc/my.cnf`):**

```ini
[mysqld]
server-id = 2
log-bin = /var/log/mysql/mysql-bin
binlog-format = MIXED
relay-log = /var/log/mysql/mysql-relay
log-slave-updates = 1
expire_logs_days = 7

# Offset auto-increments to avoid collisions
auto_increment_increment = 2
auto_increment_offset = 2

binlog-do-db = asterisk
replicate-do-db = asterisk
```

The `auto_increment_increment` and `auto_increment_offset` settings ensure Master 1 generates odd-numbered IDs (1, 3, 5...) and Master 2 generates even-numbered IDs (2, 4, 6...). This eliminates the most common source of replication conflicts in VICIdial — `vicidial_log`, `call_log`, and other auto-increment tables.

**Critical: Configure both servers as slaves of each other.** Run the `CHANGE MASTER TO` command on each server pointing to the other, using the same procedure as master-slave but in both directions.

---

## Automated Failover with Keepalived

Manual failover means waking someone up at 2 AM, SSH-ing into servers, running commands, and hoping they don't fat-finger something critical. Automated failover with keepalived means the system detects the failure, promotes the standby, and repoints the cluster — all within seconds.

[Keepalived](http://www.keepalived.org/) uses VRRP (Virtual Router Redundancy Protocol) to manage a floating IP address — a [virtual IP (VIP)](/glossary/load-balancing/) that automatically moves from the primary server to the backup if the primary becomes unreachable.

Here's how it works in a VICIdial context: your cluster nodes don't connect to the database server's real IP. They connect to a VIP. Keepalived runs on both the primary and standby database servers. If the primary dies, the VIP moves to the standby within 1-3 seconds. Every cluster node's existing MySQL connections fail and reconnect — to the same VIP, which now resolves to the standby. Total interruption: under 10 seconds.

### Keepalived Configuration

**On the primary database server (`/etc/keepalived/keepalived.conf`):**

```
vrrp_script chk_mysql {
    script "/usr/local/bin/check_mysql.sh"
    interval 2
    weight -20
    fall 3
    rise 2
}

vrrp_instance VI_MYSQL {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass your_secret_here
    }

    virtual_ipaddress {
        192.168.1.100/24
    }

    track_script {
        chk_mysql
    }

    notify_master "/usr/local/bin/failover_promote.sh"
    notify_backup "/usr/local/bin/failover_demote.sh"
}
```

**On the standby database server — same config but:**

```
    state BACKUP
    priority 90
```

### The MySQL Health Check Script

This is what keepalived runs every 2 seconds to determine if MySQL is alive and healthy:

```bash
#!/bin/bash
# /usr/local/bin/check_mysql.sh

# Check if MySQL process is running
if ! systemctl is-active --quiet mariadb; then
    exit 1
fi

# Check if MySQL is accepting connections and responding
if ! mysqladmin ping --connect-timeout=3 &>/dev/null; then
    exit 1
fi

# Check if replication is healthy (on slave/standby)
SLAVE_STATUS=$(mysql -N -e "SHOW SLAVE STATUS\G" 2>/dev/null)
if [ -n "$SLAVE_STATUS" ]; then
    SQL_RUNNING=$(echo "$SLAVE_STATUS" | grep "Slave_SQL_Running:" | awk '{print $2}')
    if [ "$SQL_RUNNING" != "Yes" ]; then
        exit 1
    fi
fi

exit 0
```

Make it executable: `chmod +x /usr/local/bin/check_mysql.sh`

The `fall 3` parameter means the primary must fail three consecutive checks (6 seconds) before failover triggers. This prevents false positives from momentary MySQL stalls during heavy queries or table locks — something that happens regularly in VICIdial during report generation and log archiving.

### The Promotion Script

When the standby takes over the VIP, it needs to stop replication and prepare to accept writes:

```bash
#!/bin/bash
# /usr/local/bin/failover_promote.sh
# Called by keepalived when this node becomes MASTER

LOGFILE="/var/log/vicidial_failover.log"
echo "$(date '+%Y-%m-%d %H:%M:%S') FAILOVER: Promoting to master" >> $LOGFILE

# Stop slave replication — this server is now the primary
mysql -e "STOP SLAVE;" 2>/dev/null
mysql -e "RESET SLAVE ALL;" 2>/dev/null

# Remove read-only restriction if set
mysql -e "SET GLOBAL read_only = 0;"

# Log the current binary log position for later resync
mysql -e "SHOW MASTER STATUS\G" >> $LOGFILE

echo "$(date '+%Y-%m-%d %H:%M:%S') FAILOVER: Promotion complete" >> $LOGFILE

# Send alert
echo "VICIdial DB failover occurred at $(date). This server is now the primary." | \
    mail -s "CRITICAL: VICIdial Database Failover" ops@yourdomain.com
```

**What your cluster nodes need:** Every server's `/etc/astguiclient.conf` should have `$VARDB_server` set to the VIP address (`192.168.1.100` in this example), not the real IP of either database server. Same for any MySQL connection strings in custom scripts, API integrations, or cron jobs. Miss one and that component will break during failover while everything else works — maddening to debug under pressure.

> **Your 2 AM Failover Shouldn't Depend on Your Typing Accuracy.**
> ViciStack clusters ship with keepalived, automated promotion, and alerting pre-configured. The database fails over before your phone finishes buzzing. [Get Your Free DR Assessment -->](/free-audit/)

---

## Asterisk Telephony Server Failover

Database failover gets all the attention, but telephony server failures are actually more common. Hard drives die, kernel panics happen, and Asterisk has its own collection of segfaults and memory leaks that can take a server down mid-shift.

The good news: VICIdial's architecture already handles telephony failover better than most people realize.

### How VICIdial Naturally Handles Dialer Loss

When a telephony server drops out of a [cluster](/glossary/vicidial-cluster/), here's what happens automatically:

1. The keepalive processes on that server stop updating their timestamps in the `server_updater` table.
2. Within 30-60 seconds, VICIdial's `AST_VDadapt.pl` (running on your primary dialer) notices the missing server and stops assigning new calls to it.
3. Agents who were on that server get a "time synchronization" error and their calls drop.
4. The adaptive algorithm redistributes dial capacity across remaining servers.

**What this means practically:** If you have 10 telephony servers and one dies, you lose 10% of your agent capacity until it comes back. The other 90% of agents continue dialing without interruption. This is inherent to VICIdial's cluster architecture — no additional configuration required.

### The Critical Exception: The Primary Dialer

As covered in the [cluster guide](/blog/vicidial-cluster-guide/), exactly one telephony server runs keepalive flags 5 and 7 — the adaptive predictive algorithm and the fill dialer. If that specific server dies, your entire cluster stops auto-dialing within minutes because there's nothing calculating dial levels.

**The fix: A scripted primary dialer failover.**

```bash
#!/bin/bash
# /usr/local/bin/promote_primary_dialer.sh
# Run this on the backup dialer when the primary dialer is confirmed down

PRIMARY_IP="10.0.0.11"
BACKUP_IP="10.0.0.12"
DB_HOST="192.168.1.100"  # The VIP

# Verify the primary is actually down
if ping -c 3 -W 2 $PRIMARY_IP &>/dev/null; then
    echo "Primary dialer is still responding. Aborting."
    exit 1
fi

# Update this server's keepalive flags to include 5, 7, and 9
sed -i 's/^VARactive_keepalives.*/VARactive_keepalives => 123456789/' /etc/astguiclient.conf

# Restart keepalive processes to pick up new flags
killall -9 AST_VDadapt.pl AST_VDauto_dial_FILL.pl 2>/dev/null
/usr/share/astguiclient/ADMIN_keepalive_ALL.pl --force-run

echo "This server is now the primary dialer. Flags 5/7/9 activated."
echo "REMEMBER: When the original primary comes back, demote it to secondary flags (12368)"
```

**Do not automate this without a human check.** Unlike database failover where a VIP makes the switch transparent, promoting a new primary dialer while the old one is still running (just unreachable from your monitoring point) creates the duplicate-dialing scenario described in the cluster guide. A network partition between your monitoring server and the primary dialer could trick automation into promoting a backup while the original is still actively dialing. That means two servers running flag 5 simultaneously — exactly the scenario that causes double-dialed leads and erratic dial levels.

### Keeping a Hot Spare Telephony Server

For operations where every agent-minute counts, keep one telephony server running with VICIdial installed, configured, and registered in the `servers` table — but with no agents assigned and keepalive flags set to `12368`. This server is warmed up and ready. If any telephony server fails, reassign its agents to the spare through the admin interface. Time to recovery: under 2 minutes.

The spare doesn't need to be identical hardware. Even a modest 4-core server can absorb the 15-20 agents from one failed dialer while you bring the original back up.

---

## Recording Backup Strategies

Recordings are simultaneously the most storage-intensive and most legally sensitive data in your VICIdial environment. A 200-agent operation generates roughly 1-2 TB of recordings per month. Losing them can mean regulatory fines, lost evidence in disputes, and compliance audit failures.

### The Three-Tier Recording Strategy

**Tier 1: Local disk on telephony servers (hours)**

Recordings start here. Asterisk writes raw audio to `/var/spool/asterisk/monitor/` on the telephony server handling the call. The VICIdial audio pipeline processes and compresses these files every 3 minutes and stages them for transfer. Local storage is a buffer, not a backup — treat anything sitting only on a telephony server's local disk as temporary.

**Tier 2: Archive server or NFS share (weeks to months)**

The FTP pipeline (as described in the [cluster guide](/blog/vicidial-cluster-guide/)) moves processed recordings to the archive server. This is your primary working store — the web interface pulls recordings from here for playback. For clusters, use a dedicated archive server with large spinning disks (HDDs are fine for sequential read workloads). Mount an NFS share from the archive server to `/var/spool/asterisk/monitorDONE/RECORDINGS/` on each web server to simplify playback URL resolution.

**NFS mount example (on each web server's `/etc/fstab`):**

```
archive_ip:/var/recordings  /var/spool/asterisk/monitorDONE/RECORDINGS  nfs  defaults,soft,timeo=15  0  0
```

**Tier 3: Offsite/cloud storage (years)**

This is your true backup — geographically separate, survives a total site failure. Options:

**S3 or S3-compatible storage** is the most cost-effective approach for long-term recording retention:

```bash
#!/bin/bash
# /usr/local/bin/sync_recordings_s3.sh
# Cron: 0 3 * * * /usr/local/bin/sync_recordings_s3.sh

RECORDING_DIR="/var/recordings"
S3_BUCKET="s3://your-vicidial-recordings"
DAYS_OLD=1  # Only sync recordings older than 1 day (fully processed)

find $RECORDING_DIR -name "*.mp3" -mtime +$DAYS_OLD -print0 | \
    xargs -0 -P 4 -I {} aws s3 cp {} $S3_BUCKET/$(date -r {} +%Y/%m/%d)/ \
    --storage-class STANDARD_IA \
    --quiet

# Optional: delete local copies older than 90 days to free disk space
# find $RECORDING_DIR -name "*.mp3" -mtime +90 -delete
```

Using `STANDARD_IA` (Infrequent Access) storage class cuts costs roughly in half compared to standard S3 — recordings older than a day are rarely accessed but need to be retrievable. For recordings older than a year, consider S3 Glacier Deep Archive at ~$1/TB/month.

**Cost reference for S3 recording backup:**

| Agent Count | Monthly Recordings | S3 Standard IA | S3 Glacier Deep Archive |
|-------------|-------------------|----------------|------------------------|
| 50 agents | ~250 GB | ~$3/month | ~$0.25/month |
| 200 agents | ~1.5 TB | ~$19/month | ~$1.50/month |
| 500 agents | ~4 TB | ~$50/month | ~$4/month |

That's the cost of insuring terabytes of legally required recordings against total data loss. There's no rational argument against it.

**rsync to a remote server** works if you have a second physical location or a cheap VPS:

```bash
# Cron: 0 */6 * * * (every 6 hours)
rsync -avz --bwlimit=50000 /var/recordings/ backup_user@remote_ip:/backup/recordings/
```

The `--bwlimit=50000` caps transfer at 50 MB/s to prevent the backup from saturating your production network during business hours. Adjust based on your bandwidth.

---

## Database Backup and Restore

Replication protects you from hardware failure. Backups protect you from everything else — accidental deletes, corrupted tables, bad application upgrades, ransomware. Replication is not a backup. If someone runs `DELETE FROM vicidial_list WHERE 1=1` on the master, that delete replicates to the slave instantly. Both copies are now empty.

### Automated Daily Backups

```bash
#!/bin/bash
# /usr/local/bin/vicidial_backup.sh
# Cron: 0 2 * * * /usr/local/bin/vicidial_backup.sh

BACKUP_DIR="/backup/mysql"
DATE=$(date +%Y%m%d_%H%M%S)
RETENTION_DAYS=14
DB_NAME="asterisk"

mkdir -p $BACKUP_DIR

# Full dump with routines, triggers, and events
mysqldump -u root \
    --single-transaction \
    --routines \
    --triggers \
    --events \
    --master-data=2 \
    --flush-logs \
    $DB_NAME | gzip > $BACKUP_DIR/vicidial_${DATE}.sql.gz

# Verify the backup isn't zero-size or corrupt
BACKUP_SIZE=$(stat -f%z "$BACKUP_DIR/vicidial_${DATE}.sql.gz" 2>/dev/null || stat -c%s "$BACKUP_DIR/vicidial_${DATE}.sql.gz")
if [ "$BACKUP_SIZE" -lt 1000000 ]; then
    echo "WARNING: Backup file suspiciously small ($BACKUP_SIZE bytes)" | \
        mail -s "VICIdial Backup Warning" ops@yourdomain.com
    exit 1
fi

# Clean up backups older than retention period
find $BACKUP_DIR -name "vicidial_*.sql.gz" -mtime +$RETENTION_DAYS -delete

# Sync to offsite
rsync -az $BACKUP_DIR/vicidial_${DATE}.sql.gz backup_user@remote_ip:/offsite_backup/mysql/

echo "Backup completed: vicidial_${DATE}.sql.gz ($BACKUP_SIZE bytes)"
```

**The `--master-data=2` flag** embeds the binary log position as a comment in the dump file. This is essential — if you need to restore from backup and then replay binary logs to a specific point in time, you need to know exactly where the backup's snapshot ends and the binary logs should begin.

### Point-in-Time Recovery (PITR)

This is what separates "we lost a day of data" from "we lost 30 seconds of data." PITR combines a full backup with binary log replay to restore the database to any specific moment.

**Prerequisite:** Binary logging must be enabled (it already is if you configured replication).

**Recovery procedure:**

```bash
# 1. Identify the disaster time (e.g., someone dropped the vicidial_list table at 14:32:15)

# 2. Restore the most recent full backup
gunzip < /backup/mysql/vicidial_20260318_020000.sql.gz | mysql asterisk

# 3. Find which binary logs cover the gap between backup and disaster
mysqlbinlog --start-datetime="2026-03-18 02:00:00" \
            --stop-datetime="2026-03-18 14:32:14" \
            /var/log/mysql/mysql-bin.000042 \
            /var/log/mysql/mysql-bin.000043 | mysql asterisk
```

The `--stop-datetime` is set to one second before the disaster. This replays every write that happened between the backup and the bad event, without replaying the bad event itself.

**Store binary logs on a separate physical disk from your data directory.** If the disk with your MySQL data fails, the binary logs survive, and you can do PITR from the last backup plus the surviving binary logs.

### What to Back Up Beyond MySQL

The database gets all the attention, but there are other critical files:

| Path | What It Is | Backup Frequency |
|------|-----------|-----------------|
| `/etc/astguiclient.conf` | Per-server VICIdial config (server IP, DB credentials, keepalive flags) | On change |
| `/etc/asterisk/` | Asterisk configuration (sip.conf, extensions.conf, etc.) | On change |
| `/etc/my.cnf` | MySQL/MariaDB configuration | On change |
| `/etc/keepalived/keepalived.conf` | Failover configuration | On change |
| `/usr/share/astguiclient/` | VICIdial Perl scripts and custom modifications | Weekly |
| `/var/www/html/agc/` | Agent interface customizations | Weekly |
| SSL certificates | WebRTC / HTTPS | On renewal |
| Carrier trunk credentials | SIP trunk authentication | On change |

A simple approach: keep a Git repository with all configuration files. After any change, commit and push to a private remote. If you need to rebuild a server from scratch, the entire configuration history is versioned and available.

```bash
# One-time setup on each server
cd /etc
git init
git add astguiclient.conf asterisk/ my.cnf keepalived/
git commit -m "Initial configuration snapshot"
git remote add origin git@your-git-server:vicidial-configs/server01.git
git push -u origin master

# After any config change
cd /etc && git add -A && git commit -m "Updated sip.conf — added new carrier trunk" && git push
```

---

## The DR Runbook Template

A runbook isn't documentation you read. It's a step-by-step procedure you execute under pressure at 2 AM when your brain is running at 40% capacity. Keep it simple, sequential, and tested.

### Scenario 1: Database Server Total Failure (Automated Failover Active)

**Expected RTO: < 10 seconds (automated) + 5 minutes (verification)**

1. **Keepalived detects failure** — VIP migrates to standby automatically
2. **Promotion script executes** — Standby stops replication, accepts writes
3. **Verify failover occurred:**
   ```bash
   # On the new primary
   ip addr show | grep 192.168.1.100  # Confirm VIP is here
   mysql -e "SHOW SLAVE STATUS\G"      # Should show empty (no longer a slave)
   mysql -e "SELECT COUNT(*) FROM asterisk.vicidial_live_agents;"  # Agents reconnecting
   ```
4. **Monitor agent reconnection** — Agents will see brief "time synchronization" errors, then recover within 30-60 seconds as keepalive scripts reconnect
5. **Notify operations** — "DB failover occurred. Service restored. Investigating root cause."
6. **Investigate and repair original primary** — Do NOT bring it back online until replication is reconfigured. Plugging a stale master back into the cluster without resync causes data divergence.

### Scenario 2: Database Server Failure (No Automated Failover)

**Expected RTO: 15-30 minutes**

1. **Confirm the failure is real** — SSH to the DB server. Check if it's a MySQL crash (restart it) vs. hardware failure (proceed to failover).
   ```bash
   systemctl status mariadb
   # If MySQL is down but server is up:
   systemctl restart mariadb
   # If server is unreachable: proceed to step 2
   ```
2. **Promote the slave:**
   ```bash
   # On the slave
   mysql -e "STOP SLAVE;"
   mysql -e "RESET SLAVE ALL;"
   mysql -e "SET GLOBAL read_only = 0;"
   ```
3. **Update the cluster to point to the new DB:**
   ```bash
   # On EVERY telephony server, web server, and archive server:
   sed -i 's/VARDB_server.*/VARDB_server => NEW_DB_IP/' /etc/astguiclient.conf
   # Restart all VICIdial processes
   /usr/share/astguiclient/ADMIN_keepalive_ALL.pl --force-run
   ```
4. **Restart Apache on web servers:**
   ```bash
   systemctl restart httpd
   ```
5. **Verify agents can log in** and calls are being placed.

### Scenario 3: Telephony Server Failure

**Expected RTO: 2-5 minutes**

1. **Identify affected agents** — Check which agents were assigned to the failed server in Admin > Servers.
2. **If hot spare is available:** Reassign affected agents to the spare server via Admin > Servers > Server Assignment.
3. **If no spare:** Agents redistribute naturally if you have capacity on remaining servers. Temporarily reduce dial ratio to compensate for lost capacity.
4. **If the failed server was the primary dialer (flags 5/7/9):** Run the primary dialer promotion script on your designated backup dialer.
5. **Repair and return:** When the server comes back, start it with secondary flags (`12368`) only. Do NOT start it with the primary flags if another server has been promoted.

### Scenario 4: Total Site Loss (Fire, Power, Network)

**Expected RTO: 1-4 hours (with offsite backups)**

1. **Activate DR site** — If you have a standby environment at a different location, update DNS and carrier trunk routing.
2. **If no DR site:** Provision new servers (cloud dedicated instances for speed), restore database from offsite backup, restore configurations from Git, reconfigure carrier trunks, test and go live.
3. **Recording recovery:** Recordings in S3/offsite are safe. Recordings still in local pipeline on destroyed servers are lost — this is an accepted risk unless you're syncing in real-time.

---

## Testing Your DR Plan

A DR plan that hasn't been tested isn't a plan — it's a hypothesis. And hypotheses have a way of being wrong at the worst possible times.

### Testing Schedule

| Test Type | Frequency | Duration | Impact |
|-----------|-----------|----------|--------|
| Backup restore verification | Monthly | 30 min | None (test on separate instance) |
| Replication failover drill | Quarterly | 15 min | Brief agent interruption (5-10 sec) |
| Full DR simulation | Semi-annually | 2-4 hours | Scheduled maintenance window |
| Primary dialer failover | Quarterly | 5 min | Minor dial-level fluctuation |

### The Monthly Backup Test

This is non-negotiable. Every month, restore your latest backup to a separate MySQL instance and verify the data is intact:

```bash
# Spin up a test instance (or use a dev server)
gunzip < /backup/mysql/vicidial_latest.sql.gz | mysql -h test_server asterisk_test

# Verify table counts match production
mysql -h test_server -e "SELECT COUNT(*) FROM asterisk_test.vicidial_list;"
mysql -h production_vip -e "SELECT COUNT(*) FROM asterisk.vicidial_list;"

# Verify recent data exists (not a stale backup)
mysql -h test_server -e "SELECT MAX(call_date) FROM asterisk_test.vicidial_log;"
```

If the row counts are wildly different or the most recent `call_date` is from last week, your backup process is broken and you've been flying without a net.

### The Quarterly Failover Drill

Schedule this during a low-volume period. Inform your operations team. Then deliberately kill the primary database and let keepalived do its thing:

```bash
# On the primary DB server
systemctl stop mariadb
# Watch the VIP migrate (should take 3-6 seconds)
# Monitor agent screens for recovery (should take < 60 seconds)
# After verification, restart MySQL and reconfigure replication
```

Time every step. Write down what went wrong. The first drill always reveals something — a script with a hardcoded IP instead of the VIP, a cron job that doesn't reconnect, a monitoring check that fires false alerts.

> **When Was the Last Time You Tested Your Backups?**
> If the answer is "never" or "I'm not sure," you're one bad drive away from a very bad week. ViciStack runs automated DR drills and backup verification as part of every managed cluster. [Talk to Us Before the Emergency -->](/free-audit/)

---

## Monitoring for DR Readiness

Your DR plan is only as good as your ability to detect failures before agents start complaining. Here's what to monitor:

### Critical Alerts (Page someone immediately)

```bash
# MySQL replication lag > 30 seconds
mysql -e "SHOW SLAVE STATUS\G" | awk '/Seconds_Behind_Master/{if($2>30) print "CRITICAL: Replication lag "$2"s"}'

# Replication broken
mysql -e "SHOW SLAVE STATUS\G" | awk '/Slave_SQL_Running:/{if($2!="Yes") print "CRITICAL: SQL replication stopped"}'

# Keepalived state change
# (Monitor /var/log/messages or syslog for VRRP state transitions)

# Backup age > 36 hours
find /backup/mysql -name "vicidial_*.sql.gz" -mtime -1.5 | wc -l
# If 0: backup hasn't run
```

### Warning Alerts (Investigate within the hour)

```bash
# Disk space on archive server < 20%
df -h /var/recordings | awk 'NR==2{gsub(/%/,""); if($5>80) print "WARNING: Recording disk "$5"% full"}'

# Binary log disk space
du -sh /var/log/mysql/

# Failed recording FTP transfers (files accumulating on dialers)
find /var/spool/asterisk/monitorDONE/FTP/ -mmin +60 | wc -l
# If > 0: FTP pipeline is backed up
```

### The Dashboard Query

Run this on your database to get a one-glance DR health check:

```sql
SELECT
    'Replication Status' AS metric,
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS
     WHERE VARIABLE_NAME = 'Slave_running') AS value
UNION ALL
SELECT
    'Seconds Behind Master',
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS
     WHERE VARIABLE_NAME = 'Seconds_Behind_Master')
UNION ALL
SELECT
    'Last Backup Age (hours)',
    CAST(TIMESTAMPDIFF(HOUR,
        (SELECT MAX(call_date) FROM vicidial_log),
        NOW()) AS CHAR)
UNION ALL
SELECT
    'Active Servers',
    CAST(COUNT(*) AS CHAR)
FROM server_updater
WHERE last_update > NOW() - INTERVAL 2 MINUTE;
```

---

## Geographic Redundancy: The Multi-Site Architecture

For operations that genuinely cannot tolerate a site-level failure — 24/7 call centers, regulated industries, operations spanning multiple countries — geographic redundancy is the answer. This is the most complex and expensive DR strategy, but it's the only one that survives a datacenter fire, a regional ISP outage, or a natural disaster.

### The Proven Approach: Active-Passive Sites

**Site A (Primary):** Full VICIdial cluster — database, telephony servers, web servers, archive.
**Site B (DR):** Minimal cluster — one database slave (replicating from Site A over VPN), one or two pre-configured telephony servers, one web server.

MySQL replication runs continuously over an encrypted VPN tunnel between sites. Site B's servers are installed, configured, and tested — but idle. Carrier trunks are pre-configured at both sites but only active at Site A.

**Failover procedure:**
1. Promote Site B's database slave to master.
2. Activate carrier trunks at Site B (or reroute via carrier portal).
3. Update DNS for the agent web interface to point to Site B.
4. Agents log in at Site B. Operations resume.

**Expected RTO:** 15-30 minutes, dominated by DNS propagation and carrier trunk activation. Use low-TTL DNS records (60-120 seconds) to minimize propagation delay.

**Cost:** Roughly 30-40% of your primary site cost for infrastructure that sits idle 99.9% of the time. For a 200-agent operation, that's an additional $300-500/month. Insurance against a total site loss that would otherwise cost you days of recovery time and tens of thousands in lost revenue.

### Cross-Cluster Communication (CCC) for Active-Active

For the largest operations, CCC allows multiple independent VICIdial clusters to operate simultaneously and transfer calls between them. Each cluster has its own database, its own telephony servers, and its own autonomy. If one cluster fails, the others continue operating independently.

This is the architecture behind the largest confirmed VICIdial deployments — 1,000+ agents across multiple clusters. It's beyond the scope of a DR guide (it's an architectural pattern more than a failover mechanism), but understanding that it exists is important for long-term planning. See the [cluster guide](/blog/vicidial-cluster-guide/) for more details on CCC.

---

## Cloud-Specific DR Considerations

If you're running VICIdial on [cloud infrastructure](/blog/vicidial-cloud-deployment/), you have additional tools available — but also additional pitfalls.

### AWS-Specific

- **RDS for MySQL:** Don't use it for VICIdial's primary database. RDS doesn't support MyISAM MEMORY tables properly, and the latency overhead breaks real-time operations. Use RDS as a read replica for reporting only.
- **EBS Snapshots:** Automate daily EBS snapshots of your database volume. They're incremental, cheap, and fast to restore. This is the easiest backup mechanism on AWS.
- **Cross-Region Replication:** Replicate your database to a different AWS region using standard MySQL replication over VPN. AWS Transit Gateway simplifies the networking.
- **Route 53 Health Checks:** Use Route 53 to automatically failover DNS from your primary site to your DR site based on health check endpoints.

### DigitalOcean/Hetzner

- **Volume Snapshots:** Both platforms support volume snapshots. Schedule them daily.
- **Server Images:** Create a golden image of each server role (database, telephony, web) with VICIdial pre-installed and configured. Spinning up a replacement from an image takes 2-5 minutes vs. 2-5 hours for a from-scratch install.
- **Floating IPs:** DigitalOcean's floating IPs work similarly to keepalived VIPs — they can be reassigned between servers via API in seconds. Use them as your cluster's database endpoint.

### The Universal Cloud DR Advantage

The single biggest advantage of cloud for DR: you can provision replacement servers in minutes. With bare metal, a failed server means ordering hardware, waiting for shipping, and racking it. With cloud, a failed dedicated instance gets replaced with an API call. Keep your server build automated (Ansible, bash scripts, golden images — whatever works) so that "provision a replacement" means running a script, not following a 50-step manual procedure.

---

## Common DR Mistakes (And We've Seen Every One)

**Mistake 1: Treating replication as backup.** We said it above but it bears repeating. Replication protects against hardware failure. It does not protect against human error, software bugs, or malicious actions. A `DROP TABLE` replicates. A ransomware encryption replicates. You need both replication AND offline backups.

**Mistake 2: Never testing the restore.** Backups that can't be restored are worse than no backups — they give you false confidence. We've seen gzipped dump files that were silently truncated because the backup disk filled up mid-write. The file existed, had a reasonable size, and was completely useless.

**Mistake 3: Hardcoding database IPs everywhere.** Every custom script, API endpoint, and cron job should connect to the VIP, not the primary database's real IP. During a failover, you'll discover every place you forgot — each one is a separate outage you have to chase down while agents are waiting.

**Mistake 4: No alerting on replication lag.** Replication breaks silently. The slave just stops applying updates and sits there with stale data. If you don't monitor `Seconds_Behind_Master` and `Slave_SQL_Running`, you won't know your "hot standby" is actually 3 days behind until you need it.

**Mistake 5: Forgetting about carrier trunks.** Your DR database and telephony servers are ready. Great. But your SIP trunks are authenticated by IP address and only your primary site's IPs are whitelisted with the carrier. Calls can't go out. Add your DR site's IPs to every carrier trunk in advance.

**Mistake 6: No documented runbook.** The person who set up DR might not be the person executing the failover at 2 AM. If the procedure lives in one person's head, it dies when that person is on vacation, asleep, or has left the company. Write it down. Print it out. Keep a copy somewhere that doesn't depend on the infrastructure it's meant to recover.

**Mistake 7: Ignoring the recordings.** The database gets all the DR attention. Meanwhile, 3 TB of recordings are sitting on a single archive server with no backup, no RAID, and no offsite copy. When that disk fails, you lose years of legally required call recordings. The S3 sync script above costs literally dollars per month. Set it up.

---

## Where ViciStack Fits In

Everything in this guide is exactly what we implement for our managed clusters. Keepalived with automated database failover. MySQL replication with point-in-time recovery. Recording backup to S3. Monitoring with alerting on replication lag, backup freshness, and disk utilization. Quarterly failover drills. Documented runbooks.

The difference is we do it once, correctly, and maintain it permanently. You don't have to figure out why replication broke at 3 AM, why the promotion script didn't fire, or why the backup was silently failing for two weeks. We already know the answer because we've seen it before.

**ViciStack exists for operators who want enterprise-grade DR without building an ops team to maintain it.** Same architecture described in this guide. Fraction of the effort. Dramatically lower risk than hoping your one sysadmin remembers the failover procedure when the pager goes off.

[Get a free DR assessment for your VICIdial environment -->](/free-audit/)

---

## Frequently Asked Questions

### How much does a VICIdial HA setup cost compared to a single-server deployment?

For a mid-size operation (100-200 agents), adding proper HA adds roughly $150-300/month in infrastructure — primarily the standby database server and offsite backup storage. On Hetzner or OVH bare metal, a replica database server runs $50-85/month, S3 recording backup runs $20-50/month, and monitoring costs are negligible. Compare that to the cost of a single 4-hour outage ($12,000+ in lost productivity for a 200-agent operation) and the payback period is measured in days, not months. The expense of HA is trivial relative to the cost of not having it.

### Can I use Galera Cluster or InnoDB Cluster instead of traditional MySQL replication?

Not with VICIdial in its current form. VICIdial is built exclusively on MyISAM tables, and critical high-frequency tables like `vicidial_live_agents` often get converted to the MEMORY engine for performance. Galera Cluster requires InnoDB. InnoDB Cluster requires InnoDB. Both are fundamentally incompatible with VICIdial's storage engine requirements. Standard MySQL master-slave or master-master replication with MyISAM is the only supported approach. See the [MySQL optimization guide](/blog/vicidial-mysql-optimization/) for more on VICIdial's storage engine constraints.

### What happens to active calls during a database failover?

Active calls in progress will experience a brief disruption — typically 5-15 seconds with keepalived-based automated failover. Here's the sequence: the database goes down, Asterisk continues handling existing audio streams (Asterisk doesn't need the database for active RTP), but VICIdial's Perl scripts lose their MySQL connection and stop updating agent state. When the VIP migrates and MySQL reconnects, the scripts resume. Agents may see a momentary freeze on their screens. Calls that were in the process of being connected (ringing but not yet answered) will likely fail. Calls already in conversation usually survive the brief interruption. New calls won't be placed during the failover window.

### How do I handle MySQL replication breaking with "duplicate key" errors?

This is the most common replication issue in VICIdial environments, usually caused by the `auto_increment_offset` settings being misconfigured or by a brief period where both masters accepted writes simultaneously during a split-brain event. The quick fix is `SET GLOBAL SQL_SLAVE_SKIP_COUNTER = 1; START SLAVE;` to skip the conflicting transaction — but this loses data. The correct fix is to identify the conflicting row, manually resolve it on the slave, and restart replication. For persistent conflicts, verify your `auto_increment_increment` and `auto_increment_offset` settings are correct on both masters. If you're seeing frequent duplicate key errors, you likely had a period of genuine dual-write that created divergent data — the safest recovery is to re-snapshot the master and rebuild the slave from scratch.

### Should I replicate the vicidial_list table? It's enormous.

Yes. `vicidial_list` contains your leads — the core revenue-generating data in your entire operation. Losing it means losing every lead, every disposition, every callback schedule, every DNC status. The table can be tens of millions of rows and many gigabytes, which means initial replication setup takes longer and replication lag may be higher during bulk imports, but excluding it from replication is not an acceptable tradeoff. If replication lag during lead imports is a problem, use `LOAD DATA INFILE` instead of row-by-row inserts — it generates far fewer binary log events. Also consider splitting your lead data across multiple lists and archiving old lists to `_archive` tables to keep the hot table manageable.

### What's the minimum DR setup you'd recommend for a 50-agent operation?

At 50 agents, you don't need the full automated failover stack. Here's the minimum viable DR plan: (1) MySQL master-slave replication to a second server — this can be a smaller/cheaper box since it only needs to handle reads and be promotable in an emergency. (2) Automated daily database backups with offsite copy (rsync to a remote server or upload to S3). (3) Monthly backup restore tests. (4) A written one-page runbook covering "slave promotion" and "who to call." (5) Recording sync to S3. Total additional cost: $60-100/month. This gets your RTO down to 15-30 minutes and your RPO down to seconds (replication lag). It won't auto-failover, but it gives a competent sysadmin everything they need to recover quickly.

### How do I prevent split-brain in a master-master replication setup?

Split-brain — where both masters accept writes simultaneously during a network partition — is the nightmare scenario. Prevention is entirely about the STONITH principle (Shoot The Other Node In The Head): when the standby takes over, it must ensure the old primary is truly dead, not just unreachable. Keepalived's VRRP handles this at the IP level (only one server holds the VIP at a time), but add a fencing mechanism for safety. The promotion script should attempt to shut down MySQL on the old primary via SSH before accepting writes. If that fails (because the old primary is truly unreachable), proceed with promotion but immediately investigate. Also: never set both servers to `state MASTER` in keepalived.conf — one must be `MASTER` with higher priority, the other `BACKUP` with lower priority. The `nopreempt` option prevents the original primary from automatically reclaiming the VIP when it comes back, giving you time to verify data consistency before resyncing.

### How often should I run a full DR drill?

Full DR drills (simulating a complete primary site failure and activating the DR site) should happen semi-annually at minimum, with smaller component tests more frequently. Run backup restore verification monthly — it takes 30 minutes and catches the most common failure mode (silent backup corruption). Run database failover drills quarterly during low-volume windows. Test the primary dialer promotion script quarterly. And every time you make a significant infrastructure change (new server, new carrier, new VICIdial version), run an unscheduled component test to verify DR still works with the new configuration. Document the results of every drill — what worked, what broke, how long each step took. The drill results become your most valuable DR asset because they tell you exactly what your real RTO is, not what you hope it is.

---

*This guide is maintained by the ViciStack team and updated as VICIdial, Asterisk, and best practices evolve. Last updated: March 2026.*

*Running VICIdial without DR is running without a seatbelt. The crash might never come — but if it does, you'll wish you'd spent the 10 minutes it takes to buckle up. Questions this guide didn't cover? Reach out directly — we'll add it.*

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-disaster-recovery).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
