# C/C++ Connection (libmysqlclient)

## Overview

C/C++ connects to OceanBase (MySQL Mode) via `libmysqlclient`. This guide covers installation, connection, and best practices for V4.2+.

---

## Installation (Linux)

```bash
# CentOS/RHEL
yum install mysql-devel

# Ubuntu/Debian
apt-get install libmysqlclient-dev

# Verify
mysql_config --version
```

---

## Basic Connection (C)

```c
#include <mysql/mysql.h>
#include <stdio.h>
#include <stdlib.h>

int main() {
    MYSQL *conn = mysql_init(NULL);
    if (conn == NULL) {
        fprintf(stderr, "mysql_init() failed\n");
        return 1;
    }

    // Connect
    if (mysql_real_connect(conn,
            "10.0.0.100",          // OBProxy IP
            "app_user@mysql_tenant",
            "password",
            "test_db",
            2883,                    // OBProxy port
            NULL, 0) == NULL) {
        fprintf(stderr, "Connection failed: %s\n", mysql_error(conn));
        mysql_close(conn);
        return 1;
    }

    printf("Connected!\n");

    // Query
    if (mysql_query(conn, "SELECT 1 FROM DUAL") != 0) {
        fprintf(stderr, "Query failed: %s\n", mysql_error(conn));
    } else {
        MYSQL_RES *result = mysql_store_result(conn);
        MYSQL_ROW row = mysql_fetch_row(result);
        printf("Result: %s\n", row[0]);
        mysql_free_result(result);
    }

    mysql_close(conn);
    return 0;
}
```

> **Compile**: `gcc -o test test.c $(mysql_config --cflags --libs)`

---

## Basic Connection (C++)

```cpp
#include <mysql_driver.h>
#include <mysql_connection.h>
#include <cppconn/statement.h>
#include <iostream>

int main() {
    try {
        sql::mysql::MySQL_Driver *driver = sql::mysql::get_mysql_driver_instance();
        sql::Connection *conn = driver->connect(
            "tcp://10.0.0.100:2883",
            "app_user@mysql_tenant",
            "password"
        );
        conn->setSchema("test_db");

        sql::Statement *stmt = conn->createStatement();
        sql::ResultSet *res = stmt->executeQuery("SELECT 1 FROM DUAL");
        while (res->next()) {
            std::cout << "Result: " << res->getInt(1) << std::endl;
        }

        delete res;
        delete stmt;
        delete conn;
    } catch (sql::SQLException &e) {
        std::cout << "Error: " << e.what() << std::endl;
    }
    return 0;
}
```

> **Install**: `apt-get install libmysqlcppconn-dev` (Ubuntu)

---

## Connection Pool (C - Simple Implementation)

```c
// Simple connection pool (fixed size)
#define POOL_SIZE 10
MYSQL *conn_pool[POOL_SIZE];

void init_pool() {
    for (int i = 0; i < POOL_SIZE; i++) {
        conn_pool[i] = mysql_init(NULL);
        mysql_real_connect(conn_pool[i],
            "10.0.0.100", "app_user@mysql_tenant",
            "password", "test_db", 2883, NULL, 0);
    }
}

MYSQL *get_connection() {
    // Round-robin (simplified)
    static int idx = 0;
    return conn_pool[idx++ % POOL_SIZE];
}

void close_pool() {
    for (int i = 0; i < POOL_SIZE; i++) {
        mysql_close(conn_pool[i]);
    }
}
```

---

## Prepared Statements (Performance + Security)

```c
// Prepared statement (avoid SQL injection)
MYSQL_STMT *stmt = mysql_stmt_init(conn);
mysql_stmt_prepare(stmt, "INSERT INTO employees (emp_name, dept_id) VALUES (?, ?)", -1);

// Bind parameters
MYSQL_BIND bind[2];
memset(bind, 0, sizeof(bind));

bind[0].buffer_type = MYSQL_TYPE_STRING;
bind[0].buffer = "Alice";
bind[0].buffer_length = 5;

int dept_id = 10;
bind[1].buffer_type = MYSQL_TYPE_LONG;
bind[1].buffer = &dept_id;

mysql_stmt_bind_param(stmt, bind);
mysql_stmt_execute(stmt);

mysql_stmt_close(stmt);
```

---

## Common Mistakes

1. **Not checking `mysql_init()` return**
   - ❌ `MYSQL *conn = mysql_init(NULL);` then use without NULL check
   - ✅ Always check `if (conn == NULL)`

2. **Not calling `mysql_free_result()`**
   - ❌ Memory leak
   - ✅ `mysql_free_result(result);`

3. **Using `mysql_query()` with user input (SQL injection)**
   - ❌ `mysql_query(conn, "SELECT * FROM users WHERE name='" + user_input + "'")`
   - ✅ Use prepared statements (`mysql_stmt_prepare`)

4. **Not setting connection timeout**
   - ❌ Hangs forever on network issue
   - ✅ `mysql_options(conn, MYSQL_OPT_CONNECT_TIMEOUT, &timeout)`

5. **Forgetting to `mysql_library_end()` (C++)**
   - ❌ Library cleanup not done
   - ✅ Call `mysql_library_end()` at program exit

---

## OceanBase Oracle Mode (C/C++)

Same `libmysqlclient`! OceanBase Oracle mode speaks MySQL wire protocol:

```c
// Same code works; only SQL syntax changes
mysql_real_connect(conn, "10.0.0.100", "app_user@oracle_tenant", ...);
// Use Oracle syntax in queries: "SELECT * FROM employees WHERE ROWNUM <= 10"
```

---

## Sources
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/c-connectin
- https://www.oceanbase.com/docs/oceanbase-database-cn/v4.2.5/c-connectin
- https://dev.mysql.com/doc/c-api/en/
