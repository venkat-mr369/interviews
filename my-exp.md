## Senior DBA & Multi-Cloud DBA — Deep-Dive Interview Questions & Answers
### 15 Years Experience | Basic → Advanced | Scenario-Based | Issue-Based | Performance Tuning

---

### TABLE OF CONTENTS
1. PostgreSQL Administration
2. MySQL & MariaDB Administration
3. Multi-Database Management (Cassandra, CockroachDB, Aurora, MSSQL)
4. Kubernetes & Database Management
5. CI/CD Pipeline & Database Automation
6. Flyway & Liquibase — Schema Migration
7. Terraform & Infrastructure as Code
8. Performance Tuning (Deep Dive — All Databases)
9. Scenario-Based Questions
10. Real Issue-Based Troubleshooting Questions

---

### SECTION 1: POSTGRESQL ADMINISTRATION

---

### BASIC LEVEL

---

**Q1. What is the difference between pg_dump and pg_basebackup?**

**Answer:**

| Feature | pg_dump | pg_basebackup |
|---|---|---|
| Type | Logical backup | Physical backup |
| Output | SQL / custom / tar | Binary cluster copy |
| Restore | psql or pg_restore | rsync / copy to data dir |
| Point-in-time | No | Yes (with WAL) |
| Cross-version | Yes | No (same major version) |

`pg_dump` exports selected databases/tables as SQL. It is version-agnostic and great for migrations.

`pg_basebackup` takes a binary copy of the entire PostgreSQL cluster including WAL segments — required for streaming replication setup and PITR (Point-In-Time Recovery).

```bash
# pg_dump — logical backup
pg_dump -Fc -d mydb -f mydb.dump

# pg_basebackup — physical backup with WAL streaming
pg_basebackup -h primary -U replication -D /var/lib/pgsql/backup -Xs -P -R
```

---

**Q2. Explain WAL (Write-Ahead Logging) and why it is important.**

**Answer:**

WAL is PostgreSQL's transaction logging mechanism. Before any data file change is written to disk, the change is first written to the WAL log. This ensures:

- **Crash recovery** — on restart, PostgreSQL replays WAL to recover uncommitted or partially committed transactions.
- **Streaming replication** — standby servers receive and replay WAL segments from the primary in real time.
- **PITR** — by archiving WAL, you can replay to any specific point in time.

Key WAL parameters:
```sql
-- In postgresql.conf
wal_level = replica          -- enables replication
max_wal_senders = 10         -- max streaming connections
wal_keep_size = 1GB          -- retain WAL on primary
archive_mode = on            -- enable WAL archiving
archive_command = 'cp %p /wal_archive/%f'
```

---

**Q3. What is PgBouncer and when would you use it?**

**Answer:**

PgBouncer is a lightweight connection pooler for PostgreSQL. PostgreSQL creates a new OS process for every connection, which is expensive at scale. PgBouncer sits between applications and PostgreSQL, reusing backend connections.

**Pooling modes:**
- **Session pooling** — one server connection per client session (default, safe)
- **Transaction pooling** — server connection held only during a transaction (most efficient, most common)
- **Statement pooling** — per statement (rarely used, restrictions apply)

```ini
# pgbouncer.ini
[databases]
mydb = host=127.0.0.1 port=5432 dbname=mydb

[pgbouncer]
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 25
min_pool_size = 5
server_idle_timeout = 600
```

**Use case:** When your app opens hundreds or thousands of short-lived connections (microservices, APIs), PgBouncer reduces PostgreSQL process overhead drastically.

---

**Q4. What is the difference between VACUUM, VACUUM FULL, and AUTOVACUUM?**

**Answer:**

PostgreSQL uses MVCC (Multi-Version Concurrency Control). Dead tuples from UPDATE/DELETE accumulate and must be cleaned.

- **VACUUM** — marks dead tuples as reusable space. Does NOT return space to OS. Non-blocking (shares table with reads/writes).
- **VACUUM FULL** — rewrites entire table, returns space to OS. **Locks the table exclusively** — use with caution in production.
- **AUTOVACUUM** — background daemon that triggers VACUUM automatically based on thresholds.

```sql
-- Manual vacuum
VACUUM mydb.orders;

-- Verbose vacuum to see stats
VACUUM VERBOSE ANALYZE mydb.orders;

-- Check tables needing vacuum
SELECT relname, n_dead_tup, last_autovacuum
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;

-- Autovacuum tuning in postgresql.conf
autovacuum = on
autovacuum_vacuum_threshold = 50
autovacuum_vacuum_scale_factor = 0.02   -- trigger at 2% dead tuples
autovacuum_analyze_scale_factor = 0.01
autovacuum_vacuum_cost_delay = 2ms      -- reduce I/O throttling
```

---

**Q5. How do you set up streaming replication in PostgreSQL?**

**Answer:**

**On Primary:**
```sql
-- postgresql.conf
wal_level = replica
max_wal_senders = 5
wal_keep_size = 512MB

-- pg_hba.conf
host  replication  replicator  <standby_ip>/32  md5

-- Create replication user
CREATE USER replicator REPLICATION LOGIN PASSWORD 'strongpass';
```

**On Standby:**
```bash
# Take base backup from primary
pg_basebackup -h <primary_ip> -U replicator -D /var/lib/postgresql/data -Xs -P -R

# -R auto-creates standby.signal and postgresql.auto.conf with primary_conninfo
```

**Verify replication:**
```sql
-- On Primary
SELECT * FROM pg_stat_replication;

-- On Standby
SELECT * FROM pg_stat_wal_receiver;
```

---

### INTERMEDIATE LEVEL

---

**Q6. Explain Patroni and how it differs from repmgr for HA.**

**Answer:**

Both tools provide PostgreSQL high availability, but differently:

| Feature | repmgr | Patroni |
|---|---|---|
| Failover | Semi-automatic (needs witness) | Fully automatic |
| DCS dependency | None | etcd / Consul / ZooKeeper |
| Split-brain protection | Manual fencing | DCS-based leader election |
| Config management | Manual | Centralized via REST API |
| Cloud-native | Limited | Yes (Kubernetes-ready) |

**Patroni architecture:**
- Each PostgreSQL node runs a `patroni` agent
- Leader election is done via a DCS (etcd recommended)
- Only the DCS-elected leader can be primary — eliminates split-brain
- Automatic failover in seconds

```yaml
# patroni.yml
scope: postgres-cluster
namespace: /service/
name: node1

etcd:
  hosts: etcd1:2379,etcd2:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576

postgresql:
  listen: 0.0.0.0:5432
  connect_address: node1:5432
  data_dir: /var/lib/postgresql/data
  authentication:
    replication:
      username: replicator
      password: strongpass
```

```bash
# Check cluster status
patronictl -c /etc/patroni/patroni.yml list

# Manual failover
patronictl -c /etc/patroni/patroni.yml failover postgres-cluster
```

---

**Q7. How do you perform a PostgreSQL major version upgrade with minimal downtime?**

**Answer:**

**Method 1: pg_upgrade (offline — minutes of downtime)**
```bash
# Stop old cluster
pg_ctlcluster 14 main stop

# Run pg_upgrade
/usr/lib/postgresql/16/bin/pg_upgrade \
  -b /usr/lib/postgresql/14/bin \
  -B /usr/lib/postgresql/16/bin \
  -d /var/lib/postgresql/14/main \
  -D /var/lib/postgresql/16/main \
  --link  # hard links, much faster

# Analyze new cluster
./analyze_new_cluster.sh
```

