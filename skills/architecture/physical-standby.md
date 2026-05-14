# Physical Standby and Disaster Recovery

## Overview

OceanBase supports **physical standby** for disaster recovery (DR). A standby cluster replicates data from the primary cluster in real-time. This guide covers standby cluster setup, role switchover, and DR best practices for OceanBase V4.2+.

---

## Architecture

```
Primary Cluster (Zone1, Zone2, Zone3)
        ↓ (Log Archive → Replication)
Standby Cluster (Zone4, Zone5, Zone6)
```

> **Use Case**: Disaster recovery, regional HA, read-only reporting (with limitations).

---

## Standby Cluster Types

| Type | Description |
|------|-------------|
| **Physical Standby** | Exact replica of primary (recommended for DR) |
| **Logical Standby** | (Not supported in OceanBase; use OMS for logical replication) |

---

## Setting Up Physical Standby

### Prerequisites

1. Primary cluster has log archive enabled
2. Network connectivity between primary and standby
3. Standby cluster installed and empty

### Step 1: Configure Primary for Standby

```sql
-- On primary cluster, ensure log archive is running
ALTER SYSTEM ARCHIVE LOG;
SELECT * FROM oceanbase.DBA_OB_ARCHIVELOG;

-- Configure archival to shared storage (accessible by standby)
ALTER SYSTEM SET log_archive_dest = 'oss://.../primary_archive';
```

### Step 2: Restore Standby from Primary Backup

```sql
-- On standby cluster, restore from primary's backup
ALTER SYSTEM RESTORE TENANT standby_tenant
  FROM 'oss://.../primary_backup'
  UNTIL TIME = 'MAX';
-- 'MAX' means restore to latest available point
```

### Step 3: Start Real-Time Replication

```sql
-- On standby cluster, start applying archived logs
ALTER SYSTEM START REPLAY ARCHIVE LOG;
```

---

## Managing Standby Cluster

### Check Replication Status

```sql
-- On standby: check replay progress
SELECT * FROM oceanbase.DBA_OB_ARCHIVELOG_REPLAY_STATUS;

-- Check lag (should be < 5 seconds for healthy DR)
SELECT round(lag_seconds, 2) AS lag_sec
FROM oceanbase.DBA_OB_ARCHIVELOG_REPLAY_STATUS;
```

### Stop/Start Replication

```sql
-- Stop replay (maintenance)
ALTER SYSTEM STOP REPLAY ARCHIVE LOG;

-- Resume replay
ALTER SYSTEM START REPLAY ARCHIVE LOG;
```

---

## Switchover (Planned)

Switchover promotes standby to primary (no data loss).

```sql
-- Step 1: On primary, switch to standby role
ALTER SYSTEM SWITCH TO STANDBY;

-- Step 2: On standby, switch to primary role
ALTER SYSTEM SWITCH TO PRIMARY;
```

> **Tip**: Always test switchover during maintenance window.

---

## Failover (Disaster)

Failover promotes standby when primary is unrecoverable (may lose data after last archived log).

```sql
-- On standby, force promote to primary
ALTER SYSTEM ACTIVATE STANDBY CLUSTER;
```

> **Warning**: Only use failover when primary is completely lost.

---

## Best Practices

1. **Monitor replication lag** — alert if > 60 seconds
2. **Test switchover quarterly** — don't wait for real disaster
3. **Separate network for replication** — avoid congestion with client traffic
4. **Use 3 zones for standby** — same as primary
5. **Document DR procedure** — runbook for failover/switchover

---

## MySQL Mode vs Oracle Mode

| Aspect | MySQL Mode | Oracle Mode |
|--------|-----------|-------------|
| System views | `oceanbase.DBA_OB_*` | `SYS.DBA_OB_*` |
| Switchover command | Same | Same |

---

## Common Mistakes

1. **Standby not in same zone configuration as primary**
   - ❌ Primary has 3 zones, standby has 1 zone
   - ✅ Standby should match primary's zone count

2. **Not monitoring replication lag**
   - ❌ Lag grows to hours undetected
   - ✅ Alert at > 60s lag

3. **Testing failover on production without runbook**
   - ❌ Surprise during real disaster
   - ✅ Test switchover quarterly with documented procedure

4. **Forgetting to rebuild standby after primary reinstall**
   - ❌ Standby points to old primary
   - ✅ Rebuild standby after major primary changes

5. **Network bottleneck between primary and standby**
   - ❌ Replication can't keep up
   - ✅ Use dedicated link; monitor throughput

---

## OceanBase Version Notes

### V4.2 (Baseline)
- Physical standby supported
- Real-time log replay
- Switchover and failover

### V4.4+
- Faster replay performance
- Improved lag monitoring

---

## Sources
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/physical-standby
- https://www.oceanbase.com/docs/oceanbase-database-cn/v4.2.5/physical-standby
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/disaster-recovery
