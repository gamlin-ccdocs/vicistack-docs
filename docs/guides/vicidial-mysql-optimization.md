# VICIdial MySQL Optimization: Queries, Indexes & Tuning

*It starts the same way every time. Your VICIdial system runs fine for six months. Then one Monday morning, agents can't log in. Real-time reports take 30 seconds to load. The hopper empties faster than it fills. You check the server — CPU is pegged, but not by Asterisk. It's MySQL, grinding through a query that used to take milliseconds and now takes 14 seconds because your `vicidial_log` table has 47 million rows and nobody set up archiving.*

*This is the MySQL optimization guide that saves your next Monday morning.*

The database is VICIdial's single most critical component. Every agent action, every call event, every disposition, every real-time report, every hopper query, every dial-level calculation — all of it flows through one MySQL (or MariaDB) instance. In a [VICIdial cluster](/blog/vicidial-cluster-guide/), you can add telephony servers for more call capacity and web servers for more agent connections. But you cannot cluster the database. It's a single instance, and when it slows down, everything slows down.

We've optimized MySQL on hundreds of VICIdial deployments, from 10-agent single-server setups to 500+ agent clusters processing millions of calls per month. This guide covers the database internals you actually need to understand: table architecture, growth patterns, buffer tuning, slow query identification, MEMORY table conversions, archival strategies, index optimization, the TIMESTAMP schema migration, corruption recovery, and replication for reporting. We include actual `my.cnf` configurations calibrated for real-world deployment sizes.

---

## Understanding VICIdial's Database Architecture

Before you can optimize VICIdial's MySQL instance, you need to understand how VICIdial uses its database — because it's not like most web applications.

### The Access Pattern Problem

VICIdial's database access patterns are unusual and demanding:

1. **High-frequency writes:** Every second, multiple Perl daemons update agent states, call statuses, and dial metrics across dozens of tables. The `AST_update.pl` process alone performs hundreds of UPDATEs per second in a busy 50-agent campaign.

2. **Constant polling:** Real-time reports don't use WebSockets or push notifications. They poll the database every second. Ten managers with real-time reports open means 10 SELECT queries per second against `vicidial_live_agents` and `vicidial_auto_calls` — on top of everything else.

3. **Mixed workload:** Short, fast transactional queries (agent state updates) run simultaneously with long analytical queries (campaign reports spanning millions of call records). The transactional queries need low latency. The analytical queries need throughput. MySQL has to serve both from the same instance.

4. **Table-level locking:** VICIdial uses MyISAM exclusively. Matt Florell has been unambiguous: "We do not recommend using InnoDB under any circumstances." MyISAM's table-level locks mean that a single slow SELECT on `vicidial_log` blocks every INSERT, UPDATE, and DELETE on that same table until the SELECT completes. This is the fundamental scaling constraint.

### Key Tables and Their Growth Patterns

Understanding which tables grow and how fast tells you where to focus optimization. Here are the tables that matter, with approximate growth rates for a 50-agent outbound operation running 8 hours per day:

| Table | Purpose | Growth Rate | Typical Size at 6 Months |
|-------|---------|-------------|--------------------------|
| `vicidial_log` | Primary call log (one row per call attempt) | ~1M rows/month | 6M rows, ~2 GB |
| `vicidial_log_extended` | Extended call data (carrier info, etc.) | ~1M rows/month | 6M rows, ~3 GB |
| `call_log` | Raw Asterisk CDR data | ~1M rows/month | 6M rows, ~2 GB |
| `vicidial_carrier_log` | Trunk-level call events | ~2-3M rows/month | 15M rows, ~4 GB |
| `vicidial_manager` | AMI command queue/history | **~5-10M rows/month** | 40M+ rows, ~8 GB |
| `recording_log` | Call recording metadata | ~500K rows/month | 3M rows, ~1 GB |
| `vicidial_closer_log` | Inbound/transfer call log | ~200K rows/month | 1.2M rows, ~500 MB |
| `server_performance` | Server stats (1 row/6 sec/server) | ~15K rows/day/server | 2.7M rows/server |

**`vicidial_manager` is the fastest-growing table in most deployments** and the one most often overlooked. It stores every AMI command VICIdial sends to Asterisk — Originate commands for dialing, Hangup commands, Monitor commands for recording. At 50 agents with a 5:1 [dial ratio](/glossary/auto-dial-level/), that's 250 Originate commands per cycle, plus hangups, plus monitors. This table can hit 100 million rows in under a year if not archived.

### MEMORY Tables: VICIdial's Real-Time Engine

Several critical VICIdial tables run in the MEMORY storage engine (also called HEAP). These tables live entirely in RAM — no disk I/O, no table-level lock contention with disk writes. They are how VICIdial achieves real-time performance:

| MEMORY Table | Purpose | Typical Row Count |
|--------------|---------|-------------------|
| `vicidial_live_agents` | Current state of every logged-in agent | 1 row per agent |
| `vicidial_live_inbound_agents` | Inbound-eligible agents | 1 row per closer agent |
| `vicidial_auto_calls` | Every active auto-dial call | 1 row per active call |
| `vicidial_hopper` | Leads queued for dialing | configurable, usually 50-500 |
| `vicidial_live_sip_channels` | Active SIP channels | dynamic |

