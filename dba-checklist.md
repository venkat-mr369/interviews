**In-depth DBA Troubleshooting Playbook** 
---
**Step 0 — Universal Pre-Checks**
CPU (per-core, iowait, steal %), Memory (swap detection, which process is swapping), Disk I/O (iostat column-by-column reference), Network latency (with thresholds), App vs DB time split methodology

**Section 1 — MySQL Deep Dive**
Full processlist with state meanings table, Performance Schema with `examined_per_sent` ratio analysis, InnoDB lock wait chain query, deadlock root cause decision tree, EXPLAIN column reference with fixes, buffer pool hit ratio, replication lag diagnosis

**Section 2 — PostgreSQL Deep Dive**
Wait event type reference table (all critical events explained), blocking chain recursive CTE, pg_stat_statements with cache hit per query + temp spill detection, autovacuum bloat analysis, transaction ID wraparound danger check, checkpoint health, EXPLAIN plan node reference table, sequential scan finder

**Section 3 — SQL Server Deep Dive**
Full blocking tree with recursive CTE, top CPU/physical reads queries with plan extraction, complete wait stats with all 15+ wait types explained with fixes, missing index DMV with auto-generated CREATE INDEX statements, unused index finder, buffer pool by database, log reuse wait diagnosis, TempDB spill analysis per session

**Section 4 — Root Cause Decision Tree**
Complete diagnostic flowchart, cross-database quick reference card, emergency kill protocol for all 3 databases, post-incident tuning checklist
===
### 🚨 DBA Troubleshooting Playbook 
### Application Slowness | MySQL • PostgreSQL • SQL Server
#### Full Command Reference + Interpretation + Root Cause + Fix

---

> **How to Use This Playbook:**
> Run checks in order — Step 0 → Step 1 → Step 2. Each step narrows the root cause.
> Every command includes: *What it does → What to look for → What it means → How to fix it.*

---

# 📋 STEP 0: UNIVERSAL PRE-CHECKS (ALL DATABASES)

Before touching any database, rule out infrastructure-level causes first. Most "slow database" reports are actually slow networks, starved CPUs, or saturated disks.

---

## 0.1 — CPU Check

```bash
# Linux
top -bn1 | head -20
mpstat -P ALL 1 3       # per-core breakdown
sar -u 1 5              # historical CPU trend

# What to look for:
# %us (user) > 80%  → App/DB query processing overhead
# %sy (system) > 20% → Kernel I/O or context switching
# %wa (iowait) > 10% → Disk I/O bottleneck — critical signal
# %st (steal) > 5%  → VM being CPU-throttled by hypervisor (cloud VMs)
```

**Interpretation:**

| Metric | Threshold | Meaning |
|---|---|---|
| CPU User % | > 80% | Heavy query processing or missing indexes |
| CPU iowait % | > 10% | Disk bottleneck — check storage |
| CPU steal % | > 5% | Cloud VM is being throttled — scale up |
| Load Average | > CPU count | System is saturated |

---

## 0.2 — Memory Check

```bash
free -h                 # available memory, swap usage
vmstat -s               # detailed memory stats
cat /proc/meminfo | grep -E "MemTotal|MemFree|Cached|SwapUsed"

# What to look for:
# SwapUsed > 0          → DB is paging to disk — severe performance hit
# MemAvailable < 10%    → Memory pressure — may cause OS to evict DB cache
# Cached memory low     → Buffer pool / shared_buffers not caching enough
```

**Critical Rule:** If swap is being used by the database process, **all query times will multiply by 100x-1000x**. This is the #1 misdiagnosed "slow DB" cause.

```bash
# Check which process is using swap
for pid in $(ls /proc | grep -E '^[0-9]+$'); do
  swap=$(grep VmSwap /proc/$pid/status 2>/dev/null | awk '{print $2}')
  [ "$swap" -gt "0" ] 2>/dev/null && echo "PID $pid: ${swap}kB swap"
done | sort -t: -k2 -rn | head -10
```

---

## 0.3 — Disk I/O Check

```bash
iostat -xz 1 5          # per-device extended I/O stats
iotop -o                # real-time per-process I/O
df -h                   # disk space (full disk = immediate crisis)
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT

# Critical iostat columns:
# %util       > 80%   → Disk is saturated
# await       > 10ms  → High I/O latency (SSD should be <1ms)
# r_await     > 20ms  → Read latency — possible missing indexes
# w_await     > 20ms  → Write latency — checkpoint/flush storm
# svctm       ≈ await → If svctm << await, queuing is occurring
```

**Disk Full Emergency:**
```bash
# Find large files fast
du -sh /var/lib/mysql/* 2>/dev/null | sort -rh | head -20
du -sh /var/lib/postgresql/* 2>/dev/null | sort -rh | head -20

# Find recently modified large files
find /var/log -size +100M -mtime -1 -ls 2>/dev/null
```

---

## 0.4 — Network Latency Check

```bash
# Latency between app server and DB server
ping -c 20 <db_server_ip>                  # baseline latency
traceroute <db_server_ip>                  # hop-by-hop path
mtr --report <db_server_ip>                # continuous path analysis

# Check DB port connectivity and response time
time mysql -h <db_host> -u root -p -e "SELECT 1;"
time psql -h <db_host> -U postgres -c "SELECT 1;"

# Packet loss check
ping -c 100 <db_server_ip> | tail -3
# Any packet loss > 0.1% under load = application timeout root cause

# Check network interface errors
ip -s link show eth0
netstat -s | grep -E "retransmit|failed|error"
```

**Interpretation:**

| Latency | Status | Action |
|---|---|---|
| < 1ms | Excellent (same subnet) | Normal |
| 1–5ms | Acceptable | Monitor |
| 5–20ms | Elevated | Check network path |
| > 20ms | Critical | Escalate to network team |
| Any packet loss | Critical | Immediate network investigation |

---

## 0.5 — App Response Time vs DB Execution Time

This is the most important diagnostic split. It tells you WHERE the slowness lives.

```
App Response Time  = DB Execution Time + Network Time + App Processing Time

Example:
  App reports: 8 seconds
  DB reports:  0.3 seconds
  Conclusion:  Problem is NOT the database — app layer or ORM issue

Example:
  App reports: 8 seconds
  DB reports:  7.8 seconds
  Conclusion:  Database IS the problem — proceed to DB-specific checks
```

**How to measure DB execution time:**

