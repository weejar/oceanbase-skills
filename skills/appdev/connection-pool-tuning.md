# Connection Pool Tuning

## Overview

Connection pool tuning is critical for performance. This guide covers pool sizing, timeout settings, and troubleshooting for OceanBase V4.2+.

---

## Connection Pool Sizing

| Pool Type | Formula | Example |
|-----------|----------|---------|
| **OLTP** | `pool_size = core_count * 2` | 8 cores → 16 connections |
| **Mixed** | `pool_size = avg_concurrent_users * 1.5` | 20 users → 30 connections |
| **OLAP** | `pool_size = core_count` | 8 cores → 8 connections |

> **Rule**: Never set `pool_size > 100` (unless benchmark proves need).

---

## HikariCP (Java — Recommended)

```java
HikariConfig config = new HikariConfig();
config.setMaximumPoolSize(20);      // Max connections
config.setMinimumIdle(5);           // Min idle connections
config.setConnectionTimeout(5000);  // 5s connect timeout
config.setIdleTimeout(600000);      // 10min idle timeout
config.setMaxLifetime(1800000);    // 30min max lifetime
config.setLeakDetectionThreshold(60000); // 60s leak detection
```

| Parameter | Recommended | Description |
|-----------|-------------|-------------|
| `maximumPoolSize` | 20-50 | Based on concurrency |
| `connectionTimeout` | 5000 | Fail fast (5s) |
| `idleTimeout` | 600000 | Recycle idle connections |
| `maxLifetime` | 1800000 | Avoid stale connections |
| `leakDetectionThreshold` | 60000 | Log connection leaks |

---

## SQLAlchemy (Python)

```python
engine = create_engine(
    'mysql+pymysql://user:pass@host:2883/db',
    pool_size=10,           # Fixed pool size
    max_overflow=5,         # Extra connections beyond pool_size
    pool_recycle=3600,      # Recycle after 1h
    pool_pre_ping=True,      # Validate before use
    pool_use_lifo=True       # LIFO (reuse recent connections)
)
```

| Parameter | Recommended | Description |
|-----------|-------------|-------------|
| `pool_size` | 10 | Fixed pool |
| `max_overflow` | 5 | Burst capacity |
| `pool_recycle` | 3600 | Avoid stale connections |
| `pool_pre_ping` | True | Validate before use |

---

## Go `database/sql` Pool

```go
db.SetMaxOpenConns(20)      // Max open connections
db.SetMaxIdleConns(5)        // Max idle connections
db.SetConnMaxLifetime(time.Hour)  // Max connection lifetime
db.SetConnMaxIdleTime(10 * time.Minute)  // Max idle time
```

---

## Monitoring Pool Usage

### Java (HikariCP JMX)

```java
HikariDataSource ds = ...;
int active = ds.getHikariPoolMXBean().getActiveConnections();
int idle = ds.getHikariPoolMXBean().getIdleConnections();
int waiting = ds.getHikariPoolMXBean().getThreadsAwaitingConnection();
```

> **Alert**: If `waiting > 0`, pool is too small.

### Python (SQLAlchemy)

```python
from sqlalchemy import event

@event.listens(engine, "checkout")
def on_checkout(dbapi_conn, connection_rec, connection_proxy):
    print(f"Active connections: {engine.pool.status()}")
```

---

## Common Mistakes

1. **Pool too large (`maximumPoolSize=200`)**
   - ❌ DB server connection limit exceeded
   - ✅ `maximumPoolSize=20-50`

2. **Not setting `connectionTimeout`**
   - ❌ Threads hang forever waiting for connection
   - ✅ `connectionTimeout=5000` (5s)

3. **Not enabling `pool_pre_ping` (Python)**
   - ❌ Stale connections cause errors
   - ✅ `pool_pre_ping=True`

4. **Pool too small for concurrency**
   - ❌ `waiting > 0` (threads waiting)
   - ✅ Increase `maximumPoolSize`

5. **Not monitoring pool metrics**
   - ❌ Pool exhaustion in production
   - ✅ Monitor `active`, `idle`, `waiting`

---

## OceanBase-Specific Tips

1. **Use OBProxy** — app connects to OBProxy (single IP), which routes to OBServer.
2. **Set `rewriteBatchedStatements=true` (JDBC)** — critical for batch performance.
3. **Connection per CPU core × 2** — good starting point for OLTP.

---

## Sources
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/connection-pool
- https://www.oceanbase.com/docs/oceanbase-database-cn/v4.2.5/connection-pool
- https://github.com/brettwooldridge/HikariCP