**Critical behavior:** MEMORY tables lose all data when MySQL restarts. This is by design. When MySQL comes back up, these tables are empty. Agents will need to log out and log back in, and the hopper will refill automatically. This is normal — but it means unexpected MySQL restarts during production hours cause a brief disruption as the system repopulates.

The MEMORY table ceiling is controlled by `max_heap_table_size` in `my.cnf`. If a MEMORY table tries to grow beyond this limit, the INSERT fails with an error that typically manifests as agents unable to log in or calls not being placed. For most deployments, 64 MB is sufficient. For 500+ agent operations, increase to 128-256 MB.

> **Your Database Is Your Bottleneck — Even If You Don't Know It Yet.**
> ViciStack deployments ship with MySQL pre-tuned, MEMORY tables optimized, and archival jobs running from day one. [See What Optimized Looks Like →](/pricing/)

---

## my.cnf Tuning: The Settings That Actually Matter

Out of the box, MySQL/MariaDB is configured for a general-purpose workload on modest hardware. VICIdial's access patterns — high-frequency writes, constant polling, mixed transactional and analytical queries — need specific tuning. Here are the parameters that make a measurable difference, organized by deployment size.

### Universal Settings (Every VICIdial Deployment)

These settings should be applied regardless of size:

```ini
[mysqld]
# THE most important single setting for VICIdial
skip-name-resolve

# Allow VICIdial's connection-heavy architecture
max_connections = 2000

# Disable query cache (more harm than good with VICIdial's write patterns)
query_cache_size = 0
query_cache_type = 0

# Allow concurrent inserts during SELECT on MyISAM tables
concurrent_insert = 2

# Table cache — VICIdial opens hundreds of tables
table_open_cache = 4096

# Temp table limits (affects complex report queries)
tmp_table_size = 64M
max_heap_table_size = 64M

# Connection timeout tuning
wait_timeout = 600
interactive_timeout = 600
connect_timeout = 30

# Thread cache — avoids thread creation overhead for frequent connections
thread_cache_size = 128

# Slow query logging (enable always, analyze periodically)
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow-query.log
long_query_time = 2
```

**The `skip-name-resolve` story:** Without this setting, MySQL performs a reverse DNS lookup on every single incoming connection. In a VICIdial cluster where telephony servers, web servers, and dozens of cron-spawned Perl scripts all maintain constant database connections, those DNS lookups pile up. The symptom is "Too many connections" errors even though `max_connections` is set high enough — because connections are stuck waiting for DNS resolution. This single line has saved more VICIdial installations than any other `my.cnf` change. We covered this in the [cluster guide](/blog/vicidial-cluster-guide/), and we're repeating it here because it's that important.

When you enable `skip-name-resolve`, you must use IP addresses (not hostnames) in MySQL GRANT statements. `GRANT ALL ON *.* TO 'cron'@'10.0.0.5'` works. `GRANT ALL ON *.* TO 'cron'@'dialer1.local'` does not.

### 50-Agent Deployment (Single Server or Small Cluster)

```ini
[mysqld]
# MyISAM key buffer — primary cache for MyISAM index blocks
key_buffer_size = 512M

# Sort buffer for ORDER BY / GROUP BY in reports
sort_buffer_size = 4M
read_buffer_size = 2M
read_rnd_buffer_size = 4M

# MyISAM-specific settings
myisam_sort_buffer_size = 64M
myisam_max_sort_file_size = 10G
myisam_repair_threads = 1

# Binlog (enable if you plan to set up replication later)
# log-bin = mysql-bin
# binlog_format = MIXED
# expire_logs_days = 7

# InnoDB settings — even though VICIdial uses MyISAM,
# MySQL system tables use InnoDB internally
innodb_buffer_pool_size = 256M
innodb_log_file_size = 64M
```

**Total RAM budget:** On a dedicated database server with 16 GB RAM, this configuration uses approximately 1-2 GB for MySQL buffers, leaving ample room for OS cache (which MyISAM leverages heavily for data file caching through the filesystem cache).

### 100-Agent Deployment (Dedicated Database Server)

```ini
[mysqld]
key_buffer_size = 1024M

sort_buffer_size = 8M
read_buffer_size = 4M
read_rnd_buffer_size = 8M
join_buffer_size = 4M

myisam_sort_buffer_size = 128M
myisam_max_sort_file_size = 20G

max_connections = 3000
table_open_cache = 8192

max_heap_table_size = 128M
tmp_table_size = 128M

thread_cache_size = 256

innodb_buffer_pool_size = 512M
innodb_log_file_size = 128M

# Enable binary logging for replication
log-bin = mysql-bin
binlog_format = MIXED
expire_logs_days = 7
sync_binlog = 0    # Async for performance
```

**Total RAM budget:** On a 32 GB dedicated database server, this uses approximately 3-4 GB for MySQL buffers. The operating system will use the remaining ~28 GB for filesystem cache, which directly benefits MyISAM's data file reads.

### 500-Agent Enterprise Deployment

