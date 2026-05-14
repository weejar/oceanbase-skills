# Query Rewrite and Optimization

## Overview

Rewriting queries can dramatically improve performance. This guide covers common query rewrite patterns for OceanBase V4.2+.

---

## Rewrite 1: OR → UNION (Index Usage)

### Bad (Index Not Used)

```sql
-- Bad: OR prevents index usage
SELECT * FROM employees
WHERE dept_id = 10 OR hire_date < '2020-01-01';
```

### Good (Rewritten)

```sql
-- Good: UNION allows index usage
SELECT * FROM employees WHERE dept_id = 10
UNION ALL
SELECT * FROM employees WHERE hire_date < '2020-01-01';
```

> **Result**: Each part can use index (faster).

---

## Rewrite 2: NOT IN → NOT EXISTS

### Bad (NULL Handling Issue)

```sql
-- Bad: NOT IN fails if subquery returns NULL
SELECT * FROM employees
WHERE dept_id NOT IN (SELECT dept_id FROM departments WHERE status = 'INACTIVE');
```

### Good (NOT EXISTS)

```sql
-- Good: NOT EXISTS handles NULL correctly
SELECT * FROM employees e
WHERE NOT EXISTS (
    SELECT 1 FROM departments d
    WHERE d.dept_id = e.dept_id
      AND d.status = 'INACTIVE'
);
```

---

## Rewrite 3: Dependent Subquery → JOIN

### Bad (Dependent Subquery)

```sql
-- Bad: subquery executed for EACH row (slow)
SELECT * FROM employees e
WHERE salary > (
    SELECT AVG(salary) FROM employees
    WHERE dept_id = e.dept_id
);
```

### Good (JOIN)

```sql
-- Good: JOIN, executed once
SELECT e.* FROM employees e
JOIN (
    SELECT dept_id, AVG(salary) AS avg_sal
    FROM employees
    GROUP BY dept_id
) d ON e.dept_id = d.dept_id
WHERE e.salary > d.avg_sal;
```

---

## Rewrite 4: OFFSET Pagination → Keyset Pagination

### Bad (Deep Pagination)

```sql
-- Bad: OFFSET 100000 (slow)
SELECT * FROM employees
ORDER BY emp_id
LIMIT 20 OFFSET 100000;
```

### Good (Keyset Pagination)

```sql
-- Good: use last ID (fast, even for deep pages)
SELECT * FROM employees
WHERE emp_id > 100000
ORDER BY emp_id
LIMIT 20;
```

> **Result**: Keysit pagination is O(1) (no OFFSET scan).

---

## Rewrite 5: SELECT * → Specify Columns>

### Bad>

```sql
-- Bad: loads all columns (slow)
SELECT * FROM employees WHERE dept_id = 10;
```

### Good>

```sql
-- Good: only needed columns
SELECT emp_id, emp_name, hire_date
FROM employees WHERE dept_id = 10;
```

> **Result**: Less I/O; may use covering index.

---

## MySQL Mode vs Oracle Mode>

| Rewrite | MySQL Mode | Oracle Mode |
|--------|-----------|-------------|
| `OR → UNION` | ✅ | ✅ |
| `NOT IN → NOT EXISTS` | ✅ | ✅ |
| `Dependent subquery → JOIN` | ✅ | ✅ |
| `OFFSET → keyset` | ✅ | ✅ |
| `SELECT * → columns` | ✅ | ✅ |

---

## Common Mistakes>

1. **Not rewriting `OFFSET` for deep pages**
   - ❌ `OFFSET 100000` (slow)
   - ✅ `WHERE id > :last_id LIMIT 20`

2. **Using `NOT IN` with nullable column**
   - ❌ `WHERE col NOT IN (SELECT ...)` (NULL issue)
   - ✅ Rewrite to `NOT EXISTS`

3. **Not rewriting `OR` to `UNION`**
   - ❌ `WHERE a = 1 OR b = 2` (index not used)
   - ✅ `SELECT ... WHERE a=1 UNION ALL SELECT ... WHERE b=2`

4. **Using dependent subquery**
   - ❌ Correlated subquery (executed per row)
   - ✅ Rewrite to JOIN

5. **Using `SELECT *` in production**
   - ❌ Loads all columns
   - ✅ Specify needed columns>

---

## OceanBase Version Notes>

### V4.2 (Baseline)
- Query rewrite support
- Optimizer improvements>

### V4.4+
- More aggressive query rewrite
- Better optimizer choices>

---

## Sources>
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/query-rewrite
- https://www.oceanbase.com/docs/oceanbase-database-cn/v4.2.5/query-rewrite
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/optimization
