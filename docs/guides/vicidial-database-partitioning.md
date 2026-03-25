# VICIdial Database Partitioning for High-Volume Call Centers

Every VICIdial deployment eventually hits the same wall: the database becomes the bottleneck. When your `vicidial_log` table has 50 million rows, your `recording_log` has grown to 30 GB, and your real-time reports take 15 seconds to load, you are past the point where adding more CPU or RAM to the dialer solves the problem. The database is doing full table scans on tables that were never designed for this volume.

This guide covers the practical steps to partition VICIdial's MySQL database, archive old data, tune InnoDB for high-volume dialing, and optimize the queries that matter most for daily operations.

## When the Database Becomes the Bottleneck

VICIdial uses MySQL (typically MariaDB on modern installs) as its sole data store. Every call event, agent status change, lead update, and recording reference writes to MySQL in real time. For a 25-agent center making 500 calls per agent per day, that is 12,500 new rows per day in `vicidial_log` alone. Add in `vicidial_agent_log`, `call_log`, `vicidial_closer_log`, `recording_log`, and `vicidial_did_log`, and you are easily generating 50,000+ rows per day.

At that rate, you hit 1 million rows in three weeks. After a year, you have 18 million rows in `vicidial_log`. After two years, you are approaching 40 million. The table does not partition itself.

### Symptoms of a Database Bottleneck

Watch for these warning signs:

- **Real-time reports lag**: The agent performance screen takes 5+ seconds to load
- **Outbound dialing hesitates**: The hopper refill query takes longer, causing gaps between calls
- **Recordings page is slow**: Searching recordings by date range takes 30+ seconds
- **Server load spikes during reports**: Running a campaign summary report pins a CPU core
- **`SHOW PROCESSLIST` shows long-running queries**: Queries against `vicidial_log` or `call_log` running for 10+ seconds

Verify with:

```sql
-- Check table sizes
SELECT table_name,
       ROUND(data_length / 1024 / 1024, 2) AS data_mb,
       ROUND(index_length / 1024 / 1024, 2) AS index_mb,
       table_rows
FROM information_schema.tables
WHERE table_schema = 'asterisk'
ORDER BY data_length DESC
LIMIT 20;
```

If `vicidial_log`, `call_log`, or `recording_log` are over 1 GB or over 10 million rows, you need partitioning and archiving.

## VICIdial Tables That Grow Unbounded

Not every VICIdial table needs attention. These are the ones that grow continuously and cause performance issues:

| Table | Grows By | Contains |
|-------|----------|----------|
| `vicidial_log` | Every outbound call attempt | Dial results, talk time, status |
| `vicidial_closer_log` | Every inbound/transfer call | Closer call details |
| `call_log` | Every call (CDR) | Asterisk CDR records |
| `vicidial_agent_log` | Every agent state change | Login, pause, wait, talk times |
| `recording_log` | Every recorded call | Recording file paths and metadata |
| `vicidial_did_log` | Every inbound DID call | DID routing log |
| `vicidial_carrier_log` | Every carrier event | SIP trunk activity |
| `server_performance` | Every update interval | Server metrics over time |
| `vicidial_dial_log` | Every dial attempt | Raw dial events |

## Partitioning vicidial_log by Date

MySQL partition pruning allows queries that filter by date to scan only the relevant partition instead of the entire table. For `vicidial_log`, which is almost always queried with a date filter, this is a massive performance win.

### Step 1: Assess the Current Table

```sql
-- Check current table structure
SHOW CREATE TABLE vicidial_log\G

-- Check row count and size
SELECT COUNT(*) FROM vicidial_log;
SELECT ROUND(data_length/1024/1024, 2) AS size_mb
FROM information_schema.tables
WHERE table_name = 'vicidial_log' AND table_schema = 'asterisk';
```

### Step 2: Backup Before Any Schema Change

This is non-negotiable. A failed ALTER TABLE on a multi-million row table can leave you with no data and no dialer.

```bash
# Full backup of the asterisk database
mysqldump --single-transaction --routines --triggers \
  asterisk vicidial_log > /backup/vicidial_log_$(date +%Y%m%d).sql

# Verify the backup
mysql -e "SELECT COUNT(*) FROM vicidial_log;" asterisk
wc -l /backup/vicidial_log_$(date +%Y%m%d).sql
```

