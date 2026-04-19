# VICIdial Database Maintenance

**Last updated: March 2026 | Reading time: ~26 minutes**

Your VICIdial server is going to crash. Not might. Will. The only question is whether it crashes at 2 AM on a Saturday when nobody cares, or at 10:30 AM on a Monday when 80 agents are mid-call and your operations manager is standing behind you asking why the phones died.

The difference between those two scenarios is database maintenance. VICIdial generates an absurd amount of data — call logs, agent events, recording metadata, hopper entries, disposition history, carrier logs — and if you don't actively manage that data, your MySQL tables will grow until they corrupt, your disk will fill until the OS stops writing, and your queries will slow until the real-time report takes 45 seconds to load.

I've rebuilt VICIdial databases from crashed systems more times than I want to admit. Every single one was preventable with a maintenance schedule that takes about 20 minutes to set up.

---

## The Tables That Will Kill You

VICIdial's `asterisk` database contains hundreds of tables. Most of them stay small. About a dozen of them grow without bound and will eventually take your server down if left unmanaged.

Here are the ones that matter, ranked by how often they cause problems:

### Tier 1: Will Definitely Crash Your Server

**`vicidial_log`** — One row per outbound call attempt. A 50-agent center doing 300 dials per agent per day generates 15,000 rows daily. After a year, you're looking at 5+ million rows. After three years, 15+ million. Queries against this table slow down every report in the system.

**`vicidial_closer_log`** — Same as vicidial_log but for inbound calls. Smaller in most shops, but blended operations can generate heavy volume.

**`vicidial_agent_log`** — Every agent state change (login, pause, ready, incall, dispo) gets a row. A 50-agent center can generate 50,000-100,000 rows per day depending on call volume and pause behavior. This table bloats fast.

**`call_log`** — Raw Asterisk CDR data. Every call leg gets a row. With predictive dialing, a single customer call might generate 3-5 rows (original dial, agent leg, conference bridge, etc.). Growth rate is 3-5x the number of actual calls.

**`recording_log`** — Metadata for every call recording. The table itself isn't huge, but if your recording file path points to a partition that fills up, MySQL crashes when it can't write to disk.

### Tier 2: Will Slow You Down Before They Kill You

**`vicidial_carrier_log`** — Carrier-level call detail for every trunk interaction. High-volume centers generate 20,000+ rows per day.

**`server_performance`** — System metrics logged every update interval. Not a crisis by itself, but it contributes to total database size.

**`vicidial_dial_log`** — Dialer-level logging. Grows proportionally to dial volume.

**`vicidial_lead_search_log`** — If you use the lead search feature, every query gets logged.

### Tier 3: The Silent Disk Killer

**Call recordings on disk** — Not a database table, but the single most common cause of VICIdial server crashes. A G.711 WAV recording averages 500 KB per minute. A 3-minute average call at 10,000 calls per day = 15 GB per day = 450 GB per month. Your 1 TB drive is full in two months.

---

## The Maintenance Stack

Here's the maintenance schedule we use on every VICIdial system we manage. Copy it, adjust the times to your operation's quiet hours, and forget about it until the monitoring alerts tell you something changed.

### Nightly: Archive Old Log Data

VICIdial ships with a script called `ADMIN_archive_log_tables.pl` that moves rows older than a configurable threshold from active tables into archive tables. The archive tables have identical schemas but no auto-increment indexes, so they don't participate in active queries.

Set this up in crontab on your VICIdial web/database server:

```bash
crontab -e
```

Add this line:

```
20 2 * * * /usr/share/astguiclient/ADMIN_archive_log_tables.pl --daily --carrier-daily 2>&1 >> /var/log/vicidial-archive.log
```

This runs at 2:20 AM every night and archives:
- `vicidial_log` entries older than the configured threshold
- `vicidial_closer_log` entries
- `vicidial_agent_log` entries
- `call_log` entries
- `vicidial_carrier_log` entries
- Various other log tables

By default, the script archives data older than 732 days (2 years). For high-volume centers, I recommend changing this to 90 or 180 days:

```
20 2 * * * /usr/share/astguiclient/ADMIN_archive_log_tables.pl --daily --carrier-daily --days=90 2>&1 >> /var/log/vicidial-archive.log
```

**Why 90 days?** Most compliance requirements (PCI, TCPA) care about having recordings and logs available, not about keeping them in the active production database. Archive tables are still queryable — they're just not in the hot path for real-time operations.

