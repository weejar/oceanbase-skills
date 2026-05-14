# Multi-Tenant Architecture

## Overview

OceanBase's multi-tenant architecture isolates multiple "tenants" (database instances) within a single cluster. Each tenant has its own resources, users, and data. This guide explains the multi-tenant architecture and management for OceanBase V4.2+.

---

## Architecture Layers

```
OceanBase Cluster
├── Sys Tenant (管理租户, 1个集群 only)
├── User Tenant-1 (业务租户, 独立资源)
├── User Tenant-2 (业务租户, 独立资源)
└── Meta Tenant (每个User Tenant对应1个Meta Tenant)
```

| Tenant Type | Purpose | Resource Isolation |
|-------------|---------|-------------------|
| **Sys Tenant** | Cluster management, metadata | Shared (small) |
| **User Tenant** | Business workload | Full isolation (unit) |
| **Meta Tenant** | User tenant's metadata | Linked to user tenant |

---

## Sys Tenant

The **Sys Tenant** manages the entire cluster. It hosts:
- Cluster parameters (`SHOW PARAMETERS`)
- Tenant management (`DBA_OB_TENANTS`)
- OBServer management (`DBA_OB_SERVERS`)

```sql
-- Connect to sys tenant (MySQL Mode)
mysql -h127.0.0.1 -P2881 -uroot@sys -p

-- View all tenants
SELECT tenant_id, tenant_name, status
FROM oceanbase.DBA_OB_TENANTS;
```

> **Important**: The sys tenant does NOT store business data.

---

## User Tenant

Each **User Tenant** is a fully isolated database instance.

```sql
-- Create a user tenant
CREATE TENANT app_tenant
  RESOURCE_POOL_LIST = ('pool_app')
  SET ob_tcp_invited_nodes = '10.0.0.%';

-- Connect to user tenant (MySQL Mode)
mysql -h127.0.0.1 -P2881 -uapp_user@app_tenant -p

-- View resources for a tenant
SELECT * FROM oceanbase.DBA_OB_UNITS
WHERE tenant_id = (SELECT tenant_id FROM oceanbase.DBA_OB_TENANTS WHERE tenant_name='app_tenant');
```

---

## Meta Tenant

Each **User Tenant** has a corresponding **Meta Tenant** that stores:
- Table metadata
- Partition metadata
- Schema definitions

> **Note**: Meta tenants are managed automatically. You don't create them manually.

---

## Resource Isolation

Resources are isolated via **Resource Unit** and **Resource Pool**:

```sql
-- Step1: Define unit (CPU, memory, disk)
CREATE RESOURCE UNIT unit_4c8g
  MAX_CPU 4,
  MEMORY_SIZE '8G',
  DATA_DISK_SIZE '50G',
  LOG_DISK_SIZE '16G';

-- Step2: Create pool (assign units to zones)
CREATE RESOURCE POOL pool_app
  UNIT = 'unit_4c8g',
  UNIT_NUM = 1,
  ZONE_LIST = ('zone1', 'zone2', 'zone3');

-- Step3: Create tenant using pool
CREATE TENANT app_tenant
  RESOURCE_POOL_LIST = ('pool_app');
```

---

## Tenant Operations

### View Tenant Resources

```sql
-- MySQL Mode: view tenant resource usage
SELECT t.tenant_name,
       u.unit_id,
       u.max_cpu,
       u.memory_size / 1024 / 1024 / 1024 AS mem_gb
FROM oceanbase.DBA_OB_TENANTS t
JOIN oceanbase.DBA_OB_UNITS u ON t.tenant_id = u.tenant_id;
```

### Scale Tenant (Online)

```sql
-- Create larger unit
CREATE RESOURCE UNIT unit_8c16g AS MAX_CPU 8, MEMORY_SIZE '16G';

-- Switch pool to larger unit (online!)
ALTER RESOURCE POOL pool_app UNIT = 'unit_8c16g';
```

### Stop/Start Tenant

```sql
-- Stop tenant (maintenance)
ALTER TENANT app_tenant STATUS = 'STOPPED';

-- Restart
ALTER TENANT app_tenant STATUS = 'NORMAL';
```

---

## MySQL Mode vs Oracle Mode

| Aspect | MySQL Mode | Oracle Mode |
|--------|-----------|-------------|
| Tenant creation | `CREATE TENANT ...` | Same (add `SET ob_compatibility_mode='oracle'`) |
| Connect | `-uuser@tenant` | Same |
| System views | `oceanbase.DBA_OB_*` | `SYS.DBA_OB_*` |

---

## Common Mistakes

1. **Sys tenant and user tenant on same hardware without resource isolation**
   - ❌ Sys tenant starves user tenant resources
   - ✅ Allocate dedicated units for sys tenant

2. **`ob_tcp_invited_nodes='%'` in production**
   - ❌ Any IP can connect
   - ✅ `ob_tcp_invited_nodes='10.0.0.%'`

3. **Not monitoring tenant resource usage**
   - ❌ Tenant runs out of CPU/memory
   - ✅ OCP alert at 80% resource usage

4. **Unit too small, causing OOM**
   - ❌ `MEMORY_SIZE='2G'` for production workload
   - ✅ Monitor and scale proactively

5. **Deleting tenant without backup**
   - ❌ `DROP TENANT` then realize data needed
   - ✅ Always backup before drop

---

## OceanBase Version Notes

### V4.2 (Baseline)
- Multi-tenant architecture
- Resource unit/pool management
- Online tenant scaling

### V4.4+
- Faster unit migration during scaling
- Improved resource isolation (cgroup v2)

---

## Sources
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/multi-tenant
- https://www.oceanbase.com/docs/oceanbase-database-cn/v4.2.5/multi-tenant
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/tenant-management
