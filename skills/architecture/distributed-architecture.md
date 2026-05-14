# Distributed Architecture Overview

## Overview

OceanBase is a **shared-nothing distributed database** that uses the Paxos consensus protocol to replicate data across multiple servers (zones). This guide explains the core distributed architecture concepts: Zone, node, partition, log stream, and how data is distributed and replicated in OceanBase V4.2+.

---

## Architecture Diagram (Conceptual)

```
┌─────────────────────────────────────────────────────┐
│                    OBProxy (Query Router)            │
└──────────────┬──────────────────┬──────────────────┘
               │                  │
        ┌──────▼──────┐   ┌──────▼──────┐
        │   Zone-1     │   │   Zone-2     │   │   Zone-3     │
        │  OBServer-1  │   │  OBServer-2  │   │  OBServer-3  │
        │  (Leader)    │   │  (Follower)  │   │  (Follower)  │
        └──────────────┘   └──────────────┘   └──────────────┘
              │                  │                  │
              └──────────────────┴──────────────────┘
                        Paxos Log Replication
```

---

## Core Concepts

### Zone

A **Zone** is a failure domain (like a rack, server room, or data center). OceanBase distributes replicas across zones for high availability.

```sql
-- View zones
SELECT * FROM oceanbase.DBA_OB_ZONES;

-- Add a zone
ALTER SYSTEM ADD ZONE zone4;
```

> **Best Practice**: Use 3 zones for production (ensures Paxos majority quorum).

---

### OBServer (Node)

An **OBServer** is a physical or virtual server running the OceanBase process (`observer`). Each OBServer hosts multiple **partitions** (tablets).

```sql
-- View OBServers
SELECT svr_ip, svr_port, zone, status
FROM oceanbase.DBA_OB_SERVERS;
```

---

### Partition (Tablet)

OceanBase **automatically partitions** large tables for distributed storage. Each partition is a **tablet** (shard) managed by a specific OBServer.

```sql
-- View partition distribution for a table
SELECT * FROM oceanbase.DBA_OB_TABLE_LOCATIONS
WHERE table_name = 'employees';
```

| Partition Type | Use Case |
|---------------|----------|
| Hash | Even distribution, most OLTP workloads |
| Range | Time-series data, range queries |
| List | Categorical data |
| Composite | Two-level partitioning (e.g., Range + Hash) |

---

### Log Stream (LS)

A **Log Stream** is the replication pipeline for a partition's redo logs. All replicas of a partition share one log stream.

```
Partition P1 (Leader on S1) → Log Stream → Replicas (S2, S3)
```

> **Key Point**: The **Leader** replica handles all reads and writes. Follower replicas are for HA only (reads go to leader by default).

---

### Paxos Consensus

OceanBase uses **Paxos** to ensure data consistency across replicas:

1. Leader receives write → writes to Clog
2. Leader sends Clog to followers
3. Majority (≥ 2/3) acknowledge → commit
4. Leader responds to client

> **Majority rule**: With 3 replicas, the cluster tolerates 1 node failure.

---

## Data Distribution

### Partition Location

```sql
-- Find which server hosts the leader for a table
SELECT table_name,
       partition_id,
       svr_ip,
       role  -- LEADER or FOLLOWER
FROM oceanbase.DBA_OB_TABLE_LOCATIONS
WHERE table_name = 'orders'
  AND role = 'LEADER';
```

### Data Replica Types

| Type | Description |
|------|-------------|
| **Full Replica** | Complete data copy (can be leader) |
| **Log Replica** | Logs only (no data scan) |
| **Columnar Replica** (V4.2+) | Columnar format for analytical queries (HTAP) |

```sql
-- View replica distribution
SELECT table_name, partition_id, svr_ip, replica_type
FROM oceanbase.DBA_OB_TABLE_LOCATIONS;
```

---

## Replication Factor

Set the number of replicas per tenant:

```sql
-- 3 replicas (recommended for production)
CREATE RESOURCE POOL pool_3replica
  UNIT = 'unit_2c4g',
  UNIT_NUM = 1,
  ZONE_LIST = ('zone1', 'zone2', 'zone3');

-- 2 replicas (for dev/test only)
CREATE RESOURCE POOL pool_2replica
  UNIT_NUM = 1,
  ZONE_LIST = ('zone1', 'zone2');
```

> **Warning**: 2 replicas cannot tolerate ANY node failure (no majority).

---

## MySQL Mode vs Oracle Mode

Architecture is identical in both modes. Only SQL dialect and system view names differ.

| Aspect | MySQL Mode | Oracle Mode |
|--------|-----------|-------------|
| System views | `oceanbase.DBA_OB_*` | `SYS.DBA_OB_*` |
| Connection | MySQL protocol | MySQL protocol (Oracle dialect) |

---

## Common Mistakes

1. **Using 1 or 2 zones in production**
   - ❌ `ZONE_LIST=('zone1')` — no HA
   - ✅ `ZONE_LIST=('zone1','zone2','zone3')` — tolerates 1 failure

2. **All replicas land on same zone**
   - ❌ `ZONE_LIST=('zone1','zone1','zone1')` (same zone, no HA)
   - ✅ `ZONE_LIST=('zone1','zone2','zone3')`

3. **Leader and follower on same server**
   - ❌ Small cluster with 1 OBServer per zone but zone has 2 OBServers
   - ✅ Ensure `DBA_OB_TABLE_LOCATIONS` shows even distribution

4. **Not monitoring leader distribution**
   - ❌ All leaders on one server (hot spot)
   - ✅ Use OCP to rebalance leader distribution

5. **Confusing replica type**
   - ❌ Expecting follower to serve reads (default: leader only)
   - ✅ Enable weak consistency read: `SET ob_read_consistency='WEAK'`

---

## OceanBase Version Notes

### V4.2 (Baseline)
- Paxos-based replication
- Log Stream architecture
- Columnar replica (HTAP)

### V4.4+
- Faster leader switchover
- Improved log stream concurrency

---

## Sources
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/distributed-architecture
- https://www.oceanbase.com/docs/oceanbase-database-cn/v4.2.5/distributed-architecture
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/paxos
