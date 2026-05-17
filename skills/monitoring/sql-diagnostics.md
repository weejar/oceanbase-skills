# SQL Diagnostics

> **⚠️ Security Note**: 代码示例中的 `YOUR_PASSWORD` 为占位符。请通过环境变量（如 `os.getenv('OB_PASSWORD')`）或运行时传入真实密码，**不要将真实密码硬编码到文件中**。

> **⚡ Quick Commands (enmotest 集群，sys 租户)**
>
> 慢SQL Top20：
> ```bash
> python -c "
> import pymysql
> conn=pymysql.connect(host='172.20.22.213',port=2883,user='root@sys#enmotest',password='xxxx',database='oceanbase')
> c=conn.cursor()
> c.execute('SELECT TENANT_NAME,USER_NAME,SQL_ID,ROUND(ELAPSED_TIME/1000,1) AS elapsed_ms,DISK_READS,TABLE_SCAN,IS_HIT_PLAN,LEFT(QUERY_SQL,120) AS sql_text FROM oceanbase.GV\$OB_SQL_AUDIT WHERE ELAPSED_TIME>100000 AND IS_INNER_SQL=0 ORDER BY ELAPSED_TIME DESC LIMIT 20')
> [print(dict(zip([d[0] for d in c.description],r))) for r in c.fetchall()]
> c.close();conn.close()"
> ```
>
> append hint 检测：
> ```bash
> python -c "
> import pymysql
> conn=pymysql.connect(host='172.20.22.213',port=2883,user='root@sys#enmotest',password='xxxx',database='oceanbase')
> c=conn.cursor()
> c.execute(\"SELECT TENANT_NAME,USER_NAME,SQL_ID,ROUND(ELAPSED_TIME/1000,1) AS elapsed_ms,AFFECTED_ROWS,RET_CODE,LEFT(QUERY_SQL,200) AS sql_text FROM oceanbase.GV\$OB_SQL_AUDIT WHERE IS_INNER_SQL=0 AND QUERY_SQL LIKE '%/*+append*/%' ORDER BY ELAPSED_TIME DESC LIMIT 10\")
> [print(dict(zip([d[0] for d in c.description],r))) for r in c.fetchall()]
> c.close();conn.close()"
> ```

Comprehensive SQL-level diagnostics for OceanBase. All views require **sys tenant** connection via OBProxy (`root@sys#enmotest`).

---

## Connection

```python
import pymysql

conn = pymysql.connect(
    host='172.20.22.213',  # OBProxy/observer IP
    port=2883,              # OBProxy SQL port
    user='root@sys#enmotest',
    password='xxxxx5',
    database='oceanbase',
    charset='utf8mb4',
    connect_timeout=10
)
```

---

## Slow Queries (>1s)

### Top 20 by Elapsed Time

```python
cursor.execute("""
SELECT 
    TENANT_NAME,
    USER_NAME,
    DB_NAME,
    SQL_ID,
    PLAN_HASH,
    STMT_TYPE,
    ROUND(ELAPSED_TIME / 1000000, 3)   AS elapsed_sec,
    ROUND(EXECUTE_TIME / 1000000, 3)   AS execute_sec,
    ROUND(QUEUE_TIME / 1000, 3)        AS queue_ms,
    ROUND(GET_PLAN_TIME / 1000, 3)     AS get_plan_ms,
    RETURN_ROWS,
    AFFECTED_ROWS,
    DISK_READS,
    TABLE_SCAN,
    IS_HIT_PLAN,
    RETRY_CNT,
    USER_IO_WAIT_TIME,
    CONCURRENCY_WAIT_TIME,
    LEFT(QUERY_SQL, 200)               AS sql_text,
    FROM_UNIXTIME(REQUEST_TIME / 1000000) AS req_time
FROM oceanbase.GV$OB_SQL_AUDIT
WHERE ELAPSED_TIME > 1000000
  AND IS_INNER_SQL = 0
ORDER BY ELAPSED_TIME DESC
LIMIT 20
""")
```

### Count by Threshold