```bash
# MySQL
mysql -e "SET profiling=1; SELECT ...; SHOW PROFILES;"

# PostgreSQL
psql -c "\timing" -c "SELECT ...;"

# SQL Server — use SET STATISTICS TIME ON
SET STATISTICS TIME ON;
SELECT ...;
SET STATISTICS TIME OFF;
```

---

---

# 🟢 SECTION 1: MYSQL — IN-DEPTH TROUBLESHOOTING

---

## 1.1 — Active Queries & Process List

```sql
-- Full process list with all query text
SHOW FULL PROCESSLIST;

-- More detailed via information_schema (filterable)
SELECT
  id,
  user,
  host,
  db,
  command,
  time AS seconds_running,
  state,
  LEFT(info, 500) AS query_text
FROM information_schema.processlist
WHERE command != 'Sleep'
  AND time > 5               -- queries running > 5 seconds
ORDER BY time DESC;
```

**State Column — What Each State Means:**

| State | Meaning | Urgency |
|---|---|---|
| `Locked` | Waiting for table-level lock | 🔴 Critical |
| `Waiting for row lock` | Row-level InnoDB lock wait | 🔴 Critical |
| `Sending data` | Reading rows, may be doing full scan | 🟡 Warning |
| `Copying to tmp table` | Filesort or GROUP BY without index | 🟡 Warning |
| `Sorting result` | ORDER BY without index | 🟡 Warning |
| `Opening tables` | Metadata lock contention | 🟡 Warning |
| `Checking permissions` | Auth overhead | 🟢 Usually ok |
| `Sleep` | Idle connection, holding resources | 🟡 If > 300s |

**Kill a problematic query:**
```sql
-- Kill single query
KILL QUERY 1234;       -- kills only the query, keeps connection
KILL CONNECTION 1234;  -- kills connection entirely

-- Kill all long-running queries (generate KILL statements)
SELECT CONCAT('KILL QUERY ', id, ';') AS kill_cmd
FROM information_schema.processlist
WHERE command != 'Sleep'
  AND time > 30
  AND user != 'replication'
ORDER BY time DESC;
-- Review output, then execute selectively
```

---

## 1.2 — Slow Queries via Performance Schema

```sql
-- Top 10 slowest query digests (all time)
SELECT
  SCHEMA_NAME AS db,
  DIGEST_TEXT AS query_pattern,
  COUNT_STAR AS executions,
  ROUND(SUM_TIMER_WAIT / 1e12, 2) AS total_seconds,
  ROUND(AVG_TIMER_WAIT / 1e9, 2) AS avg_ms,
  ROUND(MAX_TIMER_WAIT / 1e9, 2) AS max_ms,
  SUM_ROWS_EXAMINED AS rows_examined,
  SUM_ROWS_SENT AS rows_sent,
  ROUND(SUM_ROWS_EXAMINED / NULLIF(SUM_ROWS_SENT, 0), 0) AS examined_per_sent
FROM performance_schema.events_statements_summary_by_digest
WHERE SCHEMA_NAME IS NOT NULL
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 10;
```

**Interpretation:**

| Column | Red Flag | Meaning |
|---|---|---|
| `avg_ms` | > 1000ms | Query consistently slow |
| `examined_per_sent` | > 100 | Reading far more rows than needed — missing index |
| `max_ms` >> `avg_ms` | Large variance | Occasional lock waits or plan changes |
| `executions` very high | Combined with avg_ms > 10ms | High-frequency slow query — high priority fix |

```sql
-- Queries with worst rows-examined ratio (missing index candidates)
SELECT
  DIGEST_TEXT,
  COUNT_STAR,
  SUM_ROWS_EXAMINED,
  SUM_ROWS_SENT,
  ROUND(SUM_ROWS_EXAMINED / NULLIF(SUM_ROWS_SENT,0)) AS ratio
FROM performance_schema.events_statements_summary_by_digest
WHERE SUM_ROWS_EXAMINED / NULLIF(SUM_ROWS_SENT,0) > 100
ORDER BY SUM_ROWS_EXAMINED DESC
LIMIT 15;

-- Queries with no index usage (full scans)
SELECT
  DIGEST_TEXT,
  COUNT_STAR,
  SUM_NO_INDEX_USED + SUM_NO_GOOD_INDEX_USED AS bad_index_count
FROM performance_schema.events_statements_summary_by_digest
WHERE SUM_NO_INDEX_USED + SUM_NO_GOOD_INDEX_USED > 0
ORDER BY bad_index_count DESC
LIMIT 10;
```

---

## 1.3 — Locks & Deadlocks — Deep Analysis

```sql
-- Current InnoDB lock waits
SELECT
  r.trx_id AS waiting_trx,
  r.trx_mysql_thread_id AS waiting_thread,
  r.trx_query AS waiting_query,
  r.trx_wait_started AS wait_started,
  TIMESTAMPDIFF(SECOND, r.trx_wait_started, NOW()) AS wait_seconds,
  b.trx_id AS blocking_trx,
  b.trx_mysql_thread_id AS blocking_thread,
  b.trx_query AS blocking_query,
  b.trx_started AS blocking_trx_started
FROM information_schema.innodb_lock_waits w
  JOIN information_schema.innodb_trx r ON r.trx_id = w.requesting_trx_id
  JOIN information_schema.innodb_trx b ON b.trx_id = w.blocking_trx_id;

-- All open transactions (including idle transactions holding locks)
SELECT
  trx_id,
  trx_mysql_thread_id AS thread_id,
  trx_started,
  TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS open_seconds,
  trx_query,
  trx_rows_locked,
  trx_rows_modified
FROM information_schema.innodb_trx
ORDER BY trx_started;
-- ALERT: Any transaction open > 60 seconds is dangerous
-- ALERT: trx_rows_locked > 10000 = massive lock footprint
```

**Full Deadlock Analysis:**
```sql
-- Extract latest deadlock from InnoDB status
SHOW ENGINE INNODB STATUS\G

-- In the output, find: *** LATEST DETECTED DEADLOCK ***
-- Read section carefully:
-- "TRANSACTION 1" — first party in deadlock
--   "holds lock on" — what it has
--   "waiting for lock on" — what it wants
-- "TRANSACTION 2" — second party
--   (reverse of Transaction 1)
-- "TRANSACTION 1 ROLLED BACK" — InnoDB chose victim (always the cheaper one)

-- Enable persistent deadlock logging to error log
SET GLOBAL innodb_print_all_deadlocks = ON;
-- Then check: tail -100 /var/log/mysql/error.log | grep -A 40 "DEADLOCK"
```

