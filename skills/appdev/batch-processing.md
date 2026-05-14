# Batch Processing

## Overview

Batch processing (bulk INSERT/UPDATE/DELETE) is critical for performance. This guide covers batch techniques for OceanBase (MySQL Mode) across Java/Python/Go, V4.2+.

---

## Batch INSERT (JDBC — Java)

### Bad (Slow)

```java
// DON'T: Single INSERT per row (N round-trips)
for (Employee emp : list) {
    PreparedStatement ps = conn.prepareStatement(
        "INSERT INTO employees (emp_name, dept_id) VALUES (?, ?)");
    ps.setString(1, emp.getName());
    ps.setInt(2, emp.getDeptId());
    ps.executeUpdate();  // 1 round-trip per row = SLOW
}
```

### Good (Fast — `rewriteBatchedStatements`)

```java
// Add to JDBC URL: rewriteBatchedStatements=true
String url = "jdbc:mysql://10.0.0.100:2883/test_db?rewriteBatchedStatements=true";

PreparedStatement ps = conn.prepareStatement(
    "INSERT INTO employees (emp_name, dept_id) VALUES (?, ?)");
for (Employee emp : list) {
    ps.setString(1, emp.getName());
    ps.setInt(2, emp.getDeptId());
    ps.addBatch();       // Add to batch
}
ps.executeBatch();  // Single round-trip for all rows
```

> **Key**: `rewriteBatchedStatements=true` rewrites multiple INSERTs into multi-value INSERT (much faster).

---

## Batch INSERT (MyBatis)

```xml
<!-- Mapper XML: batch insert -->
<insert id="batchInsert">
    INSERT INTO employees (emp_name, dept_id)
    VALUES
    <foreach item="emp" collection="list" separator=",">
        (#{emp.empName}, #{emp.deptId})
    </foreach>
</insert>
```

```java
// Java code
mapper.batchInsert(empList);  // All rows in 1 SQL statement
```

> **Tip**: Keep batch size ≤ 1000 per statement (avoid too large SQL).

---

## Batch INSERT (Python/PyMySQL)

```python
# Use executemany() (rewrites as multi-value INSERT)
cursor = conn.cursor()
sql = "INSERT INTO employees (emp_name, dept_id) VALUES (%s, %s)"
values = [('Alice', 10), ('Bob', 20), ('Charlie', 30)]
cursor.executemany(sql, values)  # PyMySQL rewrites as multi-value INSERT
conn.commit()
```

> **Performance**: `executemany()` is ~10x faster than loop with `execute()`.

---

## Batch UPDATE (JDBC)

```java
PreparedStatement ps = conn.prepareStatement(
    "UPDATE employees SET dept_id = ? WHERE emp_id = ?");
for (Employee emp : list) {
    ps.setInt(1, emp.getDeptId());
    ps.setLong(2, emp.getEmpId());
    ps.addBatch();
}
ps.executeBatch();
```

---

## Batch DELETE ( JDBC)

```java
PreparedStatement ps = conn.prepareStatement(
    "DELETE FROM employees WHERE emp_id = ?");
for (Long empId : empIds) {
    ps.setLong(1, empId);
    ps.addBatch();
}
ps.executeBatch();
```

---

## OceanBase-Specific Limits

| Item | Limit | Recommendation |
|------|-------|-----------------|
| Max SQL length | 64MB | Keep batch ≤ 5000 rows |
| Max INSERT values per statement | ~5000 | Split into chunks |
| `max_allowed_packet` | 64MB | Increase if needed |

```sql
-- Increase max packet size (if batch SQL too large)
ALTER SYSTEM SET max_allowed_packet = '128M';
```

---

## Common Mistakes

1. **Not using `rewriteBatchedStatements=true` (JDBC)**
   - ❌ Single INSERT per round-trip
   - ✅ Add to JDBC URL

2. **Batch too large (> 10000 rows)**
   - ❌ SQL too long, fails
   - ✅ Split into 1000-row chunks

3. **Not committing after batch**
   - ❌ Auto-commit ON (slow)
   - ✅ `conn.setAutoCommit(false);` then `conn.commit();`

4. **Using `executeUpdate()` in loop (Python)**
   - ❌ `cursor.execute()` per row
   - ✅ Use `cursor.executemany()`

5. **Not checking `executeBatch()` return**
   - ❌ Assume all rows inserted
   - ✅ Check return array for failures

---

## OceanBase Oracle Mode

Same batch techniques. Only SQL syntax may differ.

---

## Sources
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/batch
- https://www.oceanbase.com/docs/oceanbase-database-cn/v4.2.5/batch
- https://dev.mysql.com/doc/connector-j/en/connector-j-batch-insert.html