```python
cursor.execute("""
SELECT 
    SUM(ELAPSED_TIME > 100000)  AS gt_100ms,
    SUM(ELAPSED_TIME > 500000)  AS gt_500ms,
    SUM(ELAPSED_TIME > 1000000) AS gt_1s,
    SUM(ELAPSED_TIME > 5000000) AS gt_5s,
    COUNT(*)                    AS total
FROM oceanbase.GV$OB_SQL_AUDIT
WHERE IS_INNER_SQL = 0
""")
```

### By Tenant

```python
cursor.execute("""
SELECT 
    TENANT_NAME,
    COUNT(*)                          AS sql_count,
    SUM(ELAPSED_TIME > 1000000)      AS gt_1s,
    SUM(ELAPSED_TIME > 5000000)      AS gt_5s,
    ROUND(SUM(ELAPSED_TIME) / COUNT(*) / 1000, 1) AS avg_ms,
    ROUND(MAX(ELAPSED_TIME) / 1000, 1)           AS max_ms,
    SUM(TABLE_SCAN)                   AS table_scans,
    SUM(IS_HIT_PLAN)                  AS plan_hits
FROM oceanbase.GV$OB_SQL_AUDIT
WHERE IS_INNER_SQL = 0
GROUP BY TENANT_NAME
ORDER BY SUM(ELAPSED_TIME) DESC
""")
```

---

## High-Resource Queries

### High Disk Reads (I/O intensive)

```python
cursor.execute("""
SELECT 
    TENANT_NAME,
    SQL_ID,
    PLAN_HASH,
    DISK_READS,
    TABLE_SCAN,
    IS_HIT_PLAN,
    RETURN_ROWS,
    ROUND(ELAPSED_TIME / 1000, 1) AS elapsed_ms,
    LEFT(QUERY_SQL, 150)          AS sql_text
FROM oceanbase.GV$OB_SQL_AUDIT
WHERE IS_INNER_SQL = 0
  AND DISK_READS > 0
  AND ELAPSED_TIME > 0
ORDER BY DISK_READS DESC
LIMIT 20
""")
```

### Full Table Scans

```python
cursor.execute("""
SELECT 
    TENANT_NAME,
    USER_NAME,
    SQL_ID,
    RETURN_ROWS,
    ROUND(ELAPSED_TIME / 1000, 1) AS elapsed_ms,
    TABLE_SCAN,
    IS_HIT_PLAN,
    LEFT(QUERY_SQL, 150)          AS sql_text
FROM oceanbase.GV$OB_SQL_AUDIT
WHERE IS_INNER_SQL = 0
  AND TABLE_SCAN = 1
ORDER BY ELAPSED_TIME DESC
LIMIT 20
""")
```

### Retry / Contention Queries

```python
cursor.execute("""
SELECT 
    TENANT_NAME,
    SQL_ID,
    RETRY_CNT,
    CONCURRENCY_WAIT_TIME,
    ROUND(ELAPSED_TIME / 1000, 1) AS elapsed_ms,
    LEFT(QUERY_SQL, 150)           AS sql_text
FROM oceanbase.GV$OB_SQL_AUDIT
WHERE IS_INNER_SQL = 0
  AND RETRY_CNT > 0
ORDER BY RETRY_CNT DESC, ELAPSED_TIME DESC
LIMIT 20
""")
```

### Batch / Multi-Value INSERT Optimization

> OceanBase 客户端（如 JDBC）支持将多条 `INSERT INTO VALUES()` 自动合并为 `INSERT INTO VALUES(), (), …` 批处理执行，显著减少网络往返。当 `is_batched_multi_stmt = 0` 时说明未走批处理。

