# Backup and Recovery Overview

## Overview

OceanBase provides built-in physical backup and recovery capabilities without requiring third-party tools like RMAN. Backup operations are managed via `ALTER SYSTEM` SQL commands and monitored through system views. This guide covers backup strategies, configuration, and recovery procedures for OceanBase V4.2+.

OceanBase supports:
- **Physical backup**: Full and incremental data backups + log archives
- **Point-in-Time Recovery (PITR)**: Restore to any timestamp within the backup window
- **Table-level recovery**: Restore individual tables without full cluster restore (V4.2+)
- **Cross-tenant recovery**: Restore data to a different tenant

---

## Backup Architecture

OceanBase backup consists of two independent but related components:

| Component | Description | Storage |
|-----------|-------------|----------|
| **Data Backup** | Full or incremental backup of tablet data | External storage (NFS, OSS, COS, S3) |
| **Log Archive** | Continuous archiving of Clogs (redo logs) | External storage (same or separate) |

> **Important**: Log archive MUST be running before data backup. Without log archives, you cannot recover to a point in time.

### Backup Types

| Type | Command | Size | Speed |
|------|---------|------|-------|
| Full backup | `ALTER SYSTEM BACKUP DATABASE` | Largest | Slowest |
| Incremental backup | `ALTER SYSTEM BACKUP INCREMENTAL` | Small | Fast |
| Log archive | Automatic (once enabled) | Continuous | Real-time |

---

## Configuring Backup Parameters

### Step 1: Set Backup Destination

```sql
-- MySQL Mode
ALTER SYSTEM SET backup_dest = 'oss://access_id:access_key@endpoint/bucket/path?host=host';
-- or NFS
ALTER SYSTEM SET backup_dest = 'file:///data/backup';

-- Verify
SHOW PARAMETERS LIKE 'backup_dest';
```

### Step 2: Enable Log Archive

```sql
-- Start log archiving (required before data backup!)
ALTER SYSTEM ARCHIVE LOG;

-- Check archive status
SELECT * FROM oceanbase.DBA_OB_ARCHIVELOG;
```

### Step 3: Configure Backup Parameters

```sql
-- Set backup concurrency (default: 1)
ALTER SYSTEM SET backup_parallelism = 4;

-- Set log archive concurrency
ALTER SYSTEM SET log_archive_concurrency = 4;

-- Set backup compression (zstd recommended)
ALTER SYSTEM SET backup_compression_algorithm = 'zstd';
```

---

## Running Backups

### Full Backup

```sql
-- Trigger a full backup
ALTER SYSTEM BACKUP DATABASE;

-- Monitor backup progress
SELECT * FROM oceanbase.DBA_OB_BACKUP_PROGRESS;

-- View backup history
SELECT * FROM oceanbase.DBA_OB_BACKUP_JOBS ORDER BY START_TIMESTAMP DESC LIMIT 10;
```

### Incremental Backup

```sql
-- Only backs up macro blocks that have changed since last backup
ALTER SYSTEM BACKUP INCREMENTAL DATABASE;

-- Monitor same views as full backup
SELECT * FROM oceanbase.DBA_OB_BACKUP_PROGRESS;
```

### Scheduled Backup (via OCP)

OCP (OceanBase Cloud Platform) provides built-in backup scheduling UI:
1. Navigate to **Backup Recovery** → **Backup Strategy**
2. Configure full backup schedule (recommended: daily or weekly)
3. Configure incremental backup schedule (recommended: every 6-12 hours)
4. Set retention policy (recommended: 7-30 days)

---

## Recovery Procedures

### Point-in-Time Recovery (PITR)

```sql
-- Step 1: Stop the tenant (if running)
ALTER TENANT tenant_name STATUS = 'STOPPED';

-- Step 2: Restore to a specific timestamp
ALTER SYSTEM RESTORE TENANT tenant_name FROM 'oss://...'
  UNTIL TIME = '2025-01-15 14:30:00';

-- Step 3: Monitor restore progress
SELECT * FROM oceanbase.DBA_OB_RESTORE_PROGRESS;

-- Step 4: Activate the tenant after restore completes
ALTER TENANT tenant_name STATUS = 'NORMAL';
```

