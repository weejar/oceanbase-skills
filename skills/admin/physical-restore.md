# Physical Restore and Table-Level Recovery

## Overview

OceanBase supports restoring entire tenants or individual tables from physical backups. Restore uses the `ALTER SYSTEM RESTORE` command and works with both full and incremental backups plus log archives. This guide covers restore procedures for OceanBase V4.2+.

---

## Restore Architecture

```
Backup Data (external storage)
        +
Log Archives (external storage)
        ↓
ALTER SYSTEM RESTORE
        ↓
New Tenant (restored state)
```

> **Prerequisite**: Log archive MUST be available for point-in-time recovery.

---

## Tenant-Level Restore (PITR)

### Step 1: Prepare the Restore Command

```sql
-- Restore tenant to a specific point in time
ALTER SYSTEM RESTORE TENANT restored_tenant
  FROM 'oss://access_id:access_key@endpoint/bucket/path'
  UNTIL TIME = '2025-01-15 14:30:00';
```

> **Tip**: Always restore to a *new* tenant first. Verify data, then switch over if needed.

### Step 2: Monitor Restore Progress

```sql
-- Check restore progress
SELECT job_id,
       tenant_id,
       restore_type,
       round(progress, 2) AS progress_pct,
       status,
       start_timestamp,
       estimated_finish_time
FROM oceanbase.DBA_OB_RESTORE_PROGRESS;

-- Check restore history
SELECT * FROM oceanbase.DBA_OB_RESTORE_HISTORY
ORDER BY start_timestamp DESC
LIMIT 10;
```

### Step 3: Activate the Restored Tenant

```sql
-- After restore completes, activate the tenant
ALTER TENANT restored_tenant STATUS = 'NORMAL';

-- Verify tenant is accessible
SHOW TENANTS;  -- MySQL Mode
-- or
SELECT * FROM oceanbase.DBA_OB_TENANTS;  -- MySQL Mode
```

---

## Table-Level Recovery (V4.2+)

Restore individual tables without restoring the entire tenant.

```sql
-- Restore a single table
ALTER SYSTEM RESTORE TABLE db1.employees, db1.departments
  TO tenant_name
  FROM 'oss://access_id:access_key@endpoint/bucket/path'
  UNTIL TIME = '2025-01-15 14:30:00';

-- Monitor table-level restore
SELECT * FROM oceanbase.DBA_OB_RESTORE_PROGRESS;
```

> **Use case**: Accidental `DROP TABLE` or `DELETE` without `WHERE`.

---

## Cross-Tenant Restore

Restore data from one tenant's backup into a different tenant.

```sql
-- Backup was taken from tenant_A, restore to tenant_B
ALTER SYSTEM RESTORE TENANT tenant_B
  FROM 'oss://...'
  UNTIL TIME = '2025-01-15 14:30:00';
```

> **Note**: Schema must be compatible. The target tenant must have the same table structures.

---

## Restore Parameters

| Parameter | Default | Recommended | Description |
|-----------|---------|-------------|-------------|
| `restore_parallelism` | 1 | 4-8 | Parallel restore workers |
| `restore_dest` | (empty) | (set) | Temporary restore data path |

```sql
-- Increase restore speed
ALTER SYSTEM SET restore_parallelism = 8;
```

---

## Restore Best Practices

1. **Always restore to a new tenant first** — verify before overwriting production
2. **Calculate time needed** — full restore of 10TB can take 2-6 hours
3. **Ensure sufficient disk space** — restore needs space for full data + logs
4. **Test restore monthly** — don't wait for a real disaster to test
5. **Document restore procedure** — have a runbook ready for emergencies

---

## MySQL Mode vs Oracle Mode

| Aspect | MySQL Mode | Oracle Mode |
|--------|-----------|-------------|
| System view | `oceanbase.DBA_OB_RESTORE_PROGRESS` | `SYS.DBA_OB_RESTORE_PROGRESS` |
| Date literal | `'YYYY-MM-DD HH:MM:SS'` | `TO_DATE('YYYY-MM-DD HH24:MI:SS')` |
| Show tenants | `SHOW TENANTS;` | `SELECT * FROM DBA_OB_TENANTS;` |

---

## Common Mistakes

1. **Restoring directly to production tenant**
   - ❌ `RESTORE TENANT prod_tenant ...` (overwrites production!)
   - ✅ `RESTORE TENANT prod_restored ...` then verify, then switch

2. **Log archive not available for PITR**
   - ❌ Archived logs deleted before recovery window
   - ✅ Keep archives for N+1 days beyond backup retention

3. **Insufficient disk space for restore**
   - ❌ Not checking available space before restore
   - ✅ Ensure target tenant has enough resources (check `DBA_OB_UNITS`)

4. **Wrong time zone in UNTIL TIME**
   - ❌ Using local time without specifying time zone
   - ✅ Use UTC or explicitly specify `+08:00`

5. **Not monitoring restore progress**
   - ❌ Starting restore and walking away
   - ✅ Monitor `DBA_OB_RESTORE_PROGRESS` and set an alert

---

## OceanBase Version Notes

### V4.2 (Baseline)
- Tenant-level PITR supported
- Table-level recovery introduced
- Cross-tenant restore supported

### V4.4+
- Faster restore performance (parallelized log apply)
- Restore progress reporting improved

---

## Sources
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/physical-restore
- https://www.oceanbase.com/docs/oceanbase-database-cn/v4.2.5/physical-restore
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/alter-system-restore