```python
cursor.execute("""
SELECT
    TENANT_NAME,
    USER_NAME,
    SQL_ID,
    SUBSTR(QUERY_SQL, 1, 120) AS sql_preview,
    COUNT(*)                    AS exec_count,
    SUM(AFFECTED_ROWS)         AS total_rows,
    ROUND(SUM(ELAPSED_TIME) / 1000, 1)  AS total_ms,
    ROUND(AVG(ELAPSED_TIME) / 1000, 1)  AS avg_ms,
    MIN(FROM_UNIXTIME(REQUEST_TIME / 1000000)) AS first_seen,
    MAX(FROM_UNIXTIME(REQUEST_TIME / 1000000)) AS last_seen
FROM oceanbase.GV$OB_SQL_AUDIT
WHERE QUERY_SQL LIKE 'INSERT %'
  AND AFFECTED_ROWS = 1
  AND IS_BATCHED_MULTI_STMT = 0
  AND IS_INNER_SQL = 0
  AND REQUEST_TIME > (NOW() - 2 / 24)   -- 最近2小时
GROUP BY TENANT_NAME, USER_NAME, SQL_ID, SUBSTR(QUERY_SQL, 1, 120)
HAVING COUNT(*) > 2
ORDER BY exec_count DESC
LIMIT 20
""")
```

> **关键字段解读**：
> - `exec_count` — 单条执行次数，次数越高说明越需要优化
> - `is_batched_multi_stmt = 0` — 未走批处理，**性能损耗点**
> - `affected_rows = 1` — 每条仅影响1行，进一步说明是逐条插入
> - `avg_ms` — 平均耗时，高则影响更严重
>
> **优化方向**：在 JDBC 连接串添加 `rewriteBatchedStatements=true`（MySQL 兼容模式），或业务层手动拼接多值 INSERT。

---

## SQL Hint Detection

### `/*+append*/` (Direct-Path Insert)

```python
cursor.execute("""
SELECT 
    TENANT_NAME,
    USER_NAME,
    DB_NAME,
    SQL_ID,
    PLAN_HASH,
    STMT_TYPE,
    ROUND(ELAPSED_TIME / 1000, 1) AS elapsed_ms,
    AFFECTED_ROWS,
    RETURN_ROWS,
    RET_CODE,
    IS_HIT_PLAN,
    DISK_READS,
    LEFT(QUERY_SQL, 300)          AS sql_text,
    FROM_UNIXTIME(REQUEST_TIME / 1000000) AS req_time
FROM oceanbase.GV$OB_SQL_AUDIT
WHERE IS_INNER_SQL = 0
  AND QUERY_SQL LIKE '%/*+append*/%'
ORDER BY ELAPSED_TIME DESC
LIMIT 20
""")
```

> **RET_CODE values**: 0 = success, -4012 = timeout/execution error, other negative = error. AFFECTED_ROWS = 0 + non-zero RET_CODE indicates failed execution.

### `/*+ PARALLEL */` (Parallel Query)

```python
cursor.execute("""
SELECT 
    TENANT_NAME,
    SQL_ID,
    ROUND(ELAPSED_TIME / 1000, 1) AS elapsed_ms,
    RETURN_ROWS,
    IS_HIT_PLAN,
    LEFT(QUERY_SQL, 200)          AS sql_text
FROM oceanbase.GV$OB_SQL_AUDIT
WHERE IS_INNER_SQL = 0
  AND QUERY_SQL LIKE '%/*+ PARALLEL%'
ORDER BY ELAPSED_TIME DESC
LIMIT 20
""")
```

---

## SQL Audit View Key Columns

| Column | Unit | Meaning |
|--------|------|---------|
| `QUERY_SQL` | text | SQL statement text |
| `ELAPSED_TIME` | μs | Total wall-clock time |
| `EXECUTE_TIME` | μs | Pure execution time |
| `QUEUE_TIME` | μs | Queue waiting time |
| `GET_PLAN_TIME` | μs | Plan compilation time |
| `DISK_READS` | rows | Physical SSStore reads (0 = cached) |
| `TABLE_SCAN` | flag | 1 = full table/index scan |
| `IS_HIT_PLAN` | flag | 1 = plan cache hit |
| `RETRY_CNT` | count | Number of retries (>0 = contention) |
| `RETURN_ROWS` | rows | Rows returned |
| `AFFECTED_ROWS` | rows | Rows inserted/updated/deleted |
| `RET_CODE` | code | 0 = OK, negative = error |
| `USER_IO_WAIT_TIME` | μs | Disk I/O wait time |
| `CONCURRENCY_WAIT_TIME` | μs | Lock/latch contention time |
| `IS_INNER_SQL` | flag | 1 = internal SQL (skip these) |

