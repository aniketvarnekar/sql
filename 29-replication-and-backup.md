# Replication and Backup

Replication and backup are the two pillars of MySQL data availability and recovery. Replication keeps a live copy of your data on one or more replica servers for high availability and read scaling. Backups provide point-in-time snapshots for recovery from data loss, corruption, or accidental deletion.

## Overview of MySQL Replication

MySQL replication works by having the primary server record every data-modifying statement or row change in its binary log. Replica servers connect to the primary, receive the binary log events, and apply them to their own data — keeping themselves synchronized with the primary. This happens asynchronously by default.

Replication serves several purposes:
- **High availability:** if the primary fails, a replica can be promoted to become the new primary
- **Read scaling:** read-heavy queries can be directed to replicas, reducing load on the primary
- **Backup offloading:** backups can be taken from a replica without impacting primary performance
- **Geographical distribution:** replicas can be placed in different data centers

## Primary / Replica Setup Concept

The core components of MySQL replication are:

The **primary server** (formerly called "master") enables its binary log and assigns each change a binary log position. Every `INSERT`, `UPDATE`, `DELETE`, `CREATE TABLE`, and other data-modifying event is recorded.

The **replica server** (formerly called "slave") runs two threads:
1. The **I/O thread** connects to the primary, reads the binary log, and writes the events to the replica's local relay log.
2. The **SQL thread** reads the relay log and applies the events to the replica's data.

A replica can itself be a primary to another replica, creating a chain. This is called cascading or chained replication.

To view replication status on a replica:

```sql
SHOW REPLICA STATUS\G
-- MySQL 8.0+ uses REPLICA terminology
-- Older versions use: SHOW SLAVE STATUS\G
```

Key fields to monitor: `Replica_IO_Running`, `Replica_SQL_Running` (both should be 'Yes'), `Seconds_Behind_Source` (replication lag in seconds), and `Last_Error`.

## Types of Replication

### Asynchronous Replication

The default mode. The primary writes to its binary log and immediately acknowledges the transaction to the client without waiting for any replica to confirm receipt. If the primary fails before a replica has received the latest events, those events are lost — the replica is slightly behind.

This provides the best performance but the least durability guarantee in a failover scenario.

### Semi-Synchronous Replication

A plugin-based mode where the primary waits until at least one replica acknowledges receipt of the binary log events before returning success to the client. The primary does not wait for the replica to apply the events — only for receipt.

Semi-sync trades some write latency for a stronger durability guarantee: if the primary fails after a commit, at least one replica has the data. Available in MySQL 8.0 as a built-in plugin:

```sql
-- On primary:
INSTALL PLUGIN rpl_semi_sync_source SONAME 'semisync_source.so';
SET GLOBAL rpl_semi_sync_source_enabled = 1;

-- On replica:
INSTALL PLUGIN rpl_semi_sync_replica SONAME 'semisync_replica.so';
SET GLOBAL rpl_semi_sync_replica_enabled = 1;
```

### Group Replication

MySQL Group Replication (and the higher-level InnoDB Cluster built on it) uses a consensus protocol to synchronize transactions across a group of servers. All members must agree before a transaction commits. This provides strong consistency guarantees and automatic failover but requires more setup and has higher write latency.

## mysqldump for Logical Backups

`mysqldump` is the standard MySQL tool for creating logical backups — SQL files containing `CREATE TABLE` and `INSERT` statements that can reconstruct the database:

```bash
# Backup a single database:
mysqldump -u root -p database_name > backup.sql

# Backup multiple databases:
mysqldump -u root -p --databases db1 db2 > backup.sql

# Backup all databases:
mysqldump -u root -p --all-databases > all_databases.sql

# Backup with common production-safe options:
mysqldump -u root -p \
    --single-transaction \
    --routines \
    --triggers \
    --events \
    database_name > backup.sql
```

`--single-transaction` takes a consistent snapshot without locking tables, using InnoDB's MVCC. It is essential for live backups of production InnoDB databases. Without it, `mysqldump` would lock every table it reads, causing queries to wait for the duration of the backup.

`--routines` includes stored procedures and functions. `--triggers` includes triggers. `--events` includes scheduled events.

To restore from a `mysqldump` backup:

```bash
mysql -u root -p database_name < backup.sql
```

Or to restore all databases:

```bash
mysql -u root -p < all_databases.sql
```

