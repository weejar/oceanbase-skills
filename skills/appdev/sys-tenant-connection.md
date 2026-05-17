# Sys Tenant Connection & Tenant Information Query

> **⚠️ Security Note**: 代码示例中的 `YOUR_PASSWORD` 为占位符。请通过环境变量（如 `os.getenv('OB_PASSWORD')`）或运行时传入真实密码，**不要将真实密码硬编码到文件中**。

> **⚡ Quick Command**: 直接执行以下命令列出所有租户：
> ```bash
> python -c "import pymysql; conn=pymysql.connect(host='172.20.22.213',port=2883,user='root@sys#enmotest',password='YOUR_PASSWORD',database='oceanbase'); c=conn.cursor(); c.execute('SELECT tenant_id,tenant_name,status,CASE compatibility_mode WHEN 0 THEN \"MYSQL\" WHEN 1 THEN \"ORACLE\" END,zone_list FROM oceanbase.__all_tenant WHERE in_recyclebin=0 ORDER BY tenant_id'); [print(r) for r in c.fetchall()]; c.close(); conn.close()"
> ```

Connecting to the sys tenant and querying tenant information is the most common daily operation for OceanBase DBAs. This guide covers connection methods, tenant listing, and resource details.

---

## Connection Methods

### Method 1: MySQL Client (Recommended)

```bash
mysql -h <OBProxy_IP> -P <OBProxy_Port> -u "root@sys#<cluster_name>" -p
```

| Parameter | Description | Example |
|-----------|-------------|---------|
| `-h` | OBProxy or observer IP | `172.20.22.213` |
| `-P` | OBProxy SQL port (default 2883) or observer SQL port (default 2881) | `2883` |
| `-u` | `root@sys#cluster_name` (via OBProxy) or `root@sys` (direct observer) | `root@sys#enmotest` |
| `-p` | Password (prompted or inline) | |

### Method 2: OBClient

```bash
obclient -h <OBProxy_IP> -P 2883 -u "root@sys#<cluster_name>" -p -A
```

### Method 3: Python (pymysql)

```python
import pymysql

conn = pymysql.connect(
    host='172.20.22.213',
    port=2883,
    user='root@sys#enmotest',
    password='YOUR_PASSWORD',  # Replace with your actual password
    database='oceanbase',
    charset='utf8mb4'
)
cursor = conn.cursor()
cursor.execute("SELECT tenant_id, tenant_name FROM oceanbase.__all_tenant")
for row in cursor.fetchall():
    print(row)
cursor.close()
conn.close()
```

### Connection Parameters (Current Environment)

| Parameter | Value |
|-----------|-------|
| Host | `172.20.22.213` |
| Port | `2883` |
| User | `root@sys#enmotest` |
| Database | `oceanbase` |

### Connection Notes

- **Via OBProxy**: User format is `root@sys#cluster_name`. Port defaults to `2883`.
- **Direct observer**: User format is `root@sys`. Port defaults to `2881`.
- Always connect to `oceanbase` database for system table access.
- Use `-A` flag to disable MySQL client's auto-completion (speeds up connection).

---

## Tenant Information Queries

### List All Tenants (Primary Query)

```sql
SELECT
    tenant_id,
    tenant_name,
    status,
    CASE compatibility_mode
        WHEN 0 THEN 'MYSQL'
        WHEN 1 THEN 'ORACLE'
        ELSE 'UNKNOWN'
    END AS compat_mode,
    zone_list,
    primary_zone,
    locality,
    CASE locked WHEN 0 THEN 'NO' ELSE 'YES' END AS locked,
    CASE in_recyclebin WHEN 0 THEN 'NO' ELSE 'YES' END AS in_recyclebin,
    gmt_create,
    info
FROM oceanbase.__all_tenant
ORDER BY tenant_id;
```

### Output Columns Explained

| Column | Description |
|--------|-------------|
| `tenant_id` | Unique tenant identifier. `1` = sys tenant. Oracle tenants have companion META$ tenants (e.g., tenant 1034 has META$1034). |
| `tenant_name` | Tenant name. `sys` is system tenant. `META$xxxx` is internal metadata tenant for Oracle mode tenants. |
| `status` | `NORMAL` = active, other values indicate issues. |
| `compatibility_mode` | `0` = MySQL compatible, `1` = Oracle compatible. |
| `zone_list` | Zones where this tenant has units. |
| `primary_zone` | Read-write zone preference (leader location). |
| `locality` | Unit type and count per zone, e.g., `FULL{1}@zone1, FULL{1}@zone2`. |
| `locked` | Whether tenant is locked (locked tenants reject connections). |
| `in_recyclebin` | Whether tenant is in recyclebin (pending deletion). |
| `gmt_create` | Tenant creation timestamp. |
| `info` | Additional info, e.g., `system tenant` for sys. |

