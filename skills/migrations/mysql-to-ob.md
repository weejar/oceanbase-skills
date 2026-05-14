# MySQL to OceanBase Migration

## Overview

Migrating from MySQL to OceanBase (MySQL Mode) is straightforward because OceanBase is MySQL-compatible. This guide covers assessment, migration steps, and common issues for V4.2+.

---

## Compatibility

OceanBase MySQL Mode is **highly compatible** with MySQL 5.7/8.0:

| Feature | MySQL | OceanBase MySQL Mode |
|---------|-------|--------------------|
| SQL Syntax | 100% | ~98% compatible |
| Data Types | All | All (plus extensions) |
| PL/SQL | No | Yes (MySQL stored proc) |
| JSON | Yes | Yes |
| Window Functions | Yes | Yes |
| CTE (WITH) | Yes | Yes |

> **Result**: Most MySQL apps work unchanged with OceanBase.

---

## Migration Steps

### Step 1: Assessment

```bash
# Use OceanBase Migration Assessment Tool (OMAT)
omat assess --source-mysql-host 10.0.0.1 --source-mysql-port 3306 \
       --source-db test_db \
       --target-ob-host 10.0.0.100 --target-ob-port 2881
```

> **Output**: Compatibility report (SQL incompatibilities, data type mapping).

### Step 2: Data Migration (OMS)

```bash
# Use OceanBase Migration Service (OMS) for online migration
# OMS supports: full data + incremental replication (minimal downtime)

# 1. Create migration task in OMS console
# 2. Configure source (MySQL) and target (OceanBase)
# 3. Select tables to migrate
# 4. Start full migration
# 5. Start incremental replication (catches up real-time)
# 6. Cutover (switch app to OceanBase)
```

### Step 3: Verify Data

```sql
-- Compare row counts
SELECT 'employees', COUNT(*) FROM employees
UNION ALL
SELECT 'departments', COUNT(*) FROM departments;

-- Compare data samples
SELECT * FROM employees ORDER BY emp_id LIMIT 10;
```

---

## MySQL → OceanBase Data Type Mapping

| MySQL | OceanBase MySQL Mode | Notes |
|--------|----------------------|-------|
| `INT` | `INT` | Same |
| `BIGINT` | `BIGINT` | Same |
| `VARCHAR(n)` | `VARCHAR(n)` | Same |
| `TEXT` | `TEXT` | Same |
| `BLOB` | `BLOB` | Same |
| `JSON` | `JSON` | Same |
| `DATETIME` | `DATETIME` | Same |
| `TIMESTAMP` | `TIMESTAMP` | Same |

> **No data type changes needed** for MySQL → OceanBase MySQL Mode.

---

## Application Changes

| Area | MySQL | OceanBase | Change Needed? |
|-------|-------|------------|----------------|
| JDBC URL | `jdbc:mysql://mysql-host:3306/db` | `jdbc:mysql://ob-proxy:2883/db` | Only host/port |
| SQL Syntax | MySQL syntax | MySQL syntax | None (compatible) |
| User | `app_user@%` | `app_user@mysql_tenant` | Username format |

> **Result**: Most apps only need JDBC URL change.

---

## Common Issues

| Issue | Fix |
|-------|-----|
| `ERROR 1235 (42000): Not supported` | Check OMAT report; rewrite SQL |
| `Character set not supported` | Use `utf8mb4` (OceanBase default) |
| `max_allowed_packet` exceeded | `ALTER SYSTEM SET max_allowed_packet='128M'` |
| Performance degradation | Check indexes; run `ANALYZE TABLE` |

---

## Verification Checklist

| Item | Command |
|------|---------|
| Row count match | `SELECT COUNT(*) FROM table` (source vs target) |
| Sample data match | `SELECT * FROM table ORDER BY pk LIMIT 10` |
| Index exists | `SHOW INDEX FROM table` |
| Stored proc works | Call a few procedures |
| App connects | Run app with OB JDBC URL |

---

## Common Mistakes

1. **Not running OMAT assessment**
   - ❌ Surprise incompatibilities during cutover
   - ✅ Run OMAT first

2. **Using OMS without testing**
   - ❌ First cutover attempt is production
   - ✅ Test migration in staging 3 times

3. **Not updating JDBC URL in app**
   - ❌ App still connects to MySQL after cutover
   - ✅ Update all app configs to OBProxy IP:2883

4. **Not checking character set**
   - ❌ Data corruption due to charset mismatch
   - ✅ Ensure `utf8mb4` on both sides

5. **Forgetting stored procedures/triggers**
   - ❌ Procedures missing after migration
   - ✅ Export and import all stored procedures

---

## OceanBase Version Notes

### V4.2 (Baseline)
- MySQL 5.7/8.0 compatibility
- OMS full + incremental migration

### V4.4+
- Faster OMS replication
- More MySQL-feature compatibility

---

## Sources
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/mysql-migration
- https://www.oceanbase.com/docs/oceanbase-database-cn/v4.2.5/mysql-migration
- https://en.oceanbase.com/docs/oms/mysql-to-ob
