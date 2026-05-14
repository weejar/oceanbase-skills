# Log Archive Management

## Overview

Log archiving is the foundation of OceanBase backup and recovery. Without archived logs, point-in-time recovery is impossible. This guide covers configuring, monitoring, and troubleshooting log archives in OceanBase V4.2+.

> **Critical**: Always enable log archive BEFORE taking your first data backup.

---

## Architecture

OceanBase uses a **Paxos-based log replication** mechanism. Each transaction's redo log (Clog) is synchronously replicated to multiple replicas. Log archive copies these logs to external storage for long-term retention.

```
Transaction → Clog (mem) → Clog (disk) → Log Archive (external storage)
```

---

## Enabling Log Archive

### Step 1: Configure Archive Destination

```sql
-- MySQL Mode: Set archive destination (NFS example)
ALTER SYSTEM SET log_archive_dest = 'file:///data/ob_archive';

-- Or OSS (Alibaba Cloud Object Storage Service)
ALTER SYSTEM SET log_archive_dest = 'oss://access_id:access_key@endpoint/bucket/archive';

-- Verify
SHOW PARAMETERS LIKE 'log_archive_dest';
```

### Step 2: Start Archiving

```sql
-- Start archiving (run on sys tenant)
ALTER SYSTEM ARCHIVE LOG;

-- Verify archive is running
SELECT * FROM oceanbase.DBA_OB_ARCHIVELOG;
```

### Step 3: Monitor Archive Lag

```sql
-- Check archive lag (should be < 5 seconds)
SELECT tenant_id,
       destination,
       round(lag_seconds, 2) AS lag_sec,
       status
FROM oceanbase.DBA_OB_ARCHIVELOG;
```

---

## Log Archive Parameters

| Parameter | Default | Recommended | Description |
|-----------|---------|-------------|-------------|
| `log_archive_dest` | (empty) | `file:///path` or `oss://...` | Archive destination URL |
| `log_archive_concurrency` | 4 | 4-8 | Parallel archive workers |
| `log_archive_piece_switch_interval` | 0 (disabled) | `7d` | Switch archive piece interval |
| `enable_syslog_recycle` | FALSE | TRUE | Auto-recycle observer logs |

```sql
-- Recommended settings
ALTER SYSTEM SET log_archive_concurrency = 8;
ALTER SYSTEM SET log_archive_piece_switch_interval = '7d';
ALTER SYSTEM SET enable_syslog_recycle = TRUE;
ALTER SYSTEM SET max_syslog_file_count = 50;
```

---

## Monitoring Log Archive

### Check Archive Status

```sql
-- Detailed archive status per tenant
SELECT tenant_id,
       destination,
       status,               -- DOING = running, STOP = stopped
       round(lag_seconds, 2) AS lag_sec,
       start_scn,
       checkpoint_scn
FROM oceanbase.DBA_OB_ARCHIVELOG
ORDER BY tenant_id;
```

### Check Archive Size

```sql
-- Space used by archived logs
SELECT tenant_id,
       round(sum(file_size) / 1024 / 1024 / 1024, 2) AS archive_gb
FROM oceanbase.DBA_OB_ARCHIVELOG_FILE
GROUP BY tenant_id;
```

### Alert: Archive Lag > 60 Seconds

```sql
-- Find tenants with dangerous archive lag
SELECT tenant_id,
       destination,
       round(lag_seconds, 2) AS lag_sec
FROM oceanbase.DBA_OB_ARCHIVELOG
WHERE lag_seconds > 60;
```
> **Action**: Check network to archive destination, disk space, or increase `log_archive_concurrency`.

---

## Stopping and Restarting Archive

```sql
-- Stop log archive (CAUTION: stops point-in-time recovery!)
ALTER SYSTEM CANCEL ARCHIVE LOG;

-- Restart archive
ALTER SYSTEM ARCHIVE LOG;

-- Check status after restart
SELECT status, lag_seconds FROM oceanbase.DBA_OB_ARCHIVELOG;
```

> **Warning**: Stopping log archive breaks the backup chain. Only stop for maintenance, and restart as soon as possible.

---

## Cleaning Up Old Archives

OceanBase does NOT auto-delete archived logs. You must manage retention manually or via OCP.

```sql
-- Via OCP UI: Backup Recovery → Backup Strategy → Retention Policy
-- Set: "Keep backups for N days"

-- Manual cleanup (delete archives older than 7 days)
-- NOTE: Only delete after confirming backups are no longer needed!
ALTER SYSTEM DELETE ARCHIVELOG UNTIL TIME = '2025-01-01 00:00:00';
```

---

## MySQL Mode vs Oracle Mode

| Aspect | MySQL Mode | Oracle Mode |
|--------|-----------|-------------|
| System view | `oceanbase.DBA_OB_ARCHIVELOG` | `SYS.DBA_OB_ARCHIVELOG` |
| Date literal | `'YYYY-MM-DD HH:MM:SS'` | `TO_DATE('YYYY-MM-DD HH24:MI:SS')` |

---

## Common Mistakes

1. **Not starting archive before first backup**
   - ❌ `BACKUP DATABASE` fails with "log archive not enabled"
   - ✅ Always run `ALTER SYSTEM ARCHIVE LOG;` first

2. **Archive destination runs out of space**
   - ❌ Not monitoring archive storage
   - ✅ Set up alert at 80% full on archive destination

3. **Archive lag growing undetected**
   - ❌ Assuming archive is running fine
   - ✅ Monitor `lag_seconds` daily; alert if > 60s

4. **Deleting archives that are still needed for PITR**
   - ❌ Running `DELETE ARCHIVELOG` without checking recovery window
   - ✅ Only delete archives older than your recovery window (e.g., 7 days)

5. **Using local disk for archive in multi-node cluster**
   - ❌ `log_archive_dest = 'file:///local/path'` (only visible to one node)
   - ✅ Use NFS or cloud storage accessible by all OBServers

---

## OceanBase Version Notes

### V4.2 (Baseline)
- Log archive to NFS, OSS, COS, S3 supported
- `DBA_OB_ARCHIVELOG` view available

### V4.4+
- Archive to additional cloud storage (HCS OBS, AWS S3)
- Improved archive performance for high-throughput workloads

---

## Sources
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/log-archive
- https://www.oceanbase.com/docs/oceanbase-database-cn/v4.2.5/log-archive
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/alter-system-archive-log