**Deadlock Root Cause Decision Tree:**
```
Deadlock detected?
├── Same two tables always involved?
│   ├── YES → Fix lock ordering (always lock Table A before Table B)
│   └── NO  → Multiple lock paths — add covering indexes to reduce lock scope
├── DELETE + INSERT pattern?
│   └── Gap lock issue — consider READ COMMITTED isolation level
├── Bulk operations?
│   └── Break into smaller batches (1000 rows per transaction)
└── ORM generating unexpected locks?
    └── Add logging to capture actual SQL; review N+1 patterns
```

---

## 1.4 — Index Analysis & EXPLAIN Deep Dive

```sql
-- EXPLAIN output — full column analysis
EXPLAIN SELECT o.*, c.name
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.status = 'PENDING'
  AND o.created_at > DATE_SUB(NOW(), INTERVAL 7 DAY)
ORDER BY o.created_at DESC
LIMIT 100;
```

**EXPLAIN Column Reference:**

| Column | Dangerous Value | Good Value | Fix |
|---|---|---|---|
| `type` | `ALL` (full scan) | `ref`, `eq_ref`, `range` | Add index |
| `rows` | Millions | < 1000 for OLTP | Index or rewrite |
| `Extra` | `Using filesort` | `Using index` | Add ORDER BY to index |
| `Extra` | `Using temporary` | — | Fix GROUP BY or DISTINCT |
| `Extra` | `Using join buffer` | — | Add index on join column |
| `key` | `NULL` | index name | Add missing index |

```sql
-- EXPLAIN ANALYZE (MySQL 8.0+) — actual vs estimated rows
EXPLAIN ANALYZE
SELECT * FROM orders WHERE status='PENDING' AND created_at > '2024-01-01';

-- Look for: "actual rows=50000" vs "rows=100" — huge divergence = stale statistics
-- Fix stale statistics:
ANALYZE TABLE orders;

-- Find missing indexes via sys schema
SELECT * FROM sys.schema_unused_indexes;         -- indexes never used
SELECT * FROM sys.schema_redundant_indexes;      -- duplicate indexes
SELECT * FROM sys.statements_with_full_table_scans   -- queries doing full scans
ORDER BY no_index_used_count DESC LIMIT 10;
```

---

## 1.5 — Buffer Pool & Memory Analysis

```sql
-- Buffer pool hit ratio (target: > 99%)
SELECT
  ROUND((1 - (
    (SELECT variable_value FROM information_schema.global_status WHERE variable_name='Innodb_buffer_pool_reads') /
    (SELECT variable_value FROM information_schema.global_status WHERE variable_name='Innodb_buffer_pool_read_requests')
  )) * 100, 4) AS buffer_pool_hit_pct;

-- If < 99%: increase innodb_buffer_pool_size
-- If = 100%: all data in memory — slowness is elsewhere

-- Buffer pool dirty page ratio
SELECT
  variable_name,
  variable_value
FROM information_schema.global_status
WHERE variable_name IN (
  'Innodb_buffer_pool_pages_total',
  'Innodb_buffer_pool_pages_dirty',
  'Innodb_buffer_pool_pages_free',
  'Innodb_buffer_pool_wait_free'    -- > 0 means pool is full and thrashing
);

-- Check temp table spills to disk (work_mem equivalent)
SHOW STATUS LIKE 'Created_tmp_disk_tables';
SHOW STATUS LIKE 'Created_tmp_tables';
-- If disk_tables / tmp_tables > 0.05 (5%) → increase tmp_table_size
```

---

## 1.6 — MySQL Replication Lag (if using replicas for reads)

```sql
-- Check replica lag
SHOW REPLICA STATUS\G
-- Key fields:
-- Seconds_Behind_Source     → 0 = in sync, > 10 = lagging, > 60 = critical
-- Replica_IO_Running        → Must be YES
-- Replica_SQL_Running       → Must be YES
-- Last_Error                → Any value here = replication broken

-- Check parallel replication status
SHOW VARIABLES LIKE 'replica_parallel_workers';
-- If 0 and lag exists → enable parallel replication:
SET GLOBAL replica_parallel_type = 'LOGICAL_CLOCK';
SET GLOBAL replica_parallel_workers = 8;
STOP REPLICA SQL_THREAD;
START REPLICA SQL_THREAD;
```

---

---

# 🔵 SECTION 2: POSTGRESQL — IN-DEPTH TROUBLESHOOTING

---

## 2.1 — Active Sessions & Wait Events

```sql
-- All non-idle sessions with wait events
SELECT
  pid,
  usename,
  application_name,
  client_addr,
  state,
  wait_event_type,
  wait_event,
  NOW() - query_start AS duration,
  LEFT(query, 200) AS query_snippet
FROM pg_stat_activity
WHERE state != 'idle'
  AND pid != pg_backend_pid()
ORDER BY duration DESC NULLS LAST;
```

**Wait Event Reference — Critical Ones:**

| wait_event_type | wait_event | Meaning | Fix |
|---|---|---|---|
| `Lock` | `relation` | Waiting for table lock | Find and kill blocking query |
| `Lock` | `tuple` | Row-level lock wait | Reduce transaction duration |
| `Lock` | `transactionid` | Waiting for another txn to commit/rollback | Locate long-running transaction |
| `LWLock` | `BufferMapping` | Buffer pool contention | Increase shared_buffers |
| `LWLock` | `WALWrite` | WAL write contention | Increase wal_buffers |
| `IO` | `DataFileRead` | Reading from disk (not in cache) | Increase shared_buffers |
| `IO` | `WALWrite` | WAL sync | Use wal_sync_method=fdatasync on SSD |
| `Client` | `ClientRead` | Waiting for app to send query | App-side issue |

```sql
-- Aggregated wait event summary (current snapshot)
SELECT
  wait_event_type,
  wait_event,
  COUNT(*) AS waiting_sessions,
  SUM(EXTRACT(EPOCH FROM (NOW() - query_start))) AS total_wait_seconds
FROM pg_stat_activity
WHERE wait_event IS NOT NULL
  AND state != 'idle'
GROUP BY wait_event_type, wait_event
ORDER BY waiting_sessions DESC;
```

---

## 2.2 — Long Running Queries — Kill Protocol