Logical backups are portable — they can be restored to a different MySQL version, operating system, or server — but they are slower to create and restore than physical backups on large databases.

## mysqlpump and MySQL Shell for Modern Backups

`mysqlpump` (not to be confused with `mysqldump`) is a parallel backup tool included with MySQL 5.7+. It backs up multiple tables or databases simultaneously, which is significantly faster than `mysqldump` for large databases:

```bash
mysqlpump -u root -p --default-parallelism=4 database_name > backup.sql
```

MySQL Shell's `util.dumpInstance()`, `util.dumpSchemas()`, and `util.loadDump()` provide the most modern backup approach, with parallel execution, compression, progress tracking, and the ability to stream directly to or from cloud storage:

```bash
mysqlsh -- util dumpSchemas '["database_name"]' --outputUrl='/backups/dump'
mysqlsh -- util loadDump '/backups/dump'
```

For large production databases (hundreds of GB or more), physical backup tools like **Percona XtraBackup** or **MySQL Enterprise Backup** are standard. They copy the raw InnoDB data files without SQL generation, making backups and restores orders of magnitude faster.

## Point-in-Time Recovery Concept

Point-in-time recovery (PITR) allows you to restore the database to any specific moment, not just the moment of the last backup. It requires:

1. A base backup (a full `mysqldump` or physical backup)
2. Binary log files from after the backup was taken

To recover to a specific point in time:

```bash
# Step 1: Restore the base backup:
mysql -u root -p database_name < backup_from_yesterday.sql

# Step 2: Apply binary log events up to (but not including) the moment of the bad event:
mysqlbinlog \
    --start-datetime="2024-03-15 02:00:00" \
    --stop-datetime="2024-03-15 14:29:59" \
    /var/lib/mysql/binlog.000042 | mysql -u root -p
```

This replays all changes from the binary log between the backup time and the target time, effectively restoring the database to the moment before the disaster.

## Binary Logs and Their Role in Recovery

The binary log (`binlog`) is a record of all data-modifying events on the primary, used for both replication and PITR:

```sql
-- Check if binary logging is enabled:
SHOW VARIABLES LIKE 'log_bin';

-- List available binary log files:
SHOW BINARY LOGS;

-- View the contents of a binary log:
SHOW BINLOG EVENTS IN 'binlog.000042' LIMIT 20;

-- Purge old binary logs (up to but not including the named file):
PURGE BINARY LOGS TO 'binlog.000042';

-- Purge binary logs older than a specified time:
PURGE BINARY LOGS BEFORE '2024-01-01 00:00:00';
```

Binary logs should be retained long enough to cover your recovery window (typically 7-30 days) but purged periodically to avoid filling the disk. The `binlog_expire_logs_seconds` variable (MySQL 8.0) controls automatic purging:

```sql
SHOW VARIABLES LIKE 'binlog_expire_logs_seconds';
SET GLOBAL binlog_expire_logs_seconds = 604800; -- 7 days
```

## Practice Problems

### Problem 1
Write the `mysqldump` command to take a production-safe backup of a database called `ecommerce`, including all routines, triggers, and events, using single-transaction mode to avoid table locks. Then show how to restore it.

**Schema used:**
```sql
-- No SQL schema needed; this is a shell command exercise.
```

**Your query:**
```bash
-- Write the backup command and the restore command.
```

**Solution:**
```bash
# Backup:
mysqldump \
    -u root -p \
    --single-transaction \
    --routines \
    --triggers \
    --events \
    --hex-blob \
    --set-gtid-purged=OFF \
    ecommerce > ecommerce_backup_$(date +%Y%m%d_%H%M%S).sql

# Restore:
mysql -u root -p ecommerce < ecommerce_backup_20240315_020000.sql
```

**Explanation:**
`--single-transaction` wraps the entire backup in a `START TRANSACTION` with `REPEATABLE READ` isolation, capturing a consistent snapshot of all InnoDB tables without any locks. Without this flag, `mysqldump` locks each table while dumping it, blocking writes. `--routines` includes stored procedures and functions, which `mysqldump` omits by default. `--triggers` and `--events` are also not included by default. `--hex-blob` encodes BLOB and BINARY columns in hexadecimal, preventing encoding issues in the SQL file. `--set-gtid-purged=OFF` avoids GTID-related errors if you're restoring to a server without GTID mode enabled. The `$(date +%Y%m%d_%H%M%S)` appends a timestamp to the filename so each backup has a unique name.