### Table-Level Recovery (V4.2+)

```sql
-- Restore a single table from backup without full tenant restore
ALTER SYSTEM RESTORE TABLE db1.tbl1 TO tenant_name
  FROM 'oss://...'
  UNTIL TIME = '2025-01-15 14:30:00';
```

### Restoring to a New Tenant

```sql
-- Create a new tenant for restore (don't overwrite production!)
CREATE TENANT restored_tenant ...;

-- Restore to the new tenant
ALTER SYSTEM RESTORE TENANT restored_tenant FROM 'oss://...'
  UNTIL TIME = '2025-01-15 14:30:00';
```

---

## Monitoring Backup & Recovery

### Key System Views

| View | Purpose |
|------|---------|
| `DBA_OB_BACKUP_JOBS` | History of backup jobs |
| `DBA_OB_BACKUP_PROGRESS` | Currently running backup |
| `DBA_OB_ARCHIVELOG` | Log archive status |
| `DBA_OB_RESTORE_PROGRESS` | Currently running restore |
| `DBA_OB_RESTORE_HISTORY` | Restore history |

### Check Log Archive Lag

```sql
-- Critical: log archive lag should be < 5 seconds
SELECT tenant_id,
       destination,
       round(lag_seconds, 2) AS lag_sec
FROM oceanbase.DBA_OB_ARCHIVELOG
WHERE status = 'DOING';


```

## Backup & Restore with OBD tool

### Set Backup Config

```bash
obd cluster tenant set-backup-config <deploy_name> <tenant_name> -d <data_uri> -a <log_uri>
```

### Run Backup

```bash
obd cluster tenant backup <deploy_name> <tenant_name>
```

### Restore from Backup

```bash
obd cluster tenant restore <deploy_name> <tenant_name> <data_uri> <log_uri>
```

See [references/backup-restore.md](references/backup-restore.md) for detailed backup and restore procedures.

---

## MySQL Mode vs Oracle Mode

| Aspect | MySQL Mode | Oracle Mode |
|--------|-----------|-------------|
| System views | `oceanbase.DBA_OB_*` | `SYS.DBA_OB_*` |
| Date literals | `'YYYY-MM-DD HH:MM:SS'` | `TO_DATE('YYYY-MM-DD HH24:MI:SS')` |
| Parameter setting | `ALTER SYSTEM SET` | Same |

---

## Common Mistakes

1. **Not enabling log archive before backup**
   - ❌ Running `BACKUP DATABASE` without `ARCHIVE LOG`
   - ✅ Always run `ALTER SYSTEM ARCHIVE LOG;` first

2. **Backup destination not accessible by all servers**
   - ❌ Using local disk path only visible to one OBServer
   - ✅ Use shared storage (NFS) or cloud storage (OSS/S3)

3. **Insufficient backup destination space**
   - ❌ Not monitoring backup storage usage
   - ✅ Set up alerts on backup_dest usage at 80% full

4. **Not testing recovery regularly**
   - ❌ Assuming backup works without testing
   - ✅ Perform a test restore to a staging tenant monthly

5. **Wrong UNTIL TIME during PITR**
   - ❌ Using local time without time zone consideration
   - ✅ Always use UTC or explicitly specify time zone

---

## OceanBase Version Notes

### V4.2 (Baseline)
- Physical backup and PITR supported
- Table-level recovery introduced
- Cross-tenant recovery supported

### V4.4+
- Backup to additional cloud storage types (COS, OBS)
- Improved backup compression ratios (zstd level configurable)
- Faster incremental backup (reduced overhead)

---

## Sources
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/backup-and-recovery-1
- https://www.oceanbase.com/docs/oceanbase-database-cn/v4.2.5/backup-and-recovery-1
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/alter-system-backup
