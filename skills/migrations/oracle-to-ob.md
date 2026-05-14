# Oracle to OceanBase Migration

## Overview

Migrating from Oracle to OceanBase (Oracle Mode) requires data type mapping and SQL syntax adjustments. This guide covers assessment, migration steps, and common issues for V4.2+.

---

## Compatibility

OceanBase Oracle Mode is **partially compatible** with Oracle:

| Feature | Oracle | OceanBase Oracle Mode |
|---------|--------|----------------------|
| SQL Syntax | Oracle | ~90% compatible |
| PL/SQL | Full | Supported (stored proc, trigger, package) |
| Data Types | Oracle | Most supported (see mapping) |
| Analytic Functions | Yes | Yes |
| Partitioning | Yes | Yes |
| DBLink | Yes | Yes |

> **Result**: Some SQL rewriting needed.

---

## Migration Steps

### Step1: Assessment (OMAT)

```bash
# OceanBase Migration Assessment Tool
omat assess --source-oracle-host 10.0.0.1 --source-oracle-port 1521 \
       --source-oracle-service testdb \
       --target-ob-host 10.0.0.100 --target-ob-port 2881
```

> **Output**: Compatibility report (SQL incompatibilities, data type mapping).

### Step2: Schema Migration (OMS)

```bash
# OMS (OceanBase Migration Service) handles schema + data
# 1. Create migration task in OMS console
# 2. Configure source (Oracle) and target (OceanBase Oracle Mode)
# 3. OMS automatically converts schema (data types, PL/SQL)
# 4. Start full migration
# 5. Start incremental replication (minimal downtime)
```

### Step3: Data Migration (OMS)

OMS supports:
- **Full migration**: All data (baseline)
- **Incremental replication**: Real-time catch-up (minimal downtime)

---

## Oracle → OceanBase Data Type Mapping

| Oracle | OceanBase Oracle Mode | Notes |
|--------|--------------------------|-------|
| `NUMBER(p,s)` | `NUMBER(p,s)` | Same |
| `VARCHAR2(n)` | `VARCHAR2(n)` | Same |
| `NVARCHAR2(n)` | `NVARCHAR2(n)` | Same |
| `DATE` | `DATE` | Same |
| `TIMESTAMP` | `TIMESTAMP` | Same |
| `CLOB` | `CLOB` | Same |
| `BLOB` | `BLOB` | Same |
| `RAW(n)` | `RAW(n)` | Same |
| `LONG` | `CLOB` | Convert (LONG deprecated) |
| `ROWID` | `UROWID` | Different (pseudo-column) |

> **Note**: `LONG` must be converted to `CLOB`.

---

## SQL Syntax Differences

| Oracle | OceanBase Oracle Mode | Action |
|--------|----------------------|--------|
| `DUAL` table | Supported | None |
| `SYSDATE` | `SYSDATE` | Same |
| `ROWNUM` | `ROWNUM` | Same |
| `NVL()` | `NVL()` | Same |
| `DECODE()` | `DECODE()` | Same |
| `CONNECT BY` (hierarchical) | Not supported | Rewrite using recursive CTE |
| `MERGE` | Supported | Same |
| `PIVOT`/`UNPIVOT` | Supported (V4.2+) | Same |

---

## PL/SQL Migration

OMS automatically converts most PL/SQL objects:

| Object | Oracle | OceanBase | Conversion |
|--------|--------|------------|-------------|
| Stored Procedure | `CREATE PROCEDURE` | `CREATE PROCEDURE` | Auto-converted |
| Function | `CREATE FUNCTION` | `CREATE FUNCTION` | Auto-converted |
| Trigger | `CREATE TRIGGER` | `CREATE TRIGGER` | Auto-converted |
| Package | `CREATE PACKAGE` | `CREATE PACKAGE` | Auto-converted |
| Cursor | `CURSOR` | `CURSOR` | Auto-converted |

> **Check**: OMS validation report for conversion errors.

---

## Common Issues

| Issue | Fix |
|-------|-----|
| `CONNECT BY` not supported | Rewrite using recursive CTE |
| `LONG` data type | Convert to `CLOB` before migration |
| Package body compilation error | Manual rewrite (rare) |
| Performance degradation | Check indexes; gather stats |

---

## Verification Checklist

| Item | Command |
|------|---------|
| Row count match | `SELECT COUNT(*) FROM table` (source vs target) |
| Data sample match | `SELECT * FROM table WHERE ROWNUM <= 10` |
| PL/SQL works | Execute a few procedures |
| Trigger works | Test INSERT/UPDATE that fires trigger |

---

## Common Mistakes

1. **Not converting `LONG` to `CLOB`**
   - ❌ Migration fails on `LONG` columns
   - ✅ `ALTER TABLE t MODIFY (col CLOB);` before migration

2. **Using OMS without testing**
   - ❌ First cutover is production
   - ✅ Test migration in staging 3 times

3. **Not updating application JDBC URL**
   - ❌ App still connects to Oracle after cutover
   - ✅ Update JDBC URL to OceanBase (same Oracle syntax)

4. **Forgetting to migrate DBLink**
   - ❌ DBLinks missing after migration
   - ✅ Manually recreate DBLinks in OceanBase

5. **Not checking PL/SQL conversion report**
   - ❌ Procedures fail after migration
   - ✅ Review OMS conversion report; fix errors

---

## OceanBase Version Notes

### V4.2 (Baseline)
- Oracle Mode PL/SQL support
- OMS schema + data migration
- `PIVOT`/`UNPIVOT` support

### V4.4+
- More Oracle feature compatibility
- Faster OMS replication

---

## Sources
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/oracle-migration
- https://www.oceanbase.com/docs/oceanbase-database-cn/v4.2.5/oracle-migration
- https://en.oceanbase.com/docs/oms/oracle-to-ob
