# Go Database Connection (database/sql + go-sql-driver)

## Overview

Go connects to OceanBase (MySQL Mode) via `database/sql` with `go-sql-driver/mysql`. This guide covers connection setup, pooling, and best practices for V4.2+.

---

## Installation

```bash
go get -u github.com/go-sql-driver/mysql
```

---

## Basic Connection

```go
package main

import (
    "database/sql"
    "fmt"
    _ "github.com/go-sql-driver/mysql"
)

func main() {
    // DSN format: user@tenant:password@tcp(host:port)/dbname
    dsn := "app_user@mysql_tenant:password@tcp(10.0.0.100:2883)/test_db?timeout=5s&readTimeout=30s"
    db, err := sql.Open("mysql", dsn)
    if err != nil {
        panic(err)
    }
    defer db.Close()

    // Test connection
    var result int
    err = db.QueryRow("SELECT 1 FROM DUAL").Scan(&result)
    if err != nil {
        panic(err)
    }
    fmt.Println("Connected! Result:", result)
}
```

> **Key**: DSN username format = `user@tenant` (same as MySQL).

---

## Connection Pool Configuration

```go
// Set connection pool parameters
db.SetMaxOpenConns(20)      // Max open connections
db.SetMaxIdleConns(5)        // Max idle connections
db.SetConnMaxLifetime(time.Hour)  // Max connection lifetime
db.SetConnMaxIdleTime(10 * time.Minute)  // Max idle time
```

| Parameter | Recommended | Description |
|-----------|-------------|-------------|
| `MaxOpenConns` | 20-50 | Based on concurrency |
| `MaxIdleConns` | 5-10 | Idle connections kept |
| `ConnMaxLifetime` | 1h | Avoid stale connections |
| `ConnMaxIdleTime` | 10m | Recycle idle connections |

---

## Using OBProxy

```
App (Go) → OBProxy (10.0.0.100:2883) → OBServer
```

```go
// Connect via OBProxy (recommended)
dsn := "app_user@mysql_tenant:password@tcp(10.0.0.100:2883)/test_db"
```

> **Tip**: OBProxy provides transparent failover.

---

## Prepared Statements

```go
// Prepare statement (reuse for performance)
stmt, err := db.Prepare("INSERT INTO employees (emp_name, dept_id) VALUES (?, ?)")
if err != nil {
    panic(err)
}
defer stmt.Close()

// Execute prepared statement
result, err := stmt.Exec("Alice", 10)
if err != nil {
    panic(err)
}
lastId, _ := result.LastInsertId()
fmt.Println("Inserted ID:", lastId)
```

---

## Batch Processing

```go
// Use transactions for batch (faster)
tx, err := db.Begin()
if err != nil {
    panic(err)
}

stmt, err := tx.Prepare("INSERT INTO employees (emp_name, dept_id) VALUES (?, ?)")
if err != nil {
    tx.Rollback()
    panic(err)
}
defer stmt.Close()

// Batch insert
for i := 0; i < 1000; i++ {
    _, err := stmt.Exec(fmt.Sprintf("Emp%d", i), 10)
    if err != nil {
        tx.Rollback()
        panic(err)
    }
}
tx.Commit()
```

---

## Common Mistakes

1. **Not calling `db.Close()`**
   - ❌ Connection leak
   - ✅ `defer db.Close()`

2. **Not setting connection pool limits**
   - ❌ Unlimited connections (DB overload)
   - ✅ `SetMaxOpenConns(20)`

3. **Using `Query()` for writes**
   - ❌ `db.Query("INSERT ...")` (returns `sql.Rows`)
   - ✅ `db.Exec("INSERT ...")`

4. **Not using prepared statements**
   - ❌ SQL injection risk
   - ✅ `db.Prepare()` + parameterized queries

5. **Forgetting `defer stmt.Close()`**
   - ❌ Statement leak
   - ✅ `defer stmt.Close()`

---

## OceanBase Oracle Mode (Go)

Same driver! OceanBase Oracle mode speaks MySQL wire protocol:

```go
dsn := "app_user@oracle_tenant:password@tcp(10.0.0.100:2883)/test_db"
// Same go-sql-driver/mysql!
```

> **Only difference**: SQL syntax (use Oracle syntax in queries).

---

## Sources
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/go
- https://www.oceanbase.com/docs/oceanbase-database-cn/v4.2.5/go
- https://github.com/go-sql-driver/mysql
