### 🚨 Application Is Slow — Complete DBA Investigation Process
### MySQL • MariaDB • PostgreSQL • SQL Server
#### In-Depth Analysis | Root Cause to Fix

---

> **How to use this guide:**
> Always start at **Step 0** (universal pre-checks) before touching any database.
> The golden rule: **Eliminate the obvious before going deep.**
> Every command in this guide includes — what it does, what to look for, what it means, and how to fix it.

---

### 📐 The Diagnostic Mindset

When an application is reported as "slow", there are five possible culprits:

```
Application Slow
    │
    ├── 1. Network          → Latency between app and DB server
    ├── 2. OS / Hardware    → CPU, memory, disk, I/O at OS level
    ├── 3. Database Engine  → Locks, bad queries, missing indexes, poor config
    ├── 4. Application Code → ORM inefficiency, N+1 queries, connection pool exhaustion
    └── 5. External Systems → Linked servers, external APIs called from DB
```

**Always split app response time from DB execution time first.**
If DB takes 0.2s but the app takes 8s → the database is NOT the problem.
If DB takes 7.8s and the app takes 8s → the database IS the problem.

---

---

### 🌐 STEP 0: UNIVERSAL PRE-CHECKS (ALL DATABASES)

Run these before opening any database console. These 10 minutes can save hours of misdirected work.

---

### 0.1 — CPU Analysis

```bash
# Quick snapshot
top -bn1 | head -25

# Per-core breakdown (shows if one core is maxed = single-threaded bottleneck)
mpstat -P ALL 1 5

# Historical 5-second samples
sar -u 1 5

# Find which process is consuming CPU
ps aux --sort=-%cpu | head -15
```

**What Each CPU Metric Means:**

| Metric | Threshold | Meaning | Action |
|--------|-----------|---------|--------|
| `%us` (user space) | > 80% | Query processing / app logic overhead | Find top queries, check indexes |
| `%sy` (system/kernel) | > 20% | Kernel I/O handling, context switching | Check disk I/O, network syscalls |
| `%wa` (I/O wait) | > 10% | CPU stalled waiting for disk reads/writes | Critical — disk is the bottleneck |
| `%st` (steal) | > 5% | VM being CPU-throttled by hypervisor | Upgrade instance tier in cloud |
| Load Average | > CPU count × 1.5 | System is saturated, queue building | Immediate intervention needed |

> **Key insight:** `iowait` above 10% almost always points to missing indexes causing full table scans, or a buffer pool/shared_buffers that is too small to cache working data in memory.

---

## 0.2 — Memory Analysis

```bash
# Overview
free -h

# Detailed breakdown
vmstat -s

# Key fields
cat /proc/meminfo | grep -E "MemTotal|MemFree|MemAvailable|Cached|SwapTotal|SwapFree|Dirty"

# Which process is using swap (dangerous for databases)
for pid in $(ls /proc | grep -E '^[0-9]+$'); do
  swap=$(grep VmSwap /proc/$pid/status 2>/dev/null | awk '{print $2}')
  name=$(grep Name /proc/$pid/status 2>/dev/null | awk '{print $2}')
  [ "${swap:-0}" -gt "0" ] 2>/dev/null && echo "PID $pid ($name): ${swap} kB swap"
done | sort -t: -k2 -rn | head -10
```

**Memory Warning Thresholds:**

| Condition | Severity | Impact |
|-----------|----------|--------|
| SwapUsed > 0 for DB process | 🔴 Critical | All query response times multiply by 10–1000× |
| MemAvailable < 10% of total | 🔴 Critical | OS evicting DB cache pages, causing disk reads |
| Dirty memory climbing fast | 🟡 Warning | Checkpoint/flush storm about to occur |
| Cache/Buff < 50% of RAM | 🟡 Warning | DB buffer pool may not fit working data set |

> **The swap trap:** If your database process is in swap, every "slow query" diagnosis is misleading. The query itself may be fine — it's being paged out. Always check swap before analyzing SQL.

---

## 0.3 — Disk I/O Analysis

```bash
# Per-device I/O statistics (refresh every 1 second, 5 times)
iostat -xz 1 5

# Real-time per-process I/O
iotop -o -d 2

# Disk space (a full disk is an immediate emergency)
df -h
df -i         # inode exhaustion — can look like disk full even with space available

# Find large files that may have appeared recently
find /var/lib/mysql /var/lib/postgresql /var/opt/mssql -size +1G -ls 2>/dev/null | sort -k7 -rn
```

**iostat Column Reference:**

| Column | Dangerous Threshold | Meaning |
|--------|---------------------|---------|
| `%util` | > 80% | Disk is saturated — requests queuing |
| `await` | > 10ms on SSD, >30ms on HDD | High I/O latency — stalling queries |
| `r_await` | > 15ms | Slow reads — check buffer pool size and indexes |
| `w_await` | > 20ms | Slow writes — checkpoint pressure, redo log bottleneck |
| `svctm` | Approaches `await` | Disk is actively processing, not queuing |
| `rkB/s` + `wkB/s` | Near max device rating | Bandwidth saturation |

> **Read vs Write pattern matters:**
> - High reads → data not in buffer pool → increase memory or add indexes
> - High writes → checkpoint floods, bulk inserts, or undo/redo log pressure

---

## 0.4 — Network Latency Check

```bash
# Baseline round-trip time (run from app server to DB server)
ping -c 20 <db_server_ip>

# Path analysis
traceroute <db_server_ip>
mtr --report --report-cycles 50 <db_server_ip>

# Actual DB connection time (measures DNS + TCP + auth overhead)
time mysql -h <db_host> -u readonly_user -p -e "SELECT 1;" 2>/dev/null
time psql -h <db_host> -U readonly_user -c "SELECT 1;" 2>/dev/null

# Check for TCP errors and retransmits
netstat -s | grep -iE "retransmit|failed|reset|error" | head -20
ss -s   # socket summary

# Check network interface errors
ip -s link show eth0 | grep -A2 "RX\|TX"
```

**Latency Interpretation:**

| Round-trip Latency | Status | What It Means |
|--------------------|--------|---------------|
| < 1ms | ✅ Excellent | Same rack/subnet |
| 1–3ms | ✅ Good | Same datacenter |
| 3–10ms | 🟡 Monitor | Cross-AZ or cross-region |
| 10–30ms | 🟡 Elevated | Check routing, load balancer |
| > 30ms | 🔴 Critical | Network path problem — escalate |
| Any packet loss > 0% | 🔴 Critical | Causes TCP retransmits → application timeouts |

---

## 0.5 — App Response Time vs DB Execution Time Split

This single comparison tells you where to focus your investigation. Do not skip it.

```bash
# MySQL — measure pure DB execution time
mysql -h <host> -u <user> -p -e "SET profiling=1; <YOUR QUERY>; SHOW PROFILES;"

# PostgreSQL — timing enabled
psql -h <host> -U <user> -c "\timing on" -c "<YOUR QUERY>"

# SQL Server — in SSMS or sqlcmd
SET STATISTICS TIME ON;
SET STATISTICS IO ON;
<YOUR QUERY>
SET STATISTICS TIME OFF;
SET STATISTICS IO OFF;
-- Output: CPU time, elapsed time, logical reads, physical reads
```

**Interpreting the Split:**

```
Scenario A:
  App response time:  8,200ms
  DB execution time:    250ms
  → Gap = ~8 seconds OUTSIDE the database
  → Root cause: Application code, ORM, result set processing, network between layers
  → Action: Profile the application, not the database

Scenario B:
  App response time:  8,200ms
  DB execution time:  7,900ms
  → Gap = ~300ms (normal overhead)
  → Root cause: Database IS the bottleneck
  → Action: Proceed to DB-specific investigation below

Scenario C:
  App response time:  8,200ms
  DB execution time:  varies (50ms to 7000ms across runs)
  → Inconsistent DB times = lock contention or parameter sniffing
  → Action: Check active locks and query plan cache
```

---

---

# 🟢 SECTION 1: MYSQL — IN-DEPTH INVESTIGATION

---

## Overview: MySQL Slowness Categories

```
MySQL Slow Application
    │
    ├── A. Lock Contention    → Rows/tables locked, queries queuing
    ├── B. Slow Queries       → Missing indexes, full scans, bad plans
    ├── C. Buffer Pool Miss   → Data not cached, disk reads on every query
    ├── D. Connection Limits  → Too many connections, pool exhausted
    ├── E. Replication Lag    → Read queries hitting stale replica
    └── F. Configuration      → Undersized InnoDB settings
```

---

## 1.1 — Check Active Queries & Process List

```sql
-- Full process list (every active database connection)
SHOW FULL PROCESSLIST;

-- More powerful version via information_schema (supports filtering)
SELECT
  id                              AS thread_id,
  user,
  host,
  db                              AS database_name,
  command,
  time                            AS seconds_running,
  state,
  LEFT(info, 300)                 AS query_text
FROM information_schema.processlist
WHERE command != 'Sleep'
ORDER BY time DESC;
```

**State Column — Complete Reference:**

