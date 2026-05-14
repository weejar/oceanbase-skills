# Data Type Mapping (Cross-Database)

## Overview

Migrating between databases requires accurate data type mapping. This guide provides mapping tables for common source databases to OceanBase (MySQL Mode or Oracle Mode) for V4.2+.

---

## MySQL → OceanBase MySQL Mode

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
| `DECIMAL(p,s)` | `DECIMAL(p,s)` | Same |

> **Result**: No data type changes needed.

---

## Oracle → OceanBase Oracle Mode

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
| `LONG` | `CLOB` | **Convert** (LONG deprecated) |
| `ROWID` | `UROWID` | Different (pseudo-column) |

> **Action**: Convert `LONG` to `CLOB` before migration.

---

## PostgreSQL → OceanBase MySQL Mode

| PostgreSQL | OceanBase MySQL Mode | Notes |
|------------|----------------------|-------|
| `INTEGER` | `INT` | Same |
| `BIGINT` | `BIGINT` | Same |
| `VARCHAR(n)` | `VARCHAR(n)` | Same |
| `TEXT` | `TEXT` | Same |
| `JSON` | `JSON` | Same |
| `JSONB` | `JSON` | **Convert** (no binary JSON) |
| `TIMESTAMP` | `TIMESTAMP` | Same |
| `DATE` | `DATE` | Same |
| `BOOLEAN` | `TINYINT(1)` | **Convert** |
| `UUID` | `VARCHAR(36)` | **Convert** |
| `ARRAY` | `JSON` | **Convert** |

> **Action**: Convert `JSONB`, `BOOLEAN`, `UUID`, `ARRAY` before migration.

---

## DB2 → OceanBase MySQL Mode

| DB2 | OceanBase MySQL Mode | Notes |
|-----|----------------------|-------|
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

> **Result**: Most types map directly.

---

## SQL Server → OceanBase MySQL Mode

| SQL Server | OceanBase MySQL Mode | Notes |
|-------------|----------------------|-------|
| `INT` | `INT` | Same |
| `BIGINT` | `BIGINT` | Same |
| `VARCHAR(n)` | `VARCHAR(n)` | Same |
| `NVARCHAR(n)` | `NVARCHAR(n)` | Same |
| `TEXT` | `TEXT` | Same |
| `DATETIME` | `DATETIME` | Same |
| `DATETIME2` | `DATETIME(6)` | **Convert** |
| `UNIQUEIDENTIFIER` | `VARCHAR(36)` | **Convert** |
| `BIT` | `TINYINT(1)` | **Convert** |
| `VARBINARY` | `BLOB` | **Convert** |

> **Action**: Convert `DATETIME2`, `UNIQUEIDENTIFIER`, `BIT`, `VARBINARY`.

---

## Mapping Strategy

### Step1: Generate Source DDL

```sql
-- MySQL
SELECT CONCAT('CREATE TABLE ', table_name, ' (', column_list, ');')
FROM infoation_schema.columns
WHERE table_schema = 'test_db';
```

### Step2: Convert Data Types

Use OMS (OceanBase Migration Service) — it automatically converts data types.

### Step3: Validate Converted DDL

```sql
-- OceanBase: verify table structure
DESCRIBE employees;  -- MySQL Mode

-- Oracle Mode
DESC employees;
```

---

## Common Mistakes

1. **Not converting `LONG` (Oracle)**
   - ❌ Migration fails on `LONG` column
   - ✅ `ALTER TABLE t MODIFY (col CLOB);` before migration

2. **Not converting `JSONB` (PostgreSQL)**
   - ❌ OceanBase doesn't support `JSONB`
   - ✅ Convert to `JSON` before migration

3. **Assuming `BOOLEAN` maps directly (PostgreSQL)**
   - ❌ PostgreSQL `BOOLEAN` → OceanBase error
   - ✅ Convert to `TINYINT(1)`

4. **Not validating converted DDL**
   - ❌ Assume OMS conversion is perfect
   - ✅ `DESCRIBE` / `DESC` every table after migration

5. **Forgetting character set differences**
   - ❌ Data corruption due to charset mismatch
   - ✅ Ensure `utf8mb4` on both sides

---

## OceanBase Version Notes

### V4.2 (Baseline)
- Data type mapping for all major databases
- OMS automatic conversion

### V4.4+
- More data type compatibility
- Faster OMS conversion

---

## Sources
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/data-type-mapping
- https://www.oceanbase.com/docs/oceanbase-database-cn/v4.2.5/data-type-mapping
- https://en.oceanbase.com/docs/oms/data-type-mapping
