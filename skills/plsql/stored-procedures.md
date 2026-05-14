# Stored Procedures (PL/SQL)

## Overview

OceanBase Oracle Mode supports PL/SQL stored procedures. This guide covers stored procedure creation, calling, and best practices for V4.2+.

---

## Creating Stored Procedure

```sql
-- Oracle Mode
CREATE OR REPLACE PROCEDURE sp_create_employee(
    p_emp_name IN VARCHAR2,
    p_dept_id IN NUMBER,
    p_emp_id OUT NUMBER
) AS
BEGIN
    INSERT INTO employees (emp_name, dept_id)
    VALUES (p_emp_name, p_dept_id)
    RETURNING emp_id INTO p_emp_id;
    COMMIT;
END;
/
```

> **Tip**: Use `/` on separate line to execute (SQL*Plus format).

---

## Calling Stored Procedure

### In SQL (Oracle Mode)

```sql
DECLARE
    v_emp_id NUMBER;
BEGIN
    sp_create_employee('Alice', 10, v_emp_id);
    DBMS_OUTPUT.PUT_LINE('Created emp_id: ' || v_emp_id);
END;
/
```

### In JDBC (Java)

```java
// CallableStatement for stored procedure
CallableStatement cs = conn.prepareCall("{call sp_create_employee(?, ?, ?)}");
cs.setString(1, "Alice");
cs.setInt(2, 10);
cs.registerOutParameter(3, Types.BIGINT);
cs.execute();
long empId = cs.getLong(3);
System.out.println("Created emp_id: " + empId);
```

---

## Parameters (IN/OUT/IN OUT)

| Type | Description |
|------|-------------|
| `IN` | Input only (default) |
| `OUT` | Output only |
| `IN OUT` | Input + output |

```sql
CREATE OR REPLACE PROCEDURE sp_get_emp_info(
    p_emp_id IN NUMBER,
    p_emp_name OUT VARCHAR2,
    p_salary IN OUT NUMBER  -- Input: current salary, Output: new salary
) AS
BEGIN
    SELECT emp_name, salary
    INTO p_emp_name, p_salary
    FROM employees
    WHERE emp_id = p_emp_id;

    p_salary := p_salary * 1.1;  -- 10% raise
END;
/
```

---

## Transaction Control

```sql
CREATE OR REPLACE PROCEDURE sp_transfer_funds(
    p_from_acct IN NUMBER,
    p_to_acct IN NUMBER,
    p_amount IN NUMBER
) AS
BEGIN
    UPDATE accounts SET balance = balance - p_amount WHERE acct_id = p_from_acct;
    UPDATE accounts SET balance = balance + p_amount WHERE acct_id = p_to_acct;
    COMMIT;  -- Commit transaction
EXCEPTION
    WHEN OTHERS THEN
        ROLLBACK;  -- Rollback on error
        RAISE;
END;
/
```

> **Best Practice**: Always include `EXCEPTION` handler.

---

## Common Mistakes

1. **Not committing in procedure**
   - ❌ `INSERT` without `COMMIT`
   - ✅ Add `COMMIT;` before `END;`

2. **Not handling exceptions**
   - ❌ No `EXCEPTION` block
   - ✅ Always add `EXCEPTION WHEN OTHERS THEN ...`

3. **Using `SELECT *` in procedure**
   - ❌ `SELECT * INTO variables FROM ...`
   - ✅ Specify columns: `SELECT col1, col2 INTO var1, var2`

4. **Not parameterizing inputs**
   - ❌ SQL injection risk
   - ✅ Use `IN` parameters (never concatenate strings)

5. **Overly complex procedure**
   - ❌ 500-line procedure (hard to maintain)
   - ✅ Break into smaller procedures

---

## MySQL Mode (Stored Procedure)

```sql
-- MySQL Mode (different syntax)
DELIMITER //
CREATE PROCEDURE sp_create_employee(
    IN p_emp_name VARCHAR(100),
    IN p_dept_id INT,
    OUT p_emp_id BIGINT
)
BEGIN
    INSERT INTO employees (emp_name, dept_id) VALUES (p_emp_name, p_dept_id);
    SET p_emp_id = LAST_INSERT_ID();
END //
DELIMITER ;
```

> **Tip**: MySQL Mode uses `DELIMITER` (different from Oracle Mode).

---

## Sources
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/stored-procedures
- https://www.oceanbase.com/docs/oceanbase-database-cn/v4.2.5/stored-procedures
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/plsql
