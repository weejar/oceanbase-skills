# DDL Best Practices

## Overview

Data Definition Language (DDL: CREATE, ALTER, DROP) affects schema and performance. This guide covers DDL best practices for OceanBase V4.2+.

---

## CREATE TABLE Best Practices

### Specify Column Constraints Explicitly

```sql
-- Good: explicit constraints
CREATE TABLE employees (
    emp_id INT NOT NULL AUTO_INCREMENT,
    emp_name VARCHAR(100) NOT NULL,
    dept_id INT,
    salary DECIMAL(10,2) DEFAULT 0.00,
    hire_date DATE DEFAULT CURDATE(),
    PRIMARY KEY (emp_id),
    FOREIGN KEY (dept_id) REFERENCES departments(dept_id)
) COMMENT='Employee master table';
```

> **Tip**: Always add `COMMENT` for maintainability.

### Choose Correct Data Types

| Scenario | Recommended | Avoid |
|-----------|-------------|-------|
| Primary key | `BIGINT` (large table) | `INT` (overflow risk) |
| Short string | `VARCHAR(n)` | `TEXT` (no index prefix) |
| Large text | `TEXT` / `CLOB` | `VARCHAR(4000)` (too short) |
| Decimal | `DECIMAL(p,s)` | `FLOAT` (precision loss) |
| Date | `DATE` / `DATETIME` | `VARCHAR` (no date functions) |

---

## ALTER TABLE Best Practices

### Add Column (Online, Non-Blocking)

```sql
-- Good: add column (OceanBase supports online DDL)
ALTER TABLE employees ADD COLUMN phone VARCHAR(20);
```

> **OceanBase**: Most `ALTER TABLE` operations are online (non-blocking).

### Add Index (Online)

```sql
-- Good: online add index
ALTER TABLE employees ADD INDEX idx_emp_name (emp_name);
```

> **Tip**: OceanBase supports online index creation (no table lock).

### Avoid Dropping Columns (If Possible)

```sql
-- Alternative: mark as unused (logical delete)
ALTER TABLE employees MODIFY COLUMN old_col SET UNUSED;
```

---

## DROP TABLE Best Practices

### Use TRUNCATE Instead of DELETE (Empty Table)

```sql
-- Good: TRUNCATE (instant, no undo)
TRUNCATE TABLE temp_table;

-- Bad: DELETE (slow, generates undo)
DELETE FROM temp_table;
```

### Drop Table with Caution

```sql
-- Good: double-check before drop
SELECT COUNT(*) FROM employees;  -- Verify table has data?
-- Then:
DROP TABLE employees;
```

---

## INDEX Best Practices (DDL)

### Create Index with Correct Column Order

```sql
-- Good: index on (dept_id, hire_date) for this query
CREATE INDEX idx_dept_hire ON employees(dept_id, hire_date);

-- Query benefiting:
SELECT * FROM employees
WHERE dept_id = 10
ORDER BY hire_date;
```

> **Rule**: Most selective column first (for composite index).

### Avoid Over-Index

| Table Size | Recommended Index Count |
|-------------|----------------------------|
| < 10,000 rows | 0-1 (full scan is faster) |
| 10K-1M rows | 1-3 |
| > 1M rows | 3-5 (max) |

---

## Partitioning (DDL)

### When to Partition

| Scenario | Partition Strategy |
|-----------|----------------------|
| Time-series data | Range partition (by month) |
| Large table (> 1TB) | Hash partition |
| Fast delete by range | Range partition (drop partition) |

```sql
-- Example: range partition by month
CREATE TABLE orders (
    order_id BIGINT,
    order_date DATE,
    ...
) PARTITION BY RANGE (order_date) (
    PARTITION p202401 VALUES LESS THAN ('2024-02-01'),
    PARTITION p202402 VALUES LESS THAN ('2024-03-01')
);
```

---

## Common Mistakes

1. **Using `FLOAT` for monetary values**
   - ❌ `salary FLOAT(10,2)` (precision loss)
   - ✅ `salary DECIMAL(10,2)`

2. **Not adding `COMMENT` to columns/table**
   - ❌ `CREATE TABLE t (id INT);` (no comment)
   - ✅ `CREATE TABLE t (id INT COMMENT 'Primary key');`

3. **Over-indexing ( > 5 indexes per table)**
   - ❌ 10 indexes on `orders` (slow writes)
   - ✅ 3-5 strategic indexes`

4. **Using `DELETE` to empty large table**
   - ❌ `DELETE FROM huge_table;` (slow, generates undo)
   - ✅ `TRUNCATE TABLE huge_table;`

5. **Not using partitioning for large tables**
   - ❌ 10TB table, no partition (maintenance nightmare)
   - ✅ Range partition by month (easy to drop old data)

---

## MySQL Mode vs Oracle Mode

| Aspect | MySQL Mode | Oracle Mode |
|--------|-----------|-------------|
| AUTO_INCREMENT | ✅ | ❌ (use SEQUENCE) |
| SEQUENCE | ❌ | ✅ |
| PARTITION BY | ✅ | ✅ |

---

## Sources
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/ddl-best-practices
- https://www.oceanbase.com/docs/oceanbase-database-cn/v4.2.5/ddl-best-practices
