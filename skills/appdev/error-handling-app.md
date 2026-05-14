# Error Handling (Application)

## Overview

Proper error handling prevents data corruption and improves user experience. This guide covers error handling patterns for OceanBase in Java/Python/Go, V4.2+.

---

## Common OceanBase Error Codes

| Error Code | Meaning | Action |
|------------|---------|--------|
| `1205` | Lock wait timeout | Retry transaction |
| `1213` | Deadlock detected | Retry transaction |
| `-4030` | Disk full | Add storage |
| `-4012` | Memory limit exceeded | Increase `MEMORY_SIZE` |
| `2006` | MySQL server gone away | Reconnect |
| `1045` | Access denied | Check credentials |

---

## Java — Retry on Deadlock

```java
import java.sql.SQLException;

public void saveWithRetry(Employee emp, int maxRetries) {
    for (int i = 0; i < maxRetries; i++) {
        try {
            employeeRepo.save(emp);
            return;  // Success
        } catch (SQLException e) {
            if (e.getErrorCode() == 1213) {  // Deadlock
                // Wait random backoff before retry
                try { Thread.sleep((long) (Math.random() * 100)); } catch (InterruptedException ie) {}
                continue;  // Retry
            }
            throw e;  // Other error, rethrow
        }
    }
    throw new RuntimeException("Failed after " + maxRetries + " retries");
}
```

> **Tip**: Always use exponential backoff for retries.

---

## Java — Connection Error (Reconnect)

```java
public Connection getConnectionWithRetry(String url, String user, String pass, int maxRetries) {
    for (int i = 0; i < maxRetries; i++) {
        try {
            return DriverManager.getConnection(url, user, pass);
        } catch (SQLException e) {
            if (e.getErrorCode() == 2006) {  // MySQL server gone away
                // Wait and retry
                try { Thread.sleep(1000 * (i + 1)); } catch (InterruptedException ie) {}
                continue;
            }
            throw new RuntimeException(e);
        }
    }
    throw new RuntimeException("Connection failed after " + maxRetries + " retries");
}
```

---

## Python — Retry on Deadlock

```python
import pymysql
import time
from pymysql import OperationalError, InternalError

def save_with_retry(emp, max_retries=3):
    for attempt in range(max_retries):
        try:
            with conn.cursor() as cursor:
                cursor.execute(
                    "INSERT INTO employees (emp_name, dept_id) VALUES (%s, %s)",
                    (emp['name'], emp['dept_id'])
                )
            conn.commit()
            return
        except InternalError as e:
            if e.args[0] == 1213:  # Deadlock
                time.sleep(0.1 * (2 ** attempt))  # Exponential backoff
                continue
            raise
    raise Exception(f"Failed after {max_retries} retries")
```

---

## Go — Retry Pattern

```go
import (
    "database/sql"
    "time"
)

func saveWithRetry(db *sql.DB, empName string, deptId int, maxRetries int) error {
    for i := 0; i < maxRetries; i++ {
        _, err := db.Exec(
            "INSERT INTO employees (emp_name, dept_id) VALUES (?, ?)",
            empName, deptId,
        )
        if err != nil {
            if strings.Contains(err.Error(), "Deadlock") {
                // Wait and retry
                time.Sleep(time.Millisecond * time.Duration(100*(i+1)))
                continue
            }
            return err
        }
        return nil  // Success
    }
    return fmt.Errorf("Failed after %d retries", maxRetries)
}
```

---

## Circuit Breaker Pattern (Advanced)

```java
// Use Resilience4j CircuitBreaker
CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("ob-circuit");

Supplier<String> decoratedSupplier = CircuitBreaker
    .decorateSupplier(circuitBreaker, () -> employeeRepo.findById(1));

// If DB down, circuit opens and fails fast (no waiting)
String result = Try.ofSupplier(decoratedSupplier)
    .recover(throwable -> "fallback").get();
```

> **Use Case**: Prevent cascading failures when OceanBase is down.

---

## Common Mistakes

1. **Not retrying on deadlock (1213)**
   - ❌ Fail immediately on deadlock
   - ✅ Retry 3 times with exponential backoff

2. **Retrying on non-retryable errors (e.g., syntax error)**
   - ❌ Retry `INSERT INTO employees (invalid_col) ...`
   - ✅ Only retry on deadlock/timeout/connection errors

3. **Not logging errors**
   - ❌ Silent failure
   - ✅ Log error code, message, stack trace

4. **Infinite retry loop**
   - ❌ `while(true) { retry(); }`
   - ✅ Max retries (3 or 5), then fail

5. **Not using exponential backoff**
   - ❌ Retry immediately (thundering herd)
   - ✅ Wait 100ms, 200ms, 400ms, ...

---

## OceanBase-Specific Errors

| Error | Meaning | Fix |
|-------|---------|-----|
| `-4030` (disk full) | Add storage | `ALTER SYSTEM ADD TODO...` |
| `-4012` (memory limit) | Increase `MEMORY_SIZE` | `ALTER RESOURCE UNIT ...` |
| `1213` (deadlock) | Retry transaction | Implement retry logic |
| `1205` (lock timeout) | Increase `lock_wait_timeout` | `SET lock_wait_timeout=50` |

---

## Sources
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/error-codes
- https://www.oceanbase.com/docs/oceanbase-database-cn/v4.2.5/error-codes
- https://resilience4j.readme.io/
