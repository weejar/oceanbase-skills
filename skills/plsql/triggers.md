# Triggers

## Overview

Triggers automatically execute on INSERT/UPDATE/DELETE. This guide covers trigger creation, best practices, and limitations in OceanBase Oracle Mode V4.2+.

---

## Creating Trigger>

```sql
-- Oracle Mode
CREATE OR REPLACE TRIGGER trg_emp_audit
    AFTER INSERT OR UPDATE OR DELETE ON employees
    FOR EACH ROW
BEGIN
    IF INSERTING THEN
        INSERT INTO employee_audit (emp_id, action, action_date)
        VALUES (:NEW.emp_id, 'INSERT', SYSDATE);
    ELSIF UPDATING THEN
        INSERT INTO employee_audit (emp_id, action, action_date)
        VALUES (:NEW.emp_id, 'UPDATE', SYSDATE);
    ELSIF DELETING THEN
        INSERT INTO employee_audit (emp_id, action, action_date)
        VALUES (:OLD.emp_id, 'DELETE', SYSDATE);
    END IF;
END;
/
```

> **Tip**: `:NEW` = new value, `:OLD` = old value.

---

## Trigger Types>

| Type | Firing Time | Use Case |
|------|--------------|----------|
| `BEFORE EACH ROW` | Before change | validation, modify `:NEW` |
| `AFTER EACH ROW` | After change | audit logging |
| `INSTEAD OF` (view) | Instead of DML | redirect to base table |

---

## MySQL Mode Trigger>

```sql
-- MySQL Mode (different syntax)
DELIMITER //
CREATE TRIGGER trg_emp_audit
    AFTER INSERT ON employees
    FOR EACH ROW
BEGIN
    INSERT INTO employee_audit (emp_id, action, action_date)
    VALUES (NEW.emp_id, 'INSERT', NOW());
END //
DELIMITER ;
```

> **Limitation**: MySQL Mode triggers are simpler (no `INSERTING`/`UPDATING`/`DELETING`).

---

## Best Practices>

1. **Keep trigger logic short** — triggers fire for EACH ROW (performance hit if complex).
2. **Avoid triggers that modify other tables** — can cause mutating table error.
3. **Document trigger purpose** — `COMMENT ON TRIGGER` (if supported).
4. **Test trigger thoroughly** — hard to debug.

---

## Common Mistakes>

1. **Trigger modifies same table**  
   - ❌ `UPDATE employees SET ...` inside trigger on `employees`  
   - ✅ Use `BEFORE` trigger to modify `:NEW.col` instead of `UPDATE`.

2. **Not handling multi-row operations**  
   - ❌ Assumes single-row INSERT  
   - ✅ Trigger fires for EACH ROW (handles multi-row).

3. **Forgetting to enable trigger**  
   - ❌ Trigger created but not enabled  
   - ✅ `ALTER TRIGGER trg_name ENABLE;`

4. **Using `COMMIT` inside trigger**  
   - ❌ `COMMIT;` inside trigger (not allowed)  
   - ✅ Trigger runs in parent transaction (no `COMMIT`).

5. **Not testing trigger with bulk operations**  
   - ❌ Test with single row only  
   - ✅ Test with `INSERT INTO ... SELECT ...` (multi-row).

---

##OceanBase Version Notes>

### V4.2 (Baseline)>
- Trigger support (Oracle Mode & MySQL Mode)>
- `BEFORE`/`AFTER` triggers>

### V4.4+>
- Faster trigger execution>
- Improved error messaging>

---

## Sources>
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/triggers  
- https://www.oceanbase.com/docs/oceanbase-database-cn/v4.2.5/triggers  
- https://docs.oracle.com/en/database/oracle/oracle-database/21/lnpls/triggers.html  