---

## Full Diagnostic Script

```python
import pymysql

def sql_diagnostics(threshold_ms=1000, limit=20):
    conn = pymysql.connect(
        host='172.20.22.213', port=2883,
        user='root@sys#enmotest', password='YOUR_PASSWORD',
        database='oceanbase', charset='utf8mb4', connect_timeout=10
    )

    def query(sql, params=None):
        cursor = conn.cursor()
        cursor.execute(sql, params)
        cols = [d[0] for d in cursor.description]
        rows = cursor.fetchall()
        cursor.close()
        return [dict(zip(cols, r)) for r in rows]

    threshold_us = threshold_ms * 1000

    print("=" * 60)
    print(f"SQL Diagnostics — threshold: {threshold_ms}ms, limit: {limit}")
    print("=" * 60)

    # 1. Slow SQL count
    counts = query("""
        SELECT 
            SUM(ELAPSED_TIME > 100000)  AS gt_100ms,
            SUM(ELAPSED_TIME > 500000)  AS gt_500ms,
            SUM(ELAPSED_TIME > 1000000) AS gt_1s,
            SUM(ELAPSED_TIME > 5000000) AS gt_5s,
            COUNT(*)                    AS total
        FROM oceanbase.GV$OB_SQL_AUDIT WHERE IS_INNER_SQL = 0
    """)[0]
    print(f"\n[1] Slow SQL Count:")
    print(f"  >100ms: {counts['gt_100ms']}  >500ms: {counts['gt_500ms']}  "
          f">1s: {counts['gt_1s']}  >5s: {counts['gt_5s']}  Total: {counts['total']}")

    # 2. Top slow queries
    print(f"\n[2] Top {limit} Slow SQL (>{threshold_ms}ms):")
    slow = query(f"""
        SELECT TENANT_NAME, USER_NAME, DB_NAME, SQL_ID,
               ROUND(ELAPSED_TIME/1000,1) AS elapsed_ms,
               ROUND(EXECUTE_TIME/1000,1) AS execute_ms,
               DISK_READS, TABLE_SCAN, IS_HIT_PLAN, RETRY_CNT,
               LEFT(QUERY_SQL, 120) AS sql_text
        FROM oceanbase.GV$OB_SQL_AUDIT
        WHERE ELAPSED_TIME > %s AND IS_INNER_SQL = 0
        ORDER BY ELAPSED_TIME DESC LIMIT %s
    """, (threshold_us, limit))
    for i, s in enumerate(slow, 1):
        flag = "⚠全表扫" if s['TABLE_SCAN'] else ""
        hit  = "✓命中" if s['IS_HIT_PLAN'] else "✗未命中"
        print(f"  [{i}] {s['TENANT_NAME']}/{s['USER_NAME']}  {s['elapsed_ms']}ms {flag} {hit}")
        print(f"      SQL: {s['sql_text']}")

    # 3. High disk reads
    print(f"\n[3] High Disk Reads (Top 10):")
    disk = query("""
        SELECT TENANT_NAME, SQL_ID, DISK_READS, RETURN_ROWS,
               ROUND(ELAPSED_TIME/1000,1) AS elapsed_ms,
               LEFT(QUERY_SQL, 100) AS sql_text
        FROM oceanbase.GV$OB_SQL_AUDIT
        WHERE IS_INNER_SQL=0 AND DISK_READS>0
        ORDER BY DISK_READS DESC LIMIT 10
    """)
    for d in disk:
        print(f"  租户={d['TENANT_NAME']}  磁盘读={d['DISK_READS']}  耗时={d['elapsed_ms']}ms  SQL={d['sql_text']}")

    # 4. Hint usage
    for hint, label in [('/*+append*/','append'), ('/*+ PARALLEL','parallel')]:
        print(f"\n[4] Hint Usage: /*{hint}*/")
        h = query(f"""
            SELECT TENANT_NAME, SQL_ID, ROUND(ELAPSED_TIME/1000,1) AS elapsed_ms,
                   AFFECTED_ROWS, RET_CODE, IS_HIT_PLAN,
                   LEFT(QUERY_SQL, 120) AS sql_text
            FROM oceanbase.GV$OB_SQL_AUDIT
            WHERE IS_INNER_SQL=0 AND QUERY_SQL LIKE %s
            ORDER BY ELAPSED_TIME DESC LIMIT 10
        """, (f'%{hint}%',))
        if not h:
            print("  未发现使用此 hint 的 SQL")
        for x in h:
            err = f" ⚠错误码={x['RET_CODE']}" if x['RET_CODE'] else ""
            print(f"  {x['TENANT_NAME']}/{x['USER_NAME']}  {x['elapsed_ms']}ms  "
                  f"影响行={x['AFFECTED_ROWS']}{err}  SQL={x['sql_text']}")

    # 5. Batch INSERT optimization check
    print(f"\n[5] Batch INSERT 未优化 (is_batched_multi_stmt=0):")
    batch = query("""
        SELECT TENANT_NAME, USER_NAME, SQL_ID,
               COUNT(*)                           AS exec_count,
               SUM(AFFECTED_ROWS)                 AS total_rows,
               ROUND(SUM(ELAPSED_TIME)/1000,1)  AS total_ms,
               ROUND(AVG(ELAPSED_TIME)/1000,1)  AS avg_ms,
               SUBSTR(QUERY_SQL, 1, 100)        AS sql_preview
        FROM oceanbase.GV$OB_SQL_AUDIT
        WHERE UPPER(QUERY_SQL) LIKE 'INSERT %'
          AND AFFECTED_ROWS = 1
          AND IS_BATCHED_MULTI_STMT = 0
          AND IS_INNER_SQL = 0
          AND REQUEST_TIME > (NOW() - 1/24)
        GROUP BY TENANT_NAME, USER_NAME, SQL_ID, SUBSTR(QUERY_SQL, 1, 100)
        HAVING COUNT(*) > 2
        ORDER BY exec_count DESC LIMIT 10
    """)
    if not batch:
        print("  未发现未优化的单条 INSERT")
    for b in batch:
        print(f"  租户={b['TENANT_NAME']}  执行{b['exec_count']}次  "
              f"总行={b['total_rows']}  总耗时={b['total_ms']}ms  均耗时={b['avg_ms']}ms")
        print(f"      SQL: {b['sql_preview']}")

    conn.close()

sql_diagnostics(threshold_ms=1000, limit=20)
```