```sql
-- All queries running longer than 30 seconds
SELECT
  pid,
  usename,
  NOW() - pg_stat_activity.query_start AS duration,
  query,
  state,
  wait_event_type,
  wait_event
FROM pg_stat_activity
WHERE state = 'active'
  AND query_start < NOW() - INTERVAL '30 seconds'
ORDER BY duration DESC;

-- Safe cancel (sends interrupt — query stops, connection survives)
SELECT pg_cancel_backend(pid) FROM pg_stat_activity
WHERE state = 'active'
  AND duration > INTERVAL '5 minutes'
  AND query NOT LIKE '%vacuum%'
  AND query NOT LIKE '%analyze%';

-- Hard kill (closes connection — use if cancel doesn't work after 30s)
SELECT pg_terminate_backend(pid) FROM pg_stat_activity
WHERE pid = 12345;

-- Auto-terminate queries over N minutes (production safety setting)
-- In postgresql.conf:
-- statement_timeout = '10min'       -- global limit
-- lock_timeout = '30s'              -- prevent indefinite lock waits
-- idle_in_transaction_session_timeout = '5min'  -- kill abandoned transactions
```

---

## 2.3 — Blocking Chains — Full Dependency Tree

```sql
-- Full blocking chain with recursive CTE (PG13+)
WITH RECURSIVE lock_chain AS (
  -- Base: sessions being blocked
  SELECT
    blocked.pid AS blocked_pid,
    blocked.query AS blocked_query,
    blocked.wait_event_type,
    blocked.wait_event,
    blocking.pid AS blocker_pid,
    blocking.query AS blocker_query,
    NOW() - blocked.query_start AS blocked_duration,
    1 AS depth
  FROM pg_stat_activity blocked
  JOIN pg_locks blocked_locks ON blocked.pid = blocked_locks.pid
  JOIN pg_locks blocking_locks
    ON blocking_locks.locktype = blocked_locks.locktype
    AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
    AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
    AND blocking_locks.pid != blocked_locks.pid
    AND blocking_locks.granted = true
    AND blocked_locks.granted = false
  JOIN pg_stat_activity blocking ON blocking.pid = blocking_locks.pid

  UNION ALL

  -- Recursive: find who is blocking the blocker
  SELECT
    lc.blocked_pid,
    lc.blocked_query,
    lc.wait_event_type,
    lc.wait_event,
    blocking.pid,
    blocking.query,
    lc.blocked_duration,
    lc.depth + 1
  FROM lock_chain lc
  JOIN pg_locks blocked_locks ON lc.blocker_pid = blocked_locks.pid
  JOIN pg_locks blocking_locks
    ON blocking_locks.locktype = blocked_locks.locktype
    AND blocking_locks.pid != blocked_locks.pid
    AND blocking_locks.granted = true
    AND blocked_locks.granted = false
  JOIN pg_stat_activity blocking ON blocking.pid = blocking_locks.pid
  WHERE lc.depth < 5
)
SELECT * FROM lock_chain ORDER BY depth, blocked_duration DESC;

-- Quick: find the single root blocker (most common case)
SELECT
  pid,
  usename,
  NOW() - query_start AS running_for,
  state,
  query
FROM pg_stat_activity
WHERE pid IN (
  SELECT DISTINCT blocking_locks.pid
  FROM pg_locks blocked_locks
  JOIN pg_locks blocking_locks
    ON blocking_locks.locktype = blocked_locks.locktype
    AND blocking_locks.pid != blocked_locks.pid
    AND blocking_locks.granted = true
    AND blocked_locks.granted = false
)
ORDER BY query_start;
```

---

## 2.4 — pg_stat_statements — Query Performance Analysis

```sql
-- Ensure extension is enabled
SELECT * FROM pg_available_extensions WHERE name = 'pg_stat_statements';
-- If not installed: CREATE EXTENSION pg_stat_statements;
-- Add to postgresql.conf: shared_preload_libraries = 'pg_stat_statements'

-- Top 10 queries by TOTAL time (what's costing the most overall)
SELECT
  LEFT(query, 120) AS query_snippet,
  calls,
  ROUND(total_exec_time::numeric, 2) AS total_ms,
  ROUND(mean_exec_time::numeric, 2) AS avg_ms,
  ROUND(stddev_exec_time::numeric, 2) AS stddev_ms,
  rows,
  ROUND(total_exec_time / NULLIF(calls, 0), 2) AS ms_per_call,
  ROUND((total_exec_time / SUM(total_exec_time) OVER()) * 100, 2) AS pct_of_total
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- Top 10 by AVERAGE time (slowest per call — often the most fixable)
SELECT
  LEFT(query, 120) AS query_snippet,
  calls,
  ROUND(mean_exec_time::numeric, 2) AS avg_ms,
  ROUND(max_exec_time::numeric, 2) AS max_ms,
  ROUND(stddev_exec_time::numeric, 2) AS stddev_ms
FROM pg_stat_statements
WHERE calls > 10            -- ignore one-off queries
ORDER BY mean_exec_time DESC
LIMIT 10;

-- Cache hit ratio per query (low = reading from disk)
SELECT
  LEFT(query, 100) AS query_snippet,
  calls,
  shared_blks_hit,
  shared_blks_read,
  ROUND(shared_blks_hit::numeric / NULLIF(shared_blks_hit + shared_blks_read, 0) * 100, 2) AS cache_hit_pct,
  temp_blks_written    -- > 0 means query is spilling to disk (work_mem too low)
FROM pg_stat_statements
WHERE shared_blks_hit + shared_blks_read > 1000
ORDER BY shared_blks_read DESC
LIMIT 10;

-- Reset stats after tuning (start fresh measurement)
SELECT pg_stat_statements_reset();
```

---

## 2.5 — Autovacuum & Table Bloat

```sql
-- Tables with high dead tuple count (bloat candidates)
SELECT
  schemaname || '.' || relname AS table_name,
  n_live_tup,
  n_dead_tup,
  ROUND(n_dead_tup * 100.0 / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_pct,
  pg_size_pretty(pg_total_relation_size(schemaname || '.' || relname)) AS total_size,
  last_autovacuum,
  last_autoanalyze,
  autovacuum_count
FROM pg_stat_user_tables
WHERE n_dead_tup > 10000
ORDER BY n_dead_tup DESC;

-- Tables where autovacuum is blocked or overdue
SELECT
  relname,
  last_autovacuum,
  NOW() - last_autovacuum AS since_last_vacuum,
  n_dead_tup
FROM pg_stat_user_tables
WHERE (last_autovacuum IS NULL OR last_autovacuum < NOW() - INTERVAL '24 hours')
  AND n_dead_tup > 50000
ORDER BY n_dead_tup DESC;

-- Check if autovacuum is currently running
SELECT pid, relid::regclass, phase, heap_blks_scanned, heap_blks_total,
       ROUND(heap_blks_scanned * 100.0 / NULLIF(heap_blks_total,0), 1) AS pct_done
FROM pg_stat_progress_vacuum;

-- Transaction ID wraparound danger (critical — causes DB freeze if ignored)
SELECT datname,
       age(datfrozenxid) AS txid_age,
       ROUND(age(datfrozenxid) * 100.0 / 2147483648, 2) AS pct_to_wraparound
FROM pg_database
ORDER BY age(datfrozenxid) DESC;
-- ALERT: pct_to_wraparound > 50% → Schedule emergency VACUUM FREEZE
-- CRITICAL: > 80% → DB will refuse writes to protect itself
```