### Tenant Count Summary by Compatibility Mode

```sql
SELECT
    CASE compatibility_mode WHEN 0 THEN 'MYSQL' WHEN 1 THEN 'ORACLE' END AS compat_mode,
    COUNT(*) AS tenant_count
FROM oceanbase.__all_tenant
WHERE in_recyclebin = 0
  AND tenant_name NOT LIKE 'META$%'
GROUP BY compatibility_mode;
```

### Tenant Resource Details (Units & Pools)

```sql
SELECT
    t.tenant_name,
    CASE t.compatibility_mode WHEN 0 THEN 'MYSQL' WHEN 1 THEN 'ORACLE' END AS compat,
    g.unit_id,
    g.zone,
    g.svr_ip,
    g.unit_config_name,
    g.max_cpu,
    g.min_cpu,
    g.memory_size / 1024 / 1024 / 1024 AS memory_gb,
    g.log_disk_size / 1024 / 1024 / 1024 AS log_disk_gb,
    g.data_disk_in_use / 1024 / 1024 / 1024 AS data_disk_gb
FROM oceanbase.__all_tenant t
JOIN oceanbase.__all_virtual_unit_stat g ON t.tenant_id = g.tenant_id
WHERE t.in_recyclebin = 0
  AND t.tenant_name NOT LIKE 'META$%'
ORDER BY t.tenant_id, g.zone;
```

### Tenant Database List

```sql
SELECT tenant_id, database_name
FROM oceanbase.__all_database
ORDER BY tenant_id, database_name;
```

### Tenant User List

```sql
SELECT
    tenant_id,
    user_name,
    user_type,
    CASE locked WHEN 0 THEN 'NO' ELSE 'YES' END AS locked
FROM oceanbase.__all_user
WHERE tenant_id > 1000
ORDER BY tenant_id, user_name;
```

### Tenant Object Count Summary

```sql
SELECT
    t.tenant_name,
    CASE t.compatibility_mode WHEN 0 THEN 'MYSQL' WHEN 1 THEN 'ORACLE' END AS compat,
    (SELECT COUNT(*) FROM oceanbase.__all_database d WHERE d.tenant_id = t.tenant_id) AS databases,
    (SELECT COUNT(*) FROM oceanbase.__all_user u WHERE u.tenant_id = t.tenant_id) AS users
FROM oceanbase.__all_tenant t
WHERE t.in_recyclebin = 0
  AND t.tenant_name NOT LIKE 'META$%'
ORDER BY t.tenant_id;
```

---

## Practical Workflow: Full Tenant Health Check (Python)

