# JDBC Connection and Configuration

## Overview

JDBC is the standard Java API for connecting to OceanBase (MySQL mode). This guide covers JDBC connection setup, connection parameters, and troubleshooting for OceanBase V4.2+.

---

## Basic JDBC Connection (MySQL Mode)

```java
import java.sql.*;

public class OBTest {
    public static void main(String[] args) throws Exception {
        String url = "jdbc:mysql://10.0.0.1:2881/test_db?useSSL=false&serverTimezone=UTC";
        String user = "app_user@mysql_tenant";
        String password = "password";

        Connection conn = DriverManager.getConnection(url, user, password);
        Statement stmt = conn.createStatement();
        ResultSet rs = stmt.executeQuery("SELECT 1 FROM DUAL");
        while (rs.next()) {
            System.out.println(rs.getInt(1));
        }
        conn.close();
    }
}
```

> **Key**: Username format = `user@tenant#cluster` (similar to MySQL connection string).

---

## JDBC URL Parameters

| Parameter | Default | Recommended | Description |
|-----------|---------|-------------|-------------|
| `useSSL` | TRUE | FALSE (internal network) | Enable SSL |
| `serverTimezone` | (none) | `UTC` or `Asia/Shanghai` | Timezone |
| `connectTimeout` | 0 | 5000 | Connect timeout (ms) |
| `socketTimeout` | 0 | 30000 | Socket timeout (ms) |
| `rewriteBatchedStatements` | FALSE | TRUE | Batch performance |
| `useServerPrepStmts` | FALSE | TRUE | Use server-side prepared statements |
| `cachePrepStmts` | FALSE | TRUE | Cache prepared statements |

### Optimized URL

```java
String url = "jdbc:mysql://10.0.0.1:2881/test_db?" +
    "useSSL=false&" +
    "serverTimezone=Asia/Shanghai&" +
    "rewriteBatchedStatements=true&" +
    "useServerPrepStmts=true&" +
    "cachePrepStmts=true&" +
    "prepStmtCacheSize=500&" +
    "connectTimeout=5000&" +
    "socketTimeout=30000";
```

---

## Using OBProxy with JDBC

```
App → OBProxy (10.0.0.100:2883) → OBServer
```

```java
// Connect via OBProxy
String url = "jdbc:mysql://10.0.0.100:2883/test_db?" +
    "useSSL=false&" +
    "serverTimezone=Asia/Shanghai";

// User format for OBProxy
String user = "app_user@mysql_tenant#obcluster";
```

> **Tip**: OBProxy provides transparent failover. App doesn't need to know OBServer addresses.

---

## Connection Pool (HikariCP)

```java
import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;

HikariConfig config = new HikariConfig();
config.setJdbcUrl("jdbc:mysql://10.0.0.100:2883/test_db?useSSL=false");
config.setUsername("app_user@mysql_tenant");
config.setPassword("password");
config.setMaximumPoolSize(20);
config.setMinimumIdle(5);
config.setConnectionTimeout(5000);
config.setIdleTimeout(600000);

HikariDataSource ds = new HikariDataSource(config);
Connection conn = ds.getConnection();
```

| Parameter | Recommended | Description |
|-----------|-------------|-------------|
| `maximumPoolSize` | 20-50 | Based on concurrency |
| `connectionTimeout` | 5000 | 5 seconds |
| `idleTimeout` | 600000 | 10 minutes |

---

## Common Mistakes

1. **Not using `rewriteBatchedStatements=true`**
   - ❌ Batch INSERT slow (single INSERT per statement)
   - ✅ Add `rewriteBatchedStatements=true` to URL

2. **Connecting directly to OBServer (not OBProxy)**
   - ❌ App configured with 3 OBServer IPs (no failover)
   - ✅ Use OBProxy (single IP: 10.0.0.100:2883)

3. **Not setting `socketTimeout`**
   - ❌ Query hangs forever on network issue
   - ✅ `socketTimeout=30000` (30 seconds)

4. **Using `SELECT *` in Java code**
   - ❌ `ResultSet` loads all columns (slow)
   - ✅ Select only needed columns

5. **Not closing `ResultSet`/`Statement`/`Connection`**
   - ❌ Connection leak
   - ✅ Use try-with-resources:
   ```java
   try (Connection conn = ds.getConnection();
        Statement stmt = conn.createStatement();
        ResultSet rs = stmt.executeQuery(sql)) {
       // ...
   }
   ```

---

## Oracle Mode (JDBC)

OceanBase Oracle mode also uses MySQL protocol for JDBC:

```java
String url = "jdbc:mysql://10.0.0.1:2881/test_db?useSSL=false";
String user = "app_user@oracle_tenant";
// Same JDBC driver! OceanBase Oracle mode speaks MySQL wire protocol.
```

> **Important**: You still use MySQL JDBC driver for Oracle mode.

---

## Sources
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/jdbc
- https://www.oceanbase.com/docs/oceanbase-database-cn/v4.2.5/jdbc
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/jdbc-url-parameters