```ini
[mysqld]
key_buffer_size = 4096M

sort_buffer_size = 16M
read_buffer_size = 8M
read_rnd_buffer_size = 16M
join_buffer_size = 8M
bulk_insert_buffer_size = 64M

myisam_sort_buffer_size = 256M
myisam_max_sort_file_size = 50G

max_connections = 6000
table_open_cache = 16384
table_definition_cache = 4096
open_files_limit = 65535

max_heap_table_size = 256M
tmp_table_size = 256M

thread_cache_size = 512

innodb_buffer_pool_size = 2G
innodb_log_file_size = 256M

log-bin = mysql-bin
binlog_format = MIXED
expire_logs_days = 3
sync_binlog = 0

# Per-thread buffers — be careful here, these multiply by max_connections
# 6000 connections × 16M sort_buffer = 96 GB if all allocated simultaneously
# In practice, MySQL allocates these on-demand, not all at once
```

**Critical note on per-thread buffer sizing at scale:** Settings like `sort_buffer_size`, `read_buffer_size`, and `join_buffer_size` are allocated per-thread when needed. With `max_connections = 6000`, the theoretical maximum memory for sort buffers alone would be 6000 × 16 MB = 96 GB. In reality, MySQL allocates these on-demand and most connections don't need large sort buffers simultaneously. But if you see unexpected memory pressure, these per-thread buffers are the first place to look.

**Total RAM budget:** 64 GB dedicated database server. MySQL buffers use approximately 8-10 GB. The remaining 54+ GB acts as filesystem cache for MyISAM data files, which is where the real performance comes from at scale.

### The key_buffer_size vs innodb_buffer_pool_size Question

VICIdial uses MyISAM, so `key_buffer_size` is your primary tuning lever — it caches MyISAM index blocks in memory. `innodb_buffer_pool_size` caches InnoDB data and indexes. Since VICIdial's tables are MyISAM, why set `innodb_buffer_pool_size` at all?

Because MySQL's internal system tables (`mysql.user`, `information_schema`, `performance_schema`) use InnoDB. MariaDB also uses InnoDB for some system catalogs. A small InnoDB buffer pool prevents these internal operations from thrashing disk. Don't set it to zero — just keep it modest (256M-2G depending on total RAM).

The real performance lever for MyISAM data access is the **operating system filesystem cache**. MyISAM stores data in `.MYD` files and indexes in `.MYI` files. `key_buffer_size` caches the `.MYI` index blocks. But the `.MYD` data files are read through the OS filesystem cache. This is why you don't want MySQL to consume all available RAM — the OS needs that memory for file caching. The rule of thumb: MySQL buffers should use no more than 40-50% of total RAM on a dedicated database server, leaving the rest for the OS cache.

---

## Slow Query Identification and Analysis

The slow query log is your single best diagnostic tool for database performance issues. Enable it always — the overhead is negligible compared to the debugging time it saves.

### Enabling the Slow Query Log

```ini
[mysqld]
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow-query.log
long_query_time = 2         # Log queries taking >2 seconds
log_queries_not_using_indexes = 1   # Also log queries missing indexes
```

Setting `long_query_time = 2` catches the genuinely problematic queries without flooding the log. For initial diagnosis on a system with known performance issues, temporarily set it to `0.5` to catch more borderline queries.

### Using pt-query-digest

Percona Toolkit's `pt-query-digest` is the industry standard for slow query analysis. Install it:

```bash
yum install percona-toolkit
```

Analyze your slow query log:

```bash
pt-query-digest /var/log/mysql/slow-query.log > /tmp/slow-query-report.txt
```

The output ranks queries by total execution time, showing you exactly which queries are consuming the most database resources. A typical VICIdial slow query report reveals:

**The usual suspects:**

1. **Report queries on `vicidial_log`** — `SELECT` with date ranges spanning millions of rows, often triggered by managers running "last 90 days" reports
2. **Hopper fills on `vicidial_list`** — The hopper query scans `vicidial_list` looking for callable leads. On a list with 10M+ rows and inadequate indexes, this query can take 10+ seconds
3. **Archive table queries** — If `recording_log` or `vicidial_closer_log` has millions of rows, the recording search and call detail queries slow to a crawl
4. **`vicidial_manager` cleanup** — The periodic DELETE on `vicidial_manager` to remove old entries can lock the table for seconds on a table with 50M+ rows

### The PROCESSLIST Diagnostic

When MySQL is actively struggling, check what's running:

```sql
SHOW FULL PROCESSLIST;
```

Look for:
- Queries in "Locked" state — MyISAM table-level lock contention
- Queries in "Copying to tmp table" — complex queries creating temporary tables on disk
- Queries in "Sorting result" — ORDER BY on large result sets without appropriate indexes
- Multiple connections with identical queries — real-time report polling storms

For a more targeted view:

```sql
SELECT * FROM information_schema.PROCESSLIST
WHERE COMMAND != 'Sleep'
  AND TIME > 2
ORDER BY TIME DESC;
```

This shows only active queries that have been running longer than 2 seconds — the ones actively causing problems.

---