---

## Common Issues

### 1. Unknown column error
- `GV$OB_SQL_AUDIT` fields differ from documentation. Run `DESC oceanbase.GV$OB_SQL_AUDIT` first to verify.

### 2. Empty results
- SQL audit may be disabled: `SHOW PARAMETERS LIKE 'enable_sql_audit'`
- Enable if needed: `ALTER SYSTEM SET enable_sql_audit = 'TRUE';`

### 3. High disk reads = slow
- `DISK_READS > 0` means data not in cache → full table scan or large index range scan
- Add covering index on WHERE columns to reduce disk I/O

### 4. INSERT with RET_CODE != 0
- Check table structure, sequence existence, and user permissions
- `/*+append*/` is Oracle-mode hint for direct-path insert; does NOT work in MySQL mode

### 5. INSERT 未走批处理优化
- `is_batched_multi_stmt = 0` 且 `affected_rows = 1` → 每条 INSERT 单独执行
- MySQL 兼容模式：JDBC URL 添加 `rewriteBatchedStatements=true`
- Oracle 兼容模式：确认驱动版本支持批处理（Oracle JDBC 12c+ 默认支持，需显式 `addBatch()`/`executeBatch()`）
- 应用层优化：手动拼接 `INSERT INTO t VALUES (...), (...), (...)` 而非循环单条执行

---

## Related Skills

- `skills/performance/slow-query-analysis.md` — 慢SQL综合分析（高磁盘读/全表扫/hint/一键诊断脚本）
- `skills/performance/explain-plan.md` — 执行计划分析
- `skills/monitoring/session-monitor.md` — 活动会话监控
- `skills/appdev/sys-tenant-connection.md` — Sys租户连接

---

*Version: OceanBase V4.x | Last updated: 2026-05-15*
