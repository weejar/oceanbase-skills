# Packages (PL/SQL)

## Overview

Packages group related PL/SQL objects (procedures, functions, variables). This guide covers package creation, usage, and best practices in OceanBase Oracle Mode V4.2+.

---

## Package Structure

A package has **2 parts**:
1. **Specification** (spec) — public interface
2. **Body** — implementation

```sql
-- 1. Package Specification (public)
CREATE OR REPLACE PACKAGE pkg_employee AS
    -- Public constant
    c_STATUS_ACTIVE CONSTANT VARCHAR2(10) := 'ACTIVE';

    -- Public procedure
    PROCEDURE hire_emp(
        p_name IN VARCHAR2,
        p_dept_id IN NUMBER
    );

    -- Public function
    FUNCTION get_emp_count(p_dept_id IN NUMBER) RETURN NUMBER;
END pkg_employee;
/

-- 2. Package Body (private implementation)
CREATE OR REPLACE PACKAGE BODY pkg_employee AS
    -- Private variable
    g_hire_count NUMBER := 0;

    -- Procedure implementation
    PROCEDURE hire_emp(
        p_name IN VARCHAR2,
        p_dept_id IN NUMBER
    ) AS
        v_emp_id NUMBER;
    BEGIN
        INSERT INTO employees (emp_name, dept_id)
        VALUES (p_name, p_dept_id)
        RETURNING emp_id INTO v_emp_id;

        g_hire_count := g_hire_count + 1;
        COMMIT;
    END hire_emp;

    -- Function implementation
    FUNCTION get_emp_count(p_dept_id IN NUMBER) RETURN NUMBER IS
        v_count NUMBER;
    BEGIN
        SELECT COUNT(*) INTO v_count
        FROM employees
        WHERE dept_id = p_dept_id;
        RETURN v_count;
    END get_emp_count;
END pkg_employee;
/
```

> **Key**: Private variables/procedures (not in spec) are hidden from outside.

---

## Calling Package Subprograms

```sql
-- Call procedure
BEGIN
    pkg_employee.hire_emp('Alice', 10);
END;
/

-- Call function
DECLARE
    v_count NUMBER;
BEGIN
    v_count := pkg_employee.get_emp_count(10);
    DBMS_OUTPUT.PUT_LINE('Count: ' || v_count);
END;
/
```

---

## Package State (Persistence)

Package variables retain value for **session duration**:

```sql
-- Session 1
BEGIN
    pkg_employee.hire_emp('Alice', 10);
    -- g_hire_count = 1 (session state)
END;
/

-- Session 2 (different session)
BEGIN
    -- g_hire_count = 0 (fresh state)
    NULL;
END;
/
```

> **Use Case**: Session-level caching.

---

## Best Practices

| Practice | Reason |
|-----------|--------|
| Separate spec and body | Encapsulation |
| Use private subprograms | Hide implementation |
| Initialize variables in body | Avoid NULL errors |
| Document public API | Maintainability |
| Overload procedures (V4.2+) | Flexible API |

---

## Overloading (V4.2+)

```sql
CREATE OR REPLACE PACKAGE pkg_math AS
    FUNCTION calculate(a NUMBER, b NUMBER) RETURN NUMBER;
    FUNCTION calculate(a NUMBER, b NUMBER, c NUMBER) RETURN NUMBER;  -- Overload
END pkg_math;
/
```

> **Use Case**: Same operation, different parameters.

---

## Common Mistakes

1. **Not compiling body after spec**
   - ❌ Compile spec, forget body → `PACKAGE BODY does not exist`
   - ✅ Always compile body after spec

2. **Using package variables for transactional data**
   - ❌ `g_total_sales` (lost on session disconnect)
   - ✅ Store in table (persistent)

3. **Overloading with same parameter types**
   - ❌ `PROCEDURE p(a NUMBER); PROCEDURE p(a NUMBER);` (compile error)
   - ✅ Different parameter types/counts

4. **Not handling exceptions in package**
   - ❌ No `EXCEPTION` block
   - ✅ Add `WHEN OTHERS THEN ...` in each subprogram

5. **Assuming package state persists across sessions**
   - ❌ Expect `g_hire_count` to persist after disconnect
   - ✅ Package state is session-scoped (per-session)

---

## MySQL Mode (No Packages)

MySQL Mode does **not** support packages. Use separate stored procedures instead:

```sql
-- MySQL Mode: separate procedures
CREATE PROCEDURE hire_emp(...)
BEGIN
    ...
END;

CREATE FUNCTION get_emp_count(...)
RETURNS NUMBER
BEGIN
    ...
END;
```

---

## OceanBase Version Notes

### V4.2 (Baseline)
- Package support (Oracle Mode)
- Overloading
- Session-level package state

### V4.4+
- Faster package compilation
- Improved dependency tracking

---

## Sources
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/packages
- https://www.oceanbase.com/docs/oceanbase-database-cn/v4.2.5/packages
- https://docs.oracle.com/en/database/oracle/oracle-database/21/lnpls/packages.html
