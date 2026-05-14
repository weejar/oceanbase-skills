# Tenant Management

## Overview

Multi-tenancy is the **core architecture** of OceanBase. Each tenant is an isolated database instance with its own resources (CPU, memory, disk) and users. This guide covers tenant creation, scaling, resource management, and cloning in OceanBase V4.2+.

---

## Key Concepts

| Concept | Description |
|---------|-------------|
| **Tenant** | Isolation unit (like a database instance). Each tenant has its own `sys` user. |
| **Resource Unit (Unit)** | Hardware resource definition: CPU, memory, disk, IOPS. |
| **Resource Pool** | Collection of units distributed across zones. |
| **Zone** | Failure domain (like a rack or data center). |

```
Zone-1     Zone-2     Zone-3
  |           |           |
  +-- Unit 1  Unit 2  Unit 3 --+
  |           |           |
  +------- Resource Pool --------+
              |
           Tenant
```

---

## Creating a Tenant

### Step 1: Create Resource Units

```sql
-- MySQL Mode: Define unit specs
CREATE RESOURCE UNIT unit_small
  MAX_CPU 2,
  MEMORY_SIZE '4G',
  LOG_DISK_SIZE '8G',
  DATA_DISK_SIZE '20G',
  MAX_IOPS 10000;

-- View units
SELECT * FROM oceanbase.DBA_OB_UNITS;
```

### Step 2: Create Resource Pool

```sql
-- Assign units to zones
CREATE RESOURCE POOL pool_small
  UNIT = 'unit_small',
  UNIT_NUM = 1,
  ZONE_LIST = ('zone1', 'zone2', 'zone3');
```

### Step 3: Create Tenant

```sql
-- Create tenant (MySQL Mode)
CREATE TENANT tenant_app
  RESOURCE_POOL_LIST = ('pool_small')
  SET ob_tcp_invited_nodes = '%';  -- Allowed client IPs

-- Create tenant (Oracle Mode)
CREATE TENANT tenant_app_oracle
  RESOURCE_POOL_LIST = ('pool_small')
  SET ob_compatibility_mode = 'oracle',
      ob_tcp_invited_nodes = '%';
      
 -- OBD tool
 obd cluster tenant create <deploy_name> -n <tenant_name> [options]
```

---

## Scaling a Tenant

### Vertical Scaling (Change Unit Size)

```sql
-- Step 1: Create a larger unit
CREATE RESOURCE UNIT unit_medium
  MAX_CPU 4,
  MEMORY_SIZE '8G',
  LOG_DISK_SIZE '16G',
  DATA_DISK_SIZE '40G';

-- Step 2: Alter resource pool to use new unit
ALTER RESOURCE POOL pool_small
  UNIT = 'unit_medium';

-- OceanBase automatically migrates data to new unit (online)
```

### Horizontal Scaling (Add More Replicas)

```sql
-- Add more units (increase UNIT_NUM)
ALTER RESOURCE POOL pool_small UNIT_NUM = 2;

-- This adds more replicas across zones for higher concurrency
```

---

## Tenant Operations

### View Tenants

```sql
-- MySQL Mode
SHOW TENANTS;

-- Both modes (system view)
SELECT tenant_id, tenant_name, status
FROM oceanbase.DBA_OB_TENANTS;

-- obd tool
obd cluster tenant show <deploy_name>
```

### Stop/Start Tenant

```sql
-- Stop tenant (maintenance mode)
ALTER TENANT tenant_app STATUS = 'STOPPED';

-- Start tenant
ALTER TENANT tenant_app STATUS = 'NORMAL';
```

### Drop Tenant

```sql
-- Drop tenant (IRREVERSIBLE!)
DROP TENANT tenant_app;
-- WITH FORCE (if tenant has active sessions)
DROP TENANT tenant_app FORCE;

-- obd tool
obd cluster tenant drop <deploy_name> -n <tenant_name>
```

### Clone Tenant (V4.2+)

```sql
-- Clone an existing tenant (for testing/staging)
CREATE TENANT tenant_app_clone
  FROM tenant_app
  RESOURCE_POOL_LIST = ('pool_clone');
```

---

## Resource Unit Parameters

| Parameter | Description | Example |
|-----------|-------------|---------|
| `MAX_CPU` | CPU cores | `2`, `4`, `16` |
| `MEMORY_SIZE` | Total memory | `'4G'`, `'16G'` |
| `LOG_DISK_SIZE` | Disk for CLog | `'8G'`, `'32G'` |
| `DATA_DISK_SIZE` | Disk for data | `'20G'`, `'100G'` |
| `MAX_IOPS` | IOPS limit | `10000`, `50000` |

```
-- Check unit configuration
SELECT unit_id, name, max_cpu, memory_size, data_disk_size
FROM oceanbase.DBA_OB_UNITS;
```



### Optimize Tenant

```bash
obd cluster tenant optimize <deploy_name> <tenant_name> -o <workload>
```

Supported workloads: `express_oltp`, `olap`, etc.

---

## Tenant Best Practices

1. **Right-size units** — monitor CPU/memory usage before scaling
2. **Use 3 zones for production** — ensures Paxos majority quorum
3. **Separate sys tenant and user tenants** — don't share resources
4. **Set `ob_tcp_invited_nodes` restrictively** — avoid `'%'` in production
5. **Monitor tenant resource usage** — set up OCP alerts

---

## MySQL Mode vs Oracle Mode

| Aspect | MySQL Mode | Oracle Mode |
|--------|-----------|-------------|
| Tenant creation | `CREATE TENANT ...` (default MySQL) | `SET ob_compatibility_mode='oracle'` |
| Connection | `mysql -h -P -u -p` | Same (wire protocol compatible) |
| System views | `oceanbase.DBA_OB_TENANTS` | `SYS.DBA_OB_TENANTS` |

---

## Common Mistakes

1. **Setting `ob_tcp_invited_nodes='%'` in production**
   - ❌ Allows connections from any IP
   - ✅ `ob_tcp_invited_nodes='10.0.0.%'`

2. **Unit too small for workload**
   - ❌ `MAX_CPU=1, MEMORY_SIZE='2G'` for production
   - ✅ Monitor and scale: `ALTER RESOURCE POOL ... UNIT='unit_larger'`

3. **Forgetting to create resource pool before tenant**
   - ❌ `CREATE TENANT` without `RESOURCE_POOL_LIST`
   - ✅ Always create unit → pool → tenant

4. **Dropping tenant without backup**
   - ❌ `DROP TENANT` then realize data is needed
   - ✅ Always take a backup before dropping

5. **Not monitoring tenant resources**
   - ❌ Tenant runs out of disk space
   - ✅ Set OCP alert at 80% disk usage

---

## OceanBase Version Notes

### V4.2 (Baseline)
- Multi-tenant architecture
- Tenant clone
- Online resource scaling

### V4.4+
- Faster unit migration during scaling
- Improved resource isolation (cgroup v2)

---

## Sources
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/multi-tenant
- https://www.oceanbase.com/docs/oceanbase-database-cn/v4.2.5/multi-tenant
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/create-tenant