---

## 2.6 — Cache Hit Ratio & Buffer Analysis

```sql
-- Database-level cache hit ratio
SELECT
  datname,
  blks_hit,
  blks_read,
  ROUND(blks_hit * 100.0 / NULLIF(blks_hit + blks_read, 0), 4) AS cache_hit_pct,
  tup_fetched,
  tup_returned,
  ROUND(tup_fetched * 100.0 / NULLIF(tup_returned, 0), 4) AS index_hit_pct
FROM pg_stat_database
WHERE datname NOT IN ('template0', 'template1', 'postgres')
ORDER BY blks_read DESC;

-- Per-table cache hit ratio (find cold tables)
SELECT
  relname,
  heap_blks_read,
  heap_blks_hit,
  ROUND(heap_blks_hit * 100.0 / NULLIF(heap_blks_hit + heap_blks_read, 0), 2) AS table_cache_hit_pct,
  idx_blks_read,
  idx_blks_hit,
  ROUND(idx_blks_hit * 100.0 / NULLIF(idx_blks_hit + idx_blks_read, 0), 2) AS index_cache_hit_pct
FROM pg_statio_user_tables
WHERE heap_blks_read + heap_blks_hit > 1000
ORDER BY heap_blks_read DESC
LIMIT 20;

-- Overall checkpoint & background writer health
SELECT
  checkpoints_timed,
  checkpoints_req,           -- high = checkpoints forced by wal size, not schedule
  checkpoint_write_time,
  checkpoint_sync_time,
  buffers_checkpoint,
  buffers_clean,
  maxwritten_clean,          -- > 0 = bgwriter falling behind, may need tuning
  buffers_backend,           -- high = apps forcing their own buffer writes (bad)
  buffers_alloc
FROM pg_stat_bgwriter;
```

---

## 2.7 — EXPLAIN ANALYZE — Reading the Plan

```sql
-- Gold standard EXPLAIN: shows actual rows, buffers, timing
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, FORMAT TEXT)
SELECT o.id, o.status, c.name
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.status = 'PENDING'
  AND o.created_at BETWEEN '2024-01-01' AND '2024-03-01'
ORDER BY o.created_at DESC;
```

**Plan Node Reference:**

| Node | What It Does | Red Flag |
|---|---|---|
| `Seq Scan` | Reads every row in table | On large tables with WHERE clause |
| `Index Scan` | Navigates index, fetches heap row | Normal — check rows estimate |
| `Index Only Scan` | Reads index only, no heap | Best possible — means covering index |
| `Bitmap Heap Scan` | Index → bitmap → heap | OK for range scans |
| `Hash Join` | Builds hash table of smaller side | Check "Batches > 1" = disk spill |
| `Merge Join` | Requires both sides sorted | Check sort cost |
| `Nested Loop` | For each outer row, scan inner | Bad when inner is large |
| `Sort` | External sort | Large sort cost = increase work_mem |

```sql
-- Find tables with missing indexes (sequential scans on large tables)
SELECT
  schemaname,
  relname AS table_name,
  seq_scan,
  seq_tup_read,
  idx_scan,
  ROUND(seq_scan * 100.0 / NULLIF(seq_scan + idx_scan, 0), 2) AS seq_scan_pct,
  pg_size_pretty(pg_total_relation_size(schemaname || '.' || relname)) AS table_size
FROM pg_stat_user_tables
WHERE seq_scan > 100
  AND pg_total_relation_size(schemaname || '.' || relname) > 100 * 1024 * 1024  -- > 100MB
ORDER BY seq_tup_read DESC;
-- High seq_scan_pct on large tables = strong index candidate
```

---

---

# 🟣 SECTION 3: SQL SERVER — IN-DEPTH TROUBLESHOOTING

---

## 3.1 — Active Requests & Session State

```sql
-- All active requests with complete context
SELECT
  r.session_id,
  r.status,
  r.blocking_session_id,
  r.wait_type,
  r.wait_time / 1000.0 AS wait_seconds,
  r.cpu_time / 1000.0 AS cpu_seconds,
  r.total_elapsed_time / 1000.0 AS elapsed_seconds,
  r.logical_reads,
  r.writes,
  r.row_count,
  DB_NAME(r.database_id) AS database_name,
  s.login_name,
  s.host_name,
  s.program_name,
  LEFT(t.text, 500) AS query_text,
  qp.query_plan
FROM sys.dm_exec_requests r
JOIN sys.dm_exec_sessions s ON r.session_id = s.session_id
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) t
OUTER APPLY sys.dm_exec_query_plan(r.plan_handle) qp
WHERE r.session_id > 50
  AND r.status != 'background'
ORDER BY r.total_elapsed_time DESC;
```

**Status Column Reference:**

| Status | Meaning | Action |
|---|---|---|
| `running` | Actively using CPU | Check execution plan |
| `runnable` | Waiting for CPU scheduler | CPU saturation |
| `suspended` | Waiting on resource (I/O, lock) | Check wait_type |
| `sleeping` | Idle open connection | May be holding locks |
| `background` | System task | Usually ignore |

---

## 3.2 — Blocking Analysis — Full Blocking Tree

