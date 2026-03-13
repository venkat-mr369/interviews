# Oracle DBA Deep Interview Questions & Answers
## Covering: Oracle Versions | RMAN | Data Pump (expdp/impdp) | Performance Tuning | RAC | Troubleshooting

---

# SECTION 1: ORACLE CORE & VERSIONS

## Oracle Architecture & Version History

### Q1. What are the key architectural differences between Oracle 11g, 12c, 18c, 19c, 21c, and 23c?

**Answer:**

| Version | Key Features |
|---------|-------------|
| **Oracle 11g** | Result Cache, Invisible Indexes, Advanced Compression, SecureFiles LOBs, Automatic Memory Management (AMM), Online Patching |
| **Oracle 12c R1** | Multitenant Architecture (CDB/PDB), In-Memory Column Store, Adaptive Query Optimization, Identity Columns, Row Limiting Clause |
| **Oracle 12c R2** | Sharding, Active Data Guard DML Redirection, JSON native enhancements, Online Table Move, Approximate Query Processing |
| **Oracle 18c** | Polymorphic Table Functions, Private Temporary Tables, Active Directory integration (first annual release) |
| **Oracle 19c** | Real-Time Statistics, Automatic Indexing, SQL Quarantine, Hybrid Partitioned Tables, long-term support release (LTS) |
| **Oracle 21c** | Native Blockchain Tables, Automatic Zone Maps, JSON data type (native binary), In-Memory Hybrid Columnar Compression |
| **Oracle 23c** | True Cache, SQL Domains, Table Value Constructors, BOOLEAN data type, developer-free edition, schema-level privileges |

---

### Q2. Explain Oracle's Multitenant Architecture (CDB/PDB) introduced in 12c. How does it differ from a traditional non-CDB?

**Answer:**

**Traditional Non-CDB:**
- Single database with one set of data files, control files, redo logs
- System-wide users and roles
- One SYSTEM/SYSAUX tablespace per database
- Patching requires downtime per database

**CDB (Container Database):**
- Root container (CDB$ROOT): Stores Oracle metadata, common users
- Seed PDB (PDB$SEED): Template for new PDBs (read-only)
- PDBs: Each is an isolated, portable database with its own objects, users, tablespaces

```sql
-- Create a PDB from seed
CREATE PLUGGABLE DATABASE mypdb
  ADMIN USER pdbadmin IDENTIFIED BY password
  FILE_NAME_CONVERT = ('/pdbseed/','/mypdb/');

-- Open a PDB
ALTER PLUGGABLE DATABASE mypdb OPEN;

-- Connect to PDB
ALTER SESSION SET CONTAINER = mypdb;

-- Check current container
SELECT SYS_CONTEXT('USERENV','CON_NAME') FROM DUAL;
```