| State | What's Happening | Severity | What To Do |
|-------|-----------------|----------|------------|
| `Locked` | Waiting for a table-level lock | 🔴 Critical | Find what holds the lock, kill if safe |
| `Waiting for row lock` | InnoDB row-level lock wait | 🔴 Critical | Check `INNODB_LOCK_WAITS` |
| `Sending data` | Scanning rows to build result set | 🟡 Warning | Check EXPLAIN for full scans |
| `Copying to tmp table` | GROUP BY / ORDER BY with no index | 🟡 Warning | Add appropriate index |
| `Sorting result` | filesort happening — no ORDER BY index | 🟡 Warning | Add index on sort column |
| `Opening tables` | Metadata lock contention or high `table_open_cache` miss | 🟡 Warning | Increase `table_open_cache` |
| `Waiting for metadata lock` | DDL blocking DML (ALTER TABLE held) | 🔴 Critical | Kill DDL or wait it out |
| `Creating sort index` | Query sorting on disk | 🟡 Warning | Increase `sort_buffer_size` |
| `init` | Optimizer doing plan setup | 🟢 OK | Usually fast |
| `Sleep` | Idle connection — may hold locks | 🟡 Watch | Check `wait_timeout` setting |

```sql
-- Kill a specific query (keeps the connection alive)
KILL QUERY <thread_id>;

-- Kill the entire connection
KILL CONNECTION <thread_id>;

-- Generate kill statements for all queries running > 60 seconds
SELECT
  CONCAT('KILL QUERY ', id, ';') AS kill_statement,
  id,
  user,
  time,
  LEFT(info, 200) AS query
FROM information_schema.processlist
WHERE command = 'Query'
  AND time > 60
  AND user NOT IN ('replication', 'system user')
ORDER BY time DESC;
-- REVIEW this output carefully before executing — never blindly kill
```

---

## 1.2 — Slow Query Log — Enable & Analyze

```ini
# Add to /etc/mysql/mysql.conf.d/mysqld.cnf and restart (or use SET GLOBAL)
[mysqld]
slow_query_log          = 1
slow_query_log_file     = /var/log/mysql/slow.log
long_query_time         = 1          # Log queries taking > 1 second
log_queries_not_using_indexes = 1    # Log ALL full-scan queries regardless of time
log_slow_admin_statements = 1        # Include ALTER, OPTIMIZE, etc.
min_examined_row_limit  = 100        # Only log if > 100 rows examined
```

```sql
-- Enable without restart (takes effect immediately, lost on restart)
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;
SET GLOBAL log_queries_not_using_indexes = 'ON';

-- Verify it's active
SHOW VARIABLES LIKE 'slow_query_log%';
SHOW VARIABLES LIKE 'long_query_time';
```

```bash
# Analyze the slow query log with Percona's pt-query-digest (best tool)
pt-query-digest /var/log/mysql/slow.log | head -200

# Simple manual grep analysis
grep "Query_time" /var/log/mysql/slow.log | awk '{print $3}' | sort -rn | head -20

# mysqldumpslow — built-in MySQL analysis tool
mysqldumpslow -s t -t 10 /var/log/mysql/slow.log    # top 10 by total time
mysqldumpslow -s c -t 10 /var/log/mysql/slow.log    # top 10 by call count
```

---

## 1.3 — Performance Schema — Deep Query Analysis

```sql
-- Top 10 slowest query patterns (aggregated by digest/fingerprint)
SELECT
  SCHEMA_NAME                                        AS database_name,
  LEFT(DIGEST_TEXT, 150)                             AS query_pattern,
  COUNT_STAR                                         AS total_executions,
  ROUND(SUM_TIMER_WAIT   / 1e12, 3)                 AS total_exec_seconds,
  ROUND(AVG_TIMER_WAIT   / 1e9,  3)                 AS avg_exec_ms,
  ROUND(MAX_TIMER_WAIT   / 1e9,  3)                 AS max_exec_ms,
  ROUND(MIN_TIMER_WAIT   / 1e9,  3)                 AS min_exec_ms,
  SUM_ROWS_EXAMINED                                  AS total_rows_examined,
  SUM_ROWS_SENT                                      AS total_rows_returned,
  ROUND(SUM_ROWS_EXAMINED / NULLIF(SUM_ROWS_SENT,0)) AS rows_examined_per_row_sent,
  SUM_NO_INDEX_USED                                  AS full_scan_executions,
  SUM_CREATED_TMP_DISK_TABLES                        AS tmp_disk_tables_created
FROM performance_schema.events_statements_summary_by_digest
WHERE SCHEMA_NAME IS NOT NULL
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 10;
```

**How to Read the Output:**

| Column | Red Flag Value | What It Tells You |
|--------|---------------|-------------------|
| `avg_exec_ms` | > 500ms | Query consistently slow — top priority |
| `rows_examined_per_row_sent` | > 50 | Reading far more rows than needed → missing index |
| `full_scan_executions` | > 0 (on large tables) | No index being used |
| `tmp_disk_tables_created` | > 0 | Temp table spilling to disk → increase `tmp_table_size` |
| `max_exec_ms >> avg_exec_ms` | Large gap | Occasional lock waits or plan instability |

```sql
-- Queries with worst index efficiency (missing index candidates)
SELECT
  LEFT(DIGEST_TEXT, 150) AS query_pattern,
  COUNT_STAR,
  SUM_ROWS_EXAMINED,
  SUM_ROWS_SENT,
  ROUND(SUM_ROWS_EXAMINED / NULLIF(SUM_ROWS_SENT,0)) AS rows_per_returned
FROM performance_schema.events_statements_summary_by_digest
WHERE SUM_ROWS_EXAMINED / NULLIF(SUM_ROWS_SENT,0) > 100
  AND COUNT_STAR > 5
ORDER BY SUM_ROWS_EXAMINED DESC
LIMIT 15;

-- Queries never using indexes (confirmed full scan queries)
SELECT
  LEFT(DIGEST_TEXT, 150)                  AS query_pattern,
  COUNT_STAR                              AS executions,
  SUM_NO_INDEX_USED + SUM_NO_GOOD_INDEX_USED AS bad_index_count,
  ROUND(AVG_TIMER_WAIT / 1e9, 2)          AS avg_ms
FROM performance_schema.events_statements_summary_by_digest
WHERE SUM_NO_INDEX_USED + SUM_NO_GOOD_INDEX_USED > 0
  AND COUNT_STAR > 5
ORDER BY bad_index_count DESC, SUM_TIMER_WAIT DESC
LIMIT 10;
```

---

## 1.4 — Lock Contention & Deadlock Analysis

```sql
-- Current InnoDB row lock waits (who is waiting and who is blocking)
SELECT
  r.trx_id                                               AS waiting_transaction_id,
  r.trx_mysql_thread_id                                  AS waiting_thread,
  LEFT(r.trx_query, 200)                                 AS waiting_query,
  TIMESTAMPDIFF(SECOND, r.trx_wait_started, NOW())       AS waiting_seconds,
  b.trx_id                                               AS blocking_transaction_id,
  b.trx_mysql_thread_id                                  AS blocking_thread,
  LEFT(b.trx_query, 200)                                 AS blocking_query,
  TIMESTAMPDIFF(SECOND, b.trx_started, NOW())            AS blocking_trx_age_seconds
FROM information_schema.innodb_lock_waits w
  JOIN information_schema.innodb_trx r ON r.trx_id = w.requesting_trx_id
  JOIN information_schema.innodb_trx b ON b.trx_id = w.blocking_trx_id;

-- All open InnoDB transactions (long-open = lock holder suspect)
SELECT
  trx_id,
  trx_mysql_thread_id                                    AS thread_id,
  trx_started,
  TIMESTAMPDIFF(SECOND, trx_started, NOW())              AS open_seconds,
  LEFT(trx_query, 200)                                   AS current_query,
  trx_rows_locked,
  trx_rows_modified,
  trx_operation_state
FROM information_schema.innodb_trx
ORDER BY trx_started ASC;
-- WARNING: Any transaction open > 60 seconds is dangerous
-- trx_rows_locked > 10000 = large lock footprint — may cause cascade blocking
```

**Deadlock Investigation:**
```sql
-- View latest deadlock details (run immediately after deadlock reported)
SHOW ENGINE INNODB STATUS\G

-- Look for this section in the output:
-- ========================
-- LATEST DETECTED DEADLOCK
-- ========================
-- Shows: Transaction 1 and Transaction 2
--        What each holds ("holds lock on")
--        What each waits for ("waiting for lock on")
--        Which transaction was chosen as victim and rolled back
```

```sql
-- Enable persistent deadlock logging to error log
SET GLOBAL innodb_print_all_deadlocks = ON;

-- Then check error log
-- tail -f /var/log/mysql/error.log | grep -A 40 "DEADLOCK"
```

**Deadlock Root Cause Patterns:**

