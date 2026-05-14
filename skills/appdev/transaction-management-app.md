# Transaction Management (Application)

## Overview

Transactions must be managed correctly in applications to ensure data consistency. This guide covers transaction management in Java/Python/Go for OceanBase V4.2+.

---

## Java (Spring @Transactional)

```java
@Service
public class EmployeeService {
    @Autowired
    private EmployeeRepository repo;

    @Transactional  // Start transaction
    public void createWithDependents(Employee emp, List<Dependent> deps) {
        repo.save(emp);
        for (Dependent d : deps) {
            d.setEmpId(emp.getEmpId());
            dependentRepo.save(d);
        }
        // Auto-commit if no exception; rollback if exception
    }

    @Transactional(readOnly = true)  // Read-only transaction
    public Employee getById(Long id) {
        return repo.findById(id).orElse(null);
    }
}
```

| Propagation | Description |
|-------------|-------------|
| `REQUIRED` (default) | Use existing or create new |
| `REQUIRES_NEW` | Always create new (suspend current) |
| `SUPPORTS` | Use if exists, else non-transactional |
| `NOT_SUPPORTED` | Execute non-transactional |

---

## Java (Manual Transaction)

```java
Connection conn = ds.getConnection();
try {
    conn.setAutoCommit(false);  // Start transaction

    PreparedStatement ps1 = conn.prepareStatement(
        "INSERT INTO employees (emp_name, dept_id) VALUES (?, ?)");
    ps1.setString(1, "Alice");
    ps1.setInt(2, 10);
    ps1.executeUpdate();

    PreparedStatement ps2 = conn.prepareStatement(
        "UPDATE departments SET emp_count = emp_count + 1 WHERE dept_id = ?");
    ps2.setInt(1, 10);
    ps2.executeUpdate();

    conn.commit();  // Commit transaction
} catch (Exception e) {
    conn.rollback();  // Rollback on error
    throw e;
} finally {
    conn.setAutoCommit(true);
    conn.close();
}
```

---

## Python (Exception-Based Rollback)

```python
import pymysql

conn = pymysql.connect(...)

try:
    with conn.cursor() as cursor:
        cursor.execute("INSERT INTO employees (emp_name, dept_id) VALUES (%s, %s)",
                      ('Alice', 10))
        cursor.execute("UPDATE departments SET emp_count = emp_count + 1 WHERE dept_id = %s",
                      (10,))
    conn.commit()  # Commit transaction
except Exception as e:
    conn.rollback()  # Rollback on error
    raise
finally:
    conn.close()
```

> **Tip**: Use `with` statement for automatic cleanup.

---

## Go (Explicit Transaction)

```go
tx, err := db.Begin()  // Start transaction
if err != nil {
    panic(err)
}

// Defer rollback (if commit not called)
defer tx.Rollback()

// Execute statements
_, err = tx.Exec("INSERT INTO employees (emp_name, dept_id) VALUES (?, ?)",
    "Alice", 10)
if err != nil {
    panic(err)
}

_, err = tx.Exec("UPDATE departments SET emp_count = emp_count + 1 WHERE dept_id = ?",
    10)
if err != nil {
    panic(err)
}

// Commit transaction
err = tx.Commit()
if err != nil {
    panic(err)
}
```

---

## Isolation Levels (OceanBase)

| Level | OceanBase Support | Use Case |
|-------|-------------------|----------|
| `READ UNCOMMITTED` | ❌ | - |
| `READ COMMITTED` | ✅ (default) | Most OLTP |
| `REPEATABLE READ` | ✅ | Consistent read |
| `SERIALIZABLE` | ✅ | Strict consistency |

```java
// Set isolation level (Java)
conn.setTransactionIsolation(Connection.TRANSACTION_READ_COMMITTED);
```

---

## Common Mistakes

1. **`@Transactional` on private methods (Spring)**
   - ❌ Spring AOP doesn't intercept private methods
   - ✅ Use public methods

2. **Long-running transactions**
   - ❌ Transaction open for 10 minutes (locks held)
   - ✅ Keep transactions short (< 5s)

3. **Not handling deadlocks**
   - ❌ Assume transaction always succeeds
   - ✅ Catch deadlock error (err 1213) and retry

4. **`readOnly=true` for writes (Spring)**
   - ❌ `@Transactional(readOnly = true)` for INSERT
   - ✅ Remove `readOnly=true` for writes

5. **Not setting `setAutoCommit(false)` (Java)**
   - ❌ Each statement auto-commit (slow, no atomicity)
   - ✅ `conn.setAutoCommit(false);` for multi-statement transactions

---

## OceanBase-Specific Notes

1. **Deadlock detection**: OceanBase detects deadlocks and rolls back one transaction (err 1213).
2. **Optimistic locking**: Use `SELECT ... FOR UPDATE` for pessimistic locking.
3. **Batch + transaction**: Combine batch operations in one transaction (fewer commits).

---

## Sources
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/transactions
- https://www.oceanbase.com/docs/oceanbase-database-cn/v4.2.5/transactions
- https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/annotation/Transactional.html