### Problem 2
Demonstrate how to check the binary log status, list available binary log files, and show the events in a binary log. Also show how to set automatic binary log expiration.

**Schema used:**
```sql
-- No schema needed; these are administrative queries.
```

**Your query:**
```sql
-- Write the administrative SQL statements.
```

**Solution:**
```sql
-- Check if binary logging is enabled and find the current log file:
SHOW VARIABLES LIKE 'log_bin';
SHOW VARIABLES LIKE 'log_bin_basename';
SHOW VARIABLES LIKE 'binlog_format';

-- List all binary log files and their sizes:
SHOW BINARY LOGS;

-- Show recent events in the current binary log:
SHOW BINLOG EVENTS LIMIT 20;

-- Show events in a specific binary log file:
SHOW BINLOG EVENTS IN 'binlog.000001' LIMIT 50;

-- Check current expiration setting (MySQL 8.0):
SHOW VARIABLES LIKE 'binlog_expire_logs_seconds';

-- Set binary logs to expire after 7 days (604800 seconds):
SET GLOBAL binlog_expire_logs_seconds = 604800;

-- Manually purge binary logs older than a specific date:
PURGE BINARY LOGS BEFORE DATE_SUB(NOW(), INTERVAL 7 DAY);

-- Force creation of a new binary log file:
FLUSH BINARY LOGS;
```

**Explanation:**
`SHOW BINARY LOGS` lists each binary log file, its size, and whether it is encrypted. Binary log files accumulate over time; without regular purging, they fill the disk. `binlog_expire_logs_seconds` (replacing the deprecated `expire_logs_days` in MySQL 8.0) controls automatic expiration. The `PURGE BINARY LOGS BEFORE` command immediately removes files older than the specified date. `FLUSH BINARY LOGS` closes the current binary log and opens a new one with the next sequential number, which is useful before taking a backup — the backup can note which binary log file it corresponds to, simplifying PITR.

### Problem 3
Explain the steps for point-in-time recovery after an accidental `DROP TABLE` at a known time, given that you have a daily backup and binary logs.

**Schema used:**
```sql
-- No schema needed; this is a recovery scenario.
```

**Your query:**
```bash
-- Write the recovery steps as shell commands with explanations.
```

**Solution:**
```bash
# Scenario: The table 'orders' was accidentally dropped at 2024-03-15 14:30:00.
# Last backup was taken at 2024-03-15 02:00:00.
# Goal: Restore to the state at 2024-03-15 14:29:59 (one second before the drop).

# Step 1: Identify which binary log contains the DROP TABLE event.
# Find binary logs written after 02:00:00 on 2024-03-15:
mysqlbinlog --start-datetime="2024-03-15 02:00:00" /var/lib/mysql/binlog.000042 | grep -i "DROP TABLE"
# Note the position or timestamp just before the DROP TABLE event.

# Step 2: Restore the most recent backup to a clean database:
mysql -u root -p -e "CREATE DATABASE ecommerce_recovery"
mysql -u root -p ecommerce_recovery < ecommerce_backup_20240315_020000.sql

# Step 3: Apply binary log events from after the backup up to just before the accident.
# Using stop-datetime to exclude the DROP TABLE and everything after it:
mysqlbinlog \
    --start-datetime="2024-03-15 02:00:00" \
    --stop-datetime="2024-03-15 14:29:59" \
    /var/lib/mysql/binlog.000042 | mysql -u root -p ecommerce_recovery

# Step 4: Verify the recovered database has the orders table:
mysql -u root -p ecommerce_recovery -e "SHOW TABLES; SELECT COUNT(*) FROM orders;"

# Step 5: Once verified, rename/swap the recovered database with production.
# (In practice, done during a maintenance window with application downtime.)
```

**Explanation:**
Point-in-time recovery works by combining a full backup (which captures the database state at backup time) with binary log replay (which replays every change made after the backup). The key is using `--stop-datetime` (or `--stop-position`) to stop replaying events at the moment just before the accidental `DROP TABLE`. Restoring to a separate database (`ecommerce_recovery`) rather than directly to production allows you to verify the recovery before the cutover. Binary log-based PITR can recover to any second in time, which is far more granular than recovering from daily backups alone — but it only works if binary logging was enabled before the incident and the log files are still intact.