## Essential Indexes VICIdial Sometimes Misses

VICIdial's installation scripts create most necessary indexes, but some indexes that significantly improve performance at scale are either missing or need to be added manually. Here are the ones that matter:

### vicidial_log Indexes

The default `vicidial_log` indexes cover `lead_id` and `uniqueid`, but report queries almost always filter by `call_date`. Add:

```sql
ALTER TABLE vicidial_log ADD INDEX idx_call_date (call_date);
ALTER TABLE vicidial_log ADD INDEX idx_campaign_call_date (campaign_id, call_date);
ALTER TABLE vicidial_log ADD INDEX idx_user_call_date (user, call_date);
```

The composite index on `(campaign_id, call_date)` dramatically improves campaign-specific report queries — the most common analytical pattern in VICIdial.

### vicidial_list Indexes

The hopper fill query is one of the most performance-critical queries in VICIdial. It scans `vicidial_list` for callable leads based on list ID, status, and various filter criteria. The default indexes are often insufficient for large lists:

```sql
ALTER TABLE vicidial_list ADD INDEX idx_list_status (list_id, status);
ALTER TABLE vicidial_list ADD INDEX idx_status_modify (status, modify_date);
ALTER TABLE vicidial_list ADD INDEX idx_phone_number (phone_number);
```

### recording_log Indexes

Recording playback searches use `lead_id` and date ranges:

```sql
ALTER TABLE recording_log ADD INDEX idx_lead_date (lead_id, start_time);
ALTER TABLE recording_log ADD INDEX idx_start_time (start_time);
```

### call_log Indexes

```sql
ALTER TABLE call_log ADD INDEX idx_start_time (start_time);
ALTER TABLE call_log ADD INDEX idx_channel_start (channel, start_time);
```

**Before adding indexes:** Run `SHOW INDEX FROM tablename` to verify the index doesn't already exist. Adding a duplicate index wastes space and slows writes. Also, adding an index to a large table (10M+ rows) locks the table for the duration of the ALTER TABLE operation on MyISAM. Schedule index additions during off-hours.

**Index maintenance:** MyISAM indexes can become fragmented over time, especially on heavily-written tables. Periodically run:

```sql
OPTIMIZE TABLE vicidial_log;
OPTIMIZE TABLE vicidial_list;
```

This rebuilds indexes and reclaims space from deleted rows. It locks the table for the duration — on a 10M-row table, expect 5-30 minutes depending on hardware. Always run during maintenance windows.

> **Stop Guessing at Database Bottlenecks.**
> ViciStack runs `pt-query-digest` analysis on every deployment and adds the indexes your specific workload needs. [Get Your Free Database Audit →](/free-audit/)

---

## Table Archival: The Maintenance Task That Saves Your Cluster

If your VICIdial deployment has been running for more than 90 days without archival, your database is carrying dead weight that's actively degrading performance. Every report query, every table lock, every OPTIMIZE TABLE operation takes longer because it's scanning rows that nobody will ever look at again.

### What to Archive and When

The archival principle is simple: **move historical data out of hot tables into archive tables, keeping only recent data in the tables VICIdial actively queries.**

| Table | Archive After | Reason |
|-------|---------------|--------|
| `vicidial_log` | 90 days | Report queries rarely go back further; 90 days covers quarterly analysis |
| `vicidial_log_extended` | 90 days | Paired with vicidial_log |
| `call_log` | 60 days | Raw CDR data, rarely accessed after initial reporting |
| `vicidial_carrier_log` | 30 days | High volume, low long-term value |
| `vicidial_manager` | 7 days | Fastest-growing table, almost never queried historically |
| `recording_log` | 180 days | Keep metadata longer for compliance; actual recordings stay on disk |
| `server_performance` | 30 days | Monitoring data, useless after analysis window |
| `vicidial_closer_log` | 90 days | Inbound/transfer logs |
| `vicidial_agent_log` | 90 days | Agent session data |

### Using ADMIN_archive_log_tables.pl

VICIdial includes a built-in archival script. Set it up as a daily cron job:

```bash
# Run archival at 1 AM server time, daily
0 1 * * * /usr/share/astguiclient/ADMIN_archive_log_tables.pl --daily 2>&1 >> /var/log/astguiclient/archive.log
```

The `--daily` flag moves records older than 24 hours to `_archive` suffix tables. For most deployments, this is too aggressive — you want to keep 90 days in the hot tables. Use `--days=90` if your VICIdial version supports it, or write a custom archival script:

```bash
#!/bin/bash
# Custom archival script for VICIdial
# Moves records older than 90 days to archive tables

DAYS=90
MYSQL_CMD="mysql -u cron -p'your_password' asterisk"

# Archive vicidial_log
$MYSQL_CMD -e "INSERT INTO vicidial_log_archive SELECT * FROM vicidial_log WHERE call_date < DATE_SUB(NOW(), INTERVAL $DAYS DAY);"
$MYSQL_CMD -e "DELETE FROM vicidial_log WHERE call_date < DATE_SUB(NOW(), INTERVAL $DAYS DAY) LIMIT 100000;"

# Archive call_log
$MYSQL_CMD -e "INSERT INTO call_log_archive SELECT * FROM call_log WHERE start_time < DATE_SUB(NOW(), INTERVAL 60 DAY);"
$MYSQL_CMD -e "DELETE FROM call_log WHERE start_time < DATE_SUB(NOW(), INTERVAL 60 DAY) LIMIT 100000;"

# Archive vicidial_manager (aggressive — 7 days)
$MYSQL_CMD -e "DELETE FROM vicidial_manager WHERE entry_date < DATE_SUB(NOW(), INTERVAL 7 DAY) LIMIT 500000;"
```