```sql
-- Complete blocking chain with depth
WITH BlockingTree AS (
  -- Root: sessions not blocked by anyone
  SELECT
    session_id,
    blocking_session_id,
    wait_type,
    wait_time / 1000.0 AS wait_sec,
    CAST(LEFT(t.text, 200) AS VARCHAR(200)) AS query_text,
    CAST(0 AS INT) AS level,
    CAST(CONVERT(VARCHAR, session_id) AS VARCHAR(1000)) AS chain
  FROM sys.dm_exec_requests r
  CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) t
  WHERE blocking_session_id = 0
    AND session_id IN (
      SELECT blocking_session_id FROM sys.dm_exec_requests WHERE blocking_session_id != 0
    )

  UNION ALL

  -- Recursive: blocked sessions
  SELECT
    r.session_id,
    r.blocking_session_id,
    r.wait_type,
    r.wait_time / 1000.0,
    CAST(LEFT(t.text, 200) AS VARCHAR(200)),
    bt.level + 1,
    CAST(bt.chain + ' -> ' + CAST(r.session_id AS VARCHAR) AS VARCHAR(1000))
  FROM sys.dm_exec_requests r
  CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) t
  JOIN BlockingTree bt ON r.blocking_session_id = bt.session_id
  WHERE bt.level < 10
)
SELECT
  REPLICATE('  ', level) + CAST(session_id AS VARCHAR) AS session_tree,
  blocking_session_id,
  wait_type,
  wait_sec,
  query_text,
  chain
FROM BlockingTree
ORDER BY chain;

-- Quick: show only sessions that are blocking others (root blockers)
SELECT
  blocking_session_id AS root_blocker,
  COUNT(*) AS sessions_blocked,
  MAX(wait_time) / 1000.0 AS max_wait_sec
FROM sys.dm_exec_requests
WHERE blocking_session_id != 0
GROUP BY blocking_session_id
ORDER BY sessions_blocked DESC;

-- Get query text of root blocker
DBCC INPUTBUFFER(<<root_blocker_session_id>>);
```

---

## 3.3 — Top CPU Queries — Full Breakdown

```sql
-- Top 10 queries by total CPU (cumulative cost)
SELECT TOP 10
  qs.total_worker_time / 1000.0 AS total_cpu_ms,
  qs.execution_count,
  qs.total_worker_time / qs.execution_count / 1000.0 AS avg_cpu_ms,
  qs.total_elapsed_time / qs.execution_count / 1000.0 AS avg_elapsed_ms,
  qs.total_logical_reads / qs.execution_count AS avg_logical_reads,
  qs.total_physical_reads / qs.execution_count AS avg_physical_reads,
  qs.total_logical_writes / qs.execution_count AS avg_logical_writes,
  DB_NAME(qt.dbid) AS database_name,
  SUBSTRING(
    qt.text,
    (qs.statement_start_offset / 2) + 1,
    ((CASE qs.statement_end_offset
        WHEN -1 THEN DATALENGTH(qt.text)
        ELSE qs.statement_end_offset
      END - qs.statement_start_offset) / 2) + 1
  ) AS query_text,
  qp.query_plan
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) qt
OUTER APPLY sys.dm_exec_query_plan(qs.plan_handle) qp
ORDER BY qs.total_worker_time DESC;

-- Top 10 by PHYSICAL reads (reading from disk — missing index or memory pressure)
SELECT TOP 10
  total_physical_reads,
  total_physical_reads / execution_count AS avg_physical_reads,
  execution_count,
  SUBSTRING(t.text, (qs.statement_start_offset/2)+1,
    ((CASE qs.statement_end_offset WHEN -1 THEN DATALENGTH(t.text)
      ELSE qs.statement_end_offset END - qs.statement_start_offset)/2)+1) AS query_text
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) t
ORDER BY total_physical_reads DESC;
-- High physical reads → data not in buffer pool → add memory or fix index
```

---

## 3.4 — Wait Stats — System-Wide Health Diagnosis

```sql
-- Current waits (live snapshot)
SELECT
  wait_type,
  waiting_tasks_count,
  wait_time_ms / 1000.0 AS wait_time_sec,
  max_wait_time_ms / 1000.0 AS max_wait_sec,
  signal_wait_time_ms / 1000.0 AS signal_wait_sec,
  ROUND(wait_time_ms * 100.0 / SUM(wait_time_ms) OVER(), 2) AS pct_of_total
FROM sys.dm_os_wait_stats
WHERE wait_type NOT IN (
  -- Exclude benign waits
  'SLEEP_TASK', 'WAIT_XTP_OFFLINE_CKPT_NEW_LOG', 'DISPATCHER_QUEUE_SEMAPHORE',
  'BROKER_TO_FLUSH', 'BROKER_TASK_STOP', 'CLR_AUTO_EVENT', 'DBMIRROR_EVENTS_QUEUE',
  'SQLTRACE_BUFFER_FLUSH', 'CLR_MANUAL_EVENT', 'DISPATCHER_QUEUE_SEMAPHORE',
  'FT_IFTS_SCHEDULER_IDLE_WAIT', 'XE_DISPATCHER_WAIT', 'XE_TIMER_EVENT',
  'WAITFOR', 'LAZYWRITER_SLEEP', 'CHECKPOINT_QUEUE', 'REQUEST_FOR_DEADLOCK_SEARCH',
  'RESOURCE_QUEUE', 'SERVER_IDLE_CHECK', 'SLEEP_DBSTARTUP', 'SLEEP_DCOMSTARTUP',
  'SLEEP_MASTERDBREADY', 'SLEEP_MASTERMDREADY', 'SLEEP_MASTERUPGRADED',
  'SLEEP_MSDBSTARTUP', 'SLEEP_SYSTEMTASK', 'SLEEP_TEMPDBSTARTUP', 'SNI_HTTP_ACCEPT',
  'SP_SERVER_DIAGNOSTICS_SLEEP', 'SQLTRACE_INCREMENTAL_FLUSH_SLEEP', 'WAIT_XTP_OFFLINE_CKPT_NEW_LOG'
)
ORDER BY wait_time_ms DESC;
```

**Wait Type Diagnostic Guide:**

| Wait Type | Root Cause | Fix |
|---|---|---|
| `PAGEIOLATCH_SH` / `_EX` | Reading/writing data from disk — buffer pool miss | Increase max server memory, add indexes |
| `LCK_M_X` / `LCK_M_S` | Row/page/table exclusive or shared lock waits | Investigate blocking tree, reduce transaction duration |
| `CXPACKET` / `CXCONSUMER` | Parallel query skew — one thread finishing last | Set MAXDOP, adjust cost threshold for parallelism |
| `SOS_SCHEDULER_YIELD` | CPU pressure, runnable queue high | Scale CPU, fix parameter sniffing issues |
| `WRITELOG` | Transaction log writes are bottleneck | Move log to faster disk, increase log file size |
| `ASYNC_NETWORK_IO` | App not reading results fast enough | Check network, app result set processing |
| `RESOURCE_SEMAPHORE` | Memory grant waiting — queries need more RAM | Increase max server memory, fix missing indexes |
| `THREADPOOL` | Worker thread starvation | Increase max worker threads, reduce blocking |
| `OLEDB` | Linked server latency | Remove linked servers if possible |
| `HADR_SYNC_COMMIT` | AlwaysOn synchronous replica lagging | Check replica health, consider async |

