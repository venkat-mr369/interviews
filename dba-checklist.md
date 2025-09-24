**DBA Troubleshooting Playbook (One-Page Checklist)** you can quickly run when an application is reported **slow**.
**MySQL, PostgreSQL, SQL Server**, each with **exact commands & interpretation**.

---

### 🚨 DBA Troubleshooting Playbook – Application Slowness

---

###### 🔎 Step 0: Universal Pre-Checks (All DBs)

* ✅ Check **server health** (CPU, Memory, Disk, I/O).
* ✅ Confirm **network latency** between app & DB.
* ✅ Compare **app response time vs DB execution time**.

---

###### 🟢 MySQL Quick Checks

######### 1. Active Queries

```sql
SHOW FULL PROCESSLIST;
```

👉 Look for `"Locked"`, `"Sending data"`, `"Copying to tmp table"`.

######### 2. Long / Slow Queries

```sql
SELECT *
FROM performance_schema.events_statements_summary_by_digest
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 10;
```

######### 3. Lock & Deadlocks

```sql
SHOW ENGINE INNODB STATUS\G;
```

👉 Look at **LATEST DETECTED DEADLOCK** section.

######### 4. Index Usage

```sql
EXPLAIN ANALYZE SELECT ...;
```

👉 Ensure **index scans** instead of **full table scans**.

---

###### 🔵 PostgreSQL Quick Checks

######### 1. Active Sessions

```sql
SELECT pid, usename, state, wait_event_type, wait_event, query
FROM pg_stat_activity
WHERE state != 'idle';
```

######### 2. Long Running Queries

```sql
SELECT pid, now() - query_start AS duration, query
FROM pg_stat_activity
WHERE state = 'active'
ORDER BY duration DESC;
```

######### 3. Blocking Sessions

```sql
SELECT blocked_locks.pid AS blocked_pid,
       blocked_activity.query AS blocked_query,
       blocking_locks.pid AS blocking_pid,
       blocking_activity.query AS blocking_query
FROM pg_locks blocked_locks
JOIN pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
JOIN pg_locks blocking_locks ON blocking_locks.locktype = blocked_locks.locktype
JOIN pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;
```

######### 4. Expensive Queries (needs `pg_stat_statements`)

```sql
SELECT query, total_exec_time, calls
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

---

###### 🟣 SQL Server Quick Checks

######### 1. Active Requests

```sql
SELECT session_id, status, blocking_session_id, wait_type, wait_time, text
FROM sys.dm_exec_requests r
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle)
WHERE session_id > 50;
```

######### 2. Blocking Queries

```sql
SELECT blocking_session_id AS blocker, session_id AS blocked, wait_type, text
FROM sys.dm_exec_requests r
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle)
WHERE blocking_session_id <> 0;
```

######### 3. Top CPU Queries

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

######### 4. Wait Stats (system-wide health)

```sql
SELECT wait_type, wait_time_ms, waiting_tasks_count
FROM sys.dm_os_wait_stats
ORDER BY wait_time_ms DESC;
```

👉 Look for:

* **PAGEIOLATCH** = disk I/O bottleneck
* **LCK\_M\_**\* = locking issues
* **CXPACKET** = parallelism

---

###### 🚦 Root Cause Indicators

* 🔴 **Locks/Deadlocks** → Check transactions, isolation levels.
* 🔴 **Slow Queries** → Missing indexes, bad joins, ORMs.
* 🔴 **I/O Waits** → Disk subsystem issues.
* 🔴 **High CPU** → Poor query plans, functions in WHERE.
* 🟢 **DB Clean** → Issue likely at **application / network** layer.

---