**Method 2: Logical Replication (near-zero downtime)**
```sql
-- On OLD primary (source)
CREATE PUBLICATION mypub FOR ALL TABLES;

-- On NEW cluster (target — same schema loaded)
CREATE SUBSCRIPTION mysub
  CONNECTION 'host=old_primary dbname=mydb user=repuser'
  PUBLICATION mypub;

-- Monitor lag
SELECT * FROM pg_stat_subscription;

-- When lag = 0, point app to new cluster, drop subscription
DROP SUBSCRIPTION mysub;
```

**Method 3: pglogical or AWS DMS for cross-version migrations**

---

**Q8. What is logical replication and how does it differ from streaming replication?**

**Answer:**

| Feature | Streaming Replication | Logical Replication |
|---|---|---|
| Level | Physical (byte-for-byte) | Logical (row-level changes) |
| Cross-version | No | Yes |
| Selective tables | No (entire cluster) | Yes (publication/subscription) |
| Write on standby | No | Yes (subscriber can have other tables) |
| Use case | HA / DR | Migrations, selective sync, upgrades |

```sql
-- Create publication (publisher side)
CREATE PUBLICATION orders_pub FOR TABLE orders, customers;

-- Create subscription (subscriber side)
CREATE SUBSCRIPTION orders_sub
  CONNECTION 'host=source dbname=mydb'
  PUBLICATION orders_pub;

-- Monitor
SELECT * FROM pg_publication_tables;
SELECT * FROM pg_stat_subscription;
```

---

**Q9. How do you implement and troubleshoot connection pooling issues in PgBouncer?**

**Answer:**

**Common issues and solutions:**

**Issue 1: "too many clients" error**
```ini
# Increase max_client_conn in pgbouncer.ini
max_client_conn = 2000
default_pool_size = 50
```

**Issue 2: Prepared statements failing with transaction pooling**
```ini
# Transaction pooling breaks prepared statements — either use session pooling
# or disable prepared statements at app level
server_reset_query = DISCARD ALL
```

**Issue 3: Long-running idle connections**
```ini
client_idle_timeout = 60
server_idle_timeout = 600
```

**Monitoring PgBouncer:**
```bash
# Connect to pgbouncer admin console
psql -h 127.0.0.1 -p 6432 -U pgbouncer pgbouncer

SHOW POOLS;       -- see pool states
SHOW CLIENTS;     -- connected clients
SHOW SERVERS;     -- backend connections
SHOW STATS;       -- query rates
```

---

### ADVANCED LEVEL

---

**Q10. Explain MVCC and how it causes table bloat. How do you handle it at scale?**

**Answer:**

**MVCC (Multi-Version Concurrency Control):**
PostgreSQL never overwrites a row in-place. UPDATE creates a new version (tuple) and marks the old one as dead. DELETE only marks the row as dead. This means:
- Readers never block writers
- But dead tuples accumulate → table bloat

**At scale — bloat management strategy:**

```sql
-- 1. Monitor bloat
SELECT schemaname, tablename,
       pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total_size,
       n_dead_tup, n_live_tup,
       round(n_dead_tup * 100.0 / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_pct
FROM pg_stat_user_tables
WHERE n_dead_tup > 10000
ORDER BY n_dead_tup DESC;

-- 2. Aggressive autovacuum for high-churn tables
ALTER TABLE orders SET (
  autovacuum_vacuum_scale_factor = 0.01,
  autovacuum_vacuum_threshold = 100,
  autovacuum_vacuum_cost_delay = 2
);

-- 3. For severe bloat — pg_repack (online, no full lock)
-- Install: apt install postgresql-15-repack
pg_repack -d mydb -t orders --no-superuser-check

-- 4. Check index bloat
SELECT indexname, pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
JOIN pg_index USING (indexrelid)
ORDER BY pg_relation_size(indexrelid) DESC;

-- Reindex concurrently (no downtime)
REINDEX INDEX CONCURRENTLY idx_orders_customer_id;
```

---

**Q11. How do you diagnose and resolve replication lag in PostgreSQL?**

**Answer:**

```sql
-- Step 1: Check lag on primary
SELECT
  application_name,
  state,
  sent_lsn,
  write_lsn,
  flush_lsn,
  replay_lsn,
  (sent_lsn - replay_lsn) AS replication_lag_bytes,
  write_lag,
  flush_lag,
  replay_lag
FROM pg_stat_replication;

-- Step 2: Check WAL receiver on standby
SELECT * FROM pg_stat_wal_receiver;

-- Step 3: Identify lag cause
-- a) Network bandwidth: monitor sent_lsn vs replay_lsn gap over time
-- b) Standby under I/O pressure: check iostat on standby
-- c) Long transactions on standby blocking replay (pg_stat_activity)
-- d) Hot standby conflict: check pg_stat_database_conflicts
```

**Resolution strategies:**
```sql
-- If hot standby conflicts are causing cancels:
-- Increase recovery_min_apply_delay or set:
hot_standby_feedback = on   -- prevents vacuum on primary from canceling standby queries
max_standby_streaming_delay = 30s

-- If standby is falling behind due to bulk loads:
-- Temporarily increase wal_keep_size on primary
ALTER SYSTEM SET wal_keep_size = '2GB';
SELECT pg_reload_conf();

-- Force WAL sync
SELECT pg_switch_wal();  -- on primary
```

---

### SECTION 2: MYSQL & MARIADB ADMINISTRATION

---

### BASIC LEVEL

---

**Q12. What is the difference between InnoDB and MyISAM?**

**Answer:**

| Feature | InnoDB | MyISAM |
|---|---|---|
| Transactions | Yes (ACID) | No |
| Foreign Keys | Yes | No |
| Row-level locking | Yes | Table-level |
| Crash recovery | Yes (redo log) | Limited |
| Full-text search | Yes (5.6+) | Yes |
| MVCC | Yes | No |

InnoDB is the default and recommended engine. MyISAM is essentially deprecated for OLTP.

```sql
-- Check engine
SHOW TABLE STATUS LIKE 'orders'\G

-- Convert to InnoDB
ALTER TABLE myisam_table ENGINE = InnoDB;
```

---

**Q13. Explain MySQL Group Replication vs InnoDB Cluster.**

**Answer:**

- **MySQL Group Replication** — the underlying replication technology. Uses Paxos-based distributed consensus for multi-primary or single-primary mode.
- **InnoDB Cluster** — the complete HA solution that combines Group Replication + MySQL Shell + MySQL Router for automatic failover and client routing.

```
InnoDB Cluster
├── Group Replication (replication layer)
├── MySQL Router (connection routing — reads/writes)
└── MySQL Shell (management interface)
```

```bash
# Set up InnoDB Cluster via MySQL Shell
mysqlsh

# Initialize cluster
dba.configureInstance('root@node1:3306')
var cluster = dba.createCluster('myCluster')
cluster.addInstance('root@node2:3306')
cluster.addInstance('root@node3:3306')

# Check status
cluster.status()
```

---

**Q14. How do you perform MySQL 5.7 to 8.0 upgrade safely?**

**Answer:**

