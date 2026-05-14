# Resource Isolation (cgroup)

## Overview

OceanBase uses **cgroup** (Linux Control Groups) to isolate CPU and memory resources between tenants and between OceanBase and OS. This guide covers resource isolation configuration and monitoring for OceanBase V4.2+.

---

## Architecture

```
Physical Server (128 CPU, 512GB RAM)
├── cgroup: observer (OceanBase process)
│   ├── cpu_subbsystem (limit: 80 CPU)
│   └── memory_subbsystem (limit: 400GB)
└── cgroup: system (OS, other processes)
    ├── cpu_subbsystem (remaining CPU)
    └── memory_subbsystem (remaining RAM)
```

> **Goal**: Prevent OceanBase from using all CPU/memory and crashing the OS.

---

## Configuring Resource Isolation

### CPU Isolation

```sql
-- Set CPU limit for observer process
ALTER SYSTEM SET cpu_quota_concurrency = 80;  -- 80% of total CPU

-- Set tenant CPU (within observer limit)
ALTER RESOURCE UNIT unit_4c8g MAX_CPU = 4;
```

### Memory Isolation

```sql
-- Set memory limit for observer process
ALTER SYSTEM SET memory_limit = '400G';  -- Of 512GB total

-- Set tenant memory (within observer limit)
ALTER RESOURCE UNIT unit_4c8g MEMORY_SIZE = '8G';
```

### Disk I/O Isolation (V4.2+)

```sql
-- Set IOPS limit per unit
ALTER RESOURCE UNIT unit_4c8g MAX_IOPS = 10000;
```

---

## cgroup Configuration (Linux)

### Check cgroup Setup

```bash
# Check if cgroup is enabled
cat /proc/cgroups | grep cpu

# Check observer's cgroup settings
cat /sys/fs/cgroup/cpu/observer/cpu.cfs_quota_us
```

### Manual cgroup Configuration (if not auto-configured)

```bash
# Create cgroup for observer
cgcreate -g cpu,memory:observer

# Set CPU limit (80 CPUs)
cgset -r cpu.cfs_quota_us=8000000 observer

# Set memory limit (400GB)
cgset -r memory.limit_in_bytes=429496729600 observer

# Attach observer process
cgclassify -g cpu,memory:observer $(pidof observer)
```

> **Note**: OceanBase OBD auto-configures cgroup. Manual setup is rarely needed.

---

## Tenant Resource Isolation

Each tenant's resources are isolated via **Resource Unit**:

```sql
-- Create unit with resource limits
CREATE RESOURCE UNIT unit_isolated
  MAX_CPU 4,
  MEMORY_SIZE '8G',
  DATA_DISK_SIZE '50G',
  LOG_DISK_SIZE '16G',
  MAX_IOPS 10000;

-- Assign to tenant
CREATE RESOURCE POOL pool_isolated UNIT = 'unit_isolated';
CREATE TENANT tenant_isolated RESOURCE_POOL_LIST = ('pool_isolated');
```

---

## Monitoring Resource Usage

### Observer Process (OS Level)

```bash
# CPU usage
top -p $(pidof observer)

# Memory usage
ps aux | grep observer | awk '{print $6}'

# I/O usage
iotop -p $(pidof observer)
```

### Tenant Resource Usage (SQL)

```sql
-- MySQL Mode: tenant CPU/memory usage
SELECT tenant_id,
       round(sum(cpu_usage), 2) AS cpu_used,
       round(sum(mem_usage) / 1024 / 1024 / 1024, 2) AS mem_gb_used
FROM oceanbase.DBA_OB_TENANT_RUNTIME;

-- Oracle Mode
SELECT * FROM SYS.DBA_OB_TENANT_RUNTIME;
```

---

## Best Practices

1. **Reserve 20% CPU/Memory for OS** — prevent system instability
2. **Monitor cgroup limits** — ensure they're applied correctly
3. **Use separate units for mixed workloads** — OLTP + OLAP on same cluster
4. **Set IOPS limits** — prevent one tenant from saturating disk
5. **Test resource isolation** — run stress test on one tenant, verify others unaffected

---

## MySQL Mode vs Oracle Mode

Resource isolation configuration is identical in both modes.

---

## Common Mistakes

1. **Not setting `memory_limit`**
   - ❌ Observer uses all RAM → OOM kill
   - ✅ `ALTER SYSTEM SET memory_limit='400G'`

2. **Over-allocating CPU to tenants**
   - ❌ Sum of `MAX_CPU` > `cpu_quota_concurrency`
   - ✅ Ensure tenant CPU sum ≤ observer CPU limit

3. **Not reserving resources for sys tenant**
   - ❌ User tenants use all CPU → sys tenant starved
   - ✅ Create dedicated unit for sys tenant

4. **Ignoring I/O isolation**
   - ❌ One tenant saturates disk IOPS
   - ✅ Set `MAX_IOPS` per unit

5. **cgroup not enabled**
   - ❌ No resource isolation on Linux
   - ✅ Verify `cat /proc/cgroups | grep cpu`

---

## OceanBase Version Notes

### V4.2 (Baseline)
- cgroup-based resource isolation
- CPU, memory, IOPS isolation
- Per-tenant resource units

### V4.4+
- Improved cgroup v2 support
- Better I/O isolation (mutil-queue)

---

## Sources
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/resource-isolation
- https://www.oceanbase.com/docs/oceanbase-database-cn/v4.2.5/resource-isolation
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/cgroup