### Nightly: Automated Table Check and Repair

MySQL tables can corrupt from unexpected shutdowns, disk errors, or heavy write load. VICIdial's MyISAM tables (yes, many core tables still use MyISAM, not InnoDB) are especially vulnerable to corruption from unclean shutdowns.

Add a nightly check-and-repair cron job:

```
0 3 * * * /usr/bin/mysqlcheck -u root -p'YOUR_PASSWORD' --auto-repair --optimize asterisk 2>&1 >> /var/log/vicidial-mysqlcheck.log
```

This runs at 3:00 AM and does three things:
1. Checks every table in the `asterisk` database for corruption
2. Automatically repairs any corrupted tables it finds
3. Optimizes tables to reclaim fragmented space

**Schedule this after the archive script finishes** (archive runs at 2:20, check runs at 3:00). The archive script moves data out, then the optimize reclaims the freed space.

For systems with a large number of tables, you should run the optimize only on the biggest tables to avoid the full run taking too long:

```bash
#!/bin/bash
# /usr/local/bin/vicidial-optimize.sh
TABLES="vicidial_log vicidial_closer_log vicidial_agent_log call_log vicidial_carrier_log recording_log"
for TABLE in $TABLES; do
 /usr/bin/mysql -u root -p'YOUR_PASSWORD' -e "OPTIMIZE TABLE $TABLE" asterisk >> /var/log/vicidial-optimize.log 2>&1
done
```

Then cron it:

```
0 3 * * * /usr/local/bin/vicidial-optimize.sh
```

### Weekly: Recording Rotation

This is the one that saves you from disk-full crashes. Create a script that moves recordings older than your retention period to archive storage:

```bash
#!/bin/bash
# /usr/local/bin/rotate-recordings.sh
# Move recordings older than 90 days to archive

RECORDING_DIR="/var/spool/asterisk/monitor"
ARCHIVE_DIR="/mnt/archive/recordings"
DAYS_TO_KEEP=90

find "$RECORDING_DIR" -name "*.wav" -mtime +$DAYS_TO_KEEP -exec mv {} "$ARCHIVE_DIR/" \;
find "$RECORDING_DIR" -name "*.mp3" -mtime +$DAYS_TO_KEEP -exec mv {} "$ARCHIVE_DIR/" \;
find "$RECORDING_DIR" -name "*.gsm" -mtime +$DAYS_TO_KEEP -exec mv {} "$ARCHIVE_DIR/" \;

# Log what we moved
echo "$(date): Rotated recordings older than $DAYS_TO_KEEP days" >> /var/log/recording-rotation.log
```

The archive location should be a separate mount point — an NFS share, an S3-compatible object store via s3fs, or a dedicated archive server. Never archive to the same disk that holds your active system.

Cron it weekly:

```
0 4 * * 0 /usr/local/bin/rotate-recordings.sh
```

### Monthly: Full Database Health Check

Once a month, run a full health assessment:

```bash
#!/bin/bash
# /usr/local/bin/vicidial-monthly-health.sh

echo "=== VICIdial Monthly Health Check: $(date) ===" >> /var/log/vicidial-health.log

# Disk usage
echo "--- Disk Usage ---" >> /var/log/vicidial-health.log
df -h >> /var/log/vicidial-health.log

# Top 20 largest tables
echo "--- Top 20 Largest Tables ---" >> /var/log/vicidial-health.log
mysql -u root -p'YOUR_PASSWORD' -e "
SELECT table_name,
 ROUND(data_length/1024/1024, 2) AS data_mb,
 ROUND(index_length/1024/1024, 2) AS index_mb,
 ROUND((data_length + index_length)/1024/1024, 2) AS total_mb,
 table_rows
FROM information_schema.tables
WHERE table_schema = 'asterisk'
ORDER BY (data_length + index_length) DESC
LIMIT 20;
" >> /var/log/vicidial-health.log

# Recording disk usage
echo "--- Recording Storage ---" >> /var/log/vicidial-health.log
du -sh /var/spool/asterisk/monitor/ >> /var/log/vicidial-health.log

# Archive table sizes
echo "--- Archive Table Sizes ---" >> /var/log/vicidial-health.log
mysql -u root -p'YOUR_PASSWORD' -e "
SELECT table_name,
 ROUND((data_length + index_length)/1024/1024, 2) AS total_mb,
 table_rows
FROM information_schema.tables
WHERE table_schema = 'asterisk' AND table_name LIKE '%_archive'
ORDER BY (data_length + index_length) DESC
LIMIT 10;
" >> /var/log/vicidial-health.log

echo "=== End Health Check ===" >> /var/log/vicidial-health.log
```