```bash
# Step 1: Pre-upgrade checks (on 5.7)
mysqlcheck -u root -p --all-databases --check-upgrade
mysql_upgrade --check-only

# Step 2: Backup everything
mysqldump --all-databases --single-transaction --routines --events \
  -u root -p > full_backup.sql

# Step 3: On a test server — restore and upgrade to 8.0
# Install MySQL 8.0, point to same data dir or restore backup

# Step 4: Run mysql_upgrade (8.0 auto-runs on start)
# MySQL 8.0 auto-upgrades on first start

# Step 5: Key breaking changes to validate:
# - utf8mb4 default charset (was latin1 in 5.7)
# - Strict SQL mode by default
# - Reserved words (e.g., RANK, GROUPS, CUBE)
# - Authentication plugin: caching_sha2_password (was mysql_native_password)

# Fix auth plugin if apps break:
ALTER USER 'appuser'@'%' IDENTIFIED WITH mysql_native_password BY 'password';
```

---

### INTERMEDIATE LEVEL

---

**Q15. How does semi-synchronous replication work and when do you use it?**

**Answer:**

In **asynchronous replication** (default), primary commits and does NOT wait for replica acknowledgment — data loss risk on failover.

In **semi-synchronous replication**, primary waits for at least ONE replica to acknowledge writing the relay log before committing to the client.

```sql
-- Enable on primary
INSTALL PLUGIN rpl_semi_sync_source SONAME 'semisync_source.so';
SET GLOBAL rpl_semi_sync_source_enabled = 1;
SET GLOBAL rpl_semi_sync_source_timeout = 10000; -- 10s timeout, fallback to async

-- Enable on replica
INSTALL PLUGIN rpl_semi_sync_replica SONAME 'semisync_replica.so';
SET GLOBAL rpl_semi_sync_replica_enabled = 1;

-- Verify
SHOW STATUS LIKE 'Rpl_semi_sync%';
```

**Use case:** Financial transactions, banking apps, healthcare — where losing even a single committed transaction is unacceptable.

---

**Q16. How do you troubleshoot and fix MySQL replication errors?**

**Answer:**

```sql
-- Check replica status
SHOW REPLICA STATUS\G

-- Common errors:
-- Error 1062: Duplicate entry (row already exists on replica)
-- Error 1032: Row not found for update/delete

-- Fix 1062: Skip the error (use sparingly)
STOP REPLICA;
SET GLOBAL SQL_REPLICA_SKIP_COUNTER = 1;
START REPLICA;

-- Better fix: Use idempotent mode
SET GLOBAL SLAVE_EXEC_MODE = 'IDEMPOTENT';

-- Fix 1032: Inject the missing row from primary
-- 1. Stop replica
STOP REPLICA;
-- 2. Dump the specific row from primary
mysqldump -h primary -u root -p mydb orders --where="id=12345" > fix.sql
-- 3. Apply on replica
mysql mydb < fix.sql
-- 4. Start replica
START REPLICA;

-- For GTID-based replication error skip:
STOP REPLICA;
SET GTID_NEXT = 'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx:N';
BEGIN; COMMIT;
SET GTID_NEXT = 'AUTOMATIC';
START REPLICA;
```

---

**Q17. Explain ProxySQL and its role in MySQL HA architecture.**

**Answer:**

ProxySQL is an advanced MySQL proxy that provides:
- **Read/Write splitting** — writes to primary, reads to replicas
- **Connection pooling** — reduces connection overhead
- **Query routing** — route specific queries to specific servers
- **Query mirroring** — send queries to secondary for testing
- **Failover** — automatic traffic rerouting on node failure

```sql
-- ProxySQL admin console
mysql -u admin -padmin -h 127.0.0.1 -P 6032

-- Add MySQL servers
INSERT INTO mysql_servers(hostgroup_id, hostname, port) VALUES (1,'primary',3306);
INSERT INTO mysql_servers(hostgroup_id, hostname, port) VALUES (2,'replica1',3306);
INSERT INTO mysql_servers(hostgroup_id, hostname, port) VALUES (2,'replica2',3306);

-- Define read/write split rules
INSERT INTO mysql_query_rules(rule_id, active, match_pattern, destination_hostgroup)
VALUES (1, 1, '^SELECT', 2);   -- reads to replicas (hostgroup 2)
-- Default writes go to hostgroup 1

-- Apply changes
LOAD MYSQL SERVERS TO RUNTIME;
SAVE MYSQL SERVERS TO DISK;
```

---

**Q18. How do you configure and use Percona XtraBackup for hot backups?**

**Answer:**

Percona XtraBackup performs non-blocking physical backups of InnoDB databases — no table locks.

```bash
# Full backup
xtrabackup --backup --user=root --password='pass' \
  --target-dir=/backup/full_$(date +%Y%m%d)

# Incremental backup (after full)
xtrabackup --backup --user=root --password='pass' \
  --target-dir=/backup/inc_$(date +%Y%m%d_%H) \
  --incremental-basedir=/backup/full_20240101

# Prepare (apply redo logs — BEFORE restore)
xtrabackup --prepare --target-dir=/backup/full_20240101

# Apply incremental
xtrabackup --prepare --target-dir=/backup/full_20240101 \
  --incremental-dir=/backup/inc_20240101_12

# Restore
xtrabackup --copy-back --target-dir=/backup/full_20240101
chown -R mysql:mysql /var/lib/mysql
```

---

## ADVANCED LEVEL

---

**Q19. How do you handle MySQL InnoDB deadlocks at scale?**

**Answer:**

```sql
-- Step 1: Identify deadlocks
SHOW ENGINE INNODB STATUS\G
-- Look for LATEST DETECTED DEADLOCK section

-- Step 2: Enable deadlock logging
SET GLOBAL innodb_print_all_deadlocks = ON;
-- Check /var/log/mysql/error.log

-- Step 3: Analyze the deadlock
-- Typical pattern: TXN1 holds lock on Row A, wants Row B
--                  TXN2 holds lock on Row B, wants Row A

-- Step 4: Fix strategies:
-- a) Consistent lock ordering (always lock tables/rows in same order)
-- b) Smaller transactions (shorter lock hold time)
-- c) Use SELECT ... FOR UPDATE only when necessary
-- d) Add covering indexes to reduce lock scope

-- Step 5: Monitor lock waits
SELECT
  r.trx_id waiting_trx_id,
  r.trx_mysql_thread_id waiting_thread,
  r.trx_query waiting_query,
  b.trx_id blocking_trx_id,
  b.trx_mysql_thread_id blocking_thread,
  b.trx_query blocking_query
FROM information_schema.innodb_lock_waits w
INNER JOIN information_schema.innodb_trx b ON b.trx_id = w.blocking_trx_id
INNER JOIN information_schema.innodb_trx r ON r.trx_id = w.requesting_trx_id;
```

---

### SECTION 3: MULTI-DATABASE MANAGEMENT

---

### CASSANDRA

---

**Q20. Explain Cassandra's consistency levels and how you choose them.**

**Answer:**

Cassandra uses the formula: **R + W > RF** (Reads + Writes > Replication Factor) for strong consistency.

| Consistency Level | Description |
|---|---|
| ONE | Fastest, least durable |
| QUORUM | Majority (RF/2 + 1) — balanced |
| LOCAL_QUORUM | Quorum within local DC only |
| ALL | All replicas — strongest, slowest |
| ANY | At least one node — highest availability |

```sql
-- For RF=3, QUORUM = 2 nodes must acknowledge

-- Write with LOCAL_QUORUM (recommended for multi-DC)
CONSISTENCY LOCAL_QUORUM;
INSERT INTO orders (id, customer, amount) VALUES (uuid(), 'Alice', 100.00);

-- Read with LOCAL_QUORUM
SELECT * FROM orders WHERE id = ?;
```

**Rule of thumb:** Use `LOCAL_QUORUM` for both reads and writes in multi-DC setups — strong consistency within DC, performance across DCs.

