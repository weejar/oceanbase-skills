# SQL Server to OceanBase Migration

## Overview

Migrating from SQL Server to OceanBase (MySQL Mode) requires data type mapping and SQL syntax changes. This guide covers assessment, migration steps, and common issues for V4.2+.

---

## Compatibility

OceanBase MySQL Mode has **partial compatibility** with SQL Server:

| Feature | SQL Server | OceanBase MySQL Mode |
|---------|-------------|----------------------|
| SQL Syntax | T-SQL | MySQL syntax (different) |
| Data Types | SQL Server types | Most supported (see mapping) |
| T-SQL | Yes | MySQL stored proc (different) |
| Window Functions | Yes | Yes |

> **Result**: Significant SQL rewriting needed.

---

## Migration Steps

### Step1: Assessment (OMAT)

```bash
# OceanBase Migration Assessment Tool
omat assess --source-sqlserver-host 10.0.0.1 --source-sqlserver-port 1433 \
       --source-db test_db \
       --target-ob-host 10.0.0.100 --target-ob-port 2881
```

> **Output**: Compatibility report (SQL incompatibilities, data type mapping).

### Step2: Schema Conversion (OMS)

```bash
# OMS (OceanBase Migration Service) handles schema + data
# 1. Create migration task in OMS console
# 2. Configure source (SQL Server) and target (OceanBase MySQL Mode)
# 3. OMS automatically converts schema (data types, SQL)
# 4. Start full migration
# 5. Start incremental replication (minimal downtime)
```

### Step3: Data Migration (OMS)

OMS supports:
- **Full migration**: All data
- **Incremental replication**: Real-time catch-up (minimal downtime)

---

## SQL Server → OceanBase Data Type Mapping

| SQL Server | OceanBase MySQL Mode | Notes |
|-------------|--------------------------|-------|
| `INT` | `INT` | Same |
| `BIGINT` | `BIGINT` | Same |
| `VARCHAR(n)` | `VARCHAR(n)` | Same |
| `NVARCHAR(n)` | `NVARCHAR(n)` | Same |
| `TEXT` | `TEXT` | Same |
| `DATETIME` | `DATETIME` | Same |
| `DATETIME2` | `DATETIME(6)` | Convert |
| `UNIQUEIDENTIFIER` | `VARCHAR(36)` | Convert to string |
| `BIT` | `TINYINT(1)` | Convert |
| `VARBINARY` | `BLOB` | Convert |

> **Note**: `DATETIME2` and `UNIQUEIDENTIFIER` need conversion.

---

## SQL Syntax Differences

| SQL Server | OceanBase MySQL Mode | Action |
|-------------|----------------------|--------|
| `GETDATE()` | `NOW()` | Rewrite |
| `ISNULL()` | `IFNULL()` | Rewrite |
| `TOP n` | `LIMIT n` | Rewrite |
| `IDENTITY(1,1)` | `AUTO_INCREMENT` | Rewrite |
| `ROW_NUMBER() OVER (ORDER BY col)` | `ROW_NUMBER() OVER (ORDER BY col)` | Same (window functions) |

---

## T-SQL Migration

SQL Server T-SQL must be rewritten as MySQL stored procedures:

```sql
-- SQL Server (T-SQL)
CREATE PROCEDURE get_emp_count
    @dept_id INT
AS
BEGIN
    SELECT COUNT(*) FROM employees WHERE dept_id = @dept_id;
END;
```

```sql
-- OceanBase MySQL Mode (rewrite)
CREATE PROCEDURE get_emp_count(IN p_dept_id INT)
BEGIN
    SELECT COUNT(*) FROM employees WHERE dept_id = p_dept_id;
END;
```

> **Tip**: OMS converts simple T-SQL; complex ones need manual rewrite.

---

## Common Issues

| Issue | Fix |
|-------|-----|
| `GETDATE()` syntax error | Rewrite to `NOW()` |
| `TOP n` syntax error | Rewrite to `LIMIT n` |
| `ISNULL()` error | Rewrite to `IFNULL()` |
| T-SQL compilation error | Manual rewrite |

---

## Verification Checklist

| Item | Command |
|------|---------|
| Row count match | `SELECT COUNT(*) FROM table` (source vs target) |
| Data sample match | `SELECT * FROM table LIMIT 10` |
| Stored proc works | Call migrated procedures |

---

## Common Mistakes

1. **Not rewriting T-SQL**
   - ❌ Assume OMS converts all
   - ✅ Review OMS conversion report; manually rewrite complex procedures

2. **Using `TOP n` without conversion**
   - ❌ `TOP 10` fails on OceanBase
   - ✅ Rewrite to `LIMIT 10`

3. **Not testing app with OceanBase**
   - ❌ Cutover without app testing
   - ✅ Test app with OceanBase (MySQL Mode) in staging

4. **Forgetting to update JDBC URL**
   - ❌ App still connects to SQL Server after cutover
   - ✅ Update JDBC URL to OceanBase

5. **Not checking SQL syntax differences**
   - ❌ `GETDATE()` fails on OceanBase
   - ✅ Rewrite to `NOW()`

---

## OceanBase Version Notes

### V4.2 (Baseline)
- MySQL Mode (target for SQL Server migration)
- OMS schema + data migration
- `LIMIT` support

### V4.4+
- More MySQL-feature compatibility
- Faster OMS replication

---

## Sources
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/sqlserver-migration
- https://www.oceanbase.com/docs/oceanbase-database-cn/v4.2.5/sqlserver-migration
- https://en.oceanbase.com/docs/oms/sqlserver-to-ob
