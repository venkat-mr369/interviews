**slow**, one of the first suspects is the **database layer**. 



---

#### 🔎 Deep Dive: Checking Application Slowness from Database Side

---

#### 1️⃣ Common First Steps (All Databases)

Before going DB-specific:

* ✅ Confirm **network latency** (ping between app and DB).
* ✅ Check **DB server resource usage** (CPU, memory, I/O, disk).
* ✅ Check **active sessions** (blocked queries, locks, long-running queries).
* ✅ Correlate **app request time** vs **DB execution time**.

---

#### 2️⃣ MySQL Deep Dive

### Step 1: Check Active Queries

```sql
SHOW FULL PROCESSLIST;
```

* Look for queries in `"Locked"`, `"Copying to tmp table"`, `"Sending data"`.
* Long `"Query"` states = performance issue.

### Step 2: Slow Query Log

Enable it if not already:

```ini
[mysqld]
slow_query_log = 1
long_query_time = 2
log_queries_not_using_indexes = 1
```

Check queries in log file.

### Step 3: Performance Schema

Find top resource consumers:

```sql
SELECT *
FROM performance_schema.events_statements_summary_by_digest
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 10;
```

### Step 4: Check Index Usage

```sql
EXPLAIN ANALYZE SELECT ...;
```

* Look for **full table scans**, missing indexes.
* Check if `Using temporary; Using filesort` appears.

### Step 5: System Health

```sql
SHOW GLOBAL STATUS LIKE 'Threads_connected';
SHOW GLOBAL STATUS LIKE 'Handler%';
SHOW ENGINE INNODB STATUS\G;
```

* Look for **row lock waits**, deadlocks, buffer pool misses.

---

## 3️⃣ PostgreSQL Deep Dive

### Step 1: Check Active Queries

```sql
SELECT pid, usename, state, query, wait_event_type, wait_event
FROM pg_stat_activity
WHERE state != 'idle';
```

* Look for queries stuck in `waiting`.

### Step 2: Long Running Queries

```sql
SELECT pid, now() - query_start AS duration, query
FROM pg_stat_activity
WHERE state = 'active'
ORDER BY duration DESC;
```

### Step 3: Check Locks

```sql
SELECT blocked_locks.pid     AS blocked_pid,
       blocked_activity.query AS blocked_query,
       blocking_locks.pid     AS blocking_pid,
       blocking_activity.query AS blocking_query
FROM pg_locks blocked_locks
JOIN pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
JOIN pg_locks blocking_locks ON blocking_locks.locktype = blocked_locks.locktype
JOIN pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;
```

### Step 4: Index & Plan Analysis

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;
```

* Look at **seq scans vs index scans**.
* High buffer reads = bad plan.

### Step 5: Statistics & Wait Events

```sql
SELECT * FROM pg_stat_database;
SELECT * FROM pg_stat_user_tables;
SELECT * FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 10;
```

* Requires `pg_stat_statements` extension.

### Step 6: System Views

```sql
SELECT * FROM pg_stat_bgwriter;
SELECT * FROM pg_stat_io;
```

* Helps check **I/O bottlenecks**.

---

## 4️⃣ SQL Server (MSSQL) Deep Dive

### Step 1: Active Requests

```sql
SELECT session_id, status, blocking_session_id, wait_type, wait_time, last_wait_type, text
FROM sys.dm_exec_requests r
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle)
WHERE session_id > 50;
```

### Step 2: Blocking & Deadlocks

```sql
SELECT
    blocking_session_id AS blocker,
    session_id AS blocked,
    wait_type, wait_time,
    TEXT
FROM sys.dm_exec_requests r
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle)
WHERE blocking_session_id <> 0;
```

### Step 3: Top Resource-Consuming Queries

```sql
SELECT TOP 10
    total_worker_time/1000 AS CPU_ms,
    execution_count,
    total_elapsed_time/1000 AS Elapsed_ms,
    SUBSTRING(qt.text, (qs.statement_start_offset/2)+1,
        ((CASE qs.statement_end_offset
          WHEN -1 THEN DATALENGTH(qt.text)
          ELSE qs.statement_end_offset END
          - qs.statement_start_offset)/2)+1)
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) as qt
ORDER BY total_worker_time DESC;
```

### Step 4: Wait Stats (very important in SQL Server)

```sql
SELECT wait_type, wait_time_ms, signal_wait_time_ms, waiting_tasks_count
FROM sys.dm_os_wait_stats
ORDER BY wait_time_ms DESC;
```

* Look for **PAGEIOLATCH** (I/O bottleneck), **LCK\_M\_**\* (locking), **CXPACKET** (parallelism).

### Step 5: Index Health

```sql
SELECT OBJECT_NAME(OBJECT_ID), index_id, avg_fragmentation_in_percent
FROM sys.dm_db_index_physical_stats (DB_ID(), NULL, NULL, NULL, 'LIMITED')
ORDER BY avg_fragmentation_in_percent DESC;
```

### Step 6: Query Plans

* Use **Actual Execution Plan** in SSMS (`Ctrl+M`).
* Identify **missing indexes, scans, key lookups**.

---

# 🚦 Decision Making

* **If CPU high & queries slow** → bad query plans / missing indexes.
* **If I/O waits high** → slow disk, bad schema design.
* **If locks frequent** → concurrency, transaction isolation issues.
* **If no DB bottleneck** → issue is at **application code / ORM / network**.

---
