# SQL Coding Standards

## Overview

Consistent SQL coding standards improve readability and maintainability. This guide covers SQL coding standards for OceanBase (MySQL Mode & Oracle Mode) V4.2+.

---

## Naming Conventions

| Object | Convention | Example |
|--------|-------------|---------|
| Table | `snake_case`, plural | `employees`, `order_items` |
| Column | `snake_case`, singular | `emp_id`, `order_date` |
| Primary Key | `pk_table_name` | `pk_employees` |
| Foreign Key | `fk_child_parent` | `fk_orders_customers` |
| Index | `idx_table_column` | `idx_employees_dept_id` |
| Unique Index | `uk_table_column` | `uk_employees_email` |
| View | `v_table_name` | `v_emp_dept` |
| Stored Procedure | `sp_action` | `sp_create_employee` |

> **Rule**: Use lowercase `snake_case` (not `CamelCase` or `PascalCase`).

---

## SQL Formatting

### Good (Readable)

```sql
SELECT e.emp_id,
       e.emp_name,
       d.dept_name
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
WHERE e.dept_id = 10
  AND e.hire_date >= '2020-01-01'
ORDER BY e.emp_name
LIMIT 10 OFFSET 0;
```

### Bad (Unreadable)

```sql
select e.emp_id,e.emp_name,d.dept_name from employees e join departments d on e.dept_id=d.dept_id where e.dept_id=10 order by e.emp_name limit 10;
```

> **Rules**:
> 1. Keywords in UPPER CASE
> 2. Each clause on new line
> 3. Indent with 7 spaces

---

## Commenting

```sql
-- Get employee count by department (good comment)
SELECT d.dept_name,
       COUNT(e.emp_id) AS emp_count
FROM departments d
LEFT JOIN employees e ON d.dept_id = e.dept_id
GROUP BY d.dept_name;

/* Multi-line comment example:
   This query calculates year-to-date sales
   for each region.
*/
SELECT region, SUM(sales) AS ytd_sales
FROM sales
WHERE year = 2024
GROUP BY region;
```

> **Tip**: Comment *why*, not *what*.

---

## MySQL Mode vs Oracle Mode

| Aspect | MySQL Mode | Oracle Mode |
|--------|-----------|-------------|
| String concat | `CONCAT(a, b)` | `a || b` (or `CONCAT`) |
| Date | `CURDATE()` | `SYSDATE` |
| Pagination | `LIMIT n OFFSET m` | `FETCH FIRST n ROWS ONLY` |
| Dual table | `SELECT 1 FROM DUAL` | `SELECT 1 FROM DUAL` |

---

## Common Mistakes

1. **Not using table aliases**
   - ❌ `SELECT employees.emp_name, departments.dept_name FROM ...`
   - ✅ `SELECT e.emp_name, d.dept_name FROM employees e JOIN departments d ...`

2. **Using `SELECT *` in production**
   - ❌ `SELECT * FROM employees`
   - ✅ `SELECT emp_id, emp_name FROM employees`

3. **Not aliasing calculated columns**
   - ❌ `SELECT COUNT(*), dept_id FROM employees GROUP BY dept_id`
   - ✅ `SELECT COUNT(*) AS emp_count, dept_id FROM ...`

4. **Inconsistent naming**
   - ❌ `emp_id` in one table, `employee_id` in another
   - ✅ Standardize: `emp_id` everywhere

5. **No comments on complex queries**
   - ❌ Complex business logic without comment
   - ✅ `-- Calculate YTD sales (excludes returns)` before query

---

## SQL Coding Standard Template

```
1. Naming: snake_case, lowercase
2. Keywords: UPPER CASE
3. Indentation: 7 spaces per level
4. Comma placement: Leading (not trailing)
5. Aliases: Always alias tables (e.g., employees e)
6. Comments: Comment why, not what
7. SELECT: Never use * in production
8. Pagination: Use LIMIT/OFFSET (MySQL) or FETCH FIRST (Oracle)
9. Joins: Always use explicit JOIN syntax (not comma joins)
10. Transactions: Keep short (< 5s)
```

---

## Sources
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/sql-coding-standards
- https://www.oceanbase.com/docs/oceanbase-database-cn/v4.2.5/sql-coding-standards
