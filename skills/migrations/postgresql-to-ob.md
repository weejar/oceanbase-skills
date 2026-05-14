# PostgreSQL to OceanBase Migration

## Overview

Migrating from PostgreSQL to OceanBase (MySQL Mode) requires data type mapping and SQL syntax changes. This guide covers assessment, migration steps, and common issues for V4.2+.

---

## Compatibility

OceanBase MySQL Mode is **partially compatible** with PostgreSQL:

| Feature | PostgreSQL | OceanBase MySQL Mode |
|---------|-------------|----------------------|
| SQL Syntax | PostgreSQL | MySQL syntax (different) |
| Data Types | PostgreSQL | Most supported (see mapping) |
| PL/pgSQL | Yes | MySQL stored proc (different) |
| JSON | Yes | Yes |
| Window Functions | Yes | Yes |

> **Result**: SQL rewriting needed.

---

## Migration Steps

### Step1: Assessment (OMAT)

```bash
# OceanBase Migration Assessment Tool
omat assess --source-pg-host 10.0.0.1 --source-pg-port 5432 \
       --source-db test_db \
       --target-ob-host 10.0.0.100 --target-ob-port 2881
```

> **Output**: Compatibility report (SQL incompatibilities, data type mapping).

### Step2: Schema Conversion (OMS)

```bash
# OMS automatically converts PostgreSQL schema to OceanBase MySQL Mode:
# 1. Data type mapping (see below)
# 2. SQL syntax conversion (e.g., `ILIKE` → `LIKE`)
# 3. PL/pgSQL → MySQL stored procedure (manual review may be needed)
```

### Step3: Data Migration (OMS)

OMS supports:
- **Full migration**: All data
- **Incremental replication**: Real-time catch-up (minimal downtime)

---

## PostgreSQL → OceanBase Data Type Mapping

| PostgreSQL | OceanBase MySQL Mode | Notes |
|------------|--------------------------|-------|
| `INTEGER` | `INT` | Same |
| `BIGINT` | `BIGINT` | Same |
| `VARCHAR(n)` | `VARCHAR(n)` | Same |
| `TEXT` | `TEXT` | Same |
| `JSON` | `JSON` | Same |
| `JSONB` | `JSON` | Convert (no binary JSON) |
| `TIMESTAMP` | `TIMESTAMP` | Same |
| `DATE` | `DATE` | Same |
| `BOOLEAN` | `TINYINT(1)` | Convert |
| `UUID` | `VARCHAR(36)` | Convert |
| `ARRAY` | `JSON` | Convert |

> **Note**: `JSONB` and `ARRAY` need conversion.

---

## SQL Syntax Differences

| PostgreSQL | OceanBase MySQL Mode | Action |
|-------------|----------------------|--------|
| `ILIKE` | `LIKE` | Rewrite |
| `NOW()` | `NOW()` | Same |
| `CURRENT_TIMESTAMP` | `CURRENT_TIMESTAMP` | Same |
| `STRING_AGG()` | `GROUP_CONCAT()` | Rewrite |
| `ARRAY_AGG()` | `JSON_ARRAYAGG()` | Rewrite |
| `GENERATED ALWAYS AS IDENTITY` | `AUTO_INCREMENT` | Rewrite |
| `RETURNING` | Not supported | Rewrite (separate SELECT) |

---

## PL/pgSQL Migration

PostgreSQL PL/pgSQL must be rewritten as MySQL stored procedures:

```postgresql
-- PostgreSQL
CREATE FUNCTION get_emp_count(dept_id INT) RETURNS INT AS $$
BEGIN
    RETURN (SELECT COUNT(*) FROM employees WHERE dept_id = $1);
END;
$$ LANGUAGE plpgsql;
```

```sql
-- OceanBase MySQL Mode (rewrite)
CREATE FUNCTION get_emp_count(p_dept_id INT) RETURNS INT
BEGIN
    DECLARE emp_count INT;
    SELECT COUNT(*) INTO emp_count FROM employees WHERE dept_id = p_dept_id;
    RETURN emp_count;
END;
```

> **Tip**: OMS converts simple PL/pgSQL; complex ones need manual rewrite.

---

## Common Issues

| Issue | Fix |
|-------|-----|
| `JSONB` not supported | Convert to `JSON` |
| `ILIKE` syntax error | Rewrite to `LIKE` |
| `ARRAY` type error | Convert to `JSON` |
| PL/pgSQL compilation error | Manual rewrite |

---

## Verification Checklist

| Item | Command |
|------|---------|
| Row count match | `SELECT COUNT(*) FROM table` (source vs target) |
| Data sample match | `SELECT * FROM table LIMIT 10` |
| Function works | Call migrated functions |
| App connects | Update app with OceanBase JDBC URL |

---

## Common Mistakes

1. **Not rewriting PL/pgSQL**
   - ❌ Assume OMS converts all
   - ✅ Review OMS conversion report; manually rewrite complex procedures

2. **Using `JSONB` without conversion**
   - ❌ Migration fails on `JSONB` column
   - ✅ Convert to `JSON` before migration

3. **Not testing app with OceanBase**
   - ❌ Cutover without app testing
   - ✅ Test app with OceanBase (MySQL Mode) in staging

4. **Forgetting to update JDBC URL**
   - ❌ App still connects to PostgreSQL after cutover
   - ✅ Update JDBC URL to OceanBase

5. **Not checking SQL syntax differences**
   - ❌ `STRING_AGG()` fails on OceanBase
   - ✅ Rewrite to `GROUP_CONCAT()`

---

## OceanBase Version Notes

### V4.2 (Baseline)
- MySQL Mode (target for PostgreSQL migration)
- OMS schema + data migration
- `GROUP_CONCAT()` supported

### V4.4+
- More MySQL-feature compatibility
- Faster OMS replication

---

## Sources
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/postgresql-migration
- https://www.oceanbase.com/docs/oceanbase-database-cn/v4.2.5/postgresql-migration
- https://en.oceanbase.com/docs/oms/postgresql-to-ob
