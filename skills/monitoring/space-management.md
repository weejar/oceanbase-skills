# Space Management and Capacity Planning

## Overview

Running out of disk space causes write stalls and potential data loss. This guide covers space monitoring, cleanup, and capacity planning for OceanBase V4.2+.

---

## Space Architecture

```
Total Disk Space
├── Data Files (SSTable, macro blocks)
├── CLog (Redo logs, archived to backup)
├── Log Files (observer.log, rs.log)
└── Temp Files (Sort, hash join)
```

> **Critical**: Monitor all 4 components.

---

## Viewing Space Usage

### MySQL Mode

```sql
-- Space usage per server
SELECT svr_ip,
       round(disk_total / 1024 / 1024 / 1024, 2) AS total_gb,
       round(disk_used / 1024 / 1024 / 1024, 2) AS used_gb,
       round(disk_used / disk_total * 100, 2) AS used_pct
FROM oceanbase.DBA_OB_SERVERS;

-- Space usage per tenant
SELECT t.tenant_name,
       round(sum(u.data_disk_size) / 1024 / 1024 / 1024, 2) AS allocated_gb,
       round(sum(u.data_disk_in_use) / 1024 / 1024 / 1024, 2) AS used_gb
FROM oceanbase.DBA_OB_TENANTS t
JOIN oceanbase.DBA_OB_UNITS u ON t.tenant_id = u.tenant_id
GROUP BY t.tenant_name;
```

### Oracle Mode

```sql
-- Same data, different view names
SELECT * FROM sys.DBA_OB_SERVERS;
SELECT * FROM sys.DBA_OB_TENANTS;
```

---

## Space Alerts

| Usage | Action |
|-------|--------|
| < 70% | Normal |
| 70-80% | Plan capacity expansion |
| 80-90% | Alert! Plan immediate expansion |
| > 90% | Critical! Writes may stall |

```sql
-- Alert: servers with > 80% disk usage
SELECT svr_ip,
       round(disk_used / disk_total * 100, 2) AS used_pct
FROM oceanbase.DBA_OB_SERVERS
WHERE disk_used / disk_total > 0.8;
```

---

## Cleaning Up Space

### 1. Purge Recycle Bin

```sql
-- MySQL Mode
PURGE RECYCLEBIN;

-- View recycle bin
SELECT * FROM oceanbase.RECYCLEBIN;
```

### 2. Drop Unused Indexes

```sql
-- Find unused indexes (MySQL Mode)
SELECT * FROM oceanbase.DBA_OB_INDEX_USAGE
WHERE accesses = 0;
```

### 3. Clean Up Old Partitions

```sql
-- Drop old partitions (if using partitioning)
ALTER TABLE orders DROP PARTITION p2024Q1;
```

### 4. Reduce Log Retention

```sql
-- Reduce log archive retention (free up backup storage)
-- (Configure in OCP backup policy)
```

---

## Capacity Planning

### Formula

```
Required Disk = (Data Size × Replication Factor) / Compression Ratio + 20% buffer

Example:
  Data Size = 10TB
  Replication Factor = 3
  Compression Ratio = 3:1 (OceanBase typical)
  
  Required = (10TB × 3) / 3 + 20% = 12TB
```

### When to Add Storage

```sql
-- Check if unit needs more disk
SELECT u.unit_id,
       u.data_disk_size / 1024 / 1024 / 1024 AS allocated_gb,
       u.data_disk_in_use / 1024 / 1024 / 1024 AS used_gb,
       round(u.data_disk_in_use / u.data_disk_size * 100, 2) AS used_pct
FROM oceanbase.DBA_OB_UNITS;
```

> **Rule**: If `used_pct > 80%`, increase `DATA_DISK_SIZE`.

---

## MySQL Mode vs Oracle Mode

Space management is identical in both modes. Only system view names differ.

---

## Common Mistakes

1. **Not monitoring disk usage**
   - ❌ Writes stall at 100% disk
   - ✅ OCP alert at 80% usage

2. **Underestimating replication factor**
   - ❌ Plan for 10TB but forget ×3 replicas = 30TB
   - ✅ Always account for replication factor

3. **Not planning for data growth**
   - ❌ Buy storage for current data only
   - ✅ Plan for 12-24 months growth

4. **Forgetting backup storage**
   - ❌ Only plan for data disk, not backup
   - ✅ Backup needs 2-3× data size (full + incremental)

5. **Not compressing backups**
   - ❌ Backup uses same size as data
   - ✅ `ALTER SYSTEM SET backup_compression_algorithm='zstd';`

---

## OceanBase Version Notes

### V4.2 (Baseline)
- Space monitoring views
- Data compression (typically 3:1)
- Recycle bin

### V4.4+
- Better compression ratios
- Faster space reclamation after DROP/TRUNCATE

---

## Sources
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/space-management
- https://www.oceanbase.com/docs/oceanbase-database-cn/v4.2.5/space-management
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/capacity-planning