**Critical:** Notice the `LIMIT` clause on the DELETE statements. Deleting millions of rows from a MyISAM table in a single operation locks the table for the entire duration. The LIMIT breaks it into smaller chunks. Run the script in a loop or multiple times if needed.

### Creating Archive Tables

Before archiving, the `_archive` tables need to exist. VICIdial creates them automatically in newer SVN revisions, but verify:

```sql
-- Check if archive table exists
SHOW TABLES LIKE 'vicidial_log_archive';

-- If it doesn't exist, create it as a copy of the structure
CREATE TABLE vicidial_log_archive LIKE vicidial_log;

-- Add indexes appropriate for archive queries (different from hot table)
ALTER TABLE vicidial_log_archive ADD INDEX idx_call_date (call_date);
```

### The Nuclear Option: TRUNCATE and Rebuild

If your tables have grown to hundreds of millions of rows and archival takes too long, you may need a more aggressive approach during a maintenance window:

1. Rename the bloated table: `RENAME TABLE vicidial_log TO vicidial_log_old;`
2. Create a fresh table: `CREATE TABLE vicidial_log LIKE vicidial_log_old;`
3. Copy recent data back: `INSERT INTO vicidial_log SELECT * FROM vicidial_log_old WHERE call_date > DATE_SUB(NOW(), INTERVAL 90 DAY);`
4. Verify data: `SELECT COUNT(*) FROM vicidial_log;`
5. Drop or archive the old table: `DROP TABLE vicidial_log_old;` or rename to archive

This approach is faster than DELETE because it builds the new table sequentially rather than performing random deletes. But it requires enough disk space for both tables temporarily, and VICIdial must be stopped (or at least not writing to that table) during the swap.

---

## MEMORY Table Optimization for High-Scale Deployments

For deployments above 150-200 agents, converting high-traffic tables from MyISAM to the MEMORY engine is one of the most impactful single optimizations you can make. We touched on this in the [cluster guide](/blog/vicidial-cluster-guide/), but here's the complete implementation guide.

### Why MEMORY Tables Matter

`vicidial_live_agents` is the table that tracks every logged-in agent's real-time state: current call, pause status, [talk time](/glossary/talk-time/), [wrap time](/glossary/wrap-time/), etc. It gets read by every real-time report, every dialer process, every [AGI](/glossary/agi/) script that routes calls. And it gets written to every time an agent's state changes — which is every few seconds per agent.

On MyISAM, table-level locks on `vicidial_live_agents` create contention. A manager's real-time report running a SELECT locks the table, blocking the dialer's UPDATE that needs to mark an agent as available, which blocks the AGI script trying to route a call to that agent. The result: agents freeze, calls don't get placed, and the system feels sluggish for no apparent reason.

MEMORY engine eliminates disk I/O entirely (the table is in RAM) and uses a hash index structure that provides O(1) lookups. The table-level locking still exists, but the lock duration drops from milliseconds-to-seconds (disk I/O) to microseconds (RAM-only). Forum users who've done this conversion consistently describe the difference as "night and day."

### Tables to Convert

| Table | Convert? | Notes |
|-------|----------|-------|
| `vicidial_live_agents` | **Yes** | Highest impact, most contended |
| `vicidial_live_inbound_agents` | **Yes** | Same access pattern |
| `vicidial_auto_calls` | **Yes** | Tracks active auto-dial calls |
| `vicidial_hopper` | **Yes** | Frequently inserted/deleted |
| `vicidial_live_sip_channels` | **Yes** | High-frequency updates |
| `vicidial_log` | **No** | Too large, grows indefinitely |
| `vicidial_list` | **No** | Too large, persistent data |

### The Conversion Process

MEMORY engine requires fixed-width CHAR fields — it doesn't support VARCHAR. You need to alter the column types before converting:

```sql
-- Step 1: Check current table structure
DESCRIBE vicidial_live_agents;

-- Step 2: Alter VARCHAR columns to CHAR
-- (VICIdial may have already done this in recent SVN revisions)
ALTER TABLE vicidial_live_agents
  MODIFY user CHAR(20) NOT NULL DEFAULT '',
  MODIFY server_ip CHAR(15) NOT NULL DEFAULT '',
  MODIFY campaign_id CHAR(8) NOT NULL DEFAULT '',
  MODIFY status CHAR(10) NOT NULL DEFAULT '';
-- ... repeat for all VARCHAR columns

-- Step 3: Convert to MEMORY engine
ALTER TABLE vicidial_live_agents ENGINE=MEMORY;
```

**Important caveats:**