---

**Q21. How do you handle Cassandra compaction strategies?**

**Answer:**

| Strategy | Best For |
|---|---|
| STCS (SizeTieredCompactionStrategy) | Write-heavy, time-series |
| LCS (LeveledCompactionStrategy) | Read-heavy, random reads |
| TWCS (TimeWindowCompactionStrategy) | Time-series with TTL |

```sql
-- Check current strategy
DESCRIBE TABLE orders;

-- Change to LCS for read-heavy table
ALTER TABLE orders WITH compaction = {
  'class': 'LeveledCompactionStrategy',
  'sstable_size_in_mb': 160
};

-- For time-series data (IoT, logs)
ALTER TABLE sensor_data WITH compaction = {
  'class': 'TimeWindowCompactionStrategy',
  'compaction_window_unit': 'DAYS',
  'compaction_window_size': 1
};

-- Monitor compaction
nodetool compactionstats
nodetool compactionhistory
```

---

### AWS RDS AURORA

---

**Q22. How does Aurora differ from standard RDS MySQL/PostgreSQL?**

**Answer:**

| Feature | RDS | Aurora |
|---|---|---|
| Storage | Instance-local EBS | Shared distributed storage |
| Read replicas | Up to 5 | Up to 15 |
| Failover time | 60-120 seconds | 30 seconds (typically <30s) |
| Storage auto-scale | Manual | Auto (10GB → 128TB) |
| PITR | Yes | Yes (down to the second) |
| Backtrack | No | Yes (Aurora MySQL) |
| Global DB | No | Yes |

```sql
-- Aurora-specific: check cluster endpoints
-- Writer endpoint: always points to primary
-- Reader endpoint: load-balances across read replicas

-- Check Aurora replication lag
SELECT SERVER_ID, SESSION_ID, LAST_UPDATE_TIMESTAMP, REPLICA_LAG_IN_MILLISECONDS
FROM information_schema.replica_host_status;

-- Aurora backtrack (roll back without restore)
-- In AWS console or CLI
aws rds backtrack-db-cluster \
  --db-cluster-identifier my-aurora-cluster \
  --backtrack-to "2024-03-01T10:00:00+00:00"
```

---

### TDE (TRANSPARENT DATA ENCRYPTION)

---

**Q23. How do you implement TDE across MySQL, PostgreSQL, and MSSQL?**

**Answer:**

**MySQL (Enterprise — InnoDB TDE):**
```sql
-- Enable keyring plugin (mysql_enterprise_encryption or keyring_file)
-- my.cnf:
early-plugin-load=keyring_file.so
keyring_file_data=/var/lib/mysql/keyring/keyring

-- Encrypt existing tablespace
ALTER TABLE sensitive_data ENCRYPTION='Y';

-- Encrypt new tables by default
SET GLOBAL default_table_encryption = ON;

-- Verify encryption
SELECT NAME, ENCRYPTION FROM information_schema.INNODB_TABLESPACES
WHERE NAME = 'mydb/sensitive_data';
```

**PostgreSQL (pgcrypto + tablespace encryption or Transparent Data Encryption via pg_tde):**
```sql
-- Using pgcrypto for column-level encryption
CREATE EXTENSION pgcrypto;

INSERT INTO users (name, ssn)
VALUES ('Alice', pgp_sym_encrypt('123-45-6789', 'encryption_key'));

SELECT pgp_sym_decrypt(ssn::bytea, 'encryption_key') FROM users WHERE name='Alice';
```

**MSSQL TDE:**
```sql
-- Step 1: Create master key
USE master;
CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'StrongPass@123';

-- Step 2: Create certificate
CREATE CERTIFICATE TDE_Cert WITH SUBJECT = 'TDE Certificate';

-- Step 3: Create database encryption key
USE mydb;
CREATE DATABASE ENCRYPTION KEY
  WITH ALGORITHM = AES_256
  ENCRYPTION BY SERVER CERTIFICATE TDE_Cert;

-- Step 4: Enable encryption
ALTER DATABASE mydb SET ENCRYPTION ON;

-- Verify
SELECT name, is_encrypted FROM sys.databases;
```

---

### SECTION 4: KUBERNETES & DATABASE MANAGEMENT

---

**Q24. How do you deploy a stateful MySQL cluster on Kubernetes?**

**Answer:**

```yaml
# mysql-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: prod
spec:
  serviceName: mysql
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: root-password
        ports:
        - containerPort: 3306
        resources:
          requests:
            memory: "2Gi"
            cpu: "1"
          limits:
            memory: "4Gi"
            cpu: "2"
        volumeMounts:
        - name: mysql-data
          mountPath: /var/lib/mysql
        readinessProbe:
          exec:
            command: ["mysqladmin", "ping", "-h", "localhost"]
          initialDelaySeconds: 30
          periodSeconds: 10
  volumeClaimTemplates:
  - metadata:
      name: mysql-data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: "fast-ssd"
      resources:
        requests:
          storage: 100Gi
```

```yaml
# mysql-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
type: Opaque
stringData:
  root-password: "StrongPass@123"

# mysql-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-config
data:
  my.cnf: |
    [mysqld]
    innodb_buffer_pool_size=2G
    max_connections=500
    slow_query_log=1
    long_query_time=1
```

---

**Q25. How do you troubleshoot a PostgreSQL pod in CrashLoopBackOff?**

**Answer:**

```bash
# Step 1: Check pod status and events
kubectl get pods -n prod
kubectl describe pod postgres-0 -n prod
# Look for: OOMKilled, failed liveness probe, volume mount errors

# Step 2: Check logs
kubectl logs postgres-0 -n prod --previous
kubectl logs postgres-0 -n prod -c postgres

# Step 3: Common causes and fixes:

# --- OOMKilled ---
# Pod ran out of memory. Check:
kubectl top pod postgres-0 -n prod
# Fix: increase memory limits in StatefulSet
# limits: memory: "8Gi"
# Or tune shared_buffers, work_mem in postgresql.conf

# --- PVC Binding Failure ---
kubectl get pvc -n prod
kubectl describe pvc postgres-data-postgres-0 -n prod
# Fix: check StorageClass, ensure PV exists, check AccessMode

# --- Data directory permission issues ---
kubectl exec -it postgres-0 -n prod -- ls -la /var/lib/postgresql/data
# Fix: add initContainer to chown
initContainers:
- name: fix-permissions
  image: busybox
  command: ["sh", "-c", "chown -R 999:999 /var/lib/postgresql/data"]
  volumeMounts:
  - name: postgres-data
    mountPath: /var/lib/postgresql/data

# --- Config errors ---
kubectl exec -it postgres-0 -- postgres --config-file=/etc/postgresql/postgresql.conf --dry-run
```

---

**Q26. How do you perform a zero-downtime database upgrade on Kubernetes?**

**Answer:**

```bash
# Strategy: Rolling update with readiness probe validation

# Step 1: Update image in StatefulSet
kubectl set image statefulset/postgres postgres=postgres:16 -n prod

# Step 2: Monitor rollout
kubectl rollout status statefulset/postgres -n prod

# Step 3: StatefulSet rolling update behavior
# - Updates pods one at a time (highest ordinal first)
# - Waits for readiness probe before proceeding to next pod
# - If a pod fails — rollout pauses automatically

# Step 4: For MySQL — use Orchestrator or ProxySQL to:
# a) Remove the pod being upgraded from the load balancer
# b) Upgrade the pod
# c) Re-add to load balancer after health check passes

# Rollback if needed
kubectl rollout undo statefulset/postgres -n prod

# For major version upgrades (e.g., PG14 → PG16)
# Use init container with pg_upgrade or
# Provision new StatefulSet, migrate via logical replication,
# switch traffic at ingress/service level
```

