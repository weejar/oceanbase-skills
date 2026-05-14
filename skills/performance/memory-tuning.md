# Memory Tuning

## Overview

OceanBase uses **MemStore** (in-memory store) for write operations and **KVCache** for read caching. Proper memory sizing is critical for performance. This guide covers memory parameters and tuning for OceanBase V4.2+.

---

## Memory Architecture

```
Total Memory (MEMORY_SIZE)
├── MemStore (58%) — Write buffer
├── KVCache (32%) — Read cache
└── Other (10%) — SQL worker, network, etc.
```

> **Key**: If MemStore fills up, writes stall. If KVCache is too small, reads hit disk.

---

## Key Memory Parameters

| Parameter | Default | Recommended | Description |
|-----------|---------|-------------|-------------|
| `memory_limit` | 80% of RAM | 80% of RAM | Total memory for observer |
| `memstore_limit_percentage` | 58% | 50-60% | MemStore as % of memory_limit |
| `freeze_trigger_percentage` | 70% | 60-70% | When to trigger MemStore flush |
| `kv_cache_size_percentage` | 32% | 30-40% | KVCache as % of memory_limit |

```sql
-- View current settings
SHOW PARAMETERS LIKE '%memory%';
SHOW PARAMETERS LIKE '%memstore%';
SHOW PARAMETERS LIKE '%kv_cache%';
```

---

## Tuning MemStore

### Check MemStore Usage

```sql
-- MySQL Mode
SELECT tenant_id,
       round(active_memstore / 1024 / 1024 / 1024, 2) AS active_gb,
       round(total_memstore / 1024 / 1024 / 1024, 2) AS total_gb
FROM oceanbase.DBA_OB_MEMSTORE;

-- Oracle Mode
SELECT * FROM sys.DBA_OB_MEMSTORE;
```

### If MemStore Fills Up (Write Stall)

```sql
-- Manually trigger MemStore flush (if needed)
ALTER SYSTEM MINOR FREEZE;

-- Increase MemStore percentage
ALTER SYSTEM SET memstore_limit_percentage = 65;  -- was 58
```

---

## Tuning KVCache

### Check KVCache Hit Rate

```sql
-- MySQL Mode
SELECT round(hit_rate, 2) AS kv_hit_rate
FROM oceanbase.DBA_OB_KV_CACHE;
```

> **Target**: KVCache hit rate > 95%. If lower, increase `kv_cache_size_percentage`.

### Increase KVCache

```sql
ALTER SYSTEM SET kv_cache_size_percentage = 40;  -- was 32
```

---

## Tenant Memory Sizing

When creating a tenant, right-size `MEMORY_SIZE`:

```sql
-- Create unit with appropriate memory
CREATE RESOURCE UNIT unit_8c16g
  MAX_CPU 8,
  MEMORY_SIZE '16G',       -- Total memory for tenant
  LOG_DISK_SIZE '32G',
  DATA_DISK_SIZE '100G';
```

> **Rule**: `MEMORY_SIZE` should be 60-70% of total tenant RAM needed.

---

## MySQL Mode vs Oracle Mode

Memory architecture is identical in both modes. Only system view names differ.

| Aspect | MySQL Mode | Oracle Mode |
|--------|-----------|-------------|
| MemStore view | `oceanbase.DBA_OB_MEMSTORE` | `sys.DBA_OB_MEMSTORE` |
| KVCache view | `oceanbase.DBA_OB_KV_CACHE` | `sys.DBA_OB_KV_CACHE` |

---

## Common Mistakes

1. **`memory_limit` set too high (OOM risk)**
   - ❌ `memory_limit = '90%'` (OS may OOM)
   - ✅ `memory_limit = '80%'` (leave 20% for OS)

2. **MemStore too small (write stall)**
   - ❌ `memstore_limit_percentage = 30` (fills up fast)
   - ✅ `memstore_limit_percentage = 58` (default, adjust based on workload)

3. **KVCache too small (read I/O)**
   - ❌ `kv_cache_size_percentage = 10` (lots of disk read)
   - ✅ `kv_cache_size_percentage = 32` (default, increase if hit rate < 95%)

4. **Not monitoring memory usage**
   - ❌ Only notice when write stalls
   - ✅ OCP alert at 80% MemStore usage

5. **UNIT `MEMORY_SIZE` too small for tenant**
   - ❌ `MEMORY_SIZE = '4G'` for production workload
   - ✅ Monitor and scale: `ALTER RESOURCE UNIT ... MEMORY_SIZE='16G'`

---

## OceanBase Version Notes

### V4.2 (Baseline)
- MemStore + KVCache architecture
- Auto MemStore flush (freeze/merge)
- Memory monitoring views

### V4.4+
- Improved KVCache eviction algorithm
- Better memory sizing recommendations in OCP

---

## Sources
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/memory-management
- https://www.oceanbase.com/docs/oceanbase-database-cn/v4.2.5/memory-management
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/memstore
