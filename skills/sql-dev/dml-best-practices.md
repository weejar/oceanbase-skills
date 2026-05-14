# DML Best Practices

## Overview

Data Manipulation Language (DML: INSERT, UPDATE, DELETE, SELECT) performance depends on correct usage. This guide covers DML best practices for OceanBase V4.2+.

---

## INSERT Best Practices

### Use Batch Insert

```sql
-- Good: batch insert (MySQL syntax)
INSERT INTO employees (emp_name, dept_id) VALUES
    ('Alice', 10),
    ('Bob', 20),
    ('Charlie', 30);

-- Bad: single insert in loop (N round-trips)
INSERT INTO employees (emp_name, dept_id) VALUES ('Alice', 10);
INSERT INTO employees (emp_name, dept_id) VALUES ('Bob', 20);
```

> **Performance**: Batch is ~10x faster.

### Use PreparedStatement (in App)

```java
// Good: PreparedStatement (rewriteBatchedStatements=true)
String sql = "INSERT INTO employees (emp_name, dept_id) VALUES (?, ?)";
PreparedStatement ps = conn.prepareStatement(sql);
for (Employee emp : list) {
    ps.setString(1, emp.getName());
    ps.setInt(2, emp.getDeptId());
    ps.addBatch();
}
ps.executeBatch();
```

---

## UPDATE Best Practices

### Use Index for WHERE Clause

```sql
-- Good: WHERE uses index
UPDATE employees SET salary = salary * 1.1 WHERE emp_id = 123;

-- Bad: Full table scan
UPDATE employees SET salary = salary * 1.1 WHERE emp_name = 'Alice';
```

> **Fix**: Add index on `emp_name` if frequently used in WHERE.

### Limit Affected Rows

```sql
-- Good: Limit updates
UPDATE employees SET salary = salary * 1.1
WHERE dept_id = 10
LIMIT 100;  -- MySQL syntax (OceanBase MySQL Mode)
```

---

## DELETE Best Practices

### Use Batched DELETE

```sql
-- Good: Batched delete (avoid long transaction)
DELETE FROM employees WHERE dept_id = 10 LIMIT 1000;

-- Repeat until 0 rows affected
```

> **Warning**: `DELETE FROM huge_table` (no WHERE) locks entire table.

### Use TRUNCATE for Empty Table

```sql
-- Good: TRUNCATE (instant, no undo)
TRUNCATE TABLE temp_table;

-- Bad: DELETE (slow, generates undo)
DELETE FROM temp_table;
```

---

## SELECT Best Practices

### Use Covering Index

```sql
-- Good: Covering index (index only, no table access)
CREATE INDEX idx_emp_dept_covering ON employees(dept_id) INCLUDE (emp_name, salary);

SELECT emp_name, salary FROM employees WHERE dept_id = 10;
-- Uses index only (fast)
```

### Avoid SELECT *

```sql
-- Bad: SELECT *
SELECT * FROM employees WHERE dept_id = 10;

-- Good: specify columns
SELECT emp_id, emp_name, hire_date FROM employees WHERE dept_id = 10;
```

### Use Pagination (LIMIT/OFFSET)

```sql
-- Good: Pagination
SELECT emp_id, emp_name FROM employees
ORDER BY emp_id
LIMIT 20 OFFSET 0;  -- Page 1

SELECT emp_id, emp_name FROM employees
ORDER BY emp_id
LIMIT 20 OFFSET 20;  -- Page 2
```

> **Deep pagination**: Use keyset pagination for `OFFSET > 10000`.

---

## Common Mistakes

1. **Not using batch INSERT**
   - ❌ Single INSERT per loop iteration
   - ✅ Use multi-value INSERT or `addBatch()`

2. **DELETE without LIMIT**
   - ❌ `DELETE FROM table WHERE ...` (locks many rows)
   - ✅ `DELETE ... LIMIT 1000` (batched)

3. **Using SELECT *** 
   - ❌ Loads all columns (slow)
   - ✅ Specify needed columns

4. **Not using covering index**
   - ❌ Index lookup + table access
   - ✅ `INCLUDE` columns in index

5. **Deep pagination with OFFSET**
   - ❌ `LIMIT 20 OFFSET 100000` (slow)
   - ✅ Keyset pagination: `WHERE id > :last_id LIMIT 20`

---

## MySQL Mode vs Oracle Mode

| Aspect | MySQL Mode | Oracle Mode |
|--------|-----------|-------------|
| Batch insert | `VALUES (),(),()` | Same |
| Pagination | `LIMIT n OFFSET m` | `OFFSET m ROWS FETCH FIRST n ROWS ONLY` |
| TRUNCATE | Supported | Supported |

---

## Sources
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/dml-best-practices
- https://www.oceanbase.com/docs/oceanbase-database-cn/v4.2.5/dml-best-practices