| Pattern | Cause | Fix |
|---------|-------|-----|
| Two transactions always on same two tables | Inconsistent lock ordering | Always lock Table A before Table B in all code paths |
| INSERT + DELETE on same range | Gap lock conflict | Use `READ COMMITTED` isolation level |
| High-frequency batch deletes | Large delete range locks | Break into smaller batches (< 1000 rows per commit) |
| ORM-generated queries deadlocking | N+1 generating unexpected lock sequences | Review ORM query logging, add explicit ordering |

---

## 1.5 — EXPLAIN & Index Analysis

```sql
-- Standard EXPLAIN (shows query execution plan)
EXPLAIN
SELECT o.id, o.status, c.name
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.status = 'PENDING'
  AND o.created_at > DATE_SUB(NOW(), INTERVAL 7 DAY)
ORDER BY o.created_at DESC
LIMIT 100;

-- EXPLAIN ANALYZE (MySQL 8.0+) — actual row counts vs estimates
EXPLAIN ANALYZE
SELECT * FROM orders WHERE status='PENDING' AND created_at > '2024-01-01';
```

**EXPLAIN Column Reference:**

| Column | Value | Meaning | Fix |
|--------|-------|---------|-----|
| `type` | `ALL` | Full table scan — every row read | Add index on WHERE columns |
| `type` | `index` | Full index scan (better, not great) | Consider covering index |
| `type` | `range` | Index range scan | Good for inequality conditions |
| `type` | `ref` | Index lookup with non-unique key | Good |
| `type` | `eq_ref` | Index lookup on unique/primary key | Best possible for joins |
| `type` | `const` | Single-row lookup via primary key | Perfect |
| `rows` | Millions | Estimating millions of rows | Full scan — add index |
| `Extra` | `Using filesort` | Sorting without index | Add sort column to index |
| `Extra` | `Using temporary` | Temp table for GROUP BY/DISTINCT | Rewrite or add index |
| `Extra` | `Using join buffer` | Join column has no index | Add index on join column |
| `key` | `NULL` | No index chosen by optimizer | Add appropriate index |

```sql
-- Find tables relying on full scans (via sys schema — MySQL 5.7+)
SELECT * FROM sys.statements_with_full_table_scans
ORDER BY no_index_used_count DESC
LIMIT 15;

-- Find unused indexes (pure write overhead — safe to remove)
SELECT * FROM sys.schema_unused_indexes
WHERE object_schema NOT IN ('performance_schema', 'sys', 'mysql');

-- Find duplicate / redundant indexes (wasted space and write overhead)
SELECT * FROM sys.schema_redundant_indexes;

-- Force statistics update if optimizer making bad estimates
ANALYZE TABLE orders;
ANALYZE TABLE customers;
```

---

## 1.6 — InnoDB Buffer Pool & System Health

```sql
-- Buffer pool hit ratio (the most important single MySQL metric)
SELECT
  ROUND(
    (1 - (
      (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME='Innodb_buffer_pool_reads')
      /
      NULLIF((SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME='Innodb_buffer_pool_read_requests'), 0)
    )) * 100, 4
  ) AS buffer_pool_hit_pct;
-- Target: > 99.0%
-- If < 99%: Buffer pool is too small — data being read from disk on every query

-- Buffer pool detailed stats
SELECT variable_name, variable_value
FROM information_schema.global_status
WHERE variable_name IN (
  'Innodb_buffer_pool_pages_total',
  'Innodb_buffer_pool_pages_free',
  'Innodb_buffer_pool_pages_dirty',
  'Innodb_buffer_pool_wait_free',     -- > 0 = pool thrashing (critical)
  'Innodb_buffer_pool_read_requests',
  'Innodb_buffer_pool_reads',
  'Innodb_row_lock_current_waits',    -- > 0 = active lock waits
  'Innodb_row_lock_waits',
  'Innodb_row_lock_time_avg'          -- ms average — > 100ms is concerning
);

-- Check temp table disk spills (increase tmp_table_size if > 5%)
SELECT
  variable_name,
  variable_value
FROM information_schema.global_status
WHERE variable_name IN ('Created_tmp_tables', 'Created_tmp_disk_tables');
-- Ratio = Created_tmp_disk_tables / Created_tmp_tables
-- If > 5% → increase tmp_table_size and max_heap_table_size (both must match)

-- Connection utilization
SELECT
  (SELECT variable_value FROM information_schema.global_status WHERE variable_name='Threads_connected') AS current_connections,
  (SELECT variable_value FROM information_schema.global_variables WHERE variable_name='max_connections') AS max_connections,
  (SELECT variable_value FROM information_schema.global_status WHERE variable_name='Max_used_connections') AS peak_connections;
-- If current ≥ 90% of max_connections → connection pool about to exhaust
-- Fix: Add connection pooler (ProxySQL / MaxScale), or increase max_connections
```

---

## 1.7 — Replication Lag (When Reads Go to a Replica)

```sql
-- Check replication health and lag
SHOW REPLICA STATUS\G    -- MySQL 8.0+
SHOW SLAVE STATUS\G      -- MySQL 5.7 and MariaDB

-- Critical fields to examine:
-- Replica_IO_Running        → Must be YES (IO thread receiving binlog)
-- Replica_SQL_Running       → Must be YES (SQL thread applying relay log)
-- Seconds_Behind_Source     → 0 = in sync; > 10 = lagging; > 60 = problem
-- Last_IO_Error             → Any value = IO thread broken
-- Last_SQL_Error            → Any value = SQL thread broken
-- Exec_Master_Log_Pos vs Read_Master_Log_Pos → Gap = relay log backlog

-- If replica is lagging and parallel replication is off:
SHOW VARIABLES LIKE 'replica_parallel_workers';  -- If 0, enable parallel:
SET GLOBAL replica_parallel_type    = 'LOGICAL_CLOCK';
SET GLOBAL replica_parallel_workers = 8;
STOP REPLICA SQL_THREAD;
START REPLICA SQL_THREAD;
```

---

---

# 🔵 SECTION 2: MARIADB — IN-DEPTH INVESTIGATION

MariaDB shares the MySQL foundation but has distinct tools and features. Key differences are highlighted throughout.

---

## Overview: MariaDB-Specific Considerations

```
MariaDB vs MySQL — Key Diagnostic Differences:
  ├── Uses SHOW SLAVE STATUS (not SHOW REPLICA STATUS in older versions)
  ├── Galera Cluster adds wsrep_ metrics for cluster health
  ├── Spider and ColumnStore storage engines have separate diagnostics
  ├── Thread pool (default in MariaDB) replaces MySQL one-thread-per-connection
  └── EXPLAIN output includes r_rows (actual rows) in EXPLAIN FORMAT=JSON
```

---

## 2.1 — Active Queries & Process List

```sql
-- Full process list
SHOW FULL PROCESSLIST;

-- Via information_schema (MariaDB compatible)
SELECT
  id, user, host, db, command, time, state,
  LEFT(info, 300) AS query_text
FROM information_schema.processlist
WHERE command != 'Sleep'
ORDER BY time DESC;

-- MariaDB-specific: check thread pool status (if thread_handling=pool-of-threads)
SHOW STATUS LIKE 'threadpool%';
-- threadpool_threads         = current worker threads
-- threadpool_queued_requests = requests waiting in queue
-- If queued_requests > 0 constantly → thread pool saturated, increase thread_pool_size
```

---

## 2.2 — Slow Query Log (MariaDB)

```sql
-- Enable via SET GLOBAL (no restart needed)
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;
SET GLOBAL log_queries_not_using_indexes = 'ON';
SET GLOBAL log_slow_verbosity = 'query_plan,innodb,explain';  -- MariaDB-specific: includes EXPLAIN in log
SET GLOBAL log_slow_admin_statements = 'ON';

-- Verify
SHOW VARIABLES LIKE 'slow_query%';
SHOW VARIABLES LIKE 'long_query_time';
SHOW VARIABLES LIKE 'log_slow_verbosity';
```

```bash
# Analyze with mysqldumpslow (works on MariaDB logs too)
mysqldumpslow -s t -t 10 /var/lib/mysql/slow.log

# Or Percona's pt-query-digest
pt-query-digest /var/lib/mysql/slow.log --limit=10 --order-by Query_time:sum
```

---

## 2.3 — Performance Schema & MariaDB Specific Views

```sql
-- Top slow queries (same as MySQL but MariaDB has this view built-in from 10.0)
SELECT
  SCHEMA_NAME,
  LEFT(DIGEST_TEXT, 150)                     AS query_pattern,
  COUNT_STAR                                 AS executions,
  ROUND(AVG_TIMER_WAIT / 1e9, 2)             AS avg_ms,
  ROUND(MAX_TIMER_WAIT / 1e9, 2)             AS max_ms,
  SUM_ROWS_EXAMINED,
  SUM_ROWS_SENT,
  ROUND(SUM_ROWS_EXAMINED / NULLIF(SUM_ROWS_SENT,0)) AS examined_per_returned
FROM performance_schema.events_statements_summary_by_digest
WHERE SCHEMA_NAME IS NOT NULL
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 10;

-- MariaDB EXPLAIN with actual rows (10.1+)
EXPLAIN FORMAT=JSON
SELECT * FROM orders WHERE status='PENDING' AND customer_id = 42;
-- Look for "r_rows" (actual rows read) vs "rows" (estimated)
-- Large difference = stale statistics → run ANALYZE TABLE

-- Update statistics (MariaDB uses persistent statistics)
ANALYZE TABLE orders PERSISTENT FOR ALL;
-- Or rebuild statistics engine-wide:
-- mariadb-check --analyze --all-databases -u root -p
```

