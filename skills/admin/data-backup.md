# Data Backup Operations

## Overview

OceanBase supports full and incremental data backups using `ALTER SYSTEM BACKUP` commands. Backups are stored on external storage (NFS, OSS, COS, S3) and work with log archives to enable point-in-time recovery. This guide covers backup operations, monitoring, and best practices for OceanBase V4.2+.

---

## Backup Types

| Type | Command | What It Backs Up | Size | Frequency |
|------|---------|-------------------|------|------------|
| Full | `BACKUP DATABASE` | All macro blocks | Largest | Weekly (recommended) |
| Incremental | `BACKUP INCREMENTAL DATABASE` | Changed macro blocks only | Small | Every 6-12 hours |
| Log Archive | Auto (after `ARCHIVE LOG`) | CLog records continuously | Continuous | Always running |

> **Rule**: Incremental backup is only valid if log archive is running. Without logs, you cannot recover the incremental backup.

---

## Running a Full Backup

### Trigger Full Backup

```sql
-- Trigger full backup (run from sys tenant)
ALTER SYSTEM BACKUP DATABASE;

-- For a specific tenant (Oracle Mode)
ALTER SYSTEM BACKUP TENANT tenant_name DATABASE;
```

### Monitor Backup Progress

```sql
-- Check running backup jobs
SELECT job_id,
       backup_type,
       status,
       round(progress, 2) AS progress_pct,
       start_timestamp,
       estimated_finish_time
FROM oceanbase.DBA_OB_BACKUP_PROGRESS;

-- Check backup history
SELECT job_id,
       backup_type,
       status,
       start_timestamp,
       end_timestamp,
       result
FROM oceanbase.DBA_OB_BACKUP_JOBS
ORDER BY start_timestamp DESC
LIMIT 20;
```

---

## Running an Incremental Backup

```sql
-- Only backs up macro blocks changed since last backup
ALTER SYSTEM BACKUP INCREMENTAL DATABASE;

-- Monitor same views
SELECT * FROM oceanbase.DBA_OB_BACKUP_PROGRESS;
```

> **Tip**: Incremental backups are much faster and smaller. Schedule them every 6-12 hours between full backups.

---

## Backup Parameters

| Parameter | Default | Recommended | Description |
|-----------|---------|-------------|-------------|
| `backup_parallelism` | 1 | 4-8 | Parallel backup workers |
| `backup_compression_algorithm` | `none` | `zstd` | Compression for backup files |
| `backup_dest` | (empty) | (set) | Backup destination URL |
| `log_archive_dest` | (empty) | (set) | Log archive destination |

```sql
-- Recommended settings
ALTER SYSTEM SET backup_parallelism = 8;
ALTER SYSTEM SET backup_compression_algorithm = 'zstd';
```

---

## Viewing Backup Metadata

### List All Backups

```sql
-- Available backups for recovery
SELECT backup_type,
       start_timestamp,
       end_timestamp,
       round(backup_size / 1024 / 1024 / 1024, 2) AS size_gb,
       result
FROM oceanbase.DBA_OB_BACKUP_SET_DETAIL
ORDER BY start_timestamp DESC;
```

### Check Backup Destination Usage

```sql
-- Check how much space backups are using
-- (Run on backup destination server)
du -sh /data/backup/*

-- Or check via SQL (if using OSS/S3, check cloud console)
```

---

## Backup Scheduling via OCP

OCP provides a UI for backup scheduling:

1. **Navigate**: OCP → Cluster → Backup Recovery → Backup Strategy
2. **Full backup**: Set schedule (e.g., weekly on Sunday 02:00)
3. **Incremental backup**: Set schedule (e.g., daily every 6 hours)
4. **Retention**: Set retention period (e.g., keep 7 days)
5. **Enable**: Activate the backup strategy

---

## Backup Best Practices

1. **Always enable log archive first** — without it, backups are useless for PITR
2. **Use compression** — `zstd` reduces backup size by 60-80%
3. **Schedule full backups weekly** — incremental daily or every 6 hours
4. **Monitor backup duration** — if full backup takes > 24h, increase `backup_parallelism`
5. **Test recovery monthly** — perform a test restore to a staging tenant
6. **Use separate storage for backup and archive** — avoid single point of failure

---

## MySQL Mode vs Oracle Mode

| Aspect | MySQL Mode | Oracle Mode |
|--------|-----------|-------------|
| System views | `oceanbase.DBA_OB_*` | `SYS.DBA_OB_*` |
| Tenant backup | `BACKUP DATABASE` (all tenants) | `BACKUP TENANT name DATABASE` |

---

## Common Mistakes

1. **Running backup without log archive**
   - ❌ `BACKUP DATABASE` but `log_archive_dest` is empty
   - ✅ Always run `ALTER SYSTEM ARCHIVE LOG;` and verify before backup

2. **Backup destination not shared across all servers**
   - ❌ `backup_dest = 'file:///local/path'` on one node
   - ✅ Use NFS or cloud storage accessible by all OBServers

3. **Not monitoring backup progress**
   - ❌ Assuming backup completed successfully
   - ✅ Check `DBA_OB_BACKUP_PROGRESS` and `DBA_OB_BACKUP_JOBS`

4. **Backup destination runs out of space mid-backup**
   - ❌ Not monitoring destination disk usage
   - ✅ Alert at 80% full; schedule cleanup of old backups

5. **Using default `backup_parallelism=1` on large clusters**
   - ❌ Single-threaded backup on a 100TB cluster
   - ✅ Set `backup_parallelism = 4-8` for faster backups

---

## OceanBase Version Notes

### V4.2 (Baseline)
- Full and incremental backup supported
- Compression with `zstd`, `lz4`, `snappy`
- Backup to NFS, OSS, COS, S3

### V4.4+
- Faster incremental backup (reduced overhead)
- Configurable compression level for `zstd`
- Backup to additional cloud storage (HCS OBS)

---

## Sources
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/data-backup
- https://www.oceanbase.com/docs/oceanbase-database-cn/v4.2.5/data-backup
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/alter-system-backup
