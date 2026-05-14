# DB2 to OceanBase Migration

## Overview

Migrating from DB2 to OceanBase (MySQL Mode) requires data type mapping and SQL syntax changes. This guide covers assessment, migration steps, and common issues for V4.2+.

---

## Compatibility

OceanBase MySQL Mode has **partial compatibility** with DB2:

| Feature | DB2 | OceanBase MySQL Mode |
|---------|-----|----------------------|
| SQL Syntax | DB2 SQL | MySQL syntax (different) |
| Data Types | DB2 types | Most supported (see mapping) |
| PL/SQL | SQL PL | MySQL stored proc (different) |
| Window Functions | Yes | Yes |

> **Result**: Significant SQL rewriting needed.

---

## Migration Steps

### Step1: Assessment (OMAT)

```bash
# OceanBase Migration Assessment Tool
omat assess --source-db2-host 10.0.0.1 --source-db2-port 50000 \
       --source-db2-db sample \
       --target-ob-host 10.0.0.100 --target-ob-port 2881
```

> **Output**: Compatibility report (SQL incompatibilities, data type mapping).

### Step2: Schema Conversion (OMS)

```bash
# OMS (OceanBase Migration Service) handles schema + data
# 1. Create migration task in OMS console
# 2. Configure source (DB2) and target (OceanBase MySQL Mode)
# 3. OMS automatically converts schema (data types, SQL)
# 4. Start full migration
# 5. Start incremental replication (minimal downtime)
```

### Step3: Data Migration (OMS)

OMS supports:
- **Full migration**: All data (baseline)
- **Incremental replication**: Real-time catch-up (minimal downtime)

---

## DB2 → OceanBase Data Type Mapping

| DB2 | OceanBase MySQL Mode | Notes |
|-----|--------------------------|-------|
| `INTEGER` | `INT` | Same |
| `BIGINT` | `BIGINT` | Same |
| `VARCHAR(n)` | `VARCHAR(n)` | Same |
| `CHAR(n)` | `CHAR(n)` | Same |
| `CLOB` | `CLOB` | Same |
| `BLOB` | `BLOB` | Same |
| `TIMESTAMP` | `TIMESTAMP` | Same |
| `DATE` | `DATE` | Same |
| `DECIMAL(p,s)` | `DECIMAL(p,s)` | Same |
| `FLOAT` | `FLOAT` | Same |

> **Note**: Most types map directly.

---

## SQL Syntax Differences

| DB2 | OceanBase MySQL Mode | Action |
|-----|----------------------|--------|
| `CURRENT TIMESTAMP` | `CURRENT_TIMESTAMP` | Same |
| `NVL()` | `IFNULL()` or `COALESCE()` | Rewrite |
| `ROW_NUMBER() OVER()` | `ROW_NUMBER() OVER()` | Same (window functions) |
| `FETCH FIRST n ROWS ONLY` | `LIMIT n` | Rewrite |
| `INSERT INTO ... VALUES ...` | Same | Same |

---

## Common Issues

| Issue | Fix |
|-------|-----|
| `NVL()` function error | Rewrite to `IFNULL()` or `COALESCE()` |
| `FETCH FIRST n ROWS ONLY` error | Rewrite to `LIMIT n` |
| PL/SQL conversion error | Manual rewrite (SQL PL → MySQL stored proc) |

---

## Verification Checklist

| Item | Command |
|------|---------|
| Row count match | `SELECT COUNT(*) FROM table` (source vs target) |
| Data sample match | `SELECT * FROM table LIMIT 10` |
| Stored proc works | Call migrated procedures |

---

## Common Mistakes

1. **Not running OMAT assessment**
   - ❌ Surprise incompatibilities during cutover
   - ✅ Run OMAT first

2. **Using OMS without testing**
   - ❌ First cutover attempt is production
   - ✅ Test migration in staging 3 times

3. **Not rewriting `NVL()`**
   - ❌ `NVL()` fails on OceanBase
   - ✅ Rewrite to `IFNULL()` or `COALESCE()`

4. **Forgetting to update application**
   - ❌ App still connects to DB2 after cutover
   - ✅ Update JDBC URL to OceanBase

5. **Not checking SQL syntax differences**
   - ❌ `FETCH FIRST n ROWS ONLY` fails
   - ✅ Rewrite to `LIMIT n`

---

## OceanBase Version Notes

### V4.2 (Baseline)
- MySQL Mode (target for DB2 migration)
- OMS schema + data migration
- Window function support

### V4.4+
- More MySQL-feature compatibility
- Faster OMS replication

---

## Sources
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/db2-migration
- https://www.oceanbase.com/docs/oceanbase-database-cn/v4.2.5/db2-migration
- https://en.oceanbase.com/docs/oms/db2-to-ob