1. **Fixed-width columns use more memory.** A VARCHAR(100) column storing "hello" uses 5 bytes. A CHAR(100) column storing "hello" uses 100 bytes. This is why you need adequate `max_heap_table_size`.

2. **No TEXT or BLOB columns.** MEMORY tables don't support TEXT or BLOB. If the table has any, you either need to convert them to CHAR (with a length limit) or skip that table.

3. **Data lost on restart.** MEMORY tables are empty after MySQL restarts. VICIdial handles this gracefully — agents log back in and repopulate `vicidial_live_agents`, the hopper refills automatically. But test this behavior before doing a conversion on a production system.

4. **The conversion must survive MySQL restarts.** Add the conversion to a MySQL init script or use an event:

```sql
-- Ensure MEMORY tables are recreated on MySQL restart
-- Add this to a startup script that runs after MySQL is up
-- VICIdial's keepalive scripts typically handle this automatically
```

Recent VICIdial SVN revisions (3800+) include built-in support for MEMORY tables with automatic conversion on startup. Check your VICIdial version before doing manual conversions — you might be duplicating built-in functionality.

---

## The TIMESTAMP Schema Migration

Recent VICIdial SVN updates include a significant schema migration: converting DATETIME columns to TIMESTAMP across 40+ tables. This affects every table that stores date/time data and has important implications for both performance and maintenance.

### What Changed

DATETIME fields store a date and time as a literal string value (`2026-03-18 14:30:00`) and use 8 bytes. TIMESTAMP fields store a Unix timestamp integer (seconds since 1970-01-01) and use 4 bytes. Half the storage, faster comparisons, and automatic timezone handling.

The migration script (`ADMIN_datetime_to_timestamp.pl` or the equivalent SQL migration in newer SVN revisions) alters columns like:

```sql
-- Before
call_date DATETIME NOT NULL DEFAULT '0000-00-00 00:00:00'

-- After
call_date TIMESTAMP NOT NULL DEFAULT '1970-01-01 00:00:01'
```

### Implications for Optimization

1. **Table lock duration during migration:** ALTER TABLE on a multi-million-row MyISAM table locks the table for the duration. On a 50M-row `vicidial_log`, this can take 30+ minutes. Plan accordingly.

2. **Index rebuilds:** Indexes on altered columns get rebuilt during the ALTER TABLE. This is automatic but adds to the lock duration.

3. **Application compatibility:** If you have custom scripts that insert DATETIME-formatted values into these columns, they'll still work — MySQL/MariaDB handles the conversion. But direct comparisons like `WHERE call_date = '2026-03-18 14:30:00'` should be tested.

4. **Timezone awareness:** TIMESTAMP columns are stored in UTC and converted to the session timezone on retrieval. If your VICIdial server and database are in the same timezone (which they should be), this is transparent. But if you have remote reporting connections from different timezones, the displayed times will differ from what you saw with DATETIME.

### Running the Migration

If your SVN update includes the TIMESTAMP migration:

1. **Take a full database backup first.** Not optional.
2. **Schedule during a maintenance window.** The table locks will make VICIdial unusable during the migration.
3. **Run on one table at a time** if doing it manually, so you can monitor progress:

```sql
ALTER TABLE vicidial_log MODIFY call_date TIMESTAMP NOT NULL DEFAULT '1970-01-01 00:00:01';
-- Monitor with: SHOW PROCESSLIST; (look for "copy to tmp table" state)
```

4. **Verify after migration:**

```sql
DESCRIBE vicidial_log;
-- Confirm call_date shows TIMESTAMP type
SELECT call_date FROM vicidial_log LIMIT 5;
-- Confirm dates display correctly
```

---

## Corruption Recovery: When Tables Go Bad

MyISAM tables can corrupt. Power failures, unexpected shutdowns, full disks, hardware failures — all can leave `.MYI` (index) or `.MYD` (data) files in an inconsistent state. VICIdial operators hit this more often than they'd like because VICIdial's constant write activity maximizes the window for corruption during any interruption.

### Symptoms of Table Corruption

- "Table 'tablename' is marked as crashed and should be repaired"
- Queries returning unexpected results or incomplete data
- MySQL refusing to start (if system tables are corrupted)
- `CHECK TABLE tablename` returns errors

### REPAIR TABLE: The First-Line Fix

```sql
-- Check for corruption
CHECK TABLE vicidial_log;

-- Repair the table (locks it during repair)
REPAIR TABLE vicidial_log;

-- If basic repair fails, try extended
REPAIR TABLE vicidial_log EXTENDED;

-- Nuclear option: rebuild from data file, discarding index
REPAIR TABLE vicidial_log USE_FRM;
```

`REPAIR TABLE` locks the table for the duration. On a 10M-row table, expect minutes. On a 100M-row table, expect an hour or more.

### myisamchk: Offline Repair

If MySQL can't start or the table is too corrupted for `REPAIR TABLE`, use `myisamchk` directly on the files. **MySQL must be stopped first:**

