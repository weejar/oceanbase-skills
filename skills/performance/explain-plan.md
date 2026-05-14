# EXPLAIN Execution Plan Analysis

## Overview

`EXPLAIN` shows how OceanBase executes a SQL statement. Understanding execution plans is the foundation of SQL tuning. This guide covers reading EXPLAIN output, identifying bottlenecks, and optimizing query plans in OceanBase V4.2+.

---

## Basic EXPLAIN

```sql
-- MySQL Mode
EXPLAIN SELECT * FROM employees WHERE dept_id = 10;

-- Oracle Mode
EXPLAIN PLAN FOR SELECT * FROM employees WHERE dept_id = 10;
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
```

### Sample Output

```
| ID | OPERATOR | NAME    | EST. ROWS | COST |
|----|-----------|---------|-----------|------|
| 0  | TABLE GET | employees | 1         | 1    |
```

---

## Reading the Plan

| Column | Meaning |
|--------|---------|
| `ID` | Step number (0 = root) |
| `OPERATOR` | What operation (TABLE GET, TABLE SCAN, HASH JOIN, etc.) |
| `NAME` | Object name (table, index) |
| `EST. ROWS` | Estimated rows processed |
| `COST` | Estimated CPU cost |

> **Key**: Higher `COST` = more expensive operation.

---

## Common Operators

| Operator | Meaning | Tuning |
|----------|---------|---------|
| `TABLE GET` | Primary key lookup (fast!) | Ideal for OLTP |
| `TABLE SCAN` | Full table scan (slow) | Add index |
| `INDEX SCAN` | Index lookup | Good, ensure covering index |
| `HASH JOIN` | Hash join | Acceptable for large joins |
| `NLJOIN` | Nested loop join (slow) | Rewrite or add indexes |
| `SORT` | Sorting | Add index to avoid sort |
| `MATERIAL` | Materialization | May indicate temp data |

---

## Finding Missing Indexes

```sql
-- Find slow queries (MySQL Mode)
SELECT query,
       sum_elapsed,
       executions,
       round(sum_elapsed / executions) AS avg_ms
FROM oceanbase.DBA_OB_SQL_AUDIT
ORDER BY sum_elapsed DESC
LIMIT 10;

-- EXPLAIN the slow query
EXPLAIN SELECT * FROM orders WHERE customer_id = 123;
-- If "TABLE SCAN", add index on customer_id
```
```sql
CREATE INDEX idx_orders_customer ON orders(customer_id);
```

---

## Plan Cache (Reuse Execution Plan)

OceanBase caches execution plans in **Plan Cache**.

```sql
-- View plan cache
SELECT * FROM oceanbase.DBA_OB_PLAN_CACHE_STAT
WHERE tenant_id = (SELECT tenant_id FROM oceanbase.DBA_OB_TENANTS WHERE tenant_name='app_tenant');

-- Clear plan cache (after adding index)
ALTER SYSTEM FLUSH PLAN CACHE;
```

> **Tip**: After adding an index, flush plan cache so new plans are generated.

---

## MySQL Mode vs Oracle Mode

| Aspect | MySQL Mode | Oracle Mode |
|--------|-----------|-------------|
| EXPLAIN syntax | `EXPLAIN sql` | `EXPLAIN PLAN FOR sql` + `SELECT * FROM TABLE(...)` |
| System view | `oceanbase.DBA_OB_SQL_AUDIT` | `SYS.DBA_OB_SQL_AUDIT` |

---

## Common Mistakes

1. **Not checking EST. ROWS vs actual rows**
   - ❌ Trusting the plan without verifying
   - ✅ Run query and check actual rows vs estimate

2. **Using `SELECT *` in production**
   - ❌ `TABLE SCAN` on large table
   - ✅ Select only needed columns; add covering index

3. **Not flushing plan cache after adding index**
   - ❌ Old slow plan still used
   - ✅ `ALTER SYSTEM FLUSH PLAN CACHE;`

4. **Ignoring `NLJOIN` (nested loop)**
   - ❌ Slow for large joins
   - ✅ Rewrite query or add indexes to enable `HASH JOIN`

5. **Not updating statistics**
   - ❌ Outdated stats → bad plan
   - ✅ `CALL DBMS_STATS.GATHER_TABLE_STATS(...)` (Oracle Mode) or `ANALYZE TABLE` (MySQL Mode)

---

## OceanBase Version Notes

### V4.2 (Baseline)
- EXPLAIN supported
- Plan Cache management
- `DBA_OB_SQL_AUDIT` view

### V4.4+
- More detailed plan output (operator timings)
- Improved cardinality estimation

---

## Sources
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/explain
- https://www.oceanbase.com/docs/oceanbase-database-cn/v4.2.5/explain
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/execution-plan
