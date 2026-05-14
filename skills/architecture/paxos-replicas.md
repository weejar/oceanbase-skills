# Paxos Protocol and Replica Management

## Overview

OceanBase uses the **Paxos consensus protocol** to replicate data across zones. This ensures strong data consistency and automatic failover. This guide explains Paxos in OceanBase, replica types, and how to manage replicas in V4.2+.

---

## Paxos Protocol (Simplified)

```
Leader Replica (Zone-1)
    │
    ├── Send Clog → Follower-1 (Zone-2)
    └── Send Clog → Follower-2 (Zone-3)

Commit when majority (≥2/3) acknowledge.
```

1. Client sends write to **Leader**
2. Leader writes to **Clog** (redo log)
3. Leader sends Clog to **Followers**
4. Majority (≥ N/2+1) acknowledge → **Commit**
5. Leader responds to client

> **Majority rule**: With 3 replicas, cluster tolerates 1 node failure.

---

## Replica Roles

| Role | Description | Can Be Leader? |
|------|-------------|-----------------|
| **Leader** | Handles reads + writes | Yes |
| **Follower** | Receives replicated logs | No (default) |
| **Learner** | Receives logs, not counted in majority | No |

```sql
-- View replica roles
SELECT table_name, partition_id, svr_ip, role
FROM oceanbase.DBA_OB_TABLE_LOCATIONS
WHERE role != 'FOLLOWER';
```

---

## Replica Types

| Type | Description | Use Case |
|------|-------------|----------|
| **Full Replica** | Complete data copy | OLTP (read + write) |
| **Log Replica** | Logs only (no data scan) | Reduce storage cost |
| **Columnar Replica** (V4.2+) | Columnar format | HTAP (analytical queries) |

```sql
-- View replica types
SELECT table_name, svr_ip, replica_type
FROM oceanbase.DBA_OB_TABLE_LOCATIONS;
```

---

## Managing Replicas

### View Replica Distribution

```sql
-- Check if replicas are evenly distributed
SELECT svr_ip, count(*) AS replica_count
FROM oceanbase.DBA_OB_TABLE_LOCATIONS
GROUP BY svr_ip;
```

### Rebalance Replicas

```sql
-- Trigger automatic rebalance (usually not needed; OceanBase auto-rebalances)
ALTER SYSTEM REBALANCE TENANT tenant_name;
```

### Change Replica Type (V4.2+)

```sql
-- Add a columnar replica for HTAP workload
ALTER TABLE orders SET REPLICA_NUM = 4;  -- 3 full + 1 columnar
```

---

## Leader Election and Failover

### View Leader Distribution

```sql
-- Check leader distribution across servers
SELECT svr_ip, count(*) AS leader_count
FROM oceanbase.DBA_OB_TABLE_LOCATIONS
WHERE role = 'LEADER'
GROUP BY svr_ip;
```

### Manual Leader Switch (for maintenance)

```sql
-- Switch leader away from a server (maintenance)
ALTER SYSTEM SWITCH REPLICA LEADER TO energy='zone2';
```

> **Tip**: OceanBase automatically fails over if a leader node crashes (usually < 2 seconds).

---

## MySQL Mode vs Oracle Mode

Paxos protocol is identical in both modes. Only system view names differ.

| Aspect | MySQL Mode | Oracle Mode |
|--------|-----------|-------------|
| System view | `oceanbase.DBA_OB_TABLE_LOCATIONS` | `SYS.DBA_OB_TABLE_LOCATIONS` |

---

## Common Mistakes

1. **2 replicas in production**
   - ❌ Can't tolerate ANY node failure (no majority)
   - ✅ Always use 3 replicas (`UNIT_NUM=1`, 3 zones)

2. **All replicas in same zone**
   - ❌ `ZONE_LIST=('zone1','zone1','zone1')` (no HA)
   - ✅ `ZONE_LIST=('zone1','zone2','zone3')`

3. **Uneven leader distribution**
   - ❌ All leaders on one server (hot spot)
   - ✅ Use OCP to rebalance leader distribution

4. **Not monitoring replica status**
   - ❌ Replica down for hours undetected
   - ✅ OCP alert on replica failure

5. **Confusing Full Replica with Columnar Replica**
   - ❌ Expecting columnar replica to handle writes
   - ✅ Columnar replica is read-only (HTAP only)

---

## OceanBase Version Notes

### V4.2 (Baseline)
- Paxos-based replication
- Full, Log, Columnar replica types
- Automatic leader election

### V4.4+
- Faster leader switchover (< 1s)
- Improved log replication throughput

---

## Sources
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/paxos
- https://www.oceanbase.com/docs/oceanbase-database-cn/v4.2.5/paxos
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/replica-management
