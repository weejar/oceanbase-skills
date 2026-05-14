# Top SQL Queries and SQL Audit

## Overview

OceanBase tracks all SQL execution in `v$sql_audit` (Oracle Mode) / `DBA_OB_SQL_AUDIT` (MySQL Mode). This view is the #1 tool for identifying performance issues. This guide covers SQL audit analysis for V4.2+.

---

## Key View: `DBA_OB_SQL_AUDIT` (MySQL Mode)

| Column | Meaning |
|--------|---------|
| `sql_id` | SQL fingerprint (normalized) |
| `query` / `sql_text` | Actual SQL text |
| `executions` | How many times executed |
| `sum_elapsed` | Total time (microseconds) |
| `avg_elapsed` | Average time per execution |
| `max_elapsed` | Single slowest execution |
| `disk_reads` | Physical I/O (expensive!) |
| `buffer_gets` | Logical reads (cache hits) |
| `rows_processed` | Rows returned |

> **Key Insight**: `disk_reads > 0` = index miss or full table scan.

---

## Finding Problem SQL

### Top 10 Slowest SQL (by Total Time)

```sql
-- MySQL Mode
SELECT sql_id,
       query,
       executions,
       round(sum_elapsed / 1000000, 2) AS total_sec,
       round(sum_elapsed / executions / 1000000, 2) AS avg_sec
FROM oceanbase.DBA_OB_SQL_AUDIT
WHERE executions > 0
ORDER BY sum_elapsed DESC
LIMIT 10;
```

### Top 10 Slowest SQL (by Average Time)

```sql
-- MySQL Mode
SELECT sql_id,
       query,
       executions,
       round(avg_elapsed / 1000000, 2) AS avg_sec
FROM oceanbase.DBA_OB_SQL_AUDIT
WHERE executions > 10   -- avoid 1-off slow queries
ORDER BY avg_elapsed DESC
LIMIT 10;
```

> **Tip**: `avg_sec` > 1.0 (1000ms) = needs tuning.

---

## Find Missing Indexes

```sql
-- SQL with high disk_reads (likely missing index)
SELECT sql_id,
       query,
       disk_reads,
       executions
FROM oceanbase.DBA_OB_SQL_AUDIT
WHERE disk_reads > 1000
ORDER BY disk_reads DESC
LIMIT 20;
```

> **Next Step**: `EXPLAIN` the SQL; if `TABLE SCAN`, add index.

---

## Oracle Mode: `v$sql_audit`

```sql
-- Same data, Oracle Mode syntax
SELECT sql_id,
       sql_text,
       executions,
       round(elapsed_time / 1000000, 2) AS total_sec
FROM v$sql_audit
ORDER BY elapsed_time DESC
LIMIT 10;
```

---

## Flushing SQL Audit (for Testing)

```sql
-- Clear audit data (for testing only!)
ALTER SYSTEM SET enable_sql_audit = 'FALSE';
ALTER SYSTEM SET enable_sql_audit = 'TRUE';
```

> **Warning**: Disabling audit clears history. Don't do this in production.

---

## MySQL Mode vs Oracle Mode

| Aspect | MySQL Mode | Oracle Mode |
|--------|-----------|-------------|
| View name | `oceanbase.DBA_OB_SQL_AUDIT` | `sys.v$sql_audit` |
| Column: SQL text | `query` | `sql_text` |
| Column: elapsed | `sum_elapsed` (microsec) | `elapsed_time` (microsec) |

---

## Common Mistakes

1. **Looking at `sum_elapsed` only**
   - ❌ Misses infrequent but slow queries
   - ✅ Also check `avg_elapsed` and `max_elapsed`

2. **Not enabling SQL audit**
   - ❌ `enable_sql_audit = FALSE` → no data
   - ✅ `ALTER SYSTEM SET enable_sql_audit = 'TRUE';`

3. **Analyzing SQL with `executions=1`**
   - ❌ Optimizing a query that ran once
   - ✅ Focus on `executions > 100`

4. **Forgetting to flush plan cache after adding index**
   - ❌ Old slow plan still used
   - ✅ `ALTER SYSTEM FLUSH PLAN CACHE;`

5. **Not setting `sql_audit_record_mode`**
   - ❌ All SQL recorded (huge overhead)
   - ✅ `ALTER SYSTEM SET sql_audit_record_mode = 'SAMPLE 10'` (10% sample)

---

## OceanBase Version Notes

### V4.2 (Baseline)
- `DBA_OB_SQL_AUDIT` view
- SQL audit enable/disable
- Plan cache integration

### V4.4+
- More granular audit columns (wait events, I/O)
- Faster audit log writing

---

## Sources
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/sql-audit
- https://www.oceanbase.com/docs/oceanbase-database-cn/v4.2.5/sql-audit
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/dba-ob-sql-audit