---

## 2.4 — Galera Cluster Health (if using MariaDB Galera Cluster)

```sql
-- Core Galera health check — run on every node
SHOW STATUS LIKE 'wsrep%';

-- Most important wsrep variables:
-- wsrep_cluster_size           → number of nodes in the cluster (expected count)
-- wsrep_cluster_status         → PRIMARY = healthy, NON-PRIMARY = split brain
-- wsrep_connected              → ON = node is connected to cluster
-- wsrep_ready                  → ON = node accepting queries
-- wsrep_local_state_comment    → Synced = fully replicated; Donor/Joiner = in progress
-- wsrep_local_recv_queue_avg   → > 0 = node falling behind (flow control pending)
-- wsrep_flow_control_paused    → 0 = none; > 0.1 = 10%+ time paused = serious lag
-- wsrep_cert_deps_distance     → parallelism potential; higher = more parallel replay

-- Detect flow control events (replication pressure indicator)
SELECT variable_name, variable_value
FROM information_schema.global_status
WHERE variable_name IN (
  'wsrep_flow_control_sent',
  'wsrep_flow_control_recv',
  'wsrep_flow_control_paused',
  'wsrep_local_recv_queue',
  'wsrep_local_send_queue'
);
-- wsrep_flow_control_paused > 0 = slow node is throttling the entire cluster

-- Check replication queue on this node
SHOW STATUS LIKE 'wsrep_local_recv_queue%';
-- wsrep_local_recv_queue_avg > 1 = node cannot apply transactions fast enough
-- Fix: Check slow node hardware, disable wsrep_slave_threads is too low
```

---

## 2.5 — MaxScale (MariaDB Connection Router/Load Balancer)

```bash
# If MaxScale is deployed as read/write splitter:
# Check MaxScale is routing correctly
maxctrl list servers
# Output columns: Name, Address, Port, Connections, State, GTID
# State should be: Master (for primary), Slave,Running (for replicas)
# Connections shows current connection distribution

maxctrl list services
# Shows query routing service statistics

# MaxScale real-time stats
maxctrl show service "Read-Write-Service"
# Check: Connections, Total queries, Reads/Writes split ratio

# Check if MaxScale is causing extra latency
maxctrl show maxscale | grep -i "query classification time"
```

---

---

# 🟡 SECTION 3: POSTGRESQL — IN-DEPTH INVESTIGATION

---

## Overview: PostgreSQL Slowness Categories

```
PostgreSQL Slow Application
    │
    ├── A. Lock Contention    → Row/table/advisory locks, idle transactions holding locks
    ├── B. Query Plans        → Bad planner estimates, missing indexes, stale statistics
    ├── C. Autovacuum Bloat   → Dead tuples causing sequential scan overhead
    ├── D. Connection Limits  → Max connections hit, PgBouncer pool exhausted
    ├── E. Checkpoint Storms  → WAL checkpoint flooding I/O
    └── F. Replication Lag    → Read queries hitting stale standby
```

---

## 3.1 — Active Sessions & Wait Events

```sql
-- Complete picture of all non-idle sessions
SELECT
  pid,
  usename                                              AS username,
  application_name,
  client_addr,
  state,
  wait_event_type,
  wait_event,
  NOW() - query_start                                  AS query_duration,
  NOW() - state_change                                 AS state_duration,
  LEFT(query, 250)                                     AS query_snippet,
  backend_type
FROM pg_stat_activity
WHERE state != 'idle'
  AND pid != pg_backend_pid()
ORDER BY query_duration DESC NULLS LAST;
```

**Wait Event Reference — Every Category Explained:**

| `wait_event_type` | `wait_event` | Root Cause | Fix |
|-------------------|-------------|-----------|-----|
| `Lock` | `relation` | Full table-level lock (LOCK TABLE, DDL) | Find DDL/lock holder, kill if safe |
| `Lock` | `tuple` | Waiting for specific row version | Find blocking row update transaction |
| `Lock` | `transactionid` | Waiting for a transaction to end | Find the long-running transaction |
| `Lock` | `virtualxid` | Waiting for virtual transaction | Usually brief — monitor |
| `LWLock` | `BufferMapping` | High contention on buffer pool lookup | Increase `shared_buffers` |
| `LWLock` | `WALWrite` | WAL write contention (concurrent commits) | Increase `wal_buffers`, check disk |
| `LWLock` | `lock_manager` | High lock table contention | Reduce concurrent lock holders |
| `IO` | `DataFileRead` | Reading page from disk (cache miss) | Increase `shared_buffers` or add index |
| `IO` | `DataFileWrite` | Writing dirty page to disk | Checkpoint pressure — tune checkpoint |
| `IO` | `WALWrite` | WAL write to disk | Move WAL to separate faster disk |
| `IO` | `BufFileRead` | Temp file read (sort/hash spilled to disk) | Increase `work_mem` |
| `Client` | `ClientRead` | Waiting for app to send next query | Application-side issue |
| `Client` | `ClientWrite` | App not reading result fast enough | App result processing too slow |

```sql
-- Aggregated wait summary — shows systemic patterns
SELECT
  wait_event_type,
  wait_event,
  COUNT(*)                                             AS sessions_waiting,
  ROUND(AVG(EXTRACT(EPOCH FROM (NOW() - query_start))), 2) AS avg_wait_seconds
FROM pg_stat_activity
WHERE wait_event IS NOT NULL
  AND state != 'idle'
GROUP BY wait_event_type, wait_event
ORDER BY sessions_waiting DESC;
```

---

## 3.2 — Long Running Queries & Safe Termination

```sql
-- All actively running queries sorted by duration
SELECT
  pid,
  usename,
  application_name,
  NOW() - query_start                                  AS running_for,
  state,
  wait_event_type,
  wait_event,
  LEFT(query, 300)                                     AS query_text
FROM pg_stat_activity
WHERE state IN ('active', 'idle in transaction')
  AND query_start IS NOT NULL
  AND pid != pg_backend_pid()
ORDER BY running_for DESC NULLS LAST;

-- Alert: Idle-in-transaction sessions (open transaction, not executing)
-- These sessions HOLD LOCKS without running any query
-- Extremely dangerous — will block VACUUM and block other queries
SELECT
  pid,
  usename,
  NOW() - state_change                                 AS idle_in_transaction_for,
  LEFT(query, 200)                                     AS last_query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
ORDER BY state_change ASC;
-- Fix: Set idle_in_transaction_session_timeout = '5min' in postgresql.conf
```

```sql
-- Safe cancel (query stops, connection stays open — preferred)
SELECT pg_cancel_backend(12345);

-- Hard kill (terminates connection entirely — use if cancel fails after 30s)
SELECT pg_terminate_backend(12345);

-- Cancel all long-running queries over 5 minutes (excluding maintenance tasks)
SELECT
  pid,
  pg_cancel_backend(pid)                               AS cancelled,
  query_start,
  LEFT(query, 200)                                     AS query_text
FROM pg_stat_activity
WHERE state = 'active'
  AND query_start < NOW() - INTERVAL '5 minutes'
  AND query NOT ILIKE '%vacuum%'
  AND query NOT ILIKE '%analyze%'
  AND query NOT ILIKE '%checkpoint%'
  AND pid != pg_backend_pid();
```

**Key timeout settings to prevent future incidents:**
```ini
# postgresql.conf
statement_timeout                    = '10min'   -- Max query execution time
lock_timeout                         = '30s'     -- Max wait for a lock
idle_in_transaction_session_timeout  = '5min'    -- Kill abandoned transactions
```

---

## 3.3 — Blocking Chain Analysis