---

# SECTION 5: CI/CD PIPELINE & DATABASE AUTOMATION

---

**Q27. How do you integrate database deployments into a GitLab CI/CD pipeline?**

**Answer:**

```yaml
# .gitlab-ci.yml
stages:
  - validate
  - deploy-dev
  - deploy-staging
  - deploy-prod

variables:
  FLYWAY_URL: "jdbc:postgresql://${DB_HOST}:5432/${DB_NAME}"

validate-migrations:
  stage: validate
  image: flyway/flyway:latest
  script:
    - flyway -url=$FLYWAY_URL -user=$DB_USER -password=$DB_PASS validate
  only:
    - merge_requests

deploy-dev:
  stage: deploy-dev
  image: flyway/flyway:latest
  script:
    - flyway -url=jdbc:postgresql://$DEV_DB_HOST:5432/$DB_NAME
             -user=$DB_USER -password=$DB_PASS migrate
  environment:
    name: dev
  only:
    - develop

deploy-staging:
  stage: deploy-staging
  script:
    - flyway -url=jdbc:postgresql://$STAGING_DB_HOST:5432/$DB_NAME migrate
  environment:
    name: staging
  when: manual   # Approval gate
  only:
    - main

deploy-prod:
  stage: deploy-prod
  script:
    - flyway -url=jdbc:postgresql://$PROD_DB_HOST:5432/$DB_NAME migrate
  environment:
    name: production
  when: manual
  only:
    - tags   # Only on version tags
```

---

# SECTION 6: FLYWAY & LIQUIBASE

---

**Q28. What is the difference between Flyway and Liquibase? When do you choose one over the other?**

**Answer:**

| Feature | Flyway | Liquibase |
|---|---|---|
| Config format | SQL / Java | XML, YAML, JSON, SQL |
| Versioning | Sequential (V1, V2) | Changeset-based |
| Rollback | Manual (undo scripts in Teams) | Built-in rollback |
| Learning curve | Simple | Moderate |
| Multi-DB diff apply | Limited | Strong (changelogs) |
| Dry run | Pro only | Free (updateSQL) |

**Choose Flyway when:** Simple, SQL-heavy migrations, small team, rapid iteration.

**Choose Liquibase when:** Complex multi-environment deployments, need rollbacks, multiple DB types, enterprise compliance.

```sql
-- Flyway naming convention
-- V1__Create_orders_table.sql
-- V2__Add_index_on_customer.sql
-- R__Recreate_view.sql (repeatable)

-- Check migration status
flyway info -url=jdbc:mysql://host:3306/mydb -user=root -password=pass

-- Migrate
flyway migrate

-- Repair (fix checksum mismatches)
flyway repair
```

```yaml
# Liquibase changeset (YAML)
databaseChangeLog:
  - changeSet:
      id: 001-create-orders
      author: venkat.sarath
      changes:
        - createTable:
            tableName: orders
            columns:
              - column:
                  name: id
                  type: BIGINT
                  autoIncrement: true
                  constraints:
                    primaryKey: true
              - column:
                  name: customer_id
                  type: BIGINT
                  constraints:
                    nullable: false
      rollback:
        - dropTable:
            tableName: orders
```

---

# SECTION 7: TERRAFORM & IaC FOR DATABASES

---

**Q29. How do you provision an RDS PostgreSQL instance using Terraform?**

**Answer:**

```hcl
# variables.tf
variable "env" { default = "prod" }
variable "db_password" { sensitive = true }

# main.tf
provider "aws" {
  region = "us-east-1"
}

resource "aws_db_subnet_group" "postgres" {
  name       = "${var.env}-postgres-subnet-group"
  subnet_ids = var.private_subnet_ids
}

resource "aws_db_parameter_group" "postgres" {
  name   = "${var.env}-postgres15-params"
  family = "postgres15"

  parameter {
    name  = "log_min_duration_statement"
    value = "1000"
  }
  parameter {
    name  = "shared_preload_libraries"
    value = "pg_stat_statements"
  }
}

resource "aws_db_instance" "postgres" {
  identifier           = "${var.env}-postgres-db"
  engine               = "postgres"
  engine_version       = "15.4"
  instance_class       = "db.r6g.xlarge"
  allocated_storage    = 200
  max_allocated_storage = 1000   # auto-scaling
  storage_type         = "gp3"
  storage_encrypted    = true
  kms_key_id           = aws_kms_key.rds.arn

  db_name  = "appdb"
  username = "dbadmin"
  password = var.db_password

  db_subnet_group_name   = aws_db_subnet_group.postgres.name
  parameter_group_name   = aws_db_parameter_group.postgres.name
  vpc_security_group_ids = [aws_security_group.rds.id]

  multi_az               = true
  backup_retention_period = 7
  backup_window          = "03:00-04:00"
  maintenance_window     = "Mon:04:00-Mon:05:00"

  deletion_protection = true
  skip_final_snapshot = false
  final_snapshot_identifier = "${var.env}-postgres-final-snapshot"

  tags = {
    Environment = var.env
    ManagedBy   = "Terraform"
  }
}

# Read replica
resource "aws_db_instance" "postgres_replica" {
  identifier          = "${var.env}-postgres-replica"
  replicate_source_db = aws_db_instance.postgres.identifier
  instance_class      = "db.r6g.large"
  publicly_accessible = false
  skip_final_snapshot = true
}
```

---

**Q30. How do you manage Terraform state for multi-environment database infrastructure?**

**Answer:**

```hcl
# backend.tf — Remote state with S3 + DynamoDB locking
terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state"
    key            = "databases/${var.env}/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}

# Use workspaces for environments
terraform workspace new dev
terraform workspace new staging
terraform workspace new prod
terraform workspace select prod

# Reference workspace in resources
resource "aws_db_instance" "mysql" {
  instance_class = terraform.workspace == "prod" ? "db.r6g.2xlarge" : "db.t3.medium"
}
```

---

# SECTION 8: PERFORMANCE TUNING — DEEP DIVE

---

## PostgreSQL Performance Tuning

---

**Q31. What are the most critical PostgreSQL parameters to tune for a production OLTP workload?**

**Answer:**

```ini
# postgresql.conf — OLTP tuning

# Memory
shared_buffers = 25%_of_RAM           # e.g., 16GB for 64GB RAM
effective_cache_size = 75%_of_RAM     # query planner hint
work_mem = 64MB                        # per sort/hash operation
maintenance_work_mem = 1GB            # VACUUM, CREATE INDEX

# WAL
wal_buffers = 64MB
checkpoint_completion_target = 0.9   # spread checkpoint I/O
checkpoint_timeout = 15min
max_wal_size = 4GB                    # before checkpoint forced

# Query Planner
random_page_cost = 1.1               # for SSD (default 4.0 is for HDD)
effective_io_concurrency = 200       # parallel I/O (SSD)
default_statistics_target = 200      # more histogram buckets

# Parallelism
max_parallel_workers_per_gather = 4
max_parallel_workers = 8
max_worker_processes = 16

# Connections
max_connections = 200                 # keep low, use PgBouncer
```

---

**Q32. How do you identify and fix a slow query in PostgreSQL?**

**Answer:**