### Step 3: Create the Partitioned Table

VICIdial's `vicidial_log` uses `call_date` as its primary time column. We partition by RANGE on this column, using monthly partitions:

```sql
-- First, check if the primary key includes call_date
-- MySQL requires the partition column to be part of every unique index
SHOW INDEX FROM vicidial_log;
```

If `call_date` is not part of the primary key, you need to modify the key. This is the trickiest part. VICIdial typically uses `uniqueid` as the primary key for `vicidial_log`:

```sql
-- Modify primary key to include call_date (required for partitioning)
-- WARNING: This locks the table. Do this during off-hours.
ALTER TABLE vicidial_log
  DROP PRIMARY KEY,
  ADD PRIMARY KEY (uniqueid, call_date);
```

Now add partitions:

```sql
ALTER TABLE vicidial_log PARTITION BY RANGE (TO_DAYS(call_date)) (
  PARTITION p2025_01 VALUES LESS THAN (TO_DAYS('2025-02-01')),
  PARTITION p2025_02 VALUES LESS THAN (TO_DAYS('2025-03-01')),
  PARTITION p2025_03 VALUES LESS THAN (TO_DAYS('2025-04-01')),
  PARTITION p2025_04 VALUES LESS THAN (TO_DAYS('2025-05-01')),
  PARTITION p2025_05 VALUES LESS THAN (TO_DAYS('2025-06-01')),
  PARTITION p2025_06 VALUES LESS THAN (TO_DAYS('2025-07-01')),
  PARTITION p2025_07 VALUES LESS THAN (TO_DAYS('2025-08-01')),
  PARTITION p2025_08 VALUES LESS THAN (TO_DAYS('2025-09-01')),
  PARTITION p2025_09 VALUES LESS THAN (TO_DAYS('2025-10-01')),
  PARTITION p2025_10 VALUES LESS THAN (TO_DAYS('2025-11-01')),
  PARTITION p2025_11 VALUES LESS THAN (TO_DAYS('2025-12-01')),
  PARTITION p2025_12 VALUES LESS THAN (TO_DAYS('2026-01-01')),
  PARTITION p2026_01 VALUES LESS THAN (TO_DAYS('2026-02-01')),
  PARTITION p2026_02 VALUES LESS THAN (TO_DAYS('2026-03-01')),
  PARTITION p2026_03 VALUES LESS THAN (TO_DAYS('2026-04-01')),
  PARTITION p2026_04 VALUES LESS THAN (TO_DAYS('2026-05-01')),
  PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

### Step 4: Verify Partition Pruning

```sql
-- This should only scan one partition
EXPLAIN PARTITIONS
SELECT * FROM vicidial_log
WHERE call_date BETWEEN '2026-03-01' AND '2026-03-19';