```sql
-- Standard blocking detection (which query is blocking which)
SELECT
  blocked_locks.pid                                    AS blocked_pid,
  blocked_activity.usename                             AS blocked_user,
  NOW() - blocked_activity.query_start                 AS blocked_duration,
  LEFT(blocked_activity.query, 200)                    AS blocked_query,
  blocking_locks.pid                                   AS blocking_pid,
  blocking_activity.usename                            AS blocking_user,
  NOW() - blocking_activity.query_start                AS blocker_query_duration,
  LEFT(blocking_activity.query, 200)                   AS blocking_query
FROM pg_locks blocked_locks
JOIN pg_stat_activity blocked_activity
  ON blocked_activity.pid = blocked_locks.pid
JOIN pg_locks blocking_locks
  ON blocking_locks.locktype    = blocked_locks.locktype
  AND blocking_locks.database   IS NOT DISTINCT FROM blocked_locks.database
  AND blocking_locks.relation   IS NOT DISTINCT FROM blocked_locks.relation
  AND blocking_locks.page       IS NOT DISTINCT FROM blocked_locks.page
  AND blocking_locks.tuple      IS NOT DISTINCT FROM blocked_locks.tuple
  AND blocking_locks.classid    IS NOT DISTINCT FROM blocked_locks.classid
  AND blocking_locks.objid      IS NOT DISTINCT FROM blocked_locks.objid
  AND blocking_locks.objsubid   IS NOT DISTINCT FROM blocked_locks.objsubid
  AND blocking_locks.pid        != blocked_locks.pid
  AND blocking_locks.granted    = true
  AND blocked_locks.granted     = false
JOIN pg_stat_activity blocking_activity
  ON blocking_activity.pid = blocking_locks.pid
ORDER BY blocked_duration DESC;

-- Find the ROOT blocker (top of the chain) — kill this first
SELECT
  pid,
  usename,
  NOW() - query_start                                  AS running_for,
  state,
  LEFT(query, 200)                                     AS query_text
FROM pg_stat_activity
WHERE pid IN (
  SELECT DISTINCT blocking_locks.pid
  FROM pg_locks blocked_locks
  JOIN pg_locks blocking_locks
    ON blocking_locks.locktype  = blocked_locks.locktype
    AND blocking_locks.pid     != blocked_locks.pid
    AND blocking_locks.granted  = true
    AND blocked_locks.granted   = false
)
ORDER BY running_for DESC NULLS LAST;
```

---

## 3.4 — pg_stat_statements — Query Performance Analysis

```sql
-- Verify extension is available
SELECT * FROM pg_available_extensions WHERE name = 'pg_stat_statements';
-- If not installed:
-- 1. Add to postgresql.conf: shared_preload_libraries = 'pg_stat_statements'
-- 2. Restart PostgreSQL
-- 3. CREATE EXTENSION pg_stat_statements;

-- Top 10 by total execution time (what's consuming most DB time overall)
SELECT
  LEFT(query, 150)                                     AS query_snippet,
  calls,
  ROUND(total_exec_time::numeric, 2)                   AS total_ms,
  ROUND(mean_exec_time::numeric, 2)                    AS avg_ms,
  ROUND(max_exec_time::numeric, 2)                     AS max_ms,
  ROUND(stddev_exec_time::numeric, 2)                  AS stddev_ms,
  rows,
  ROUND((total_exec_time / SUM(total_exec_time) OVER()) * 100, 2) AS pct_of_total_time
FROM pg_stat_statements
WHERE calls > 5
ORDER BY total_exec_time DESC
LIMIT 10;

-- Top 10 by average time (slowest individual calls — most actionable)
SELECT
  LEFT(query, 150)                                     AS query_snippet,
  calls,
  ROUND(mean_exec_time::numeric, 2)                    AS avg_ms,
  ROUND(max_exec_time::numeric, 2)                     AS max_ms,
  ROUND(stddev_exec_time::numeric, 2)                  AS variance_ms
FROM pg_stat_statements
WHERE calls > 10
ORDER BY mean_exec_time DESC
LIMIT 10;
-- High stddev_ms relative to avg_ms = inconsistent query — suspect lock waits or plan changes

-- Queries causing disk reads (temp file spills — work_mem too small)
SELECT
  LEFT(query, 150)                                     AS query_snippet,
  calls,
  ROUND(mean_exec_time::numeric, 2)                    AS avg_ms,
  shared_blks_hit                                      AS cache_hits,
  shared_blks_read                                     AS disk_reads,
  ROUND(shared_blks_hit::numeric / NULLIF(shared_blks_hit + shared_blks_read, 0) * 100, 2) AS cache_hit_pct,
  temp_blks_written                                    -- > 0 = spilling to disk
FROM pg_stat_statements
WHERE temp_blks_written > 0
  OR shared_blks_read > 1000
ORDER BY temp_blks_written DESC, shared_blks_read DESC
LIMIT 10;
-- temp_blks_written > 0 → increase work_mem for that session or globally
-- cache_hit_pct < 95% → data not in shared_buffers → increase shared_buffers

-- Reset stats (after applying a fix, start fresh measurement)
SELECT pg_stat_statements_reset();
```

---

## 3.5 — EXPLAIN ANALYZE — Reading the Query Plan

```sql
-- Full diagnostic EXPLAIN (always use BUFFERS to see disk vs cache)
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, FORMAT TEXT)
SELECT o.id, o.status, o.created_at, c.name, c.email
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.status = 'PENDING'
  AND o.created_at BETWEEN '2024-01-01' AND '2024-03-01'
ORDER BY o.created_at DESC
LIMIT 100;
```

**Plan Node Reference:**

| Node Type | Meaning | Red Flag | Fix |
|-----------|---------|----------|-----|
| `Seq Scan` | Reads every row in the table | On tables > 10k rows with a WHERE clause | Add index on filter columns |
| `Index Scan` | Uses index to find rows, then fetches from heap | Normal — check rows accuracy | Verify statistics are current |
| `Index Only Scan` | Reads from index only (no heap lookup) | None — this is optimal | — |
| `Bitmap Heap Scan` | Uses bitmap from index, then reads heap | Normal for range queries | Fine unless "Lossy" appears |
| `Hash Join` | Builds hash table of smaller set | `Batches > 1` = disk spill | Increase `work_mem` |
| `Merge Join` | Requires sorted input on both sides | High sort cost | Add index on join columns |
| `Nested Loop` | For each outer row, scans inner | Outer has millions of rows | Verify inner has good index |
| `Sort` | External sort (no index order) | Large "Sort Method: external" | Increase `work_mem`; add index |

```sql
-- Identify tables relying on sequential scans on large tables
SELECT
  schemaname,
  relname                                              AS table_name,
  seq_scan,
  idx_scan,
  ROUND(seq_scan::numeric * 100 / NULLIF(seq_scan + idx_scan, 0), 2) AS seq_scan_pct,
  pg_size_pretty(pg_total_relation_size(schemaname || '.' || relname)) AS table_size
FROM pg_stat_user_tables
WHERE seq_scan > 50
  AND pg_total_relation_size(schemaname || '.' || relname) > 50 * 1024 * 1024
ORDER BY seq_scan DESC;
-- Tables with high seq_scan_pct on large tables = strong index candidates
```

---

## 3.6 — Autovacuum & Table Bloat

```sql
-- Tables with dead tuple buildup (bloat candidates)
SELECT
  schemaname || '.' || relname                         AS table_name,
  n_live_tup,
  n_dead_tup,
  ROUND(n_dead_tup::numeric * 100 / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_tuple_pct,
  pg_size_pretty(pg_total_relation_size(schemaname || '.' || relname)) AS total_size,
  last_autovacuum,
  last_autoanalyze,
  autovacuum_count,
  n_mod_since_analyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC
LIMIT 20;
-- dead_tuple_pct > 10% = bloat accumulating — sequential scans become slower
-- last_autovacuum NULL = autovacuum never ran (very new table, or it's disabled)

-- Autovacuum currently running (progress tracking)
SELECT
  pid,
  relid::regclass                                      AS table_being_vacuumed,
  phase,
  heap_blks_scanned,
  heap_blks_total,
  ROUND(heap_blks_scanned::numeric * 100 / NULLIF(heap_blks_total,0), 1) AS pct_complete
FROM pg_stat_progress_vacuum;

-- Transaction ID wraparound — CRITICAL safety check
SELECT
  datname,
  age(datfrozenxid)                                    AS xid_age,
  ROUND(age(datfrozenxid)::numeric * 100 / 2147483648, 2) AS pct_to_wraparound
FROM pg_database
ORDER BY age(datfrozenxid) DESC;
-- pct_to_wraparound > 50% → Schedule VACUUM FREEZE immediately
-- pct_to_wraparound > 80% → Database will start refusing writes — emergency

-- Force vacuum on a bloated table
VACUUM VERBOSE ANALYZE orders;

-- For severe bloat — pg_repack (online, no table lock)
-- apt install postgresql-16-repack
-- pg_repack -d mydb -t orders
```

---

## 3.7 — Cache Hit Ratio & I/O Health

```sql
-- Database-level cache hit ratio
SELECT
  datname,
  blks_hit,
  blks_read,
  ROUND(blks_hit::numeric * 100 / NULLIF(blks_hit + blks_read, 0), 4) AS cache_hit_pct,
  tup_fetched,
  tup_returned,
  deadlocks,
  conflicts
FROM pg_stat_database
WHERE datname NOT IN ('template0', 'template1')
ORDER BY blks_read DESC;
-- cache_hit_pct should be > 99%
-- If < 99% and you have RAM → increase shared_buffers

-- Per-table cache efficiency (find cold hot tables)
SELECT
  relname,
  heap_blks_hit,
  heap_blks_read,
  ROUND(heap_blks_hit::numeric * 100 / NULLIF(heap_blks_hit + heap_blks_read, 0), 2) AS table_cache_hit_pct,
  idx_blks_hit,
  idx_blks_read,
  ROUND(idx_blks_hit::numeric * 100 / NULLIF(idx_blks_hit + idx_blks_read, 0), 2) AS index_cache_hit_pct
FROM pg_statio_user_tables
WHERE heap_blks_hit + heap_blks_read > 100
ORDER BY heap_blks_read DESC
LIMIT 20;

-- Background writer and checkpoint health
SELECT
  checkpoints_timed,
  checkpoints_req,         -- If high = too many forced checkpoints (increase max_wal_size)
  checkpoint_write_time,
  checkpoint_sync_time,
  buffers_checkpoint,
  buffers_clean,
  maxwritten_clean,        -- If > 0 = bgwriter falling behind (increase bgwriter_lru_maxpages)
  buffers_backend,         -- If high = app processes writing to disk directly (needs tuning)
  buffers_alloc
FROM pg_stat_bgwriter;
```