```sql
-- Step 1: Enable slow query logging
SET log_min_duration_statement = 1000;  -- log queries > 1 second

-- Step 2: Find slow queries via pg_stat_statements
SELECT
  query,
  calls,
  total_exec_time / 1000 AS total_seconds,
  mean_exec_time AS avg_ms,
  stddev_exec_time AS stddev_ms,
  rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;

-- Step 3: EXPLAIN ANALYZE the problem query
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.created_at > NOW() - INTERVAL '7 days';

-- Step 4: Read the plan
-- Look for:
-- "Seq Scan" on large tables → missing index
-- "Nested Loop" with high rows → bad cardinality estimate
-- "Hash Join" spilling to disk → increase work_mem
-- High "Buffers: read" → data not in shared_buffers

-- Step 5: Fix
-- Add missing index
CREATE INDEX CONCURRENTLY idx_orders_created_at ON orders(created_at);

-- Update statistics if estimates are wrong
ANALYZE orders;

-- Force planner to re-evaluate
ALTER TABLE orders ALTER COLUMN created_at SET STATISTICS 500;
ANALYZE orders;

-- Step 6: Verify improvement
EXPLAIN (ANALYZE, BUFFERS) SELECT ... -- same query, check cost drops
```

---

**Q33. What is index bloat and how do you handle it?**

**Answer:**

```sql
-- Check index bloat (using pgstattuple extension)
CREATE EXTENSION pgstattuple;

SELECT
  indexname,
  pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
  round((pgstattuple(indexrelid)).dead_tuple_percent::numeric, 2) AS dead_pct
FROM pg_stat_user_indexes
JOIN pg_index USING (indexrelid)
WHERE schemaname = 'public'
ORDER BY pg_relation_size(indexrelid) DESC;

-- Rebuild index with no downtime (PostgreSQL 12+)
REINDEX INDEX CONCURRENTLY idx_orders_customer_id;

-- Or create new index and drop old
CREATE INDEX CONCURRENTLY idx_orders_customer_id_new ON orders(customer_id);
DROP INDEX CONCURRENTLY idx_orders_customer_id;
ALTER INDEX idx_orders_customer_id_new RENAME TO idx_orders_customer_id;
```

---

## MySQL Performance Tuning

---

**Q34. What are the key InnoDB parameters for a high-throughput MySQL server?**

**Answer:**

```ini
# my.cnf — InnoDB tuning

# Buffer Pool (most important — 70-80% of RAM)
innodb_buffer_pool_size = 48G          # for 64GB RAM server
innodb_buffer_pool_instances = 16      # 1 per GB, max 64

# Redo Log
innodb_log_file_size = 4G             # larger = fewer checkpoints, better write throughput
innodb_log_buffer_size = 256M

# I/O
innodb_io_capacity = 2000             # IOPS your storage can handle
innodb_io_capacity_max = 4000
innodb_flush_method = O_DIRECT        # bypass OS cache (prevents double buffering)
innodb_read_io_threads = 16
innodb_write_io_threads = 16

# Transactions
innodb_flush_log_at_trx_commit = 1    # 1 = ACID, 2 = faster (1s data loss risk)
sync_binlog = 1                        # 1 = ACID for binlog

# Connections
max_connections = 500
thread_cache_size = 100

# Temp tables
tmp_table_size = 256M
max_heap_table_size = 256M
```

---

**Q35. How do you analyze and fix a slow MySQL query?**

**Answer:**

```sql
-- Step 1: Enable slow query log
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 1;
SET GLOBAL log_queries_not_using_indexes = ON;

-- Step 2: Analyze with pt-query-digest (Percona Toolkit)
pt-query-digest /var/log/mysql/slow.log > digest_report.txt

-- Step 3: EXPLAIN the query
EXPLAIN FORMAT=JSON
SELECT o.*, c.name
FROM orders o
INNER JOIN customers c ON o.customer_id = c.id
WHERE o.status = 'PENDING'
  AND o.created_at > DATE_SUB(NOW(), INTERVAL 7 DAY);

-- Step 4: Read EXPLAIN output
-- type: ALL = full table scan (BAD)
-- type: ref/eq_ref = index lookup (GOOD)
-- type: index = full index scan (OK)
-- Extra: "Using filesort" → add ORDER BY to index
-- Extra: "Using temporary" → optimize GROUP BY / DISTINCT

-- Step 5: Add composite index
-- Rule: (equality columns first, range columns last, covering index ideal)
CREATE INDEX idx_orders_status_created ON orders(status, created_at);

-- Step 6: For covering index (avoid table lookup)
CREATE INDEX idx_orders_cover ON orders(status, created_at, customer_id, id);

-- Step 7: Verify
EXPLAIN SELECT ... -- type should now be 'range' or 'ref'

-- Step 8: Check index usage
SELECT * FROM sys.schema_unused_indexes;
SELECT * FROM sys.schema_redundant_indexes;
```

---

**Q36. What is the InnoDB buffer pool and how do you monitor it?**

**Answer:**

The InnoDB buffer pool is the most critical memory component — it caches data pages and index pages to avoid disk I/O.

```sql
-- Check buffer pool hit ratio (should be > 99%)
SELECT
  (1 - (Innodb_buffer_pool_reads / Innodb_buffer_pool_read_requests)) * 100
  AS buffer_pool_hit_ratio
FROM (
  SELECT
    variable_value AS Innodb_buffer_pool_reads
  FROM information_schema.global_status
  WHERE variable_name = 'Innodb_buffer_pool_reads'
) r,
(
  SELECT
    variable_value AS Innodb_buffer_pool_read_requests
  FROM information_schema.global_status
  WHERE variable_name = 'Innodb_buffer_pool_read_requests'
) rr;

-- Detailed buffer pool stats
SHOW ENGINE INNODB STATUS\G
-- Look for: Buffer pool hit rate 999/1000 (target: > 990/1000)

-- Monitor dirty pages (pending writes)
SHOW STATUS LIKE 'Innodb_buffer_pool_pages_dirty';

-- Check if buffer pool is too small
SHOW STATUS LIKE 'Innodb_buffer_pool_wait_free';
-- If > 0 → buffer pool is thrashing, increase size
```

---

# SECTION 9: SCENARIO-BASED QUESTIONS

---

**Q37. SCENARIO: Your primary PostgreSQL server goes down at 2 AM. What do you do?**

**Answer:**

```
Step-by-step incident response:

1. ALERT ACKNOWLEDGED (0-2 min)
   - Acknowledge PagerDuty / OpsGenie alert
   - Join war room (Slack/Teams)

2. ASSESS (2-5 min)
   - ssh to primary: is it responding?
   - Check: ssh, ping, telnet port 5432
   - Check monitoring: Datadog / CloudWatch
   - Is it network? Hardware? Process crash?

3. IF USING PATRONI:
   - patronictl -c /etc/patroni.yml list
   - If primary is truly down, Patroni auto-promotes standby
   - Verify: check which node is now leader
   - Update application connection strings if not using VIP/HAProxy

4. MANUAL FAILOVER (if no automation):
   - On standby: check replication lag
     SELECT * FROM pg_stat_wal_receiver;
   - Promote standby:
     pg_ctl promote -D /var/lib/postgresql/data
     -- or --
     SELECT pg_promote();
   - Update DNS / Load balancer / HAProxy to point to new primary

5. APPLICATION RECONNECTION
   - PgBouncer: RELOAD config or RECONNECT
   - Verify application connections resuming
   - Check for any failed transactions (alert dev team)

6. POST-RECOVERY
   - Diagnose root cause of primary failure
   - Rebuild failed server as new standby
   - pg_basebackup from new primary to old server
   - Run pg_rewind if primary had diverged:
     pg_rewind --target-pgdata=/data --source-server="host=new_primary ..."

7. POST-MORTEM
   - Write RCA document
   - Review monitoring gaps
   - Test failover drill quarterly
```