```bash
# Stop MySQL
systemctl stop mariadb

# Check the table
myisamchk -c /var/lib/mysql/asterisk/vicidial_log.MYI

# Repair with recovery
myisamchk -r /var/lib/mysql/asterisk/vicidial_log.MYI

# If -r fails, try safe recovery (slower but handles more corruption)
myisamchk -o /var/lib/mysql/asterisk/vicidial_log.MYI

# Start MySQL
systemctl start mariadb
```

### InnoDB Force Recovery

If MySQL won't start due to InnoDB corruption (remember — system tables use InnoDB even though VICIdial tables are MyISAM), use the force recovery mode:

```ini
[mysqld]
innodb_force_recovery = 1
```

Start MySQL with this setting. If it starts, dump your databases:

```bash
mysqldump --all-databases > /tmp/full-backup.sql
```

Then stop MySQL, remove the corrupted InnoDB files, remove the `innodb_force_recovery` line, start MySQL, and reimport:

```bash
systemctl stop mariadb
rm /var/lib/mysql/ib_logfile*
rm /var/lib/mysql/ibdata1
systemctl start mariadb
mysql < /tmp/full-backup.sql
```

**Increase `innodb_force_recovery` incrementally (1 through 6) if lower values don't work.** Level 6 is the most aggressive and may result in data loss. Always try the lowest value first.

### Prevention

- **UPS on every database server.** Power failures are the #1 cause of MyISAM corruption.
- **RAID with battery-backed cache.** Use LSI MegaRAID with BBU (battery backup unit) — it ensures writes complete even during power loss.
- **Don't kill MySQL processes.** `kill -9 mysqld` during a write operation virtually guarantees corruption. Use `systemctl stop mariadb` and wait for a clean shutdown.
- **Monitor disk space.** A full disk during a write operation corrupts the table being written to. Alert at 80% disk usage.

---

## Replication for Reporting

Running heavy analytical reports against your production VICIdial database is a bad idea. Report queries — especially those spanning 90 days of call data with GROUP BY and ORDER BY — can lock tables for seconds, blocking the real-time operations that keep your contact center running.

The solution: MySQL replication to a dedicated reporting slave.

### Architecture

```
Primary (Production)  ──binary log──>  Replica (Reporting)
    VICIdial writes here                Reports run here
    Real-time operations                Manager dashboards
    Agent interactions                  Historical analytics
```

### Setting Up Replication

**On the primary (production) server:**

```ini
[mysqld]
server-id = 1
log-bin = mysql-bin
binlog_format = MIXED
expire_logs_days = 7
sync_binlog = 0        # Async for performance
```

Create a replication user:

```sql
CREATE USER 'replication'@'10.0.0.%' IDENTIFIED BY 'strong-password-here';
GRANT REPLICATION SLAVE ON *.* TO 'replication'@'10.0.0.%';
FLUSH PRIVILEGES;
```

Take a consistent snapshot:

```bash
mysqldump --all-databases --master-data=2 --single-transaction > /tmp/dump.sql
```

**On the replica (reporting) server:**

```ini
[mysqld]
server-id = 2
relay-log = relay-bin
read_only = 1           # Prevent accidental writes
replicate-do-db = asterisk   # Only replicate the VICIdial database
```

Import the dump and start replication:

```bash
mysql < /tmp/dump.sql

mysql -e "CHANGE MASTER TO
  MASTER_HOST='10.0.0.1',
  MASTER_USER='replication',
  MASTER_PASSWORD='strong-password-here',
  MASTER_LOG_FILE='mysql-bin.000001',
  MASTER_LOG_POS=4;
START SLAVE;"
```

Verify:

```bash
mysql -e "SHOW SLAVE STATUS\G" | grep -E "Slave_IO_Running|Slave_SQL_Running|Seconds_Behind_Master"
```

You should see both IO and SQL threads running, with `Seconds_Behind_Master` at or near 0.

### Reporting-Specific Tuning on the Replica

The replica doesn't need VICIdial's write-optimized settings. Tune it for read-heavy analytical workloads:

```ini
[mysqld]
# Larger sort buffers for complex report queries
sort_buffer_size = 32M
read_buffer_size = 16M
join_buffer_size = 16M

# Larger tmp table size for GROUP BY operations
tmp_table_size = 512M
max_heap_table_size = 512M

# Disable binary logging on the replica (unless cascading)
skip-log-bin

# MyISAM key buffer — generous for index scans on large tables
key_buffer_size = 2048M
```

Point your VICIdial reporting users (managers, supervisors) at the replica's IP for historical reports, while keeping real-time reports on the primary (since they need up-to-the-second data from MEMORY tables that don't replicate).

---

## Where ViciStack Fits In

Every optimization in this guide is something we implement and maintain on every ViciStack deployment — from the initial `my.cnf` tuning to the archival cron jobs to the MEMORY table conversions. We monitor `pt-query-digest` output weekly, add indexes proactively as workload patterns shift, and have replication pipelines set up for operations that need reporting isolation.

The reality is that MySQL optimization isn't a one-time task. Tables grow, query patterns change, VICIdial updates alter the schema, and what worked at 50 agents doesn't work at 200. Keeping a VICIdial database healthy requires ongoing attention from someone who understands both MySQL internals and VICIdial's specific access patterns.