---

---

# 🟣 SECTION 4: SQL SERVER — IN-DEPTH INVESTIGATION

---

## Overview: SQL Server Slowness Categories

```
SQL Server Slow Application
    │
    ├── A. Blocking / Deadlocks  → Lock chains, long transactions
    ├── B. Bad Query Plans       → Parameter sniffing, stale stats, missing indexes
    ├── C. Wait Stats            → I/O, CPU, memory grant, parallelism waits
    ├── D. Memory Pressure       → Buffer pool eviction, memory grant starvation
    ├── E. TempDB Contention     → Sort/hash spills, temp table storms
    └── F. Fragmentation         → Index fragmentation causing slow reads
```

---

## 4.1 — Active Requests & Session Overview

```sql
-- Complete picture of all active database requests
SELECT
  r.session_id,
  r.status,
  r.blocking_session_id,
  r.wait_type,
  r.wait_time      / 1000.0                            AS wait_seconds,
  r.cpu_time       / 1000.0                            AS cpu_seconds,
  r.total_elapsed_time / 1000.0                        AS elapsed_seconds,
  r.logical_reads,
  r.writes,
  r.row_count,
  DB_NAME(r.database_id)                               AS database_name,
  s.login_name,
  s.host_name,
  s.program_name,
  LEFT(t.text, 400)                                    AS query_text,
  qp.query_plan
FROM sys.dm_exec_requests r
JOIN sys.dm_exec_sessions s
  ON r.session_id = s.session_id
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) t
OUTER APPLY sys.dm_exec_query_plan(r.plan_handle) qp
WHERE r.session_id > 50
  AND r.status != 'background'
ORDER BY r.total_elapsed_time DESC;
```

**Request Status Reference:**

| Status | What's Happening | Action |
|--------|-----------------|--------|
| `running` | Actively consuming CPU | Check EXPLAIN / execution plan |
| `runnable` | Waiting for CPU slot | CPU saturation — check `SOS_SCHEDULER_YIELD` |
| `suspended` | Waiting for resource (I/O, lock, memory grant) | Check `wait_type` column |
| `sleeping` | Session idle, connection open — may hold locks | Check for open transactions |
| `background` | System internal task | Ignore unless causing waits |

---

## 4.2 — Blocking Chains — Complete Analysis

```sql
-- Full blocking picture: who blocks whom
SELECT
  r.session_id                                         AS blocked_session,
  r.blocking_session_id                                AS blocking_session,
  r.wait_type,
  r.wait_time / 1000.0                                 AS waiting_seconds,
  DB_NAME(r.database_id)                               AS database_name,
  s.login_name,
  s.host_name,
  LEFT(t.text, 300)                                    AS blocked_query_text,
  qp.query_plan                                        AS blocked_query_plan
FROM sys.dm_exec_requests r
JOIN sys.dm_exec_sessions s
  ON r.session_id = s.session_id
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) t
OUTER APPLY sys.dm_exec_query_plan(r.plan_handle) qp
WHERE r.blocking_session_id != 0
ORDER BY r.wait_time DESC;

-- Find the root blocker (not blocked by anyone else)
SELECT
  blocking_session_id                                  AS root_blocker,
  COUNT(*)                                             AS sessions_it_blocks,
  MAX(wait_time) / 1000.0                              AS max_wait_sec
FROM sys.dm_exec_requests
WHERE blocking_session_id != 0
GROUP BY blocking_session_id
ORDER BY sessions_it_blocks DESC;

-- Get the exact SQL being run by the root blocker
DBCC INPUTBUFFER(<root_blocker_session_id>);

-- Get full query text of root blocker via DMV
SELECT t.text AS root_blocker_query
FROM sys.dm_exec_sessions s
CROSS APPLY sys.dm_exec_sql_text(s.most_recent_sql_handle) t
WHERE s.session_id = <root_blocker_session_id>;

-- Kill root blocker (only after confirming it's safe)
KILL <root_blocker_session_id>;
```

**Blocking Root Causes & Fixes:**

| Cause | Symptom | Fix |
|-------|---------|-----|
| Long-running transaction | Old `trx_started`, holding rows locked | Find and commit/rollback |
| Missing index on join column | Table scan holding many row locks | Add index on join/filter column |
| Inappropriate isolation level | SERIALIZABLE with heavy reads | Change to `READ COMMITTED` or use RCSI |
| Missing `WITH (NOLOCK)` on reporting queries | Reports blocking OLTP | Add NOLOCK for read-only reporting (accept dirty reads) |
| Batch jobs during peak hours | Batch locks many rows | Schedule batch jobs off-peak |

---

## 4.3 — Deadlock Detection & Analysis

```sql
-- Enable system health trace to capture deadlocks (already on by default in modern SQL Server)
-- Query the system health extended event session:
SELECT
  xdr.value('@timestamp', 'datetime2') AS deadlock_time,
  xdr.query('.') AS deadlock_xml
FROM (
  SELECT CAST(target_data AS XML) AS target_data
  FROM sys.dm_xe_session_targets t
  JOIN sys.dm_xe_sessions s ON s.address = t.event_session_address
  WHERE s.name = 'system_health'
    AND t.target_name = 'ring_buffer'
) data
CROSS APPLY target_data.nodes('//RingBufferTarget/event[@name="xml_deadlock_report"]') AS xdt(xdr)
ORDER BY deadlock_time DESC;

-- Enable trace flag for deadlock logging to error log (if needed)
DBCC TRACEON(1222, -1);    -- detailed deadlock info
DBCC TRACEON(1204, -1);    -- summary deadlock info

-- Verify trace flags active
DBCC TRACESTATUS(-1);
```

---

## 4.4 — Top Resource-Consuming Queries

```sql
-- Top 10 by total CPU (cumulative cost — fix these for biggest impact)
SELECT TOP 10
  qs.total_worker_time / qs.execution_count / 1000.0  AS avg_cpu_ms,
  qs.total_worker_time / 1000.0                        AS total_cpu_ms,
  qs.execution_count,
  qs.total_elapsed_time / qs.execution_count / 1000.0  AS avg_elapsed_ms,
  qs.total_logical_reads / qs.execution_count          AS avg_logical_reads,
  qs.total_physical_reads / qs.execution_count         AS avg_physical_reads,
  qs.total_logical_writes / qs.execution_count         AS avg_writes,
  DB_NAME(qt.dbid)                                     AS database_name,
  SUBSTRING(
    qt.text,
    (qs.statement_start_offset / 2) + 1,
    ((CASE qs.statement_end_offset
        WHEN -1 THEN DATALENGTH(qt.text)
        ELSE qs.statement_end_offset
      END - qs.statement_start_offset) / 2) + 1
  )                                                    AS query_text,
  qp.query_plan
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) qt
OUTER APPLY sys.dm_exec_query_plan(qs.plan_handle) qp
ORDER BY qs.total_worker_time DESC;

-- Top 10 by physical reads (reading from disk — memory or index problem)
SELECT TOP 10
  qs.total_physical_reads / NULLIF(qs.execution_count, 0) AS avg_physical_reads,
  qs.execution_count,
  SUBSTRING(qt.text, (qs.statement_start_offset/2)+1,
    ((CASE qs.statement_end_offset WHEN -1 THEN DATALENGTH(qt.text)
      ELSE qs.statement_end_offset END - qs.statement_start_offset)/2)+1) AS query_text
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) qt
ORDER BY qs.total_physical_reads DESC;
-- High physical reads = buffer pool miss → add memory or add index to reduce rows read
```

---

## 4.5 — Wait Stats — The Most Powerful SQL Server Diagnostic