---

**Q38. SCENARIO: MySQL replication is lagging 2 hours behind. Production is using the replica for reads. How do you handle this?**

**Answer:**

```sql
-- Step 1: Assess severity
SHOW REPLICA STATUS\G
-- Check: Seconds_Behind_Source: 7200 (2 hours)

-- Step 2: Immediate mitigation
-- Route read traffic away from this replica
-- Update ProxySQL/HAProxy to remove the lagging replica from read pool
-- MySQL 8.0 ProxySQL:
UPDATE mysql_servers SET STATUS='OFFLINE_SOFT' WHERE hostname='replica2';
LOAD MYSQL SERVERS TO RUNTIME;

-- Step 3: Identify root cause
-- a) Long-running query on replica?
SHOW PROCESSLIST;
-- Kill blocking process if safe

-- b) Single-threaded replication bottleneck?
SHOW VARIABLES LIKE 'slave_parallel_workers';
-- If 0 — enable parallel replication
SET GLOBAL slave_parallel_type = 'LOGICAL_CLOCK';
SET GLOBAL slave_parallel_workers = 8;
STOP REPLICA; START REPLICA;

-- c) Network issue? Check SHOW REPLICA STATUS for I/O thread state

-- d) Large transaction on primary?
-- On primary: check binlog for large events
mysqlbinlog --start-position=<pos> /var/lib/mysql/binlog.000001 | head -100

-- Step 4: Speed up replica catch-up
SET GLOBAL innodb_flush_log_at_trx_commit = 2;   -- reduce disk sync (catch-up mode)
SET GLOBAL sync_binlog = 0;

-- Step 5: Monitor lag closing
watch -n 5 "mysql -e 'SHOW REPLICA STATUS\G' | grep Seconds_Behind"

-- Step 6: Once caught up, restore settings and add back to pool
SET GLOBAL innodb_flush_log_at_trx_commit = 1;
SET GLOBAL sync_binlog = 1;
-- Re-add to ProxySQL
UPDATE mysql_servers SET STATUS='ONLINE' WHERE hostname='replica2';
```

---

**Q39. SCENARIO: A developer ran DELETE without WHERE in production PostgreSQL. How do you recover?**

**Answer:**

```sql
-- OPTION 1: PITR (Point-In-Time Recovery) — if WAL archiving enabled
-- Most reliable. Requires downtime.

-- Step 1: Note the time the DELETE occurred (from logs/monitoring)
-- Step 2: Restore from last base backup to a recovery server
-- Step 3: Create recovery.conf (PG12-) or postgresql.conf (PG12+)

recovery_target_time = '2024-03-15 14:23:00'
recovery_target_action = 'promote'

-- Step 4: Start recovery server, it replays WAL up to target time
-- Step 5: Extract the lost data
pg_dump -t affected_table recovery_db > recovered_data.sql

-- OPTION 2: Logical Backup (if recent pg_dump exists)
pg_restore -d mydb -t orders --data-only orders_dump.dump

-- OPTION 3: Flashback via pgaudit / transaction logs
-- If pgaudit was logging, you might reconstruct INSERTs from audit log

-- OPTION 4: Using pg_undolog extension or EDB Postgres Advanced Server

-- PREVENTION:
-- 1. Wrap destructive operations in transactions
BEGIN;
DELETE FROM orders WHERE status = 'OLD';
-- Verify row count before COMMIT
SELECT count(*) FROM orders;
COMMIT;  -- or ROLLBACK

-- 2. Enable statement-level audit logging
-- 3. Use row-level security to prevent bulk deletes without WHERE
-- 4. Application-level soft deletes (is_deleted flag)
```

---

**Q40. SCENARIO: Your Kubernetes PostgreSQL pod is being OOMKilled every few hours. How do you fix it?**

**Answer:**

```bash
# Step 1: Confirm OOMKilled
kubectl describe pod postgres-0 -n prod
# Look for: Last State: Terminated — Reason: OOMKilled

kubectl get events -n prod | grep OOM

# Step 2: Check actual memory usage trend
kubectl top pod postgres-0 -n prod
# Use Prometheus/Grafana for historical trend

# Step 3: Root causes and fixes:

# --- Cause A: shared_buffers too high for container limits ---
# shared_buffers uses shared memory (SHM), counts against container memory
# If limit = 4Gi and shared_buffers = 3Gi, OS + connections exceed limit

# Fix: Set shared_buffers to 25% of container limit
kubectl exec postgres-0 -- psql -c "SHOW shared_buffers;"
# Update ConfigMap:
shared_buffers = 1GB           # for 4GB container limit
effective_cache_size = 3GB

# --- Cause B: work_mem set too high ---
# work_mem is PER OPERATION PER QUERY — can multiply to gigabytes
# work_mem = 256MB * 4 parallel workers * 10 concurrent queries = 10GB!
# Fix:
work_mem = 32MB               # conservative for high concurrency

# --- Cause C: Memory limits too low ---
# Increase in StatefulSet:
resources:
  requests:
    memory: "4Gi"
  limits:
    memory: "8Gi"

# --- Cause D: Connection storm ---
# Too many connections × per-connection overhead
# Fix: Add PgBouncer as sidecar or separate deployment

# Step 4: Add memory alerts
# Prometheus alert: container_memory_usage_bytes / container_spec_memory_limit_bytes > 0.85
```

---

# SECTION 10: REAL ISSUE-BASED TROUBLESHOOTING

---

**Q41. ISSUE: PostgreSQL queries suddenly became slow after a deployment. Nothing changed in the database. What do you check?**

**Answer:**

```sql
-- 1. Check if statistics are stale (new data loaded, old stats)
SELECT relname, last_analyze, last_autoanalyze, n_live_tup, n_dead_tup
FROM pg_stat_user_tables
WHERE last_analyze < NOW() - INTERVAL '1 day'
ORDER BY n_live_tup DESC;

-- Fix: Force analyze
ANALYZE VERBOSE;

-- 2. Check for plan cache issues (prepared statements with bad plans)
SELECT * FROM pg_prepared_statements;
-- Application may have cached a bad plan
-- Fix: DEALLOCATE ALL (or restart connection pool)

-- 3. Check for lock contention
SELECT pid, wait_event_type, wait_event, state, query
FROM pg_stat_activity
WHERE wait_event IS NOT NULL
ORDER BY wait_event_type;

-- 4. New query introduced by deployment using sequential scan
SELECT query, calls, mean_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC LIMIT 10;

-- 5. Check for bloat increase (batch job ran on deploy?)
SELECT relname, n_dead_tup FROM pg_stat_user_tables ORDER BY n_dead_tup DESC;

-- 6. Check shared_buffers hit ratio
SELECT
  blks_hit, blks_read,
  round(blks_hit * 100.0 / NULLIF(blks_hit + blks_read, 0), 2) AS cache_hit_ratio
FROM pg_stat_database WHERE datname = current_database();
-- Should be > 98%
```

---

**Q42. ISSUE: MySQL shows "Too many connections" error. How do you handle it without restart?**

**Answer:**