---

## The Real-World Numbers

Here's what database growth looks like at different scales, based on systems we've managed:

### 20-Agent Center (Low Volume)
- Daily dial attempts: ~4,000
- vicidial_log growth: ~4,000 rows/day, ~120 MB/month
- Recording storage: ~3 GB/month (compressed MP3)
- Total database growth: ~500 MB/month
- **Time to crisis without maintenance: 12-18 months**

### 50-Agent Center (Medium Volume)
- Daily dial attempts: ~15,000
- vicidial_log growth: ~15,000 rows/day, ~450 MB/month
- Recording storage: ~15 GB/month (WAV, uncompressed)
- Total database growth: ~2 GB/month
- **Time to crisis without maintenance: 6-8 months**

### 120-Agent Center (High Volume)
- Daily dial attempts: ~48,000
- vicidial_log growth: ~48,000 rows/day, ~1.5 GB/month
- Recording storage: ~45 GB/month
- Total database growth: ~6 GB/month
- **Time to crisis without maintenance: 2-3 months**

That 120-agent center we mentioned in the intro? They had 1.2 TB of unrotated recordings, 18 million rows in vicidial_log, and a vicidial_agent_log table with 42 million rows. The real-time report was taking 30+ seconds to load. MySQL was using 95% of available RAM for buffer pool, and the server was swapping to disk.

After implementing the maintenance schedule above:
- vicidial_log went from 18M to 2.3M active rows (rest in archive)
- Real-time report load time dropped from 30 seconds to under 2 seconds
- Recording storage freed up 900 GB
- MySQL memory usage dropped 40%
- Zero crash events in the following 14 months

---

## MySQL Configuration Tuning for VICIdial

The maintenance schedule handles data growth, but your MySQL configuration also needs attention. Most VICIbox installs ship with reasonable defaults, but if your system has been running for years or you've upgraded hardware, the defaults may be wrong.

### Key my.cnf Settings

Check `/etc/my.cnf` or `/etc/my.cnf.d/` for these settings:

```ini
[mysqld]
# Buffer pool should be 50-70% of total RAM (for dedicated DB server)
# For a 16 GB server, set to 8-10 GB
innodb_buffer_pool_size = 8G

# MyISAM key buffer (VICIdial still uses MyISAM for many tables)
key_buffer_size = 512M

# Query cache (deprecated in MySQL 8.0, useful in 5.7)
query_cache_size = 128M
query_cache_type = 1

# Temp table sizes for large report queries
tmp_table_size = 256M
max_heap_table_size = 256M

# Connection limits
max_connections = 500

# Slow query logging (enable for debugging, disable for production)
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 2
```

### The innodb_buffer_pool_size Rule

This is the single most impactful MySQL tuning parameter. If it's too small, MySQL reads from disk constantly. If it's too large, the OS runs out of RAM and starts swapping, which is worse.

**Rule of thumb**: Set it to 50% of total RAM on a server that also runs Asterisk, or 70% on a dedicated database server.

Check your current hit rate:

```sql
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_read%';
```

Calculate: `hit_rate = 1 - (Innodb_buffer_pool_reads / Innodb_buffer_pool_read_requests)`

If the hit rate is below 99%, your buffer pool is too small.

---

## VICIdial's Built-In Maintenance Scripts

VICIdial ships with several maintenance-related Perl scripts in `/usr/share/astguiclient/`. Most people don't know they exist:

**`ADMIN_archive_log_tables.pl`** — Archives old log data (discussed above)

**`AST_DB_optimize.pl`** — Optimizes MyISAM tables. VICIbox runs this via cron by default:

```
# Check if it's already scheduled
crontab -l | grep optimize
```

If you don't see it, add it:

```
0 4 * * * /usr/share/astguiclient/AST_DB_optimize.pl 2>&1 >> /var/log/vicidial-optimize.log
```

**`AST_reset_mysql_vars.pl`** — Resets MySQL session variables that VICIdial sets. Some deployments run this nightly to prevent variable drift:

```
30 3 * * * /usr/share/astguiclient/AST_reset_mysql_vars.pl 2>&1 >> /var/log/vicidial-reset.log
```

**`AST_cleanup_agent_log.pl`** — Cleans up orphaned agent log entries from agents who disconnected without proper logout. Run this if you notice phantom agents in your real-time report:

```
*/15 * * * * /usr/share/astguiclient/AST_cleanup_agent_log.pl 2>&1 >> /var/log/vicidial-cleanup.log
```

---

## Monitoring: Catching Problems Before They Become Outages

Maintenance schedules prevent most problems. Monitoring catches the rest. Here's what to watch:

### Disk Space Alert

If disk usage crosses 80%, you need to act. At 90%, you're in danger zone. At 95%, MySQL may stop accepting writes.

A simple monitoring script:

```bash
#!/bin/bash
# /usr/local/bin/disk-alert.sh
THRESHOLD=80
USAGE=$(df / | tail -1 | awk '{print $5}' | sed 's/%//')
if [ "$USAGE" -gt "$THRESHOLD" ]; then
 echo "ALERT: Disk usage at ${USAGE}% on $(hostname)" | mail -s "VICIdial Disk Alert" admin@yourcompany.com
fi
```

Cron it every hour:

```
0 * * * * /usr/local/bin/disk-alert.sh
```

### Table Size Monitoring

Create a script that alerts when any table exceeds a size threshold:

```bash
#!/bin/bash
# Alert if any table exceeds 5 GB
THRESHOLD_MB=5000
BIG_TABLES=$(mysql -u root -p'YOUR_PASSWORD' -N -e "
SELECT table_name, ROUND((data_length + index_length)/1024/1024) AS size_mb
FROM information_schema.tables
WHERE table_schema = 'asterisk'
 AND (data_length + index_length)/1024/1024 > $THRESHOLD_MB;
")
if [ -n "$BIG_TABLES" ]; then
 echo "ALERT: Large tables detected on $(hostname):" | mail -s "VICIdial Table Size Alert" admin@yourcompany.com <<< "$BIG_TABLES"
fi
```

### MySQL Process List Check

Long-running queries can lock tables and block VICIdial's real-time operations. Monitor for queries running longer than 60 seconds:

```bash
#!/bin/bash
LONG_QUERIES=$(mysql -u root -p'YOUR_PASSWORD' -N -e "
SELECT id, user, time, state, LEFT(info, 100)
FROM information_schema.processlist
WHERE time > 60 AND command != 'Sleep';
")
if [ -n "$LONG_QUERIES" ]; then
 echo "ALERT: Long-running queries on $(hostname):" | mail -s "VICIdial Query Alert" admin@yourcompany.com <<< "$LONG_QUERIES"
fi
```

---

## The Complete Cron Schedule

Here's the full maintenance crontab, ready to paste:

```
# VICIdial Database Maintenance Schedule
# Times are based on EST, adjust for your timezone

# Every 15 minutes: Clean up orphaned agent sessions
*/15 * * * * /usr/share/astguiclient/AST_cleanup_agent_log.pl 2>&1 >> /var/log/vicidial-cleanup.log

# Every hour: Disk space check
0 * * * * /usr/local/bin/disk-alert.sh

# Nightly 2:20 AM: Archive old log tables
20 2 * * * /usr/share/astguiclient/ADMIN_archive_log_tables.pl --daily --carrier-daily --days=90 2>&1 >> /var/log/vicidial-archive.log

# Nightly 3:00 AM: Table check and repair
0 3 * * * /usr/bin/mysqlcheck -u root -p'YOUR_PASSWORD' --auto-repair asterisk 2>&1 >> /var/log/vicidial-mysqlcheck.log

# Nightly 3:30 AM: Reset MySQL variables
30 3 * * * /usr/share/astguiclient/AST_reset_mysql_vars.pl 2>&1 >> /var/log/vicidial-reset.log

# Nightly 4:00 AM: Optimize hot tables
0 4 * * * /usr/local/bin/vicidial-optimize.sh

# Weekly Sunday 4:00 AM: Rotate old recordings
0 4 * * 0 /usr/local/bin/rotate-recordings.sh

# Monthly 1st at 5:00 AM: Full health check
0 5 1 * * /usr/local/bin/vicidial-monthly-health.sh
```

---

## Recovery: When You Didn't Maintain and Now It's Broken