-- Look for "partitions: p2026_03" in the output
-- NOT "partitions: p2025_01,p2025_02,...,p_future"
```

### Step 5: Automate Future Partitions

Create a cron job that adds new partitions before the `p_future` catchall is needed:

```sql
-- Run monthly: reorganize p_future to add next month
ALTER TABLE vicidial_log REORGANIZE PARTITION p_future INTO (
  PARTITION p2026_05 VALUES LESS THAN (TO_DAYS('2026-06-01')),
  PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

Script this:

```bash
#!/bin/bash
# /opt/vicistack/add_partition.sh
# Run via cron on the 15th of each month

NEXT_MONTH=$(date -d "next month" +%Y_%m)
NEXT_MONTH_START=$(date -d "next month" +%Y-%m-01)
FOLLOWING_MONTH_START=$(date -d "2 months" +%Y-%m-01)

mysql asterisk -e "
ALTER TABLE vicidial_log REORGANIZE PARTITION p_future INTO (
  PARTITION p${NEXT_MONTH} VALUES LESS THAN (TO_DAYS('${FOLLOWING_MONTH_START}')),
  PARTITION p_future VALUES LESS THAN MAXVALUE
);
"

echo "$(date): Added partition p${NEXT_MONTH} for vicidial_log" >> /var/log/vicistack-partitions.log
```

## Archiving Old CDR and Log Data

Partitioning makes archiving trivial. Instead of running a slow `DELETE FROM vicidial_log WHERE call_date < '2025-01-01'` that locks the table and generates massive binary log entries, you drop entire partitions instantly:

### Archive Process

```bash
#!/bin/bash
# /opt/vicistack/archive_partition.sh
# Archive and drop partitions older than 6 months

ARCHIVE_DIR="/backup/vicidial_archive"
CUTOFF=$(date -d "6 months ago" +%Y_%m)

# Export the partition data first
mysqldump --single-transaction asterisk vicidial_log \
  --where="call_date < '$(date -d '6 months ago' +%Y-%m-01)'" \
  > ${ARCHIVE_DIR}/vicidial_log_archive_${CUTOFF}.sql

# Compress the archive
gzip ${ARCHIVE_DIR}/vicidial_log_archive_${CUTOFF}.sql

# Drop the old partition (instant, no table lock)
mysql asterisk -e "ALTER TABLE vicidial_log DROP PARTITION p${CUTOFF};"

echo "$(date): Archived and dropped partition p${CUTOFF}" >> /var/log/vicistack-archive.log
```

Dropping a partition is an O(1) operation -- it removes the data file instantly regardless of how many rows are in it. Compare this to `DELETE` which would need to scan every row, update indexes, and write to the binary log.

### Apply the Same Pattern to Other Tables

Repeat the partitioning process for `call_log`, `vicidial_closer_log`, `vicidial_agent_log`, and `recording_log`. The process is identical -- just change the table name and the date column (most use `call_date` or `event_time`).

## MySQL Performance Tuning for VICIdial

Beyond partitioning, MySQL configuration has an outsized impact on VICIdial performance. Most installs run with default or near-default settings, which are tuned for general-purpose workloads, not for the write-heavy, real-time pattern VICIdial demands.

### InnoDB Buffer Pool Optimization

The InnoDB buffer pool is MySQL's in-memory cache for table data and indexes. This single setting has the largest impact on database performance.

**Rule of thumb**: Set the buffer pool to 60-70% of total system RAM on a dedicated database server, or 40-50% if MySQL shares the server with Asterisk.

```ini
# /etc/my.cnf.d/vicidial-tuning.cnf

[mysqld]
# Buffer pool - set to 60-70% of RAM on dedicated DB server
# 8 GB example (on a 12 GB server)
innodb_buffer_pool_size = 8G
innodb_buffer_pool_instances = 8    # 1 instance per GB

# Buffer pool dump/load for fast restarts
innodb_buffer_pool_dump_at_shutdown = ON
innodb_buffer_pool_load_at_startup = ON
```

Verify your buffer pool hit rate (should be > 99%):

```sql
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_read%';

-- Calculate hit rate:
-- hit_rate = 1 - (Innodb_buffer_pool_reads / Innodb_buffer_pool_read_requests) * 100
-- If below 99%, increase buffer pool size
```

### InnoDB Log File Tuning

InnoDB write performance depends on the redo log configuration:

```ini
[mysqld]
# Larger log files = less checkpoint flushing = faster writes
# Set to 25% of buffer pool size
innodb_log_file_size = 2G
innodb_log_buffer_size = 64M

# Flush behavior (trade-off: durability vs. performance)
# 1 = flush every commit (safest, slowest)
# 2 = flush to OS cache every commit, flush to disk every second (good balance)
# 0 = flush to OS cache every second (fastest, risk losing 1 sec of data on crash)
innodb_flush_log_at_trx_commit = 2

# For VICIdial, setting 2 is the right balance. The data being written
# (call logs, agent events) can be reconstructed from Asterisk logs if
# the worst happens.
```

### Write Optimization

VICIdial writes constantly. Optimize for that workload:

```ini
[mysqld]
# Reduce fsync overhead
innodb_flush_method = O_DIRECT

# Increase write throughput
innodb_io_capacity = 2000            # for SSD
innodb_io_capacity_max = 4000
innodb_write_io_threads = 8
innodb_read_io_threads = 8

# Adaptive flushing
innodb_adaptive_flushing = ON
innodb_adaptive_flushing_lwm = 10

# Change buffer (for secondary index updates)
innodb_change_buffering = all
innodb_change_buffer_max_size = 25
```

### Query Cache (MariaDB)

For MariaDB (which VICIdial commonly runs on), the query cache can help with repetitive reads like real-time report queries:

```ini
[mysqld]
query_cache_type = 1
query_cache_size = 256M
query_cache_limit = 4M
query_cache_min_res_unit = 2048
```

Note: MySQL 8.0 removed the query cache entirely. If you are on MySQL 8.0+, skip this and rely on buffer pool caching instead.

### Connection and Thread Tuning

```ini
[mysqld]
max_connections = 500              # VICIdial opens many connections
thread_cache_size = 50
table_open_cache = 4000
table_definition_cache = 2000

# Temporary tables (for complex reports)
tmp_table_size = 256M
max_heap_table_size = 256M
```

## Query Optimization for Large Datasets

Even with partitioning and tuning, poorly optimized queries can still cause problems. Here are the VICIdial queries that most commonly cause issues and how to fix them.

### The Hopper Refill Query

VICIdial's hopper (the queue of numbers to dial) is refilled by a query that joins `vicidial_list` with `vicidial_hopper` and several exclusion tables. On a large list, this query can take 10+ seconds:

```sql
-- Check if the hopper refill is slow
SHOW PROCESSLIST;
-- Look for queries hitting vicidial_list with long Time values
```

**Optimization**: Ensure proper indexes exist:

```sql
-- Critical indexes for hopper performance
ALTER TABLE vicidial_list ADD INDEX idx_list_hopper
  (list_id, status, called_since_last_reset, phone_number);

-- If you use GMT offset filtering
ALTER TABLE vicidial_list ADD INDEX idx_gmt
  (gmt_offset_now, list_id, status);
```

### Real-Time Report Queries

The real-time agent screen queries `vicidial_live_agents` and `vicidial_auto_calls`. These tables are small but queried constantly:

```sql
-- These should be in memory. Verify:
SELECT table_name, engine
FROM information_schema.tables
WHERE table_name IN ('vicidial_live_agents', 'vicidial_auto_calls')
AND table_schema = 'asterisk';

-- If engine is MEMORY, good. If InnoDB, they'll still be cached in buffer pool.
```

### Campaign Summary Reports

Reports that aggregate across `vicidial_log` benefit enormously from partitioning. But also add covering indexes:

```sql
-- Index for campaign reports by date
ALTER TABLE vicidial_log ADD INDEX idx_campaign_date
  (campaign_id, call_date, status, length_in_sec);

-- This lets the report query use an index scan instead of a table scan
-- for queries like:
SELECT status, COUNT(*), AVG(length_in_sec)
FROM vicidial_log
WHERE campaign_id = 'SALES'
AND call_date BETWEEN '2026-03-01' AND '2026-03-19'
GROUP BY status;
```

### Slow Query Log

Enable the slow query log to catch problematic queries:

```ini
[mysqld]
slow_query_log = ON
slow_query_log_file = /var/log/mysql/slow-query.log
long_query_time = 2     # log queries taking > 2 seconds
log_queries_not_using_indexes = ON
```

Review regularly:

```bash
# Find the most common slow queries
mysqldumpslow -s c -t 10 /var/log/mysql/slow-query.log
```

## Monitoring Database Performance

### Key Metrics to Track

Set up monitoring (Zabbix, Nagios, Prometheus, or a simple script) for these metrics:

```sql
-- Buffer pool hit rate (should be > 99%)
SELECT
  (1 - (
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS
     WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads') /
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS
     WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests')
  )) * 100 AS buffer_pool_hit_rate;

-- Active connections
SHOW GLOBAL STATUS LIKE 'Threads_connected';

-- Queries per second
SHOW GLOBAL STATUS LIKE 'Questions';

-- Slow queries count
SHOW GLOBAL STATUS LIKE 'Slow_queries';

-- Table lock waits (should be near zero with InnoDB)
SHOW GLOBAL STATUS LIKE 'Table_locks_waited';
```

### Automated Health Check Script

```bash
#!/bin/bash
# /opt/vicistack/db_health_check.sh

MYSQL="mysql -u root asterisk -N -e"

# Check buffer pool hit rate
READS=$($MYSQL "SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_reads';" | awk '{print $2}')
REQUESTS=$($MYSQL "SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_read_requests';" | awk '{print $2}')
HIT_RATE=$(echo "scale=2; (1 - ($READS / $REQUESTS)) * 100" | bc)
echo "Buffer pool hit rate: ${HIT_RATE}%"

# Check table sizes
echo "--- Top 10 tables by size ---"
$MYSQL "SELECT table_name, ROUND((data_length + index_length)/1024/1024, 2) AS size_mb, table_rows
FROM information_schema.tables
WHERE table_schema = 'asterisk'
ORDER BY data_length + index_length DESC LIMIT 10;"

# Check for long-running queries
echo "--- Queries running > 5 seconds ---"
$MYSQL "SELECT id, user, host, db, time, state, LEFT(info, 100)
FROM information_schema.processlist
WHERE time > 5 AND command != 'Sleep';"

# Check partition status
echo "--- Partition status for vicidial_log ---"
$MYSQL "SELECT partition_name, table_rows, ROUND(data_length/1024/1024, 2) AS size_mb
FROM information_schema.partitions
WHERE table_name = 'vicidial_log' AND table_schema = 'asterisk'
ORDER BY partition_ordinal_position;"
```

### Replication Lag (If Using Read Replicas)

For centers that run reporting queries on a read replica:

```sql
-- On the replica
SHOW SLAVE STATUS\G
-- Check Seconds_Behind_Master - should be < 5 for real-time reports
```

## Common Pitfalls

### Do Not Partition vicidial_list

The `vicidial_list` table (your lead data) might be the largest table, but partitioning it is dangerous. VICIdial queries this table by `lead_id` (not date), and the hopper refill uses complex multi-column filters. Partitioning by date would actually make these queries slower because MySQL cannot prune partitions when the query does not filter on the partition key.

Instead, manage `vicidial_list` size by:
- Archiving completed lists to a separate table or database
- Using `DELETE` with proper indexing for old, fully-worked leads
- Keeping active campaign lists under 1 million leads each

### Do Not Run ALTER TABLE During Dialing Hours

`ALTER TABLE` on a multi-million row table takes time and can lock the table depending on the operation. Schedule all schema changes for maintenance windows:

```bash
# Check current call volume before making schema changes
mysql asterisk -e "SELECT COUNT(*) FROM vicidial_auto_calls;"
# If > 0, wait for a maintenance window
```

### Do Not Skip Backups

Every database operation in this guide should be preceded by a backup. One failed ALTER TABLE can corrupt an entire table. The 20 minutes a backup takes is cheap insurance against rebuilding your call log history from Asterisk CDR files.

## How ViciStack Helps

Database performance is the foundation of everything in VICIdial. Slow queries mean delayed hopper refills, which mean fewer dials per hour, which mean fewer live connections, which directly hits your revenue. A 25-agent center losing even 5% of dials to database lag is leaving tens of thousands of dollars on the table every month.

ViciStack handles the entire database optimization stack for VICIdial call centers:

- **Table partitioning** implemented with zero downtime using online DDL where possible
- **Automated archiving** with partition rotation and compressed backup storage
- **MySQL/MariaDB tuning** specifically calibrated for VICIdial's write-heavy workload
- **Query analysis** identifying the specific queries slowing your campaigns
- **Buffer pool and InnoDB tuning** based on actual workload profiling, not generic rules of thumb
- **Ongoing monitoring** with alerts when table sizes, query times, or hit rates cross thresholds
- **Capacity planning** so you know when to move to a dedicated database server before you hit the wall

At $150/agent/month flat rate, this is included in the optimization service -- not an add-on billable at hourly rates.

**Get a free database performance analysis for your VICIdial deployment.** We will connect, profile your top 10 queries, check your table sizes, and show you exactly where the bottleneck is.

[Request your free ViciStack analysis](https://vicistack.com/proof/) -- response in under 5 minutes.

## Related Articles

- [VICIdial Remote Agent Setup: NAT Traversal, WebRTC, and SIP Configuration](/blog/vicidial-remote-agent-setup) -- optimize the connections hitting your database
- [VICIdial Quality Assurance Scoring with Call Recordings](/blog/vicidial-qa-scoring) -- recording_log is one of the fastest-growing tables
- [VICIdial CNAM Lookup Integration for Inbound Routing](/blog/vicidial-cnam-lookup-inbound) -- inbound call data adds to database volume

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-database-partitioning).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