```sql
-- System-wide wait statistics (exclude benign system waits)
SELECT
  wait_type,
  waiting_tasks_count,
  wait_time_ms / 1000.0                               AS wait_time_sec,
  max_wait_time_ms / 1000.0                           AS max_wait_sec,
  signal_wait_time_ms / 1000.0                        AS cpu_wait_sec,
  (wait_time_ms - signal_wait_time_ms) / 1000.0       AS resource_wait_sec,
  ROUND(wait_time_ms * 100.0 / SUM(wait_time_ms) OVER(), 2) AS pct_of_total
FROM sys.dm_os_wait_stats
WHERE wait_type NOT IN (
  'SLEEP_TASK','WAIT_XTP_OFFLINE_CKPT_NEW_LOG','DISPATCHER_QUEUE_SEMAPHORE',
  'BROKER_TO_FLUSH','BROKER_TASK_STOP','CLR_AUTO_EVENT','CLR_MANUAL_EVENT',
  'DBMIRROR_EVENTS_QUEUE','SQLTRACE_BUFFER_FLUSH','FT_IFTS_SCHEDULER_IDLE_WAIT',
  'XE_DISPATCHER_WAIT','XE_TIMER_EVENT','WAITFOR','LAZYWRITER_SLEEP',
  'CHECKPOINT_QUEUE','REQUEST_FOR_DEADLOCK_SEARCH','RESOURCE_QUEUE',
  'SERVER_IDLE_CHECK','SLEEP_DBSTARTUP','SLEEP_DCOMSTARTUP','SLEEP_MASTERDBREADY',
  'SLEEP_MASTERMDREADY','SLEEP_MASTERUPGRADED','SLEEP_MSDBSTARTUP',
  'SLEEP_SYSTEMTASK','SLEEP_TEMPDBSTARTUP','SNI_HTTP_ACCEPT',
  'SP_SERVER_DIAGNOSTICS_SLEEP','SQLTRACE_INCREMENTAL_FLUSH_SLEEP'
)
ORDER BY wait_time_ms DESC;
```

**Wait Type Complete Reference:**

| Wait Type | Root Cause | Diagnostic Query | Fix |
|-----------|-----------|-----------------|-----|
| `PAGEIOLATCH_SH` | Reading data pages from disk (cache miss) | Check buffer pool hit ratio | Add RAM, increase max server memory, add indexes |
| `PAGEIOLATCH_EX` | Writing dirty pages to disk | Check checkpoint frequency | Faster storage, tune max dirty pages |
| `LCK_M_X` | Exclusive row/page/table lock wait | Check blocking tree | Fix long transactions, add row-level indexes |
| `LCK_M_S` | Shared lock wait | Check blocking tree | Enable RCSI, use NOLOCK for reads |
| `CXPACKET` | Parallel query thread synchronization skew | Check MAXDOP setting | Reduce MAXDOP, tune `cost threshold for parallelism` |
| `CXCONSUMER` | Consumer thread waiting for producer | Same as CXPACKET | Fix parallel plan unbalanced distribution |
| `SOS_SCHEDULER_YIELD` | CPU saturation — thread yielding scheduler | Check CPU usage | Add CPU cores, fix bad query plans |
| `WRITELOG` | Transaction log sync to disk | Check log disk throughput | Move log to faster SSD, increase log file size |
| `ASYNC_NETWORK_IO` | App not consuming result sets fast enough | Check application code | Reduce result set size, add pagination |
| `RESOURCE_SEMAPHORE` | Memory grant queue — queries waiting for RAM | Check memory grants | Add RAM, fix missing indexes that cause huge scans |
| `THREADPOOL` | No worker thread available | Check max worker threads | Increase `max worker threads`, kill idle sessions |
| `OLEDB` | Linked server roundtrip | Check linked server calls | Remove linked servers or cache results |
| `HADR_SYNC_COMMIT` | AlwaysOn synchronous replica is lagging | Check AG replica health | Switch to async, check network to replica |

```sql
-- Clear wait stats after fixing an issue (start fresh measurement)
DBCC SQLPERF('sys.dm_os_wait_stats', CLEAR);
```

---

## 4.6 — Missing & Fragmented Index Analysis

```sql
-- Top missing indexes by estimated impact (do NOT just create all of them — review first)
SELECT TOP 20
  DB_NAME(mid.database_id)                             AS database_name,
  OBJECT_NAME(mid.object_id, mid.database_id)          AS table_name,
  migs.avg_total_user_cost * migs.avg_user_impact * (migs.user_seeks + migs.user_scans)
                                                       AS impact_score,
  migs.user_seeks,
  migs.user_scans,
  migs.avg_user_impact                                 AS estimated_improvement_pct,
  mid.equality_columns,
  mid.inequality_columns,
  mid.included_columns,
  'CREATE NONCLUSTERED INDEX [IX_' + OBJECT_NAME(mid.object_id, mid.database_id)
    + '_missing_' + CAST(ROW_NUMBER() OVER(ORDER BY migs.avg_total_user_cost * migs.avg_user_impact DESC) AS VARCHAR)
    + '] ON ' + mid.statement
    + ' (' + ISNULL(mid.equality_columns, '')
    + CASE WHEN mid.equality_columns IS NOT NULL AND mid.inequality_columns IS NOT NULL THEN ', ' ELSE '' END
    + ISNULL(mid.inequality_columns, '') + ')'
    + ISNULL(' INCLUDE (' + mid.included_columns + ')', '')
                                                       AS suggested_create_statement
FROM sys.dm_db_missing_index_details mid
JOIN sys.dm_db_missing_index_groups mig
  ON mid.index_handle = mig.index_handle
JOIN sys.dm_db_missing_index_group_stats migs
  ON mig.index_group_handle = migs.group_handle
WHERE mid.database_id = DB_ID()
ORDER BY impact_score DESC;

-- Index fragmentation check (for rebuild/reorganize decisions)
SELECT
  OBJECT_NAME(ps.OBJECT_ID)                           AS table_name,
  i.name                                              AS index_name,
  ps.index_type_desc,
  ps.avg_fragmentation_in_percent,
  ps.page_count,
  CASE
    WHEN ps.avg_fragmentation_in_percent > 30 THEN 'REBUILD'
    WHEN ps.avg_fragmentation_in_percent > 10 THEN 'REORGANIZE'
    ELSE 'OK'
  END                                                 AS recommendation
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'LIMITED') ps
JOIN sys.indexes i
  ON ps.OBJECT_ID = i.OBJECT_ID AND ps.index_id = i.index_id
WHERE ps.page_count > 1000        -- only meaningful for larger indexes
  AND ps.avg_fragmentation_in_percent > 10
ORDER BY ps.avg_fragmentation_in_percent DESC;

-- Rebuild a fragmented index (with online option to avoid blocking)
ALTER INDEX [IX_orders_customer_id] ON dbo.orders REBUILD WITH (ONLINE = ON);

-- Reorganize (lighter operation, no downtime)
ALTER INDEX [IX_orders_customer_id] ON dbo.orders REORGANIZE;
```

---

## 4.7 — Memory Analysis

```sql
-- SQL Server memory configuration and usage
SELECT
  physical_memory_in_use_kb / 1024.0                  AS memory_used_mb,
  page_fault_count,
  memory_utilization_percentage
FROM sys.dm_os_process_memory;

-- Buffer pool usage by database
SELECT
  DB_NAME(database_id)                                AS database_name,
  COUNT(*) * 8 / 1024.0                               AS buffer_pool_mb,
  SUM(CASE WHEN is_modified = 1 THEN 1 ELSE 0 END) * 8 / 1024.0 AS dirty_pages_mb
FROM sys.dm_os_buffer_descriptors
WHERE database_id != 32767
GROUP BY database_id
ORDER BY buffer_pool_mb DESC;

-- Memory grant queue (> 0 rows = queries waiting for memory = serious pressure)
SELECT
  session_id,
  requested_memory_kb / 1024.0                        AS requested_mb,
  granted_memory_kb / 1024.0                          AS granted_mb,
  used_memory_kb / 1024.0                             AS used_mb,
  queue_id,
  wait_order,
  is_next_candidate
FROM sys.dm_exec_query_memory_grants
ORDER BY requested_memory_kb DESC;
-- Any rows with grant_time IS NULL = query waiting for memory grant
-- Fix: Add more RAM, add missing indexes (reduce rows scanned), lower max server memory fragmentation

-- Plan cache efficiency
SELECT
  SUM(CASE WHEN usecounts = 1 THEN 1 ELSE 0 END)     AS single_use_plans,
  SUM(usecounts)                                      AS total_plan_usecounts,
  COUNT(*)                                            AS total_plans_cached,
  SUM(size_in_bytes) / 1024.0 / 1024.0               AS cache_size_mb
FROM sys.dm_exec_cached_plans;
-- single_use_plans / total_plans_cached > 20% = ad-hoc SQL pollution
-- Fix: Enable 'optimize for ad hoc workloads' = 1 in sp_configure
```

---

## 4.8 — TempDB & Transaction Log Health

