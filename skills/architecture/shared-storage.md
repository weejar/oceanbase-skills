# Shared Storage Architecture (V4.x)

## Overview

OceanBase V4.x introduced **shared storage support** (NFS, cloud storage) for specific use cases, reducing storage costs. This guide explains shared storage architecture, configuration, and limitations for OceanBase V4.2+.

> **Note**: Shared storage is NOT required for standard OceanBase deployments (shared-nothing is the default).

---

## Architecture

### Shared-Nothing (Default)

```
Server-1 (Local Disk)    Server-2 (Local Disk)    Server-3 (Local Disk)
     │                          │                          │
     └──────────────────────────┴──────────────────────────┘
                       3 copies (each server full data)
```

### Shared Storage (Optional, V4.x)

```
Server-1 ─┐
Server-2 ─┼──→ Shared Storage (NFS / OSS / S3)
Server-3 ─┘   (1 copy + 2 log replicas)
```

> **Benefit**: Reduces storage cost (1 data copy + 2 log replicas instead of 3 full copies).

---

## When to Use Shared Storage

| Use Case | Recommendation |
|----------|-----------------|
| OLTP (high write) | Shared-nothing (default) |
| OLAP (read-heavy) | Shared storage OK |
| Cost-sensitive | Shared storage (lower storage cost) |
| High availability | Shared-nothing (no shared storage SPOF) |

> **Rule**: If storage cost > network cost, consider shared storage.

---

## Configuring Shared Storage

### Step 1: Mount Shared Storage on All Servers

```bash
# On all OBServers, mount NFS
mount -t nfs 10.0.0.100:/export/ob_data /data/ob_shared

# Verify
df -h | grep ob_shared
```

### Step 2: Configure OceanBase to Use Shared Storage

```sql
-- Set data directory to shared storage
ALTER SYSTEM SET data_dir = '/data/ob_shared/observer1';

-- Set log directory (can be local or shared)
ALTER SYSTEM SET redo_dir = '/data/ob_redo';
```

### Step 3: Create Tenant with Shared Storage

```sql
-- Create unit with shared storage path
CREATE RESOURCE UNIT unit_shared
  MAX_CPU 4,
  MEMORY_SIZE '8G',
  DATA_DISK_SIZE '100G',
  LOG_DISK_SIZE '16G';

-- Create resource pool
CREATE RESOURCE POOL pool_shared
  UNIT = 'unit_shared',
  UNIT_NUM = 1,
  ZONE_LIST = ('zone1', 'zone2', 'zone3');
```

---

## Shared Storage Types Supported

| Type | Configuration | Use Case |
|------|---------------|----------|
| NFS | `data_dir = 'file:///nfs/path'` | On-premise |
| OSS (Alibaba Cloud) | `data_dir = 'oss://...'` | Cloud deployment |
| S3 (AWS) | `data_dir = 's3://...'` | Multi-cloud |
| COS (Tencent Cloud) | `data_dir = 'cos://...'` | Tencent Cloud |

---

## Limitations

1. **Performance**: Shared storage has higher latency than local SSD
2. **Single Point of Failure**: If shared storage fails, all servers lose data access
3. **Not for write-heavy workloads**: Use shared-nothing for OLTP
4. **Backup still needed**: Shared storage doesn't eliminate backup requirement

---

## Best Practices

1. **Use shared-nothing for production OLTP** — best performance and HA
2. **Use shared storage for OLAP/HTAP** — cost-effective for read-heavy
3. **Always have 3 log replicas** — even with shared storage
4. **Monitor shared storage IOPS** — can become bottleneck
5. **Test failover** — shared storage failure scenario

---

## MySQL Mode vs Oracle Mode

Shared storage configuration is identical in both modes.

---

## Common Mistakes

1. **Using shared storage for write-heavy OLTP**
   - ❌ High write latency
   - ✅ Use shared-nothing (local SSD) for OLTP

2. **Single NFS server (no HA for shared storage)**
   - ❌ NFS server failure = entire cluster down
   - ✅ Use HA NFS (e.g., NAS with RAID)

3. **Not monitoring shared storage performance**
   - ❌ IOPS saturation undetected
   - ✅ Monitor IOPS and latency on shared storage

4. **Assuming shared storage = backup**
   - ❌ Shared storage doesn't protect against logical errors
   - ✅ Still need regular backups

5. **Mixing shared and non-shared storage in same tenant**
   - ❌ Complex to manage
   - ✅ Use consistent storage type per tenant

---

## OceanBase Version Notes

### V4.2 (Baseline)
- Shared storage support introduced (NFS, OSS)
- Log replica separation from data

### V4.4+
- S3 and COS support added
- Improved shared storage performance

---

## Sources
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/shared-storage
- https://www.oceanbase.com/docs/oceanbase-database-cn/v4.2.5/shared-storage