---

## 3.5 — Missing Index Analysis

```sql
-- Top missing indexes by estimated impact
SELECT TOP 20
  DB_NAME(mid.database_id) AS database_name,
  OBJECT_NAME(mid.object_id, mid.database_id) AS table_name,
  migs.avg_total_user_cost * migs.avg_user_impact * (migs.user_seeks + migs.user_scans)
    AS improvement_measure,
  migs.user_seeks,
  migs.user_scans,
  migs.avg_user_impact AS estimated_improvement_pct,
  mid.equality_columns,
  mid.inequality_columns,
  mid.included_columns,
  'CREATE INDEX IX_' + REPLACE(OBJECT_NAME(mid.object_id, mid.database_id), ' ', '_')
    + '_' + REPLACE(REPLACE(ISNULL(mid.equality_columns, '') + '_'
    + ISNULL(mid.inequality_columns, ''), ', ', '_'), '[', '')
    AS suggested_index_name,
  'CREATE NONCLUSTERED INDEX [IX_missing_' + CAST(mid.object_id AS VARCHAR) + '] ON '
    + OBJECT_SCHEMA_NAME(mid.object_id, mid.database_id) + '.'
    + OBJECT_NAME(mid.object_id, mid.database_id)
    + ' (' + ISNULL(mid.equality_columns, '')
    + CASE WHEN mid.equality_columns IS NOT NULL AND mid.inequality_columns IS NOT NULL THEN ', ' ELSE '' END
    + ISNULL(mid.inequality_columns, '') + ')'
    + ISNULL(' INCLUDE (' + mid.included_columns + ')', '') AS create_index_statement
FROM sys.dm_db_missing_index_details mid
JOIN sys.dm_db_missing_index_groups mig ON mid.index_handle = mig.index_handle
JOIN sys.dm_db_missing_index_group_stats migs ON mig.index_group_handle = migs.group_handle
WHERE mid.database_id = DB_ID()
ORDER BY improvement_measure DESC;

-- Unused indexes (candidates for removal — reducing write overhead)
SELECT
  OBJECT_NAME(i.object_id) AS table_name,
  i.name AS index_name,
  i.type_desc,
  ius.user_seeks,
  ius.user_scans,
  ius.user_lookups,
  ius.user_seeks + ius.user_scans + ius.user_lookups AS total_reads,
  ius.user_updates AS total_writes,
  ius.last_user_seek,
  ius.last_user_scan
FROM sys.indexes i
LEFT JOIN sys.dm_db_index_usage_stats ius
  ON i.object_id = ius.object_id
  AND i.index_id = ius.index_id
  AND ius.database_id = DB_ID()
WHERE OBJECTPROPERTY(i.object_id, 'IsUserTable') = 1
  AND i.type_desc = 'NONCLUSTERED'
  AND ISNULL(ius.user_seeks, 0) + ISNULL(ius.user_scans, 0) + ISNULL(ius.user_lookups, 0) = 0
ORDER BY ius.user_updates DESC;
-- High user_updates + zero reads = index is pure overhead → consider dropping
```

---

## 3.6 — Memory — Buffer Pool Analysis

```sql
-- Buffer pool usage by database
SELECT
  DB_NAME(database_id) AS database_name,
  COUNT(*) * 8 / 1024.0 AS buffer_pool_mb,
  SUM(CASE WHEN is_modified = 1 THEN 1 ELSE 0 END) * 8 / 1024.0 AS dirty_pages_mb
FROM sys.dm_os_buffer_descriptors
WHERE database_id != 32767
GROUP BY database_id
ORDER BY buffer_pool_mb DESC;

-- Total memory configuration
SELECT
  physical_memory_in_use_kb / 1024.0 AS memory_used_mb,
  page_fault_count,
  memory_utilization_percentage
FROM sys.dm_os_process_memory;

-- Memory grants waiting (if > 0, queries are starved for memory)
SELECT
  grant_time,
  requested_memory_kb / 1024.0 AS requested_mb,
  granted_memory_kb / 1024.0 AS granted_mb,
  required_memory_kb / 1024.0 AS required_mb,
  used_memory_kb / 1024.0 AS used_mb,
  queue_id,
  wait_order,
  is_next_candidate
FROM sys.dm_exec_query_memory_grants
WHERE grant_time IS NULL OR is_next_candidate = 1;
-- Rows here = memory pressure → increase max server memory or fix I/O spills

-- Plan cache hit ratio
SELECT
  objtype,
  COUNT(*) AS cached_plans,
  SUM(usecounts) AS total_use_count,
  SUM(size_in_bytes) / 1024.0 / 1024.0 AS cache_size_mb
FROM sys.dm_exec_cached_plans
GROUP BY objtype
ORDER BY cache_size_mb DESC;
```

---

## 3.7 — Transaction Log & Tempdb Analysis

```sql
-- Transaction log usage (high log = long open transactions or bulk operations)
SELECT
  DB_NAME(database_id) AS db,
  name AS log_file,
  type_desc,
  size * 8 / 1024.0 AS file_size_mb,
  FILEPROPERTY(name, 'SpaceUsed') * 8.0 / 1024.0 AS space_used_mb
FROM sys.master_files
WHERE type = 1;  -- log files only

-- What is holding the log open (preventing log truncation)
SELECT
  DB_NAME(database_id) AS db,
  log_reuse_wait_desc   -- reason log can't be truncated
FROM sys.databases
WHERE log_reuse_wait_desc != 'NOTHING'
  AND database_id > 4;
-- Common values:
-- ACTIVE_TRANSACTION    → long-running transaction — find and end it
-- LOG_BACKUP           → no log backup running — check backup job
-- REPLICATION          → replication latency
-- DATABASE_MIRRORING   → mirror lag

-- TempDB usage (high = queries spilling sorts/joins or temp tables exploding)
SELECT
  DB_NAME(database_id) AS db,
  SUM(user_object_reserved_page_count) * 8 / 1024.0 AS user_objects_mb,
  SUM(internal_object_reserved_page_count) * 8 / 1024.0 AS internal_objects_mb,
  SUM(version_store_reserved_page_count) * 8 / 1024.0 AS version_store_mb,
  SUM(unallocated_extent_page_count) * 8 / 1024.0 AS free_space_mb
FROM sys.dm_db_file_space_usage
WHERE database_id = 2;  -- TempDB = database_id 2

-- Sessions using most tempdb space
SELECT TOP 10
  t.session_id,
  s.login_name,
  SUM(t.internal_objects_alloc_page_count + t.user_objects_alloc_page_count) * 8 / 1024.0 AS tempdb_used_mb,
  LEFT(st.text, 200) AS query_text
FROM sys.dm_db_task_space_usage t
JOIN sys.dm_exec_sessions s ON t.session_id = s.session_id
CROSS APPLY sys.dm_exec_sql_text(s.most_recent_sql_handle) st
GROUP BY t.session_id, s.login_name, st.text
ORDER BY tempdb_used_mb DESC;
```