If you're reading this because your server already crashed, here's the emergency procedure:

### Step 1: Check if MySQL is Running

```bash
systemctl status mariadb
# or
systemctl status mysql
```

If it's stopped, check the error log:

```bash
tail -100 /var/log/mysql/error.log
```

Common errors you'll see:
- `Table './asterisk/vicidial_log' is marked as crashed` — Table corruption
- `Disk full (/tmp/#sql...)` — No disk space for temp tables
- `InnoDB: Unable to create temporary file` — Disk full

### Step 2: Free Disk Space (If Disk Full)

Find the biggest space consumers:

```bash
du -sh /var/spool/asterisk/monitor/
du -sh /var/lib/mysql/asterisk/
du -sh /tmp/
```

If recordings are the problem, move the oldest ones to a temp location immediately:

```bash
mkdir /tmp/old-recordings
find /var/spool/asterisk/monitor/ -name "*.wav" -mtime +30 | head -10000 | xargs mv -t /tmp/old-recordings/
```

Then try starting MySQL:

```bash
systemctl start mariadb
```

### Step 3: Repair Corrupted Tables

Once MySQL is running:

```bash
mysqlcheck -u root -p --auto-repair asterisk
```

If specific tables won't repair, try:

```bash
myisamchk -r /var/lib/mysql/asterisk/vicidial_log.MYI
```

After repair, restart MySQL and verify VICIdial is functional.

### Step 4: Implement the Maintenance Schedule

You just learned why you need it. Set up the cron jobs from this guide. Now.

---

## InnoDB vs MyISAM: The VICIdial Table Engine Question

VICIdial's core tables use a mix of MyISAM and InnoDB. Most of the high-write log tables (vicidial_log, vicidial_agent_log, vicidial_carrier_log) use MyISAM by default in older installations. Newer VICIbox versions have started migrating some tables to InnoDB.

**MyISAM pros for VICIdial**:
- Faster for read-heavy workloads (the real-time report queries)
- Simpler to backup (just copy the .MYD and .MYI files)
- FULLTEXT indexing (used by lead search)

**MyISAM cons**:
- Table-level locking (one write blocks all reads on that table)
- No crash recovery (if MySQL stops unexpectedly, tables corrupt)
- No foreign key support

**InnoDB pros**:
- Row-level locking (concurrent reads and writes)
- Crash recovery (automatic recovery from unclean shutdown)
- Better performance under heavy concurrent write load

For high-volume centers (75+ agents), consider converting the biggest log tables to InnoDB. Check your current table engines:

```sql
SELECT table_name, engine
FROM information_schema.tables
WHERE table_schema = 'asterisk'
 AND table_name LIKE 'vicidial_%'
ORDER BY table_name;
```

Converting a table (do this in a maintenance window, it locks the table during conversion):

```sql
ALTER TABLE vicidial_log ENGINE=InnoDB;
ALTER TABLE vicidial_agent_log ENGINE=InnoDB;
```

After conversion, adjust your `my.cnf` to give InnoDB sufficient buffer pool memory, since it now needs to cache these tables.

---

---

**Related reading:**
- [VICIdial Database Maintenance: The Stuff Nobody Tells You Until Your Server Crashes](https://vicistack.com/blog/vicidial-database-maintenance/)
- [VICIdial Answering Machine Detection vs AI-Based AMD: Which Is Better?](https://vicistack.com/blog/vicidial-amd-vs-ai-amd/)
- [VICIdial Database Partitioning for High-Volume Call Centers](https://vicistack.com/blog/vicidial-database-partitioning/)
- [How AI Is Changing Call Center Quality Control (And Why Most Centers Are Still Stuck in 2015)](https://vicistack.com/blog/ai-call-center-quality-control/)
## When to Call for Help

Database maintenance is boring until it isn't. If you're dealing with:
- A crashed database that won't repair
- Performance degradation you can't diagnose
- A migration from MyISAM to InnoDB
- Setting up replication for high-availability
- Compliance requirements for data retention

That's what [ViciStack](https://vicistack.com/) does. We've recovered databases that other consultants gave up on, and we've built maintenance automation for centers running 200+ agents. The $5K engagement includes a complete database health audit, maintenance automation setup, and [performance tuning](/blog/vicidial-database-partitioning/) that typically cuts query times by 60-80%.

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-database-maintenance).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
