# Slow Query Analysis

## Overview

Identifying and fixing slow SQL is the #1 DBA task. OceanBase provides `v$sql_audit` (MySQL Mode) / `v$sql_audit` (Oracle Mode) to track SQL execution. This guide covers finding and fixing slow queries in V4.2+.

---

## Finding Slow Queries

### MySQL Mode

```sql
-- Find top 10 slowest queries
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

### Oracle Mode

```sql
-- Same data, different view name
SELECT sql_id,
       sql_text,
       executions,
       round(elapsed_time / 1000000, 2) AS total_sec
FROM v$sql_audit
ORDER BY elapsed_time DESC
LIMIT 10;
```

---

## Key Columns in `DBA_OB_SQL_AUDIT`

| Column | Meaning |
|--------|---------|
| `query` / `sql_text` | The SQL statement |
| `executions` | How many times executed |
| `sum_elapsed` | Total time (microseconds) |
| `max_elapsed` | Single slowest execution |
| `disk_reads` | Physical reads (expensive!) |
| `buffer_gets` | Logical reads |
| `rows_processed` | Rows returned |

> **Key**: `disk_reads > 0` = index miss or full table scan.

---

## Analyzing a Slow Query

### Step 1: Get the Execution Plan

```sql
-- MySQL Mode
EXPLAIN SELECT * FROM orders WHERE customer_id = 123;

-- Oracle Mode
EXPLAIN PLAN FOR SELECT * FROM orders WHERE customer_id = 123;
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
```

### Step 2: Identify the Problem

| Plan Operator | Problem | Fix |
|---------------|---------|-----|
| `TABLE SCAN` | Full table scan | Add index |
| `NESTED LOOP JOIN` | Slow for large joins | Add index to enable `HASH JOIN` |
| `SORT` | Expensive sort | Add index with `ORDER BY` columns |
| `TABLE ACCESS BY INDEX ROWID` | Index lookup + table access | Use covering index |

### Step 3: Apply Fix and Verify

```sql
-- Add index
CREATE INDEX idx_orders_customer ON orders(customer_id);

-- Flush plan cache (so new plan is used)
ALTER SYSTEM FLUSH PLAN CACHE;

-- Re-check performance
SELECT * FROM oceanbase.DBA_OB_SQL_AUDIT WHERE query LIKE '%customer_id = 123%';
```

---

## Common Slow Query Patterns

### 1. Missing Index

```sql
-- Slow: full table scan
SELECT * FROM orders WHERE customer_id = 123;

-- Fix: add index
CREATE INDEX idx_orders_customer ON orders(customer_id);
```

### 2. SELECT *

```sql
-- Slow: reads all columns + may cause table access
SELECT * FROM employees WHERE dept_id = 10;

-- Fix: select only needed columns
SELECT emp_id, emp_name FROM employees WHERE dept_id = 10;
```

### 3. N+1 Query Problem

```sql
-- Slow: N+1 queries (app code)
-- SELECT * FROM orders WHERE customer_id = 1;
-- SELECT * FROM orders WHERE customer_id = 2;
-- ... (N times)

-- Fix: single query with IN
SELECT * FROM orders WHERE customer_id IN (1, 2, 3, ..., N);
```

---

## MySQL Mode vs Oracle Mode

| Aspect | MySQL Mode | Oracle Mode |
|--------|-----------|-------------|
| Audit view | `oceanbase.DBA_OB_SQL_AUDIT` | `sys.v$sql_audit` |
| EXPLAIN | `EXPLAIN sql` | `EXPLAIN PLAN FOR sql` |

---

## Common Mistakes

1. **Not enabling SQL audit**
   - ❌ `SHOW PARAMETERS LIKE 'enable_sql_audit'` = FALSE
   - ✅ `ALTER SYSTEM SET enable_sql_audit = 'TRUE';`

2. **Analyzing wrong query**
   - ❌ `ORDER BY sum_elapsed` but query runs once
   - ✅ Also check `executions` and `avg_sec`

3. **Adding index then forgetting to flush plan cache**
   - ❌ Old slow plan still used
   - ✅ `ALTER SYSTEM FLUSH PLAN CACHE;`

4. **Not checking actual rows vs estimated rows**
   - ❌ Trusting the plan without verifying
   - ✅ Run query and check actual rows vs `EST. ROWS`

5. **Fixing SQL without testing on staging**
   - ❌ Adding index directly on production
   - ✅ Test index creation + query perf on staging first

---

## OceanBase Version Notes

### V4.2 (Baseline)
- `DBA_OB_SQL_AUDIT` view
- Plan cache flush
- EXPLAIN analysis

### V4.4+
- More detailed audit columns (wait events, I/O timings)
- Faster plan cache flush

---

## Sources
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/slow-query
- https://www.oceanbase.com/docs/oceanbase-database-cn/v4.2.5/slow-query
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/sql-audit