```sql
-- TempDB space usage (high = sort/hash spills or temp table explosion)
SELECT
  SUM(user_object_reserved_page_count) * 8 / 1024.0  AS user_objects_mb,
  SUM(internal_object_reserved_page_count) * 8 / 1024.0 AS sort_hash_spills_mb,
  SUM(version_store_reserved_page_count) * 8 / 1024.0 AS version_store_mb,
  SUM(unallocated_extent_page_count) * 8 / 1024.0    AS free_space_mb
FROM sys.dm_db_file_space_usage
WHERE database_id = 2;  -- TempDB is always database_id 2

-- Sessions consuming the most TempDB
SELECT TOP 10
  t.session_id,
  s.login_name,
  s.host_name,
  (t.internal_objects_alloc_page_count + t.user_objects_alloc_page_count) * 8 / 1024.0 AS tempdb_mb,
  LEFT(st.text, 200) AS query_text
FROM sys.dm_db_task_space_usage t
JOIN sys.dm_exec_sessions s ON t.session_id = s.session_id
CROSS APPLY sys.dm_exec_sql_text(s.most_recent_sql_handle) st
GROUP BY t.session_id, s.login_name, s.host_name, t.internal_objects_alloc_page_count,
         t.user_objects_alloc_page_count, st.text
ORDER BY tempdb_mb DESC;

-- Transaction log space usage and why it can't truncate
SELECT
  DB_NAME(database_id)                                AS database_name,
  name                                                AS log_file_name,
  size * 8 / 1024.0                                   AS log_file_mb,
  log_reuse_wait_desc                                 AS cannot_truncate_because
FROM sys.master_files mf
JOIN sys.databases d ON mf.database_id = d.database_id
WHERE mf.type = 1
ORDER BY size DESC;

-- log_reuse_wait_desc values and fixes:
-- ACTIVE_TRANSACTION    → Long-running transaction holding the log — find and commit/rollback
-- LOG_BACKUP            → Log backup not running — check backup job
-- REPLICATION           → Replication latency — check distributor
-- DATABASE_MIRRORING    → Mirror lag — check mirror partner
-- AVAILABILITY_REPLICA  → AlwaysOn replica not catching up
```

---

---

# 🚦 SECTION 5: ROOT CAUSE DECISION TREE & QUICK REFERENCE

---

## 5.1 — Complete Diagnostic Flowchart

```
APPLICATION REPORTED SLOW
        │
        ▼
┌─────────────────────────────────────────────────────────┐
│  STEP 0: Server Health (CPU, Memory, Disk, Network)     │
└─────────────────────────────────────────────────────────┘
        │
        ├── CPU iowait > 10%?
        │       └──→ DISK BOTTLENECK
        │             Check: iostat → which device
        │             Then: buffer pool size, index coverage
        │
        ├── Swap used by DB process?
        │       └──→ MEMORY PRESSURE (CRITICAL)
        │             Fix: Add RAM or reduce buffer pool size
        │
        ├── CPU user > 80%?
        │       └──→ QUERY PROCESSING OVERHEAD
        │             Check: pg_stat_statements, slow log, DMVs
        │
        └── All clean?
                └──→ Continue to DB-specific checks
                        │
                        ▼
┌─────────────────────────────────────────────────────────┐
│  STEP 1: Split App Time vs DB Time                      │
└─────────────────────────────────────────────────────────┘
        │
        ├── DB time ≈ App time?
        │       └──→ DATABASE IS THE PROBLEM → Step 2
        │
        └── DB time << App time?
                └──→ APPLICATION LAYER PROBLEM
                      Check: ORM logs, N+1 queries,
                             connection pool exhaustion,
                             app server CPU/memory
                        │
                        ▼
┌─────────────────────────────────────────────────────────┐
│  STEP 2: Check Locking (fastest to diagnose)            │
└─────────────────────────────────────────────────────────┘
        │
        ├── Blocking sessions found?
        │       ├── Root blocker: idle transaction
        │       │       └──→ App not releasing connection
        │       │             Fix: connection timeout settings
        │       │
        │       ├── Root blocker: long-running query
        │       │       └──→ EXPLAIN → add index or rewrite
        │       │
        │       └── Root blocker: batch/maintenance job
        │               └──→ Schedule off-peak or use lower isolation
        │
        └── No blocking?
                └──→ Continue to query analysis
                        │
                        ▼
┌─────────────────────────────────────────────────────────┐
│  STEP 3: Identify Slow Query                            │
└─────────────────────────────────────────────────────────┘
        │
        ├── Full table scan on large table?
        │       └──→ ADD INDEX on filter/join column
        │
        ├── Index exists but bad row estimate?
        │       └──→ UPDATE STATISTICS / ANALYZE TABLE
        │
        ├── Query fast alone, slow under concurrent load?
        │       └──→ RESOURCE CONTENTION → check wait events
        │
        └── Plan changed after deployment?
                └──→ PARAMETER SNIFFING (SQL Server)
                     or STATISTICS DRIFT → sp_recompile / ANALYZE
```

---

## 5.2 — Cross-Database Quick Reference Card

| Symptom | MySQL | MariaDB | PostgreSQL | SQL Server |
|---------|-------|---------|------------|------------|
| **Active queries** | `SHOW PROCESSLIST` | `SHOW PROCESSLIST` | `pg_stat_activity` | `sys.dm_exec_requests` |
| **Slow queries** | `perf_schema` digest | slow log + `log_slow_verbosity` | `pg_stat_statements` | `dm_exec_query_stats` |
| **Lock waits** | `INNODB_LOCK_WAITS` | `INNODB_LOCK_WAITS` | `pg_locks` join | `blocking_session_id` |
| **Deadlocks** | `SHOW ENGINE INNODB STATUS` | Same + `SHOW ENGINE INNODB STATUS` | `pg_locks` + `log_lock_waits` | System Health XE + Trace 1222 |
| **Cache hit ratio** | `buffer_pool_hit_pct` query | Same | `pg_stat_database` | `dm_os_buffer_descriptors` |
| **Missing indexes** | `sys.statements_with_full_table_scans` | `EXPLAIN FORMAT=JSON` r_rows | `pg_stat_user_tables` seq_scan | `dm_db_missing_index_details` |
| **Temp spills** | `Created_tmp_disk_tables` | Same | `temp_blks_written` | TempDB `internal_objects_mb` |
| **I/O bottleneck** | `iowait` + `iostat` | Same | `DataFileRead` wait event | `PAGEIOLATCH_*` waits |
| **Replication lag** | `SHOW REPLICA STATUS` | `SHOW SLAVE STATUS` + `wsrep_*` | `pg_stat_replication` | `dm_hadr_database_replica_states` |
| **Connection exhaustion** | `Threads_connected / max_connections` | `threadpool_queued_requests` | `pg_stat_activity` count | `sys.dm_exec_sessions` count |
| **Statistics refresh** | `ANALYZE TABLE t` | `ANALYZE TABLE t PERSISTENT FOR ALL` | `ANALYZE t` | `UPDATE STATISTICS t` |

---

## 5.3 — Emergency Response Cheat Sheet

```sql
-- ═══════════════════════════════════════════════════════
-- MYSQL / MARIADB EMERGENCY
-- ═══════════════════════════════════════════════════════

-- 1. Find root blocker
SELECT id, user, time, state, LEFT(info, 200)
FROM information_schema.processlist
WHERE time > 60 AND command != 'Sleep'
ORDER BY time DESC;

-- 2. Kill it
KILL QUERY <thread_id>;

-- 3. If deadlock storm — enable logging
SET GLOBAL innodb_print_all_deadlocks = ON;

-- ═══════════════════════════════════════════════════════
-- POSTGRESQL EMERGENCY
-- ═══════════════════════════════════════════════════════

-- 1. Find all blocked sessions
SELECT pid, usename, NOW()-query_start AS dur, LEFT(query,100)
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY dur DESC NULLS LAST;

-- 2. Cancel the long-running query (safe)
SELECT pg_cancel_backend(<pid>);

-- 3. Kill if cancel doesn't work
SELECT pg_terminate_backend(<pid>);

-- ═══════════════════════════════════════════════════════
-- SQL SERVER EMERGENCY
-- ═══════════════════════════════════════════════════════

-- 1. Find root blocker
SELECT blocking_session_id, COUNT(*) AS sessions_blocked
FROM sys.dm_exec_requests WHERE blocking_session_id != 0
GROUP BY blocking_session_id
ORDER BY sessions_blocked DESC;

-- 2. Get blocker's SQL
DBCC INPUTBUFFER(<blocking_session_id>);

-- 3. Kill it
KILL <blocking_session_id>;
```

---

## 5.4 — Post-Incident Checklist

```
After resolving the incident:

□ 1. Root cause documented (5-WHY analysis completed)
□ 2. Slow query captured and EXPLAIN plan reviewed
□ 3. Index added/tested in staging before production (if applicable)
□ 4. Statistics refreshed on affected tables
□ 5. Connection pool settings validated (min/max/timeout)
□ 6. Autovacuum / auto-stats settings reviewed (PG/MySQL)
□ 7. Buffer pool / shared_buffers hit ratio confirmed > 99%
□ 8. Monitoring alert thresholds reviewed — was alert too late?
□ 9. Timeout settings added (statement_timeout, lock_timeout, wait_timeout)
□ 10. Replication lag checked if replicas serve reads
□ 11. Application team informed of any query changes needed
□ 12. Postmortem meeting scheduled if incident lasted > 30 minutes
□ 13. Runbook updated with new command/finding
□ 14. Alerting added for the specific metric that triggered this (if missing)
```

---

*Playbook | Sr DBA & Multi-Cloud DBA — Y. Venkat Sarath*
*Covers: MySQL 5.7–8.0 | MariaDB 10.x + Galera | PostgreSQL 12–16 | SQL Server 2016–2022*