---

---

# 🚦 SECTION 4: ROOT CAUSE DECISION TREE

---

## 4.1 — Complete Diagnostic Flow

```
APPLICATION REPORTED SLOW
│
├── Step 0: Server Health
│   ├── CPU iowait > 10%?          → DISK I/O BOTTLENECK
│   │                                  • Check iostat, identify process
│   │                                  • Look for missing indexes (full scans)
│   │                                  • Buffer pool / shared_buffers too small
│   ├── Swap > 0?                  → MEMORY PRESSURE (critical)
│   │                                  • DB process being paged out
│   │                                  • Increase RAM or reduce buffer sizes
│   ├── CPU user > 80%?            → QUERY PROCESSING OVERHEAD
│   │                                  • Find top CPU queries (pg_stat_statements / DMVs)
│   │                                  • Look for full scans, bad plans
│   └── All clean?                 → Proceed to DB-specific checks
│
├── DB Check: LOCKING
│   ├── Blocking sessions found?
│   │   ├── Root blocker: idle transaction?
│   │   │   → App not committing — fix connection management
│   │   ├── Root blocker: long-running query?
│   │   │   → EXPLAIN/execution plan analysis
│   │   └── Root blocker: batch job?
│   │       → Schedule during off-peak or use NOLOCK hints (with care)
│   └── Deadlocks?
│       → Fix lock ordering, smaller transactions, add indexes
│
├── DB Check: SLOW QUERIES
│   ├── Full table scan on large table?
│   │   → Add index on filter/join column
│   ├── Good index, bad estimate (rows off)?
│   │   → UPDATE STATISTICS / ANALYZE
│   ├── Query fast in isolation, slow under load?
│   │   → Resource contention — look at wait events
│   └── Plan regression after code deploy?
│       → Force old plan / update statistics / sp_recompile
│
├── DB Check: I/O WAITS
│   ├── Cache hit ratio < 99%?
│   │   → Buffer pool / shared_buffers too small
│   ├── Temp file usage (PostgreSQL) / TempDB spill (SQL Server)?
│   │   → work_mem too low / missing indexes on sort columns
│   └── WAL / Log I/O high?
│       → Move log to separate disk, consider async commit for non-critical writes
│
└── DB APPEARS CLEAN → APPLICATION / NETWORK LAYER
    ├── Check ORM generating N+1 queries
    ├── Check application connection pool exhaustion
    ├── Check DNS resolution time
    └── Check TCP packet loss / retransmits
```

---

## 4.2 — Root Cause Quick Reference Card

| Symptom | MySQL Check | PostgreSQL Check | SQL Server Check |
|---|---|---|---|
| Sudden slowness | `SHOW PROCESSLIST` | `pg_stat_activity` | `sys.dm_exec_requests` |
| Lock wait | `INNODB_LOCK_WAITS` | `pg_locks` join | `blocking_session_id` |
| Full table scan | `EXPLAIN` + `type=ALL` | `seq_scan` in `pg_stat_user_tables` | `missing_index_details` |
| Memory pressure | `buffer_pool_wait_free > 0` | `cache_hit_pct < 99%` | `dm_os_process_memory` |
| Disk I/O | `iostat`, `iowait` | `DataFileRead` wait event | `PAGEIOLATCH_*` waits |
| Replication lag | `SHOW REPLICA STATUS` | `pg_stat_replication` | `sys.dm_hadr_database_replica_states` |
| Deadlock | `innodb_print_all_deadlocks` | `pg_locks` | `dm_exec_requests` + deadlock trace |
| Temp spills | `Created_tmp_disk_tables` | `temp_blks_written` in pg_stat_statements | TempDB `internal_objects_mb` |
| Bad statistics | `ANALYZE TABLE` | `ANALYZE` / `last_analyze` | `UPDATE STATISTICS` |

---

## 4.3 — Emergency Kill & Stabilize Protocol

```sql
-- === MYSQL EMERGENCY ===
-- Kill all queries running > 2 minutes (excluding replication)
SELECT GROUP_CONCAT(CONCAT('KILL QUERY ', id) SEPARATOR '; ')
FROM information_schema.processlist
WHERE time > 120 AND command = 'Query' AND user != 'replication';
-- Copy output and execute

-- === POSTGRESQL EMERGENCY ===
-- Cancel all long-running queries (safe — query stops, connection lives)
SELECT pg_cancel_backend(pid)
FROM pg_stat_activity
WHERE state = 'active'
  AND query_start < NOW() - INTERVAL '2 minutes'
  AND query NOT LIKE '%vacuum%'
  AND pid != pg_backend_pid();

-- === SQL SERVER EMERGENCY ===
-- Kill root blocking session
KILL <<blocking_session_id>>;
-- Find blocking session first:
SELECT blocking_session_id, COUNT(*) AS blocked_count
FROM sys.dm_exec_requests WHERE blocking_session_id != 0
GROUP BY blocking_session_id ORDER BY blocked_count DESC;
```

---

## 4.4 — Post-Incident Tuning Checklist

```
AFTER THE INCIDENT IS RESOLVED:

□ Root cause identified and documented (5-WHY analysis)
□ Slow query captured in pg_stat_statements / Performance Schema / DMVs
□ EXPLAIN / Execution Plan reviewed — index added if needed
□ Statistics updated on affected tables
□ Monitoring alert threshold reviewed (was alert too late?)
□ Connection pool settings validated (max connections, timeout)
□ Autovacuum / Auto-update-statistics settings verified
□ Replication lag check (was replica serving stale reads?)
□ Buffer pool / shared_buffers hit ratio confirmed > 99%
□ Any new indexes tested in staging before production deployment
□ Runbook updated with new findings
□ Postmortem scheduled if incident > 30 minutes
```

---

*Playbook Version: 2024 | Maintained by: Y. Venkat Sarath | Sr DBA & Multi-Cloud DBA*
*Covers: MySQL 5.7–8.0 | PostgreSQL 12–16 | SQL Server 2016–2022*