**ViciStack exists so you don't have to be that someone.** We handle the database so you can handle the business.

[Get a free database performance audit →](/free-audit/)

---

*This guide is maintained by the ViciStack team and updated as VICIdial, MySQL/MariaDB, and best practices evolve. Last updated: March 2026.*

---

## Frequently Asked Questions

### Should I switch VICIdial tables from MyISAM to InnoDB?

No. VICIdial is designed for MyISAM and the developers explicitly recommend against InnoDB for VICIdial tables. The application's code assumes MyISAM behaviors — concurrent inserts, table-level locking semantics, MEMORY table compatibility, and the REPAIR TABLE workflow. Switching to InnoDB introduces row-level locking (which sounds better but changes VICIdial's timing assumptions), transaction overhead, and a completely different recovery process. The performance characteristics are different in ways that break VICIdial's real-time operations. Use MyISAM for VICIdial tables and accept that InnoDB exists only for MySQL's internal system tables.

### How often should I run OPTIMIZE TABLE?

For heavily-written tables like `vicidial_log`, `call_log`, and `vicidial_carrier_log`, run OPTIMIZE TABLE weekly during a low-traffic window (Sunday night or early morning). For `vicidial_list`, run it after large-scale list operations (bulk imports, mass status changes, or large deletes). For MEMORY tables, OPTIMIZE TABLE is unnecessary — they don't fragment. Remember that OPTIMIZE TABLE locks the table for the duration, so never run it during peak hours.

### My vicidial_manager table has 200 million rows. How do I fix this?

This is the most common "database emergency" we see. The `vicidial_manager` table grows fastest and is almost never needed historically. The quickest fix: `DELETE FROM vicidial_manager WHERE entry_date < DATE_SUB(NOW(), INTERVAL 7 DAY) LIMIT 1000000;` — run this in a loop until the old rows are gone. If the table is too large for iterative deletes (each one locks the table), use the rename-and-recreate approach described in the archival section: rename to `vicidial_manager_old`, create a fresh `vicidial_manager`, copy recent rows back, then drop the old table during a maintenance window.

### What's the recommended RAID configuration for a VICIdial database server?

RAID-10 with an LSI MegaRAID controller with battery backup unit (BBU). Not RAID-5 — the write penalty on RAID-5 (read-modify-write cycle) is devastating for VICIdial's write-heavy workload. Not RAID-1 — it works for small deployments but doesn't provide the I/O parallelism needed above 100 agents. And specifically LSI MegaRAID — not 3ware, not Adaptec, not Dell PERC. Multiple VICIdial operators have paid for professional MySQL audits (including Percona engagements) and the recommendation is consistently the same: swap the RAID controller to LSI.

### How do I know if my database is the bottleneck?

Three quick checks: (1) Run `SHOW FULL PROCESSLIST` — if you see queries in "Locked" state or queries running for more than 2 seconds, you have contention. (2) Check the slow query log — if it's growing rapidly, queries are taking too long. (3) Monitor CPU and I/O wait — if `iowait` in `top` is above 20% on the database server, disk I/O is the constraint. For a more thorough assessment, enable the Performance Schema (`performance_schema = ON` in my.cnf) and query `events_statements_summary_by_digest` to find the most expensive query patterns.

### Can I use Amazon RDS or cloud-managed MySQL for VICIdial?

Technically, yes. Practically, it's a poor fit. VICIdial requires MyISAM (which RDS supports but doesn't optimize for), MEMORY tables (which work but are constrained by RDS instance class), and specific tuning parameters that RDS doesn't expose (like `myisam_repair_threads`). The latency between your EC2 telephony servers and RDS is also higher than bare-metal server-to-server communication. Most importantly, VICIdial's constant polling pattern generates a connection volume that gets expensive on RDS quickly. Bare-metal dedicated database servers with NVMe storage consistently outperform cloud database services for VICIdial workloads at a fraction of the cost.

### How do I handle the TIMESTAMP migration on a large database?

Plan a maintenance window. The ALTER TABLE operations lock each table for the duration of the conversion, which on a table with 50M+ rows can take 30-60 minutes. Run the migration table by table, starting with the smallest tables, so you can estimate timing before hitting the large ones. Take a full mysqldump backup before starting. If you can't afford the downtime for the largest tables, consider the rename-and-recreate approach: create a new table with the TIMESTAMP schema, copy recent data into it, swap the names, and archive the old table.

### What monitoring should I set up for the VICIdial database?

At minimum: (1) Disk usage alerts at 80% and 90%, (2) `Threads_connected` approaching `max_connections` (alert at 80%), (3) `Slow_queries` counter increasing faster than expected, (4) Replication lag on reporting slaves (`Seconds_Behind_Master` > 60), (5) Table corruption detection via weekly `CHECK TABLE` on critical tables, (6) MEMORY table size approaching `max_heap_table_size`. Tools like Nagios, Zabbix, or Prometheus with the mysqld_exporter work well. For quick command-line monitoring, `mysqladmin extended-status` piped through `grep` for key metrics is useful during active troubleshooting.

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-mysql-optimization).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
