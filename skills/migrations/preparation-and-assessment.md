# Migration Preparation and Assessment

## Overview

Proper preparation and assessment prevent surprises during migration. This guide covers assessment tools, compatibility checks, and preparation steps for migrating to OceanBase V4.2+.

---

## Assessment Tools

| Tool | Purpose | Use Case |
|------|---------|----------|
| **OMAT** | Offline compatiblity check | Pre-migration assessment |
| **OMS** | Online data + schema migration | Actual migration |
| `infoamation_schema` | Schema comparison | Manual verification |

---

## OMAT (OceanBase Migration Assessment Tool)

### Running OMAT

```bash
# MySQL → OceanBase
omat assess --source-mysql-host 10.0.0.1 --source-mysql-port 3306 \
       --source-db test_db \
       --target-ob-host 10.0.0.100 --target-ob-port 2881

# Oracle → OceanBase
omat assess --source-oracle-host 10.0.0.1 --source-oracle-port 1521 \
       --source-oracle-service testdb \
       --target-ob-host 10.0.0.100 --target-ob-port 2881
```

### OMAT Report Includes

| Section | Description |
|---------|-------------|
| **Compatibility Score** | % of objects compatible |
| **Incompatible Objects** | List of objects needing rewrite |
| **Data Type Mapping** | Source → OceanBase type mapping |
| **SQL Syntax Issues** | Incompatible SQL syntax |
| **Recommended Actions** | Steps to fix issues |

---

## Manual Assessment (SQL)

### Check Table Structures

```sql
-- MySQL source
SELECT table_name, column_name, data_type, is_nullable
FROM infoation_schema.columns
WHERE table_schema = 'test_db'
ORDER BY table_name, ordinal_position;

-- OceanBase target (after schema migration)
SELECT table_name, column_name, data_type, is_nullable
FROM infoation_schema.columns
WHERE table_schema = 'test_db'
ORDER BY table_name, ordinal_position;
```

### Check Indexes

```sql
-- Compare indexes
SHOW INDEX FROM employees;
```

---

## Pre-Migration Checklist

| # | Item | Check |
|---|------|-------|
| 1 | OMAT report reviewed | All incompatible objects addressed |
| 2 | OMS task created (staging) | Test migration successful |
| 3 | Data type mapping verified | No data loss |
| 4 | SQL syntax differences fixed | App works with OceanBase |
| 5 | Performance baseline captured | Can compare after migration |
| 6 | Rollback plan documented | Can revert if needed |
| 7 | Staging migration tested | 3 successful test runs |

---

## Common Mistakes

1. **Not running OMAT**
   - ❌ Surprise incompatibilities during cutover
   - ✅ Run OMAT first; fix all incompatible objects

2. **Skipping staging migration**
   - ❌ First migration attempt is production
   - ✅ Test migration in staging 3 times

3. **Not capturing performance baseline**
   - ❌ Can't verify post-migration performance
   - ✅ Capture source DB performance metrics before migration

4. **Forgetting to test app with OceanBase**
   - ❌ App fails after cutover
   - ✅ Test app with OceanBase in staging

5. **No rollback plan**
   - ❌ Can't revert if migration fails
   - ✅ Document rollback steps (reverse replication)

---

## Assessment Report Template

```
# Migration Assessment Report

## Source: MySQL 5.7
## Target: OceanBase 4.2 (MySQL Mode)

### Compatibility: 92%
### Incompatible Objects: 3
1. `get_emp_list` function (uses `GROUP_CONCAT` with `ORDER BY` — needs rewrite)
2. `employee_audit` trigger (uses `OLD`/`NEW` — compatible, but test)
3. `dept_stats` view (uses `WITH ROLLUP` — compatible)

### Data Type Mapping: No issues
### Recommended Actions:
1. Test function `get_emp_list` after migration
2. Verify trigger `employee_audit` works
3. Validate view `dept_stats`
```

---

## OceanBase Version Notes

### V4.2 (Baseline)
- OMAT supported
- OMS for MySQL/Oracle/PostgreSQL/DB2/SQL Server

### V4.4+
- More source database support
- Faster assessment

---

## Sources
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/migration-assessment
- https://www.oceanbase.com/docs/oceanbase-database-cn/v4.2.5/migration-assessment
- https://en.oceanbase.com/docs/oms/assessment