**Key differences:**
- Common users (C## prefix) exist in all containers
- Local users exist only in a specific PDB
- Each PDB has an independent data dictionary
- PDBs can be plugged/unplugged, cloned, refreshed
- A single patch upgrades the entire CDB and all PDBs

---

### Q3. What is Oracle Automatic Indexing (introduced in 19c)? How does it work?

**Answer:**

Automatic Indexing uses machine learning to automatically create, rebuild, or drop indexes based on workload patterns.

**How it works:**
1. Oracle captures SQL statements and analyzes predicates, joins, and access paths
2. Candidate indexes are identified and created as INVISIBLE indexes
3. Performance is verified — if beneficial, index becomes VISIBLE
4. Indexes unused over time are dropped automatically

```sql
-- Enable Automatic Indexing
EXEC DBMS_AUTO_INDEX.CONFIGURE('AUTO_INDEX_MODE','IMPLEMENT');

-- Check configuration
SELECT PARAMETER_NAME, PARAMETER_VALUE 
FROM DBA_AUTO_INDEX_CONFIG;

-- View auto-created indexes
SELECT INDEX_NAME, TABLE_NAME, AUTO, VISIBILITY, STATUS
FROM DBA_INDEXES
WHERE AUTO = 'YES';

-- View activity report
SELECT DBMS_AUTO_INDEX.REPORT_ACTIVITY(
  ACTIVITY_START => SYSDATE - 7,
  ACTIVITY_END   => SYSDATE,
  TYPE           => 'ALL',
  SECTION        => 'ALL'
) FROM DUAL;
```

---

### Q4. Explain Oracle's In-Memory Column Store. What problems does it solve and what are its limitations?

**Answer:**

**Purpose:** Dual-format storage — rows stored on disk (row format) for OLTP, and simultaneously in SGA memory (column format) for analytics. Eliminates the need for separate data warehouses for many workloads.

**How it works:**
- In-Memory Compression Units (IMCUs) store column data vertically
- Dictionary-based compression achieves 2-20x compression
- SIMD (Single Instruction Multiple Data) vector processing for fast scans
- Worker processes (Wnnn) populate IM column store

```sql
-- Enable In-Memory
ALTER SYSTEM SET INMEMORY_SIZE = 10G SCOPE=SPFILE;

-- Populate a table into IM column store
ALTER TABLE sales INMEMORY PRIORITY CRITICAL;

-- Check population status
SELECT SEGMENT_NAME, POPULATE_STATUS, BYTES, BYTES_NOT_POPULATED
FROM V$IM_SEGMENTS;

-- Check which columns are compressed
SELECT TABLE_NAME, COLUMN_NAME, INMEMORY_COMPRESSION
FROM V$IM_COLUMN_LEVEL;

-- Force immediate population
EXEC DBMS_INMEMORY.POPULATE('HR','EMPLOYEES');
```

**Limitations:**
- Requires Enterprise Edition + In-Memory option license
- Memory-bound (fits only what SGA can hold)
- DML operations create journal entries, slightly slower writes
- Not beneficial for OLTP point-lookup queries

---

### Q5. What are Oracle 23c's key new features — SQL Domains, Boolean data type, and True Cache?

**Answer:**

**SQL Domains (23c):**
```sql
-- Create a domain with constraints
CREATE DOMAIN email_domain AS VARCHAR2(100)
  CONSTRAINT email_chk CHECK (VALUE LIKE '%@%.%')
  DISPLAY REGEXP_SUBSTR(VALUE, '[^@]+', 1, 1);

-- Apply domain to column
CREATE TABLE employees (
  emp_id NUMBER,
  email  email_domain
);

-- Query domain metadata
SELECT * FROM USER_DOMAINS;
```

**Boolean Data Type (23c):**
```sql
-- Native boolean column (was emulated with CHAR/NUMBER before)
CREATE TABLE feature_flags (
  flag_name  VARCHAR2(50),
  is_enabled BOOLEAN DEFAULT TRUE
);

INSERT INTO feature_flags VALUES ('dark_mode', TRUE);
INSERT INTO feature_flags VALUES ('beta_ui', FALSE);

SELECT flag_name FROM feature_flags WHERE is_enabled;
```

**True Cache (23c):**
- Read-only in-memory cache that sits between application and database
- Unlike Result Cache (query level), True Cache caches full table blocks
- Scales read-heavy workloads horizontally without full RAC licenses
- Synchronizes automatically with primary database

---

### Q6. Explain Oracle's Undo Management. What is the difference between Rollback Segments and AUM?

**Answer:**

**Manual Undo (Rollback Segments — pre-9i):**
- DBAs manually created, sized, and assigned rollback segments
- Prone to "Snapshot Too Old" (ORA-01555) and "unable to extend rollback segment"
- Required expertise to estimate optimal sizes

**Automatic Undo Management (AUM — 9i+):**
```sql
-- Enable AUM (should always be enabled)
ALTER SYSTEM SET UNDO_MANAGEMENT = AUTO SCOPE=SPFILE;
ALTER SYSTEM SET UNDO_TABLESPACE = UNDOTBS1;
ALTER SYSTEM SET UNDO_RETENTION = 900; -- seconds (15 min guarantee)

-- Check undo usage
SELECT TABLESPACE_NAME, STATUS, SUM(BYTES)/1024/1024 MB
FROM DBA_UNDO_EXTENTS
GROUP BY TABLESPACE_NAME, STATUS;

-- Diagnose ORA-01555
SELECT MAX_QUERYLEN, MAXQUERYID, NOSPACEERRCNT, SSOLDERRCNT
FROM V$UNDOSTAT
ORDER BY BEGIN_TIME DESC;

-- Calculate required undo size
SELECT (UR * (UPS * DBS)) + DBS AS "Recommended Undo (bytes)"
FROM (
  SELECT MAX(UNDOBLKS/((END_TIME - BEGIN_TIME)*86400)) UPS,
         MAX(TUNED_UNDORETENTION) UR,
         8192 DBS -- block size
  FROM V$UNDOSTAT
);
```

**Tuning for ORA-01555:**
```sql
-- Option 1: Increase retention
ALTER SYSTEM SET UNDO_RETENTION = 3600; -- 1 hour

-- Option 2: Enable retention guarantee
ALTER TABLESPACE UNDOTBS1 RETENTION GUARANTEE;
-- WARNING: This can cause DML to fail instead of ORA-01555
```

---

### Q7. What is the Oracle SGA and PGA? How do you tune memory in 11g+ with AMM/ASMM?

**Answer:**

**SGA Components:**
- Buffer Cache: Cached data blocks (most critical)
- Shared Pool: Library cache (parsed SQL), Data Dictionary cache
- Large Pool: RMAN, parallel query, shared server
- Java Pool: Java stored procedures
- Streams Pool: GoldenGate, Streams replication
- Redo Log Buffer: Pre-write redo entries
- In-Memory Area (12c+): Column store

**PGA:** Per-process memory for sort areas, hash joins, bitmap operations

```sql
-- ASMM (Automatic Shared Memory Management) — 10g+
ALTER SYSTEM SET SGA_TARGET = 8G SCOPE=BOTH;      -- Oracle auto-sizes SGA components
ALTER SYSTEM SET PGA_AGGREGATE_TARGET = 2G SCOPE=BOTH;

-- AMM (Automatic Memory Management) — 11g+
ALTER SYSTEM SET MEMORY_TARGET = 10G SCOPE=SPFILE; -- Oracle manages SGA+PGA combined
ALTER SYSTEM SET MEMORY_MAX_TARGET = 12G SCOPE=SPFILE;
-- Note: AMM requires ASMM to be disabled (SGA_TARGET=0)

-- Check current SGA usage
SELECT COMPONENT, CURRENT_SIZE/1024/1024 MB, MIN_SIZE/1024/1024, MAX_SIZE/1024/1024
FROM V$SGA_DYNAMIC_COMPONENTS;

-- Check PGA usage
SELECT NAME, VALUE/1024/1024 MB
FROM V$PGASTAT
WHERE NAME IN ('total PGA allocated','maximum PGA allocated','total freeable PGA memory');

-- AMM advisory
SELECT MEMORY_SIZE/1024/1024 MB, ESTD_DB_TIME, ESTD_DB_TIME_FACTOR
FROM V$MEMORY_TARGET_ADVICE
ORDER BY MEMORY_SIZE;
```

---

### Q8. Explain Oracle's Redo Log Architecture — online redo logs, archived logs, log switching, checkpoints.

**Answer:**

**Redo Log Flow:**
1. Server process writes change vectors to Redo Log Buffer (SGA)
2. LGWR writes to online redo log group when: buffer 1/3 full, every 3 seconds, before commit, before DBWn writes
3. Log Switch occurs when current group fills → ARC process archives the full group
4. Checkpoint triggers DBWn to flush dirty blocks → advances checkpoint SCN

```sql
-- Check online redo log groups
SELECT GROUP#, MEMBERS, BYTES/1024/1024 MB, STATUS, ARCHIVED
FROM V$LOG;

-- View log member files
SELECT GROUP#, MEMBER, STATUS FROM V$LOGFILE ORDER BY GROUP#;

-- Add a redo log group
ALTER DATABASE ADD LOGFILE GROUP 4 
  ('/u01/oradata/redo04a.log', '/u02/oradata/redo04b.log') SIZE 500M;

-- Force a log switch
ALTER SYSTEM SWITCH LOGFILE;

-- Check archive log mode
SELECT LOG_MODE FROM V$DATABASE;

-- Enable archive log mode
SHUTDOWN IMMEDIATE;
STARTUP MOUNT;
ALTER DATABASE ARCHIVELOG;
ALTER DATABASE OPEN;

-- Check checkpoint performance
SELECT CHECKPOINT_CHANGE#, CHECKPOINT_TIME, 
       RESETLOGS_CHANGE#, CURRENT_SCN
FROM V$DATABASE;

-- Log switch frequency (should be every 15-30 min for OLTP)
SELECT TO_CHAR(FIRST_TIME,'YYYY-MM-DD HH24') HOUR, COUNT(*) SWITCHES
FROM V$LOG_HISTORY
GROUP BY TO_CHAR(FIRST_TIME,'YYYY-MM-DD HH24')
ORDER BY 1 DESC;
```

**Sizing redo logs:**
- Small logs → frequent log switches → I/O spikes, "checkpoint not complete"
- Large logs → longer MTTR (crash recovery time)
- Rule of thumb: 3-5 log switches per hour for OLTP

---

### Q9. What is an Oracle wait event? Name the top 10 most critical wait events and how to diagnose them.

**Answer:**

**Top Critical Wait Events:**

| Wait Event | Cause | Diagnosis |
|-----------|-------|-----------|
| `db file sequential read` | Single block reads (index scans) | High → check for missing indexes |
| `db file scattered read` | Multi-block reads (FTS) | Consider adding indexes or partitioning |
| `log file sync` | Commit waits for LGWR | Slow disk, too many commits, batch commits |
| `log file parallel write` | LGWR writing to disk | Disk latency on redo volume |
| `buffer busy waits` | Block contention | Hot blocks, sequences, freelists |
| `latch: cache buffers chains` | SGA buffer cache contention | Hot blocks being read by many sessions |
| `enq: TX - row lock contention` | Row-level locking | Long uncommitted transactions |
| `enq: HW - contention` | High-water mark contention | INSERT spikes on non-partitioned tables |
| `library cache lock/pin` | Hard parsing | Enable cursor sharing, bind variables |
| `gc buffer busy acquire` (RAC) | Remote block transfer | GC tuning, interconnect issues |

```sql
-- Top wait events right now (active sessions)
SELECT EVENT, COUNT(*) SESSIONS, SUM(SECONDS_IN_WAIT) TOTAL_WAIT
FROM V$SESSION_WAIT
WHERE WAIT_CLASS != 'Idle'
GROUP BY EVENT
ORDER BY SESSIONS DESC;

-- AWR Top Wait Events (last hour)
SELECT EVENT, TOTAL_WAITS, TIME_WAITED_MICRO/1000000 SEC_WAITED,
       ROUND(TIME_WAITED_MICRO*100/SUM(TIME_WAITED_MICRO) OVER(),2) PCT
FROM DBA_HIST_SYSTEM_EVENT
WHERE DBID = (SELECT DBID FROM V$DATABASE)
  AND SNAP_ID > (SELECT MAX(SNAP_ID)-12 FROM DBA_HIST_SNAPSHOT)
  AND WAIT_CLASS != 'Idle'
ORDER BY TIME_WAITED_MICRO DESC;

-- Find blocking sessions
SELECT BLOCKING_SESSION, SID, SERIAL#, WAIT_CLASS, EVENT, SECONDS_IN_WAIT
FROM V$SESSION
WHERE BLOCKING_SESSION IS NOT NULL;
```

---

### Q10. What are the differences between TRUNCATE, DELETE, and DROP? Explain with undo/redo implications.

**Answer:**

| Operation | Undo Generated | Redo Generated | Recoverable | Rollable Back | Triggers | Locks |
|-----------|---------------|---------------|------------|--------------|---------|-------|
| **DELETE** | Yes (row data) | Yes | Yes | Yes | Yes | Row + Table Share |
| **TRUNCATE** | Minimal (space deallocate) | Minimal | No (DDL) | No | No | Table Exclusive |
| **DROP** | Minimal | Minimal | Via Flashback/Recycle Bin | No | No | Table Exclusive |

**Performance:**
- DELETE 1M rows = 1M undo entries, 1M redo entries → very slow
- TRUNCATE 1M rows = few undo/redo entries → instantaneous
- TRUNCATE resets HWM (High Water Mark) → subsequent FTS is faster

```sql
-- Check if table goes to recycle bin after DROP
SHOW PARAMETER recyclebin;

-- View recycle bin
SELECT OBJECT_NAME, ORIGINAL_NAME, OPERATION, DROPTIME
FROM USER_RECYCLEBIN;

-- Recover from recycle bin (12c and earlier)
FLASHBACK TABLE employees TO BEFORE DROP;

-- Permanently drop (bypass recycle bin)
DROP TABLE employees PURGE;

-- TRUNCATE does NOT free extents by default in 11g+ (keeps HWM)
-- To reset HWM:
TRUNCATE TABLE employees DROP STORAGE; -- releases extents
TRUNCATE TABLE employees REUSE STORAGE; -- keeps extents allocated
```

---

# SECTION 2: RMAN — IN-DEPTH

### Q11. Explain RMAN architecture. What are the key components and processes?

**Answer:**

**RMAN Components:**
- **RMAN client**: Command-line interface that generates and interprets backup metadata
- **Target database**: Database being backed up
- **Recovery Catalog**: Optional repository database storing RMAN metadata (recommended for production)
- **Auxiliary database**: Used for DUPLICATE, TSPITR operations
- **Media Management Layer (MML)**: Interface to tape libraries (SBT)
- **Channels**: Server processes in the target database that perform I/O

**Key background processes during RMAN:**
- Server process reads/writes data blocks
- RD (Recovery Data) — reads blocks from datafiles
- WR (Writer) — writes backup sets or image copies
- No separate RMAN daemon — operates as Oracle server processes

```bash
# Connect to RMAN
rman TARGET / CATALOG rman_user/password@catdb

# Connect with dedicated channel
rman TARGET sys/password@proddb CATALOG rcat/rcat@rmancat

# Verify connectivity
RMAN> SELECT INSTANCE_NAME FROM V$INSTANCE;
```

```sql
-- Register database with catalog
RMAN> REGISTER DATABASE;

-- Resync catalog (sync RMAN catalog with controlfile)
RMAN> RESYNC CATALOG;

-- Cross-check all backups
RMAN> CROSSCHECK BACKUP;
RMAN> CROSSCHECK ARCHIVELOG ALL;
```

---

### Q12. Explain the difference between RMAN Backup Sets and Image Copies. When would you use each?

**Answer:**

**Backup Sets:**
- RMAN proprietary format (.bkp files)
- Excludes never-used blocks (sparse backups)
- Supports compression, encryption
- Requires RESTORE before use
- Can span multiple pieces
- Default format

**Image Copies:**
- Exact copy of datafiles (like cp but block-validated)
- Can be used immediately without RESTORE
- Used as "fast recover" strategy with SWITCH DATAFILE TO COPY
- Same size as original file (no block filtering)

```sql
-- Backup Set (default)
RMAN> BACKUP DATABASE PLUS ARCHIVELOG DELETE INPUT;

-- Image Copy
RMAN> BACKUP AS COPY DATABASE;
RMAN> BACKUP AS COPY DATAFILE 1 FORMAT '/backup/system.dbf';

-- Fast Recovery with Image Copy (zero restore time)
RMAN> SWITCH DATABASE TO COPY; -- change control file pointers
-- Then: RECOVER DATABASE; -- apply incremental + logs

-- Compressed Backup Set (requires Advanced Compression)
RMAN> BACKUP AS COMPRESSED BACKUPSET DATABASE;

-- Encrypted Backup
RMAN> SET ENCRYPTION ON IDENTIFIED BY "MyPass123" ONLY;
RMAN> BACKUP DATABASE;

-- List backup sets vs image copies
RMAN> LIST BACKUP;
RMAN> LIST COPY;
```

---

### Q13. What is Incremental Merge Backup strategy in RMAN? How does it achieve near-zero RTO?

**Answer:**

**Strategy: Rolling Incremental Merge**
1. Take Level 0 (full) image copy on Day 1
2. Each subsequent day: Take Level 1 incremental, THEN merge it into the Level 0 copy
3. Result: Always have an up-to-date image copy = current datafiles minus 1 day

**Benefits:** Restore = SWITCH + RECOVER (minutes, not hours). No need to restore from scratch.

```sql
-- Day 1: Level 0 image copy
RMAN> BACKUP AS COPY INCREMENTAL LEVEL 0 DATABASE 
      TAG 'ORA_LEVEL_0_COPY';

-- Day 2 onwards: Level 1 + Merge
RMAN> RECOVER COPY OF DATABASE WITH TAG 'ORA_LEVEL_0_COPY'
      UNTIL TIME 'SYSDATE';

-- This does two things atomically:
-- 1. Applies yesterday's incremental to the image copy
-- 2. Takes today's incremental backup

-- Recovery from this strategy (near-zero RTO):
RMAN> SWITCH DATABASE TO COPY; -- Instantly point control file to image copy
RMAN> RECOVER DATABASE;         -- Apply incremental + archived logs (minutes)
RMAN> ALTER DATABASE OPEN RESETLOGS;
```

---

### Q14. How do you configure RMAN Channels? Explain SBT (tape) vs DISK channels and parallelism.

**Answer:**

```sql
-- Automatic channel configuration (persistent)
RMAN> CONFIGURE CHANNEL DEVICE TYPE DISK 
      FORMAT '/backup/%U' 
      MAXPIECESIZE 10G;

RMAN> CONFIGURE DEFAULT DEVICE TYPE TO DISK;
RMAN> CONFIGURE PARALLELISM 4; -- 4 channels = 4 parallel streams

-- Allocate channels manually (overrides config for session)
RMAN> RUN {
  ALLOCATE CHANNEL d1 DEVICE TYPE DISK FORMAT '/backup/%U';
  ALLOCATE CHANNEL d2 DEVICE TYPE DISK FORMAT '/backup2/%U';
  ALLOCATE CHANNEL d3 DEVICE TYPE DISK FORMAT '/backup3/%U';
  BACKUP DATABASE;
}

-- SBT (tape) channel via MML
RMAN> RUN {
  ALLOCATE CHANNEL t1 DEVICE TYPE SBT
    PARMS 'SBT_LIBRARY=/opt/oracle/backup/lib/libobk.so,
           ENV=(NSR_SERVER=backupserver,NSR_CLIENT=dbserver)';
  BACKUP DATABASE;
}

-- Check channel configuration
RMAN> SHOW ALL;

-- Channel naming variables in FORMAT:
-- %U = unique filename (preferred)
-- %d = DB name, %t = timestamp, %s = backup set #, %p = piece #
-- %T = YYYYMMDD date
```

**Parallelism guidelines:**
- Match channels to number of disk groups/spindles
- For tape: match to physical drives available
- Too many channels → I/O contention, not more throughput

---

### Q15. Explain RMAN's Block Change Tracking (BCT). How does it improve incremental backups?

**Answer:**

**Problem without BCT:** RMAN must read all blocks to find changed ones → incremental backup = full scan time.

**BCT Solution:** A background process (CTWR) maintains a bitmap file tracking which blocks changed since last backup → RMAN only reads changed blocks.

```sql
-- Enable BCT (requires Enterprise Edition)
ALTER DATABASE ENABLE BLOCK CHANGE TRACKING 
  USING FILE '/u01/app/oracle/bct/bct.dbf';

-- Check BCT status
SELECT STATUS, FILENAME, BYTES/1024/1024 MB
FROM V$BLOCK_CHANGE_TRACKING;

-- Verify BCT is being used (after running incremental)
SELECT FILE#, BLOCKS_READ, BLOCKS, 
       ROUND(BLOCKS_READ/BLOCKS*100,1) PCT_READ
FROM V$BACKUP_DATAFILE
WHERE INCREMENTAL_LEVEL > 0
ORDER BY FILE#;
-- Low PCT_READ (e.g., 2%) = BCT working. 100% = BCT not used.

-- Disable BCT
ALTER DATABASE DISABLE BLOCK CHANGE TRACKING;
```

**BCT file sizing:**
- Oracle recommends 1/30,000 of database size
- Automatically extends as needed
- Keep on fast disk (lots of writes)

---

### Q16. Walk through a complete Point-In-Time Recovery (PITR) scenario using RMAN.

**Answer:**

**Scenario:** User dropped a critical table at 14:35 on March 10. Need to recover the database to 14:34.

```bash
# Step 1: Determine the SCN or time for recovery
sqlplus / as sysdba
SQL> SELECT TIMESTAMP_TO_SCN(TO_TIMESTAMP('2024-03-10 14:34:00','YYYY-MM-DD HH24:MI:SS')) FROM DUAL;
-- Returns SCN e.g., 5827364
```

```sql
-- Step 2: Shutdown and mount database
RMAN> SHUTDOWN IMMEDIATE;
RMAN> STARTUP MOUNT;

-- Step 3: Restore to specific time (PITR)
RMAN> RUN {
  SET UNTIL TIME "TO_DATE('2024-03-10 14:34:00','YYYY-MM-DD HH24:MI:SS')";
  -- OR: SET UNTIL SCN 5827364;
  -- OR: SET UNTIL SEQUENCE 1234 THREAD 1;
  RESTORE DATABASE;
  RECOVER DATABASE;
}

-- Step 4: Open with RESETLOGS (mandatory after incomplete recovery)
RMAN> ALTER DATABASE OPEN RESETLOGS;
```

**After recovery — protect yourself:**
```sql
-- Immediately take a full backup after RESETLOGS
RMAN> BACKUP DATABASE PLUS ARCHIVELOG;

-- Verify database open
SQL> SELECT NAME, OPEN_MODE, RESETLOGS_TIME FROM V$DATABASE;
```

**Alternative: Tablespace PITR (TSPITR) — recover one tablespace without full DB restore:**
```sql
RMAN> RECOVER TABLESPACE hr_data 
      UNTIL TIME "TO_DATE('2024-03-10 14:34:00','YYYY-MM-DD HH24:MI:SS')"
      AUXILIARY DESTINATION '/tmp/tspitr_aux';
```

---

### Q17. What is RMAN Duplicate Database? Explain ACTIVE vs BACKUP-BASED duplication.

**Answer:**

**Use cases:** Create test/dev environment, set up standby database, cloning for refresh.

**Backup-Based Duplication:**
```sql
-- On target: backup to shared/NFS location
RMAN> BACKUP DATABASE FORMAT '/nfs/backup/%U';
RMAN> BACKUP ARCHIVELOG ALL FORMAT '/nfs/backup/%U';

-- On auxiliary (new host): connect to both target and auxiliary
rman TARGET sys/pass@proddb AUXILIARY sys/pass@devdb

RMAN> DUPLICATE TARGET DATABASE TO devdb
      LOGFILE 
        GROUP 1 ('/dev/redo/redo01.log') SIZE 200M,
        GROUP 2 ('/dev/redo/redo02.log') SIZE 200M
      NOFILENAMECHECK;
```

**Active Duplication (12c+, no backup needed):**
```sql
-- Direct network copy from running target
RMAN> DUPLICATE TARGET DATABASE TO devdb 
      FROM ACTIVE DATABASE
      USING COMPRESSED BACKUPSET   -- compress during transfer
      SECTION SIZE 10G              -- parallelize large files
      SPFILE
        PARAMETER_VALUE_CONVERT 'prod_path','dev_path'
        SET DB_FILE_NAME_CONVERT 'prod_path','dev_path'
        SET LOG_FILE_NAME_CONVERT 'prod_path','dev_path'
        SET DB_NAME='devdb';
```

**Active Duplication options (19c+):**
```sql
-- Pull-based: auxiliary pulls from target (firewall friendly)
RMAN> DUPLICATE TARGET DATABASE TO devdb
      FROM ACTIVE DATABASE
      USING BACKUPSET
      NOOPEN
      SKIP TABLESPACE temp;
```

---

### Q18. How do you perform RMAN Catalog maintenance? Explain catalog reports and crosscheck operations.

**Answer:**

```sql
-- View all backups
RMAN> LIST BACKUP SUMMARY;
RMAN> LIST BACKUP OF DATABASE;
RMAN> LIST BACKUP OF ARCHIVELOG ALL;
RMAN> LIST COPY OF DATABASE;

-- List expired backups (files that no longer exist on disk/tape)
RMAN> LIST EXPIRED BACKUP;

-- Report database needing backup
RMAN> REPORT NEED BACKUP;
RMAN> REPORT NEED BACKUP DAYS 2; -- backed up more than 2 days ago

-- Report obsolete backups (based on retention policy)
RMAN> REPORT OBSOLETE;

-- Configure retention policy
RMAN> CONFIGURE RETENTION POLICY TO RECOVERY WINDOW OF 14 DAYS;
RMAN> CONFIGURE RETENTION POLICY TO REDUNDANCY 2; -- keep 2 backups

-- Crosscheck: verify backups exist where RMAN thinks they are
RMAN> CROSSCHECK BACKUP;
RMAN> CROSSCHECK ARCHIVELOG ALL;
RMAN> CROSSCHECK BACKUP OF DATAFILE 5;

-- Mark unavailable backups as EXPIRED
-- (After crosscheck, RMAN marks missing files as EXPIRED)

-- Delete expired/obsolete backups
RMAN> DELETE EXPIRED BACKUP;
RMAN> DELETE NOPROMPT OBSOLETE;
RMAN> DELETE NOPROMPT ARCHIVELOG ALL COMPLETED BEFORE 'SYSDATE-7';

-- Catalog additional backup pieces (found on disk manually)
RMAN> CATALOG BACKUPPIECE '/backup/extra/backup_01.bkp';
RMAN> CATALOG START WITH '/backup/extra/'; -- catalog entire directory
```

---

# SECTION 3: DATA PUMP (expdp/impdp)

### Q19. Explain Data Pump architecture. How does it differ from the original exp/imp?

**Answer:**

**Classic exp/imp (deprecated since 10g):**
- Client-side tool: data extracted across network to client, then pushed back
- No parallelism
- No estimate capability
- No excludes/includes, no remapping
- Data written to client filesystem

**Data Pump (expdp/impdp):**
- Server-side: All I/O happens on server filesystem (via DIRECTORY objects)
- Parallel workers (DBMS_DATAPUMP API)
- Master Control Process (MCP) + Worker Processes (WRK)
- Shadow process for API communication
- Supports: exclude/include, remap, transform, access_parameters, compression, encryption
- Dump files on server, not client

```sql
-- Create directory object (required)
CREATE DIRECTORY dp_dir AS '/u01/datapump';
GRANT READ, WRITE ON DIRECTORY dp_dir TO hr;

-- Check existing directories
SELECT DIRECTORY_NAME, DIRECTORY_PATH FROM DBA_DIRECTORIES;
```

```bash
# Full database export
expdp system/password FULL=YES DIRECTORY=dp_dir DUMPFILE=full_%U.dmp LOGFILE=full_exp.log PARALLEL=4

# Schema export
expdp hr/password SCHEMAS=HR DIRECTORY=dp_dir DUMPFILE=hr_%U.dmp LOGFILE=hr_exp.log

# Table export
expdp hr/password TABLES=HR.EMPLOYEES,HR.DEPARTMENTS DIRECTORY=dp_dir DUMPFILE=tables.dmp

# Estimate only (no actual export)
expdp hr/password SCHEMAS=HR ESTIMATE_ONLY=YES ESTIMATE=STATISTICS

# Full import
impdp system/password FULL=YES DIRECTORY=dp_dir DUMPFILE=full_%U.dmp LOGFILE=full_imp.log PARALLEL=4
```

---

### Q20. Explain Data Pump's INCLUDE, EXCLUDE, and REMAP parameters with examples.

**Answer:**

```bash
# EXCLUDE: Skip specific object types
expdp system/password SCHEMAS=HR DIRECTORY=dp_dir \
  DUMPFILE=hr_noconstraints.dmp \
  EXCLUDE=CONSTRAINT \
  EXCLUDE=INDEX:"LIKE 'SYS_%'"  \
  EXCLUDE=STATISTICS

# INCLUDE: Only export specific objects
expdp system/password SCHEMAS=HR DIRECTORY=dp_dir \
  DUMPFILE=hr_tables_only.dmp \
  INCLUDE=TABLE:"IN ('EMPLOYEES','DEPARTMENTS','JOBS')"

# REMAP_SCHEMA: Import to different schema
impdp system/password DIRECTORY=dp_dir DUMPFILE=hr.dmp \
  REMAP_SCHEMA=HR:HR_DEV \
  LOGFILE=imp.log

# REMAP_TABLESPACE: Move objects to different tablespace
impdp system/password DIRECTORY=dp_dir DUMPFILE=hr.dmp \
  REMAP_TABLESPACE=HR_DATA:HR_DEV_DATA \
  REMAP_TABLESPACE=HR_IDX:HR_DEV_IDX

# REMAP_DATAFILES: Change datafile paths
impdp system/password DIRECTORY=dp_dir DUMPFILE=full.dmp \
  REMAP_DATAFILE='/prod/oradata/hr_data.dbf':'/dev/oradata/hr_data.dbf'

# TRANSFORM: Change storage attributes
impdp system/password DIRECTORY=dp_dir DUMPFILE=hr.dmp \
  TRANSFORM=SEGMENT_ATTRIBUTES:N   -- removes storage clauses
  TRANSFORM=OID:N                   -- new OID for object types

# TABLE_EXISTS_ACTION for imports
impdp system/password DIRECTORY=dp_dir DUMPFILE=hr.dmp \
  TABLE_EXISTS_ACTION=REPLACE   -- DROP and recreate (default=SKIP)
  -- Options: SKIP, APPEND, TRUNCATE, REPLACE
```

---

### Q21. How do you use Data Pump with network_link to export/import directly without a dump file?

**Answer:**

**Network Import:** Import directly from source database without intermediate dump file. Requires DB link from target to source.

```sql
-- Step 1: Create DB link on TARGET database
CREATE DATABASE LINK source_link
  CONNECT TO system IDENTIFIED BY password
  USING 'SOURCE_DB';

-- Step 2: Test the link
SELECT * FROM DUAL@source_link;
```

```bash
# Import using network link (no dump file created)
impdp system/password \
  NETWORK_LINK=source_link \
  SCHEMAS=HR \
  REMAP_SCHEMA=HR:HR_DEV \
  LOGFILE=network_imp.log \
  PARALLEL=4

# Export using network link (pull from remote, write local dump)
expdp system/password \
  NETWORK_LINK=source_link \
  SCHEMAS=HR \
  DIRECTORY=dp_dir \
  DUMPFILE=hr_from_network.dmp \
  LOGFILE=network_exp.log

# Useful for migrations, zero-copy refreshes
# Advantage: No disk space for dump files on source
# Disadvantage: Network bandwidth consumed during operation
```

---

### Q22. How do you monitor a running Data Pump job and attach to an existing job?

**Answer:**

```bash
# Method 1: Attach to a running job by name
impdp system/password ATTACH=SYS_IMPORT_SCHEMA_01

# Method 2: List running Data Pump jobs
```

```sql
-- View all Data Pump jobs
SELECT JOB_NAME, OPERATION, JOB_MODE, STATE, DEGREE, ATTACHED_SESSIONS
FROM DBA_DATAPUMP_JOBS
WHERE STATE != 'NOT RUNNING';

-- View detailed progress
SELECT SID, SERIAL#, SOFAR, TOTALWORK, 
       ROUND(SOFAR/TOTALWORK*100,1) PCT_DONE,
       MESSAGE
FROM V$SESSION_LONGOPS
WHERE OPNAME LIKE 'KUPW%'
  AND SOFAR < TOTALWORK;

-- Check master table (exists in schema running the job)
SELECT LOG_LINE FROM SYS.SYS_IMPORT_FULL_01 ORDER BY LOG_LINE;
```

```bash
# After attaching: interactive commands
Export> STATUS        -- show detailed status
Export> ADD_FILE=dp_dir:extra_%U.dmp  -- add dump file
Export> PARALLEL=8    -- increase parallelism
Export> STOP_JOB=IMMEDIATE  -- stop without cleanup
Export> KILL_JOB     -- stop and clean up master table
Export> EXIT_CLIENT  -- detach without stopping job (job continues)
```

---

### Q23. Explain Data Pump compression and encryption options.

**Answer:**

```bash
# COMPRESSION options (requires Advanced Compression)
expdp system/password SCHEMAS=HR DIRECTORY=dp_dir DUMPFILE=hr_comp.dmp \
  COMPRESSION=ALL         -- compress data + metadata
  # Options: ALL, DATA_ONLY, METADATA_ONLY, NONE

# ENCRYPTION options (requires Advanced Security)
expdp system/password SCHEMAS=HR DIRECTORY=dp_dir DUMPFILE=hr_enc.dmp \
  ENCRYPTION=ALL \
  ENCRYPTION_ALGORITHM=AES256 \
  ENCRYPTION_MODE=PASSWORD \
  ENCRYPTION_PASSWORD=MySecurePass!
  # ENCRYPTION_MODE: PASSWORD, TRANSPARENT (TDE), DUAL

# Transparent (TDE-based) encryption - uses Oracle Wallet
expdp system/password SCHEMAS=HR DIRECTORY=dp_dir DUMPFILE=hr_tde.dmp \
  ENCRYPTION=ALL \
  ENCRYPTION_MODE=TRANSPARENT

# Import encrypted dump
impdp system/password DIRECTORY=dp_dir DUMPFILE=hr_enc.dmp \
  ENCRYPTION_PASSWORD=MySecurePass! \
  LOGFILE=imp.log
```

---

# SECTION 4: PERFORMANCE TUNING (PT)

### Q24. Explain Oracle's SQL Execution Plan. How do you read an explain plan and what are key plan operations?

**Answer:**

```sql
-- Generate execution plan
EXPLAIN PLAN FOR
SELECT e.last_name, d.department_name, SUM(e.salary)
FROM employees e JOIN departments d ON e.department_id = d.department_id
GROUP BY e.last_name, d.department_name;

-- Read plan
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(FORMAT=>'ALL'));

-- Real execution plan (after actual execution)
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(FORMAT=>'ALLSTATS LAST'));

-- Plan from AWR
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_AWR('5z8f3n7qbkhd4'));
```

**Key plan operations:**

| Operation | Meaning | Good/Bad |
|-----------|---------|----------|
| `TABLE ACCESS FULL` | Full Table Scan | Bad for small selective queries, OK for large scans |
| `INDEX RANGE SCAN` | Read range of index entries | Usually good |
| `INDEX UNIQUE SCAN` | Single index entry | Best for PK/unique lookups |
| `INDEX FAST FULL SCAN` | Read all index blocks (multi-block) | Good for index-only queries |
| `NESTED LOOPS` | For each row in outer, probe inner | Good for small driving sets |
| `HASH JOIN` | Build hash table on smaller set, probe with larger | Good for large set joins |
| `SORT MERGE JOIN` | Sort both sets, merge | Good when both sides already sorted |
| `PARTITION RANGE ALL` | Scanning all partitions | Needs attention — can partition pruning occur? |
| `PARTITION RANGE SINGLE` | Accessing one partition | Excellent — pruning working |

**Reading a plan (bottom-up, left to right):**
```
--------------------------------------------------
| Id | Operation          | Name    | Rows | Cost |
--------------------------------------------------
|  0 | SELECT STATEMENT   |         |      | 1234 |
|  1 |  HASH JOIN         |         |  100 |  850 |
|  2 |   TABLE ACCESS FULL| EMP     | 1000 |  300 |  <-- executes first
|  3 |   TABLE ACCESS FULL| DEPT    |   27 |   3  |  <-- executes second
--------------------------------------------------
-- E-Rows vs A-Rows cardinality mismatch = poor statistics
```

---

### Q25. What is Adaptive Query Optimization (12c+)? Explain Adaptive Plans and Adaptive Statistics.

**Answer:**

**Adaptive Plans:** Oracle can change the execution plan mid-execution based on actual row counts seen at runtime.

```sql
-- Enable adaptive features
ALTER SYSTEM SET OPTIMIZER_ADAPTIVE_PLANS = TRUE;
ALTER SYSTEM SET OPTIMIZER_ADAPTIVE_STATISTICS = TRUE;

-- View if plan adapted
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(FORMAT=>'ADAPTIVE'));
-- Look for "- is not" in plan notes (lines marked "-" were NOT chosen)

-- Check if adaptive join method occurred
SELECT SQL_ID, IS_ADAPTIVE_PLAN, CHILD_NUMBER
FROM V$SQL_PLAN_STATISTICS_ALL
WHERE IS_ADAPTIVE_PLAN = 'YES';
```

**Adaptive Statistics Components:**
- **Dynamic Statistics (DS):** Oracle samples data at parse time for better cardinality estimates
- **SQL Plan Directives (SPD):** Oracle learns from past plan misestimates, creates directives
- **Automatic Reoptimization:** Query recompiles on 2nd execution using runtime cardinalities

```sql
-- View SQL Plan Directives
SELECT DIRECTIVE_ID, TYPE, REASON, STATE, LAST_USED
FROM DBA_SQL_PLAN_DIRECTIVES
ORDER BY LAST_USED DESC;

-- Flush directives to disk (persists across restarts)
EXEC DBMS_SPD.FLUSH_SQL_PLAN_DIRECTIVE;

-- Drop a problematic directive
EXEC DBMS_SPD.DROP_SQL_PLAN_DIRECTIVE(directive_id=>12345);
```

---

### Q26. Explain SQL Baselines (SPM). When and how do you use them?

**Answer:**

**SQL Plan Management (SPM)** — captures accepted execution plans to prevent plan regressions during optimizer version changes, statistics updates, or patches.

```sql
-- Enable automatic capture (11g+ can capture all repeated SQL)
ALTER SYSTEM SET OPTIMIZER_CAPTURE_SQL_PLAN_BASELINES = TRUE;  -- auto capture

-- Manual baseline creation from cursor cache
DECLARE
  l_count NUMBER;
BEGIN
  l_count := DBMS_SPM.LOAD_PLANS_FROM_CURSOR_CACHE(
    sql_id => '5z8f3n7qbkhd4',
    plan_hash_value => 3456789012);
END;
/

-- Load from AWR
DECLARE
  l_count NUMBER;
BEGIN
  l_count := DBMS_SPM.LOAD_PLANS_FROM_AWR(
    begin_snap => 100,
    end_snap   => 110);
END;
/

-- View baselines
SELECT SQL_HANDLE, SQL_TEXT, PLAN_NAME, ENABLED, ACCEPTED, FIXED, AUTOPURGE,
       ORIGIN, LAST_EXECUTED
FROM DBA_SQL_PLAN_BASELINES
ORDER BY LAST_EXECUTED DESC;

-- Fix a plan (optimizer MUST use it, even if better plan found)
EXEC DBMS_SPM.ALTER_SQL_PLAN_BASELINE(
  sql_handle => 'SQL_abc123',
  plan_name  => 'SQL_PLAN_abc123_1',
  attribute_name => 'FIXED',
  attribute_value => 'YES');

-- Evolve baselines (test new plans, accept if better)
SET LONG 1000000;
SELECT DBMS_SPM.EVOLVE_SQL_PLAN_BASELINE(
  sql_handle => 'SQL_abc123',
  verify     => 'YES',
  commit     => 'YES'
) FROM DUAL;
```

---

### Q27. What are Oracle Hints? List the most important ones and when to use (and avoid) them.

**Answer:**

**Critical rule:** Hints should be a last resort. First fix statistics, indexes, schema design.

```sql
-- Access path hints
SELECT /*+ FULL(e) */ * FROM employees e;           -- Force FTS
SELECT /*+ INDEX(e EMP_IDX) */ * FROM employees e;  -- Force index
SELECT /*+ NO_INDEX(e) */ * FROM employees e;       -- Prevent index use
SELECT /*+ INDEX_FFS(e EMP_IDX) */ * FROM employees e; -- Index Fast Full Scan

-- Join hints
SELECT /*+ USE_NL(e d) */ e.*, d.*                  -- Nested Loops
FROM employees e, departments d WHERE e.dept_id = d.dept_id;

SELECT /*+ USE_HASH(e d) */ e.*, d.*                -- Hash Join
FROM employees e, departments d WHERE e.dept_id = d.dept_id;

SELECT /*+ LEADING(d e) */ e.*, d.*                 -- Control join order
FROM employees e, departments d WHERE e.dept_id = d.dept_id;

-- Cardinality hints (fix bad estimates without touching statistics)
SELECT /*+ CARDINALITY(e 100) */ * FROM employees e; -- Tell optimizer: 100 rows

-- Parallel hints
SELECT /*+ PARALLEL(e 8) */ * FROM employees e;    -- Use 8 PQ slaves
SELECT /*+ NO_PARALLEL */ * FROM employees e;       -- Prevent parallel

-- Result Cache
SELECT /*+ RESULT_CACHE */ * FROM static_lookup_table;

-- Optimizer goal hints
SELECT /*+ ALL_ROWS */ * FROM employees;   -- Optimize for throughput (default)
SELECT /*+ FIRST_ROWS(10) */ * FROM employees; -- Optimize for first 10 rows

-- Materialization
SELECT /*+ MATERIALIZE */ * FROM (SELECT ...) t;  -- Force inline view materialization
SELECT /*+ INLINE */ * FROM (SELECT ...) t;        -- Force inline (not materialize)
```

---

### Q28. Explain Oracle Partitioning — types, pruning, and performance considerations.

**Answer:**

```sql
-- Range Partitioning (most common for time-series data)
CREATE TABLE orders (
  order_id   NUMBER,
  order_date DATE,
  amount     NUMBER
)
PARTITION BY RANGE (order_date) (
  PARTITION p2023_q1 VALUES LESS THAN (DATE '2023-04-01'),
  PARTITION p2023_q2 VALUES LESS THAN (DATE '2023-07-01'),
  PARTITION p2023_q3 VALUES LESS THAN (DATE '2023-10-01'),
  PARTITION p2023_q4 VALUES LESS THAN (DATE '2024-01-01'),
  PARTITION p_future VALUES LESS THAN (MAXVALUE)
);

-- Interval Partitioning (auto-creates partitions)
CREATE TABLE orders2 (
  order_id NUMBER, order_date DATE, amount NUMBER
)
PARTITION BY RANGE (order_date)
INTERVAL (NUMTOYMINTERVAL(1,'MONTH')) (
  PARTITION p_initial VALUES LESS THAN (DATE '2020-01-01')
);

-- List Partitioning
CREATE TABLE sales_by_region (
  sale_id NUMBER, region VARCHAR2(20), amount NUMBER
)
PARTITION BY LIST (region) (
  PARTITION p_north VALUES ('NORTH', 'NORTHEAST'),
  PARTITION p_south VALUES ('SOUTH', 'SOUTHEAST'),
  PARTITION p_other VALUES (DEFAULT)
);

-- Hash Partitioning (even distribution for OLTP)
CREATE TABLE accounts (
  account_id NUMBER, customer_id NUMBER, balance NUMBER
)
PARTITION BY HASH (account_id)
PARTITIONS 16;  -- must be power of 2

-- Composite Partitioning (Range-Hash)
CREATE TABLE big_sales (
  sale_id NUMBER, sale_date DATE, region VARCHAR2(20), amount NUMBER
)
PARTITION BY RANGE (sale_date)
SUBPARTITION BY HASH (region) SUBPARTITIONS 4 (
  PARTITION p2023 VALUES LESS THAN (DATE '2024-01-01'),
  PARTITION p2024 VALUES LESS THAN (DATE '2025-01-01')
);

-- Partition pruning verification
EXPLAIN PLAN FOR
SELECT SUM(amount) FROM orders WHERE order_date BETWEEN DATE '2023-01-01' AND DATE '2023-03-31';
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
-- Look for: "PARTITION RANGE SINGLE" or "PARTITION RANGE ITERATOR"

-- Partition maintenance operations
ALTER TABLE orders TRUNCATE PARTITION p2023_q1;   -- instant, no undo
ALTER TABLE orders DROP PARTITION p2023_q1;
ALTER TABLE orders SPLIT PARTITION p_future AT (DATE '2024-04-01') 
  INTO (PARTITION p2024_q1, PARTITION p_future);
ALTER TABLE orders MERGE PARTITIONS p2023_q1, p2023_q2 INTO PARTITION p2023_h1;
ALTER TABLE orders MOVE PARTITION p2023_q1 TABLESPACE ts_archive COMPRESS;
```

---

# SECTION 5: RAC — REAL APPLICATION CLUSTERS

### Q29. Explain Oracle RAC architecture. What are the key components and how do they work together?

**Answer:**

**RAC Components:**
- **Multiple instances:** Each node runs its own Oracle instance (SGA, background processes)
- **Shared storage:** All instances access the same database files (via ASM or cluster file system)
- **Cluster Interconnect:** Private high-speed network (InfiniBand/Ethernet) for inter-instance communication
- **Clusterware (Oracle Grid Infrastructure):** CSS, CRS, EVMD, OCR, Voting Disks
- **Cache Fusion:** Global Cache Service (GCS) + Global Enqueue Service (GES) for coordinated buffer cache

**Key RAC background processes:**

| Process | Function |
|---------|----------|
| LMD (Lock Manager Daemon) | Handles GES lock requests between nodes |
| LMS (Lock Monitor Server) | Transfers blocks between nodes (Cache Fusion) |
| LMON | Global Enqueue Service Monitor, handles node failures |
| LCK0 | Instance lock management |
| DIAG | Detects and logs cluster issues |
| ACMS | Atomic Controlfile to Memory Service |

```sql
-- RAC-specific views
SELECT INST_ID, INSTANCE_NAME, HOST_NAME, STATUS FROM GV$INSTANCE;
SELECT INST_ID, NAME, VALUE FROM GV$PARAMETER WHERE NAME='db_name';

-- Check cluster interconnect
SELECT NAME, IP_ADDRESS, IS_PUBLIC FROM V$CLUSTER_INTERCONNECTS;

-- Check GCS/GES stats
SELECT CLASS, GETS, MISSES, IMMEDIATE_GETS, IMMEDIATE_MISSES
FROM GV$LATCH
WHERE NAME LIKE 'gc%'
ORDER BY MISSES DESC;
```

---

### Q30. What is Cache Fusion in RAC? Explain CR (Consistent Read) and Current block transfers.

**Answer:**

**Cache Fusion** transfers data blocks directly between instance SGAs via the cluster interconnect — eliminating the need to write dirty blocks to disk before another instance can read them.

**Two types of block transfers:**

**CR (Consistent Read) transfer:**
- Node 2 needs a consistent (past) version of a block that Node 1 has modified
- Node 1 constructs a CR image using undo and sends it to Node 2
- No disk I/O involved → fast

**Current block transfer:**
- Node 2 needs the current (latest committed) block
- Node 1 sends its current block to Node 2 (with ownership transfer)
- Common with INSERT/UPDATE/DELETE

```sql
-- Check Cache Fusion activity
SELECT INST_ID, 
       GC_CR_BLOCKS_RECEIVED,
       GC_CURRENT_BLOCKS_RECEIVED, 
       GC_CR_BLOCK_RECEIVE_TIME / DECODE(GC_CR_BLOCKS_RECEIVED,0,1,GC_CR_BLOCKS_RECEIVED) AVG_CR_TIME,
       GC_CURRENT_BLOCK_RECEIVE_TIME / DECODE(GC_CURRENT_BLOCKS_RECEIVED,0,1,GC_CURRENT_BLOCKS_RECEIVED) AVG_CUR_TIME
FROM GV$SYSSTAT
WHERE NAME IN ('gc cr blocks received','gc current blocks received')
ORDER BY INST_ID;

-- Wait events related to Cache Fusion
SELECT INST_ID, EVENT, TOTAL_WAITS, TIME_WAITED_MICRO/1000000 SEC
FROM GV$SYSTEM_EVENT
WHERE EVENT LIKE 'gc%'
ORDER BY TIME_WAITED_MICRO DESC;
```

**Performance goals for Cache Fusion:**
- CR block transfer time: < 5ms (avg)
- Current block transfer time: < 10ms (avg)
- High gc cr/current blocks receive → healthy Cache Fusion (not a problem by itself)
- High gc buffer busy → indicates contention (too many sessions competing for same block)

---

### Q31. How do you add and remove a node from a RAC cluster? What is the procedure?

**Answer:**

**Adding a node (19c Grid Infrastructure):**

```bash
# Step 1: Prepare new node (same OS, same grid user, same SSH equivalence)
ssh-copy-id grid@newnode

# Step 2: On any existing node, run cluvfy
cluvfy stage -pre nodeadd -n newnode -verbose

# Step 3: Add node to cluster (as root on existing node)
$GRID_HOME/oui/bin/addNode.sh -silent \
  "CLUSTER_NEW_NODES={newnode}" \
  "CLUSTER_NEW_VIRTUAL_HOSTNAMES={newnode-vip}"

# Step 4: Run root.sh on new node
/u01/app/grid/19.0.0/root.sh

# Step 5: Add Oracle RDBMS home to new node
$ORACLE_HOME/oui/bin/addNode.sh -silent \
  "CLUSTER_NEW_NODES={newnode}"

# Step 6: Add instance to database
srvctl add instance -db proddb -instance proddb3 -node newnode

# Step 7: Create instance using DBCA or manually
dbca -silent -addInstance -nodeList newnode -gdbName proddb \
  -instanceName proddb3 -sysDBAUserName sys -sysDBAPassword password
```

**Removing a node:**
```bash
# Step 1: Migrate/stop workload
srvctl stop instance -db proddb -instance proddb3

# Step 2: Delete instance from database
srvctl delete instance -db proddb -instance proddb3

# Step 3: Delete database home from node
$ORACLE_HOME/oui/bin/runInstaller -silent -detachHome

# Step 4: Deconfigure grid on node being removed
$GRID_HOME/crs/install/rootcrs.sh -deconfig -force

# Step 5: Remove node from cluster (on remaining node)
crsctl delete node -n removednode
```

---

### Q32. Explain Oracle Clusterware — CSS, CRS, OCR, and Voting Disks.

**Answer:**

**Cluster Synchronization Services (CSS):**
- Manages cluster membership — knows which nodes are alive
- Uses voting disks to determine quorum
- Controls node eviction (STONITH — Shoot The Other Node In The Head)
- CSSD daemon — most critical: if it fails, node restarts

**Cluster Ready Services (CRS) / OHAS:**
- Top-level daemon managing all cluster resources
- Resource types: network, database, instance, service, VIP, SCAN
- CRS profiles define resource dependencies and start/stop order

**Oracle Cluster Registry (OCR):**
- Stores cluster configuration (resource definitions, dependencies)
- Stored on shared storage (typically ASM disk group or OCFS2)
- Automatically backed up every 4 hours to $GRID_HOME/cdata/

```bash
# Check OCR status
ocrcheck

# Manual OCR backup
ocrconfig -manualbackup

# View OCR backups
ocrconfig -showbackup

# Restore OCR from backup
ocrconfig -restore /backup/ocr_backup

# View cluster resources
crsctl status resource -t
```

**Voting Disks:**
- Odd number required (1, 3, 5 — for quorum)
- 3 voting disks: node needs 2 votes to survive
- If a node can't communicate with cluster AND loses disk access → evicted
- In ASM (11gR2+): voting disks stored inside ASM disk group

```bash
# Check voting disks
crsctl query css votedisk

# Add voting disk
crsctl add css votedisk /dev/sdd1

# Check CSS configuration
crsctl get css misscount  # Max seconds for no heartbeat (default 30)
crsctl get css disktimeout # Max seconds for voting disk I/O (default 200)
```

---

# SECTION 6: TROUBLESHOOTING

### Q33. A production database is running slow. Walk through your complete diagnostic approach.

**Answer:**

**Step 1: Quick health check (first 60 seconds)**
```sql
-- Check active sessions and wait events
SELECT COUNT(*) ACTIVE FROM V$SESSION WHERE STATUS='ACTIVE' AND TYPE='USER';

SELECT EVENT, COUNT(*) CNT, 
       ROUND(AVG(SECONDS_IN_WAIT),2) AVG_WAIT
FROM V$SESSION_WAIT
WHERE WAIT_CLASS != 'Idle'
GROUP BY EVENT ORDER BY CNT DESC;

-- Check for blocking locks
SELECT BLOCKING_SESSION, SID, SERIAL#, SQL_ID, EVENT, SECONDS_IN_WAIT
FROM V$SESSION WHERE BLOCKING_SESSION IS NOT NULL;
```

**Step 2: AWR Snapshot (if not ongoing)**
```sql
EXEC DBMS_WORKLOAD_REPOSITORY.CREATE_SNAPSHOT();
-- Wait 15-30 minutes during slow period
EXEC DBMS_WORKLOAD_REPOSITORY.CREATE_SNAPSHOT();
```

**Step 3: Generate AWR Report**
```sql
SELECT * FROM TABLE(DBMS_WORKLOAD_REPOSITORY.AWR_REPORT_HTML(
  l_dbid     => (SELECT DBID FROM V$DATABASE),
  l_inst_num => 1,
  l_bid      => 100,  -- begin snap
  l_eid      => 101   -- end snap
));
```

**Step 4: Top SQL by resource**
```sql
-- Top SQL by elapsed time (from cursor cache)
SELECT ROUND(ELAPSED_TIME/1000000,2) ELAPSED_SEC,
       ROUND(CPU_TIME/1000000,2) CPU_SEC,
       EXECUTIONS, BUFFER_GETS, ROWS_PROCESSED,
       SQL_ID, SUBSTR(SQL_TEXT,1,100) SQL_TEXT
FROM V$SQL
ORDER BY ELAPSED_TIME DESC FETCH FIRST 20 ROWS ONLY;
```

**Step 5: Check for specific issues**
```sql
-- Latch contention
SELECT LATCH#, NAME, GETS, MISSES, SPIN_GETS,
       ROUND(MISSES/GETS*100,2) MISS_PCT
FROM V$LATCH WHERE MISSES > 0 ORDER BY MISSES DESC;

-- Checkpoint not complete
SELECT SEQUENCE#, STATUS, ARCHIVED FROM V$LOG;

-- Archive log destination space
SELECT DEST_ID, STATUS, TARGET, ARCHIVER, SCHEDULE, DESTINATION 
FROM V$ARCHIVE_DEST WHERE STATUS='VALID';

-- Tablespace usage
SELECT TABLESPACE_NAME, 
       ROUND((TOTAL_BYTES-FREE_BYTES)/1024/1024/1024,2) USED_GB,
       ROUND(FREE_BYTES/1024/1024/1024,2) FREE_GB,
       ROUND((TOTAL_BYTES-FREE_BYTES)/TOTAL_BYTES*100,1) PCT_USED
FROM (SELECT TABLESPACE_NAME, SUM(BYTES) TOTAL_BYTES FROM DBA_DATA_FILES GROUP BY TABLESPACE_NAME),
     (SELECT TABLESPACE_NAME TN, SUM(BYTES) FREE_BYTES FROM DBA_FREE_SPACE GROUP BY TABLESPACE_NAME)
WHERE TABLESPACE_NAME = TN;
```

---

### Q34. ORA-04031: Unable to allocate X bytes in Shared Pool. How do you diagnose and fix?

**Answer:**

**Root causes:**
1. Shared pool too small
2. Massive hard parsing (no bind variables, cursor_sharing=EXACT)
3. Library cache fragmentation (many small objects can't be coalesced)
4. Memory leak in shared pool

```sql
-- Step 1: Check current shared pool usage
SELECT NAME, BYTES/1024/1024 MB FROM V$SGAINFO
WHERE NAME LIKE '%Shared Pool%';

SELECT POOL, NAME, BYTES/1024/1024 MB
FROM V$SGASTAT
WHERE POOL = 'shared pool'
ORDER BY BYTES DESC FETCH FIRST 10 ROWS ONLY;

-- Step 2: Check free memory fragmentation
SELECT * FROM V$SHARED_POOL_ADVICE ORDER BY SHARED_POOL_SIZE_FACTOR;

-- Check sub-heaps of shared pool
SELECT KGHSSNAM, KGHSSLEN/1024 KB, KGHSSNEW, KGHSSFRE, KGHSSUSE
FROM X$KGHSS ORDER BY KGHSSLEN DESC;

-- Step 3: Identify top memory consumers
SELECT OWNER, NAME, TYPE, LOADS, INVALIDATIONS,
       SHARABLE_MEM/1024 KB
FROM V$DB_OBJECT_CACHE
ORDER BY SHARABLE_MEM DESC FETCH FIRST 20 ROWS ONLY;

-- Step 4: Immediate relief — flush shared pool (USE CAREFULLY on production)
ALTER SYSTEM FLUSH SHARED_POOL;

-- Step 5: Pin critical objects (prevent aging out)
EXEC DBMS_SHARED_POOL.KEEP('SYS.STANDARD','PACKAGE');
EXEC DBMS_SHARED_POOL.KEEP('HR.PROC_CRITICAL','PROCEDURE');

-- Step 6: Fix root cause — enable cursor sharing
ALTER SYSTEM SET CURSOR_SHARING = FORCE; -- use SIMILAR or EXACT if possible
-- Better: Fix application to use bind variables

-- Step 7: Increase shared pool
ALTER SYSTEM SET SHARED_POOL_SIZE = 2G SCOPE=BOTH;
-- Or increase SGA_TARGET if using ASMM
```

---

### Q35. How do you diagnose and resolve an ORA-01555 "Snapshot too old" error?

**Answer:**

**Root cause:** A long-running query started at time T. Between T and now, the undo blocks it needs to reconstruct a consistent read image were overwritten.

```sql
-- Step 1: Identify the frequency and affected queries
SELECT SSOLDERRCNT, NOSPACEERRCNT, MAXQUERYLEN, MAXQUERYID,
       UNDOBLKS, ACTIVEBLKS, EXPIREDBLKS, UNEXPIREDBLKS
FROM V$UNDOSTAT
ORDER BY BEGIN_TIME DESC;

-- Step 2: Find the longest running queries
SELECT SID, SQL_ID, LAST_CALL_ET, STATUS,
       TO_CHAR(LOGON_TIME,'YYYY-MM-DD HH24:MI') LOGON_TIME
FROM V$SESSION
WHERE STATUS='ACTIVE' AND TYPE='USER'
ORDER BY LAST_CALL_ET DESC;

-- Step 3: Check undo retention vs max query length
-- MAXQUERYLEN should be < UNDO_RETENTION setting
SELECT VALUE FROM V$PARAMETER WHERE NAME = 'undo_retention';

-- Step 4: Calculate required undo retention
SELECT CEIL(MAXQUERYLEN * 1.5) RECOMMENDED_RETENTION
FROM (SELECT MAX(MAXQUERYLEN) MAXQUERYLEN FROM V$UNDOSTAT);

-- Step 5: Solutions
-- Solution A: Increase undo retention
ALTER SYSTEM SET UNDO_RETENTION = 3600; -- 1 hour

-- Solution B: Guarantee retention (risky — DML fails instead of ORA-01555)
ALTER TABLESPACE UNDOTBS1 RETENTION GUARANTEE;

-- Solution C: Increase undo tablespace size
ALTER TABLESPACE UNDOTBS1 ADD DATAFILE '/u01/oradata/undo02.dbf' SIZE 10G AUTOEXTEND ON;

-- Solution D: Application fix
-- Use DBMS_FLASHBACK or flashback query for long-running reads
-- Commit more frequently in batch jobs
-- Use parallel query to reduce per-process duration

-- Step 6: Verify fix
SELECT BEGIN_TIME, END_TIME, SSOLDERRCNT, UNDO_RETENTION 
FROM V$UNDOSTAT ORDER BY BEGIN_TIME DESC;
```

---

### Q36. A RAC node was evicted (crashed). How do you diagnose the root cause?

**Answer:**

**Immediate checks:**
```bash
# Step 1: Check alert log on evicted node
tail -10000 $ORACLE_BASE/diag/rdbms/proddb/proddb1/trace/alert_proddb1.log | grep -E "evict|EVICT|CSS|CRS|fatal|FATAL"

# Step 2: Check Oracle Clusterware logs
tail -5000 $GRID_HOME/log/hostname/crsd/crsd.log | grep -i "evict\|fail\|error"
cat $GRID_HOME/log/hostname/cssd/ocssd.log | grep -E "evict|EVICT|miss"

# Step 3: Check CRS daemon logs
ls $GRID_HOME/log/hostname/

# Step 4: Determine eviction reason
grep -i "Eviction" $GRID_HOME/log/hostname/cssd/ocssd.log | tail -20
```

**Common eviction causes and solutions:**

| Cause | Log Evidence | Solution |
|-------|-------------|----------|
| **Network heartbeat missed** | "CSS daemon missed X heartbeats" | Check private interconnect NIC, switches |
| **Voting disk timeout** | "Disk timeout" | Check storage latency to voting disks |
| **OS load (missed CSS timer)** | "CSSD exiting due to operational error" | Reduce CPU contention, check hugepages |
| **Interconnect congestion** | "gc buffer busy" spikes before eviction | Separate traffic, use bonding/InfiniBand |
| **Kernel OOM killer** | Check /var/log/messages for "oom-kill" | Increase RAM, configure vm.overcommit |

```bash
# Check OS messages for OOM
dmesg | grep -i oom | tail -20
grep -i "oom\|kill\|crash" /var/log/messages | grep -i oracle | tail -20

# Check network stats on interconnect
netstat -s | grep -E "error|fail|drop"
ifconfig bondpvt | grep -E "error|drop"

# Check disk latency for voting disks
iostat -x 1 5 | grep sdd  # assuming /dev/sdd is voting disk

# Verify interconnect configuration
oifcfg getif    # should show cluster_interconnect interface
crsctl query css votedisk
```

---

### Q37. How do you rebuild a corrupt index? How do you detect index corruption?

**Answer:**

```sql
-- Step 1: Detect corruption via ANALYZE
ANALYZE INDEX hr.emp_name_idx VALIDATE STRUCTURE;
-- Check for errors in subsequent queries:
SELECT HEIGHT, BLOCKS, BR_BLKS, LF_ROWS, DEL_LF_ROWS,
       ROUND(DEL_LF_ROWS/LF_ROWS*100,2) PCT_DELETED
FROM INDEX_STATS;
-- High PCT_DELETED (>20%) = bloated, needs rebuild

-- Step 2: Detect logical corruption
DBMS_REPAIR.CHECK_OBJECT(
  schema_name    => 'HR',
  object_name    => 'EMPLOYEES',
  repair_type    => DBMS_REPAIR.INDEX_REPAIR_TYPE
);

-- Step 3: Rebuild index online (no table lock)
ALTER INDEX hr.emp_name_idx REBUILD ONLINE;

-- Step 4: Rebuild with new storage parameters
ALTER INDEX hr.emp_name_idx REBUILD 
  TABLESPACE hr_idx 
  ONLINE 
  PARALLEL 4
  NOLOGGING;  -- faster rebuild, turn on logging after

-- Step 5: Coalesce (merge index blocks without rebuilding)
ALTER INDEX hr.emp_name_idx COALESCE;
-- Less invasive than rebuild, but doesn't fix all issues

-- Step 6: If RMAN detects physical corruption
RMAN> VALIDATE DATABASE;
RMAN> SELECT * FROM V$DATABASE_BLOCK_CORRUPTION;

-- Block-level recovery
RMAN> RECOVER CORRUPTION LIST;  -- recover corrupt blocks identified in V$DATABASE_BLOCK_CORRUPTION
RMAN> BLOCKRECOVER DATAFILE 5 BLOCK 1234, 1235;  -- recover specific blocks

-- Step 7: Drop and recreate (for severe corruption)
DROP INDEX hr.emp_name_idx;
CREATE INDEX hr.emp_name_idx ON hr.employees(last_name, first_name) 
  PARALLEL 8 NOLOGGING;
ALTER INDEX hr.emp_name_idx NOPARALLEL LOGGING;
```

---

### Q38. Explain Oracle ASM (Automatic Storage Management). What disk groups, failure groups, and AU sizes should you configure for different workloads?

**Answer:**

**ASM Architecture:**
- ASM Instance (separate from DB instance): Manages metadata
- ASM Disk Groups: Collection of disks managed as a unit
- Failure Groups: Subsets of disks that can fail together (represent a single point of failure — a storage controller, RAID group, etc.)
- Allocation Units (AU): Minimum unit of space allocation (1MB default, up to 64MB)

```sql
-- Create disk group (redundancy modes)
-- EXTERNAL: No ASM mirroring (use hardware RAID)
CREATE DISKGROUP DATA EXTERNAL REDUNDANCY 
  DISK '/dev/sdb', '/dev/sdc', '/dev/sdd', '/dev/sde';

-- NORMAL: 2-way mirroring (2+ failure groups)
CREATE DISKGROUP DATA NORMAL REDUNDANCY
  FAILGROUP ctrl1 DISK '/dev/sdb', '/dev/sdc'
  FAILGROUP ctrl2 DISK '/dev/sdd', '/dev/sde'
  ATTRIBUTE 'AU_SIZE' = '4M',       -- 4MB for databases
             'COMPATIBLE.ASM' = '19.0',
             'COMPATIBLE.RDBMS' = '19.0';

-- HIGH: 3-way mirroring (3+ failure groups)
CREATE DISKGROUP RECO HIGH REDUNDANCY
  FAILGROUP ctrl1 DISK '/dev/sdh'
  FAILGROUP ctrl2 DISK '/dev/sdi'
  FAILGROUP ctrl3 DISK '/dev/sdj';

-- Check disk group status
SELECT GROUP_NUMBER, NAME, STATE, TYPE, 
       TOTAL_MB/1024 TOTAL_GB, FREE_MB/1024 FREE_GB,
       OFFLINE_DISKS
FROM V$ASM_DISKGROUP;

-- Check individual disks
SELECT GROUP_NUMBER, DISK_NUMBER, NAME, PATH, STATE, 
       TOTAL_MB, FREE_MB, READS, WRITES, READ_ERRS, WRITE_ERRS
FROM V$ASM_DISK;

-- Rebalance (automatic but can be manually triggered)
ALTER DISKGROUP DATA REBALANCE POWER 8; -- 1-1024 (higher=faster, more I/O)

-- Monitor rebalance
SELECT * FROM V$ASM_OPERATION;
```

**AU Size recommendations:**

| Workload | AU Size | Reason |
|----------|---------|--------|
| OLTP | 1MB (default) | Small random I/O |
| DW / Analytics | 4MB or 8MB | Large sequential scans |
| Redo Logs | 1MB | Sequential, small |
| VLDB | 64MB | Very large sequential |

---

### Q39. How do you perform a manual database recovery when the control file is lost?

**Answer:**

**Scenario A: Control file backup available (RMAN)**
```sql
RMAN> RESTORE CONTROLFILE FROM AUTOBACKUP;
RMAN> MOUNT;
RMAN> RECOVER DATABASE;
RMAN> ALTER DATABASE OPEN RESETLOGS;
```

**Scenario B: Create control file from scratch**
```sql
-- Step 1: Get datafile list from alert log, RMAN catalog, or OS
-- Step 2: Shutdown instance
SHUTDOWN ABORT;

-- Step 3: Create control file (NORESETLOGS if all logs available)
STARTUP NOMOUNT;

CREATE CONTROLFILE REUSE DATABASE "PRODDB" NORESETLOGS ARCHIVELOG
  MAXLOGFILES 16
  MAXLOGMEMBERS 4
  MAXDATAFILES 1024
  MAXINSTANCES 1
  MAXLOGHISTORY 2000
LOGFILE
  GROUP 1 '/u01/oradata/redo01.log' SIZE 200M,
  GROUP 2 '/u01/oradata/redo02.log' SIZE 200M,
  GROUP 3 '/u01/oradata/redo03.log' SIZE 200M
DATAFILE
  '/u01/oradata/system01.dbf',
  '/u01/oradata/sysaux01.dbf',
  '/u01/oradata/undotbs01.dbf',
  '/u01/oradata/users01.dbf',
  '/u01/oradata/hr_data01.dbf'
CHARACTER SET AL32UTF8;

-- Step 4: Recover if needed
RECOVER DATABASE;

-- Step 5: Open database
ALTER DATABASE OPEN;
-- OR if logs missing:
ALTER DATABASE OPEN RESETLOGS;

-- Step 6: Rebuild tempfiles
ALTER TABLESPACE TEMP ADD TEMPFILE '/u01/oradata/temp01.dbf' SIZE 2G;

-- Step 7: Take immediate full backup
RMAN> BACKUP DATABASE PLUS ARCHIVELOG;
```

---

### Q40. What is Oracle Flashback Technology? Explain all Flashback features with syntax.

**Answer:**

```sql
-- 1. FLASHBACK QUERY: See data as of a past time/SCN
SELECT * FROM hr.employees AS OF TIMESTAMP 
  (SYSTIMESTAMP - INTERVAL '30' MINUTE);

SELECT * FROM hr.employees AS OF SCN 5827000;

-- 2. FLASHBACK TABLE: Undo DML on a table
-- First enable row movement
ALTER TABLE hr.employees ENABLE ROW MOVEMENT;

FLASHBACK TABLE hr.employees TO TIMESTAMP 
  (SYSTIMESTAMP - INTERVAL '1' HOUR);

FLASHBACK TABLE hr.employees TO SCN 5827000;

-- After flashback: disable row movement
ALTER TABLE hr.employees DISABLE ROW MOVEMENT;

-- 3. FLASHBACK TABLE (from DROP — Recycle Bin)
FLASHBACK TABLE employees TO BEFORE DROP;
FLASHBACK TABLE "BIN$abcdef==$0" TO BEFORE DROP RENAME TO employees_old;

-- 4. FLASHBACK VERSION QUERY: See all row versions over time
SELECT VERSIONS_STARTTIME, VERSIONS_ENDTIME, VERSIONS_OPERATION,
       VERSIONS_XID, LAST_NAME, SALARY
FROM hr.employees VERSIONS BETWEEN TIMESTAMP 
  (SYSTIMESTAMP - INTERVAL '1' HOUR) AND SYSTIMESTAMP
WHERE employee_id = 100;

-- 5. FLASHBACK TRANSACTION QUERY: Find transactions that changed data
SELECT XID, START_TIMESTAMP, COMMIT_TIMESTAMP, LOGON_USER, UNDO_SQL
FROM FLASHBACK_TRANSACTION_QUERY
WHERE TABLE_OWNER = 'HR' AND TABLE_NAME = 'EMPLOYEES'
ORDER BY START_TIMESTAMP DESC;

-- 6. FLASHBACK DATABASE: Roll back entire database
RMAN> FLASHBACK DATABASE TO TIMESTAMP 
      TO_TIMESTAMP('2024-03-10 14:30:00','YYYY-MM-DD HH24:MI:SS');
RMAN> ALTER DATABASE OPEN RESETLOGS;

-- Prerequisites for Flashback Database:
-- DB_RECOVERY_FILE_DEST must be set
-- ALTER DATABASE FLASHBACK ON;
SELECT FLASHBACK_ON FROM V$DATABASE;

-- Check available flashback window
SELECT OLDEST_FLASHBACK_SCN, OLDEST_FLASHBACK_TIME FROM V$FLASHBACK_DATABASE_LOG;

-- 7. FLASHBACK DATA ARCHIVE (Total Recall, 11g+)
CREATE FLASHBACK ARCHIVE fa_longterm
  TABLESPACE flashback_ts
  RETENTION 2 YEAR;

ALTER TABLE hr.employees FLASHBACK ARCHIVE fa_longterm;
-- Now can query up to 2 years back regardless of undo retention
```

---

*End of Oracle Deep Interview Questions & Answers*

**Total Coverage:**
- Oracle 8i through 23c version features
- RMAN: Architecture, Backup Sets vs Image Copies, Incremental Merge, Channels, BCT, PITR, Duplicate
- Data Pump: Architecture, INCLUDE/EXCLUDE/REMAP, Network Link, Monitor, Compression/Encryption  
- Performance Tuning: Execution Plans, Adaptive QO, SPM, Hints, Partitioning
- RAC: Architecture, Cache Fusion, Node Add/Remove, Clusterware
- Troubleshooting: Performance diagnosis, ORA-04031, ORA-01555, Node eviction, Index corruption, ASM, Control file recovery, Flashback