```sql
-- Step 1: Check current connection count
SHOW STATUS LIKE 'Threads_connected';
SHOW VARIABLES LIKE 'max_connections';

-- Step 2: Find what's consuming connections
SELECT user, host, COUNT(*) AS conn_count, command
FROM information_schema.processlist
GROUP BY user, host, command
ORDER BY conn_count DESC;

-- Step 3: Kill idle connections (be careful in production)
SELECT CONCAT('KILL ', id, ';')
FROM information_schema.processlist
WHERE command = 'Sleep' AND time > 300;
-- Review and execute individually or use pt-kill

-- Step 4: Temporary fix — increase max_connections dynamically
SET GLOBAL max_connections = 1000;
-- Note: requires RAM — each connection ~1MB overhead

-- Step 5: Root cause fixes
-- a) App not releasing connections → check connection pool settings (min/max, timeout)
-- b) Deploy ProxySQL/PgBouncer to pool connections
-- c) Tune application thread pool
-- d) Connection leak in app → set wait_timeout / interactive_timeout
SET GLOBAL wait_timeout = 120;
SET GLOBAL interactive_timeout = 120;

-- Step 6: Monitor ongoing
SELECT variable_value FROM information_schema.global_status
WHERE variable_name = 'Max_used_connections';
-- Track peak usage
```

---

**Q43. ISSUE: Cassandra node shows "GC pause too long" alerts. How do you resolve?**

**Answer:**

```bash
# Step 1: Check GC stats
nodetool gcstats
# Look for: MaxGCPause > 200ms (alert threshold)

# Step 2: Identify GC type causing pauses
# Check cassandra/logs/system.log for GC warnings
grep "GCInspector" /var/log/cassandra/system.log | tail -50

# Step 3: Common causes and fixes:

# --- Cause A: Heap too small ---
# Edit cassandra-env.sh
MAX_HEAP_SIZE="8G"
HEAP_NEWSIZE="2G"

# --- Cause B: Large partitions being read ---
nodetool tablehistograms keyspace.table
# Look for large partition sizes — redesign data model

# --- Cause C: Memtable too large before flush ---
# Reduce memtable threshold
nodetool flush keyspace tablename  # manual flush

# In cassandra.yaml:
memtable_cleanup_threshold: 0.20   # flush earlier

# --- Cause D: Compaction backlog ---
nodetool compactionstats
# If backlog growing, increase compaction throughput:
nodetool setcompactionthroughput 256  # MB/s (default 64)

# --- Cause E: Use G1GC instead of CMS ---
# cassandra-env.sh
JVM_OPTS="$JVM_OPTS -XX:+UseG1GC"
JVM_OPTS="$JVM_OPTS -XX:G1RSetUpdatingPauseTimePercent=5"
JVM_OPTS="$JVM_OPTS -XX:MaxGCPauseMillis=500"
```

---

**Q44. ISSUE: A Flyway migration failed mid-way in production. The table is partially created. How do you recover?**

**Answer:**

```bash
# Step 1: Check migration status
flyway info -url=jdbc:postgresql://prod-db:5432/mydb -user=app -password=pass

# Output shows: V23 — FAILED

# Step 2: Check what was applied
SELECT * FROM flyway_schema_history WHERE success = false;

# Step 3: Manually fix the database state
# Option A: Complete the failed migration manually
psql -h prod-db -U app -d mydb
-- Run remaining SQL from V23__Add_payment_table.sql manually
-- Fix any partial state (DROP TABLE IF EXISTS, then recreate)

# Step 4: Mark as repaired in Flyway
flyway repair -url=... -user=... -password=...
# This removes the failed entry from flyway_schema_history

# Step 5: Re-run migration
flyway migrate

# PREVENTION: Always wrap migrations in transactions
-- V23__Add_payment_table.sql
BEGIN;
CREATE TABLE payments (
  id BIGSERIAL PRIMARY KEY,
  order_id BIGINT NOT NULL,
  amount DECIMAL(10,2),
  created_at TIMESTAMP DEFAULT NOW()
);
CREATE INDEX idx_payments_order_id ON payments(order_id);
COMMIT;
-- If any step fails, entire migration rolls back
-- (PostgreSQL supports transactional DDL — MySQL does not)
```

---

**Q45. ISSUE: Terraform apply is failing with "Error: RDS instance already exists." But Terraform state shows no instance. How do you fix it?**

**Answer:**

```bash
# Problem: Resource exists in AWS but not in Terraform state (state drift)

# Step 1: Import the existing resource into Terraform state
terraform import aws_db_instance.postgres prod-postgres-db
# Terraform fetches current state from AWS and adds to .tfstate

# Step 2: Run terraform plan to see drift
terraform plan
# Shows any differences between imported state and your .tf config

# Step 3: Reconcile config to match actual resource
# Or update resource as desired and apply

# Step 4: Prevent future drift
# a) Never create/modify RDS manually — always use Terraform
# b) Enable AWS Config rules for compliance
# c) Use terraform state list regularly to audit

# State refresh
terraform refresh

# If import is not possible (unmanaged legacy resource):
# Use data source to reference it without managing
data "aws_db_instance" "legacy_postgres" {
  db_instance_identifier = "prod-postgres-db"
}
```

---

## BONUS: Advanced Performance Tuning Checklist

---

**Q46. Give a complete PostgreSQL performance tuning checklist for a senior DBA interview.**

**Answer:**

```
LAYER 1 — HARDWARE & OS
□ Use SSD/NVMe storage (random_page_cost = 1.1)
□ Set vm.swappiness = 10 (avoid swapping)
□ Set vm.dirty_ratio = 10, vm.dirty_background_ratio = 5
□ Use NUMA-aware memory allocation
□ Set transparent hugepages = madvise or off
□ Use deadline or noop I/O scheduler for SSDs

LAYER 2 — POSTGRESQL CONFIG
□ shared_buffers = 25% RAM
□ effective_cache_size = 75% RAM
□ work_mem = 64MB (tune per workload)
□ maintenance_work_mem = 1GB
□ checkpoint_completion_target = 0.9
□ max_wal_size = 4GB
□ wal_buffers = 64MB
□ effective_io_concurrency = 200 (SSD)
□ random_page_cost = 1.1 (SSD)
□ default_statistics_target = 200

LAYER 3 — INDEXING
□ EXPLAIN ANALYZE every slow query
□ Add missing indexes (Seq Scan on large tables)
□ Use partial indexes for filtered queries
□ Use covering indexes to avoid heap fetch
□ REINDEX CONCURRENTLY bloated indexes
□ Remove unused indexes (sys.schema_unused_indexes)

LAYER 4 — VACUUM & BLOAT
□ Monitor n_dead_tup via pg_stat_user_tables
□ Tune autovacuum per high-churn tables
□ Use pg_repack for online table rebuilds
□ Monitor transaction ID wraparound (age(datfrozenxid))

LAYER 5 — CONNECTIONS
□ Use PgBouncer in transaction mode
□ max_connections = 200 (PgBouncer handles scale)
□ Set idle connection timeouts

LAYER 6 — MONITORING
□ pg_stat_statements (top queries)
□ pg_stat_bgwriter (checkpoint/buffer metrics)
□ pg_stat_replication (lag monitoring)
□ auto_explain for slow query plans
□ pgBadger for log analysis
```

---

*Document prepared for: Y. Venkat Sarath — Sr DBA & Multi Cloud DBA*
*Experience: 15 Years | Coverage: PostgreSQL, MySQL, MariaDB, Cassandra, Aurora, MSSQL, Kubernetes, Terraform, CI/CD, Flyway, Liquibase*