```python
import pymysql

def tenant_health_check(host, port, user, password):
    conn = pymysql.connect(
        host=host, port=port, user=user,
        password=password, database='oceanbase', charset='utf8mb4'
    )

    def query(sql):
        cursor = conn.cursor()
        cursor.execute(sql)
        cols = [d[0] for d in cursor.description]
        rows = [dict(zip(cols, row)) for row in cursor.fetchall()]
        cursor.close()
        return rows

    # 1. Tenant list
    tenants = query("""
        SELECT tenant_id, tenant_name,
               CASE compatibility_mode WHEN 0 THEN 'MYSQL'
                    WHEN 1 THEN 'ORACLE' END AS compat_mode,
               status, zone_list, primary_zone, locality
        FROM oceanbase.__all_tenant
        WHERE in_recyclebin = 0
        ORDER BY tenant_id
    """)

    print(f"=== Tenant Report: {len(tenants)} tenants ===")
    for t in tenants:
        print(f"  [{t['tenant_id']}] {t['tenant_name']} ({t['compat_mode']}) - {t['status']}")
        print(f"    Zones: {t['zone_list']}")
        print(f"    Locality: {t['locality']}")

    # 2. Resource usage
    resources = query("""
        SELECT t.tenant_name, g.zone, g.svr_ip,
               g.unit_config_name,
               ROUND(g.max_cpu, 1) AS max_cpu,
               ROUND(g.min_cpu, 1) AS min_cpu,
               ROUND(g.memory_size / 1024/1024/1024, 1) AS mem_gb,
               ROUND(g.data_disk_in_use / 1024/1024/1024, 1) AS data_gb
        FROM oceanbase.__all_tenant t
        JOIN oceanbase.__all_virtual_unit_stat g
             ON t.tenant_id = g.tenant_id
        WHERE t.in_recyclebin = 0 AND t.tenant_name NOT LIKE 'META$%'
        ORDER BY t.tenant_id, g.zone
    """)

    print(f"\n=== Resource Allocation ({len(resources)} units) ===")
    print(f"{'Tenant':<15} {'Zone':<10} {'Server':<16} {'Unit':<20} "
          f"{'MaxCPU':>7} {'MemGB':>7} {'DataGB':>7}")
    print("-" * 88)
    for r in resources:
        print(f"{r['tenant_name']:<15} {r['zone']:<10} {r['svr_ip']:<16} "
              f"{r['unit_config_name']:<20} {r['max_cpu']:>7} {r['mem_gb']:>7} "
              f"{r['data_gb']:>7}")

    conn.close()

# Usage
tenant_health_check(
    host='172.20.22.213', port=2883,
    user='root@sys#enmotest', password='YOUR_PASSWORD'
)
```

---

## Common Issues

### 1. Access Denied

```
ERROR 1045 (28000): Access denied for user 'root@sys#enmotest'
```

- Verify cluster name matches exactly (case-sensitive).
- Check if connecting via OBProxy (port 2883) or direct observer (port 2881).
- User format differs: `root@sys#cluster` (OBProxy) vs `root@sys` (direct).

### 2. Unknown Column Error

```
ERROR 1054 (42S22): Unknown column 'tenant_type' in 'field list'
```

- The `__all_tenant` table does NOT have a `tenant_type` column in OceanBase V4.x.
- Use `compatibility_mode` (0=MySQL, 1=Oracle) instead.
- Always run `DESCRIBE oceanbase.__all_tenant` first if unsure about schema.

### 3. Connection Timeout

```
ERROR 2003 (HY000): Can't connect to MySQL server on 'x.x.x.x' (110)
```

- Check network connectivity: `ping <host>`, `telnet <host> <port>`.
- Verify OBProxy is running: `ps -ef | grep obproxy`.
- Check firewall rules for port 2883/2881.

### 4. Empty Result for Business Tenants

- Filter out system and META tenants: `WHERE tenant_name NOT LIKE 'META$%'`.
- Check `in_recyclebin = 0` to exclude recycled tenants.

---

## Related Skills

- `skills/admin/tenant-management.md` - Tenant creation, scaling, resource pool management
- `skills/architecture/multi-tenant.md` - Multi-tenant architecture design
- `skills/monitoring/session-monitor.md` - Session monitoring, active sessions, wait events
- `skills/tools/obd-deployer.md` - OBD cluster management commands
- `skills/devops/obd-deployment.md` - OBD deployment and cluster setup
- `skills/monitoring/` - Health check, obdiag, Top SQL

---

## __all_tenant Table Schema (V4.x)

| Column | Type | Description |
|--------|------|-------------|
| gmt_create | timestamp(6) | Creation time |
| gmt_modified | timestamp(6) | Last modification time |
| tenant_id | bigint(20) | Tenant ID (PK) |
| tenant_name | varchar(128) | Tenant name |
| zone_list | varchar(8192) | Zone list |
| primary_zone | varchar(8192) | Primary zone |
| locked | bigint(20) | 0=unlocked, 1=locked |
| collation_type | bigint(20) | Collation setting |
| info | varchar(4096) | Additional info |
| locality | varchar(4096) | Unit locality per zone |
| previous_locality | varchar(4096) | Previous locality (before migration) |
| default_tablegroup_id | bigint(20) | Default table group ID |
| compatibility_mode | bigint(20) | 0=MySQL, 1=Oracle |
| drop_tenant_time | bigint(20) | Drop time (-1=not dropped) |
| status | varchar(64) | Tenant status |
| in_recyclebin | bigint(20) | 0=active, 1=in recyclebin |
| arbitration_service_status | varchar(64) | Arbitration service status |

---

*Version: OceanBase V4.x | Last updated: 2026-05-14*
