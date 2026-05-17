# Slow Query Analysis

> **⚠️ Security Note**: 代码示例中的 `YOUR_PASSWORD` 为占位符。请通过环境变量（如 `os.getenv('OB_PASSWORD')`）或运行时传入真实密码，**不要将真实密码硬编码到文件中**。

## Overview

Identifying and fixing slow SQL is the #1 DBA task. OceanBase provides `GV$OB_SQL_AUDIT` (sys tenant, cross-tenant) to track SQL execution. This guide covers finding and fixing slow queries in V4.2+.

> **⚡ Quick Command**: 直接执行以下命令查询慢SQL（sys租户跨租户查询）,替换成自己的Host,port,用户名、密码等信息：
>
> ```bash
> python -c "
> import pymysql
> conn=pymysql.connect(host='172.20.22.213',port=2883,user='root@sys#enmotest',password='xxxxxxx',database='oceanbase')
> c=conn.cursor()
> c.execute('''SELECT TENANT_NAME,USER_NAME,DB_NAME,SQL_ID,ROUND(ELAPSED_TIME/1000,1) AS elapsed_ms,ROUND(EXECUTE_TIME/1000,1) AS execute_ms,RETURN_ROWS,DISK_READS,TABLE_SCAN,IS_HIT_PLAN,LEFT(QUERY_SQL,100) AS sql_text,FROM_UNIXTIME(REQUEST_TIME/1000000) AS req_time FROM oceanbase.GV\$OB_SQL_AUDIT WHERE ELAPSED_TIME>1000000 AND IS_INNER_SQL=0 ORDER BY ELAPSED_TIME DESC LIMIT 20''')
> cols=[d[0] for d in c.description]
> [print(dict(zip(cols,r))) for r in c.fetchall()]
> c.close();conn.close()"
> ```

---

## Finding Slow Queries

### Via GV$OB_SQL_AUDIT (sys tenant, recommended — cross-tenant view)

```sql
-- Top 20 slowest SQL (>1s), all tenants except sys internal
SELECT 
    TENANT_NAME,
    USER_NAME,
    DB_NAME,
    SQL_ID,
    ROUND(ELAPSED_TIME / 1000000, 3)   AS elapsed_sec,
    ROUND(EXECUTE_TIME / 1000000, 3)   AS execute_sec,
    ROUND(QUEUE_TIME / 1000, 3)        AS queue_ms,
    ROUND(GET_PLAN_TIME / 1000, 3)     AS get_plan_ms,
    RETURN_ROWS,
    DISK_READS,
    TABLE_SCAN,
    IS_HIT_PLAN,
    RETRY_CNT,
    LEFT(QUERY_SQL, 150)               AS sql_text,
    FROM_UNIXTIME(REQUEST_TIME / 1000000) AS req_time
FROM oceanbase.GV$OB_SQL_AUDIT
WHERE ELAPSED_TIME > 1000000
  AND IS_INNER_SQL = 0
ORDER BY ELAPSED_TIME DESC
LIMIT 20;
```

### Oracle Mode (tenant-level view)

```sql
-- From within the tenant (Oracle mode)
SELECT sql_id, sql_text,
       round(elapsed_time / 1000000, 2) AS total_sec
FROM v$sql_audit
ORDER BY elapsed_time DESC
FETCH FIRST 10 ROWS ONLY;
```

---

## Aggregated Views

### Count by Threshold

```sql
SELECT 
    SUM(ELAPSED_TIME > 100000)  AS gt_100ms,
    SUM(ELAPSED_TIME > 500000)  AS gt_500ms,
    SUM(ELAPSED_TIME > 1000000) AS gt_1s,
    SUM(ELAPSED_TIME > 5000000) AS gt_5s,
    COUNT(*)                    AS total
FROM oceanbase.GV$OB_SQL_AUDIT
WHERE IS_INNER_SQL = 0;
```

### By Tenant

```sql
SELECT 
    TENANT_NAME,
    COUNT(*)                                       AS sql_count,
    SUM(ELAPSED_TIME > 1000000)                  AS gt_1s,
    SUM(ELAPSED_TIME > 5000000)                  AS gt_5s,
    ROUND(SUM(ELAPSED_TIME) / COUNT(*) / 1000, 1) AS avg_ms,
    ROUND(MAX(ELAPSED_TIME) / 1000, 1)            AS max_ms,
    SUM(TABLE_SCAN)                               AS table_scans,
    SUM(IS_HIT_PLAN)                              AS plan_hits
FROM oceanbase.GV$OB_SQL_AUDIT
WHERE IS_INNER_SQL = 0
GROUP BY TENANT_NAME
ORDER BY SUM(ELAPSED_TIME) DESC;
```

---

## High-Resource Queries

### High Disk Reads (I/O intensive)

```sql
SELECT 
    TENANT_NAME,
    SQL_ID,
    PLAN_HASH,
    DISK_READS,
    TABLE_SCAN,
    IS_HIT_PLAN,
    RETURN_ROWS,
    ROUND(ELAPSED_TIME / 1000, 1) AS elapsed_ms,
    LEFT(QUERY_SQL, 150)           AS sql_text
FROM oceanbase.GV$OB_SQL_AUDIT
WHERE IS_INNER_SQL = 0
  AND DISK_READS > 0
  AND ELAPSED_TIME > 0
ORDER BY DISK_READS DESC
LIMIT 20;
```

> `DISK_READS > 0` → 数据未命中缓存，触发物理读。常见原因：全表扫描、索引范围过宽、热点数据淘汰。

### Full Table Scans

```sql
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
LIMIT 20;
```

### Retry / Lock Contention

```sql
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
LIMIT 20;
```

> `RETRY_CNT > 0` → 锁等待或版本冲突，并发写入场景常见。

---

## SQL Hint Detection

### `/*+append*/` (Direct-Path Insert, Oracle Mode)

```sql
SELECT 
    TENANT_NAME,
    USER_NAME,
    DB_NAME,
    SQL_ID,
    STMT_TYPE,
    ROUND(ELAPSED_TIME / 1000, 1) AS elapsed_ms,
    AFFECTED_ROWS,
    RETURN_ROWS,
    RET_CODE,
    IS_HIT_PLAN,
    DISK_READS,
    LEFT(QUERY_SQL, 300)           AS sql_text,
    FROM_UNIXTIME(REQUEST_TIME / 1000000) AS req_time
FROM oceanbase.GV$OB_SQL_AUDIT
WHERE IS_INNER_SQL = 0
  AND QUERY_SQL LIKE '%/*+append*/%'
ORDER BY ELAPSED_TIME DESC
LIMIT 20;
```

> `RET_CODE`: 0 = success, negative = error。`AFFECTED_ROWS = 0` + non-zero `RET_CODE` 表示执行失败。
> `/*+append*/` 是 Oracle 模式 hint，绕过 buffer cache 直接写盘；MySQL 模式不生效。

### `/*+ PARALLEL */` (Parallel Query)

```sql
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
LIMIT 20;
```

---

## Full Diagnostic Script

```python
import pymysql

def slow_query_analysis(threshold_ms=1000, limit=20):
    """
    OceanBase 慢SQL一键诊断脚本
    连接: root@sys#enmotest via OBProxy 172.20.22.213:2883
    """
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
    print(f"  Slow Query Analysis  threshold={threshold_ms}ms  limit={limit}")
    print("=" * 60)

    # 1. Slow SQL count by threshold
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
               ROUND(ELAPSED_TIME/1000,1)  AS elapsed_ms,
               ROUND(EXECUTE_TIME/1000,1)  AS execute_ms,
               DISK_READS, TABLE_SCAN, IS_HIT_PLAN, RETRY_CNT,
               LEFT(QUERY_SQL, 120) AS sql_text
        FROM oceanbase.GV$OB_SQL_AUDIT
        WHERE ELAPSED_TIME > %s AND IS_INNER_SQL = 0
        ORDER BY ELAPSED_TIME DESC LIMIT %s
    """, (threshold_us, limit))
    for i, s in enumerate(slow, 1):
        flag = "⚠ 全表扫" if s['TABLE_SCAN'] else ""
        hit  = "✓ 命中" if s['IS_HIT_PLAN'] else "✗ 未命中"
        print(f"  [{i}] {s['TENANT_NAME']}/{s['USER_NAME']}  {s['elapsed_ms']}ms  {flag}  {hit}")
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
        print(f"  租户={d['TENANT_NAME']}  磁盘读={d['DISK_READS']}  "
              f"耗时={d['elapsed_ms']}ms  SQL={d['sql_text']}")

    # 4. Full table scans
    print(f"\n[4] Full Table Scans (Top 10):")
    scans = query("""
        SELECT TENANT_NAME, SQL_ID, RETURN_ROWS,
               ROUND(ELAPSED_TIME/1000,1) AS elapsed_ms,
               LEFT(QUERY_SQL, 100) AS sql_text
        FROM oceanbase.GV$OB_SQL_AUDIT
        WHERE IS_INNER_SQL=0 AND TABLE_SCAN=1
        ORDER BY ELAPSED_TIME DESC LIMIT 10
    """)
    for s in scans:
        print(f"  租户={s['TENANT_NAME']}  耗时={s['elapsed_ms']}ms  "
              f"返回行={s['RETURN_ROWS']}  SQL={s['sql_text']}")

    # 5. Retry / contention
    print(f"\n[5] Retry / Lock Contention (Top 10):")
    retry = query("""
        SELECT TENANT_NAME, SQL_ID, RETRY_CNT, CONCURRENCY_WAIT_TIME,
               ROUND(ELAPSED_TIME/1000,1) AS elapsed_ms,
               LEFT(QUERY_SQL, 100) AS sql_text
        FROM oceanbase.GV$OB_SQL_AUDIT
        WHERE IS_INNER_SQL=0 AND RETRY_CNT>0
        ORDER BY RETRY_CNT DESC LIMIT 10
    """)
    if not retry:
        print("  未发现重试SQL")
    for r in retry:
        print(f"  租户={r['TENANT_NAME']}  重试={r['RETRY_CNT']}  "
              f"耗时={r['elapsed_ms']}ms  SQL={r['sql_text']}")

    # 6. Hint usage
    for hint, label in [('/*+append*/', 'append'), ('/*+ PARALLEL', 'parallel')]:
        print(f"\n[6] Hint Usage: {hint}")
        h = query(f"""
            SELECT TENANT_NAME, SQL_ID, ROUND(ELAPSED_TIME/1000,1) AS elapsed_ms,
                   AFFECTED_ROWS, RET_CODE, IS_HIT_PLAN,
                   LEFT(QUERY_SQL, 120) AS sql_text
            FROM oceanbase.GV$OB_SQL_AUDIT
            WHERE IS_INNER_SQL=0 AND QUERY_SQL LIKE %s
            ORDER BY ELAPSED_TIME DESC LIMIT 10
        """, (f'%{hint}%',))
        if not h:
            print(f"  未发现使用 {label} hint 的 SQL")
        for x in h:
            err = f" ⚠ 错误码={x['RET_CODE']}" if x['RET_CODE'] else ""
            print(f"  {x['TENANT_NAME']}/{x['SQL_ID']}  {x['elapsed_ms']}ms  "
                  f"影响行={x['AFFECTED_ROWS']}{err}  SQL={x['sql_text']}")

    conn.close()
    print("\n" + "=" * 60)

# 默认执行: 阈值1s, Top20
slow_query_analysis(threshold_ms=1000, limit=20)
```

---

## Key Columns in `GV$OB_SQL_AUDIT`

| Column | Type | Meaning |
|--------|------|---------|
| `QUERY_SQL` | longtext | SQL statement text |
| `ELAPSED_TIME` | bigint (μs) | Total wall-clock time from receive to reply |
| `EXECUTE_TIME` | bigint (μs) | Pure execution time (excluding queue/parse) |
| `QUEUE_TIME` | bigint (μs) | Time spent waiting in queue |
| `GET_PLAN_TIME` | bigint (μs) | Time to get/compile execution plan |
| `DISK_READS` | bigint | Physical SSStore reads (0 = fully cached) |
| `TABLE_SCAN` | tinyint | 1 = full table scan detected |
| `IS_HIT_PLAN` | tinyint | 1 = plan cache hit, 0 = hard parse |
| `RETURN_ROWS` | bigint | Rows returned to client |
| `RETRY_CNT` | bigint | Number of retries (>0 = contention) |
| `USER_IO_WAIT_TIME` | bigint (μs) | I/O wait time |
| `CONCURRENCY_WAIT_TIME` | bigint (μs) | Lock/latch wait time |
| `REQUEST_TYPE` | bigint | 1=local, 2=remote, 3=distributed |

> **Key signals**:
> - `DISK_READS > 0` → index miss or full table scan
> - `TABLE_SCAN = 1` → missing or unused index
> - `IS_HIT_PLAN = 0` → hard parse (check plan cache)
> - `RETRY_CNT > 0` → lock contention or version conflict

---

## Analyzing a Slow Query

### Step 1: Get the Execution Plan

```sql
-- MySQL Mode
EXPLAIN SELECT * FROM orders WHERE customer_id = 123;

-- Oracle Mode
EXPLAIN PLAN FOR SELECT * FROM orders WHERE customer_id = 123;
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
```

### Step 2: Identify the Problem

| Plan Operator | Problem | Fix |
|---------------|---------|-----|
| `TABLE SCAN` | Full table scan | Add index |
| `NESTED LOOP JOIN` | Slow for large joins | Add index to enable `HASH JOIN` |
| `SORT` | Expensive sort | Add index with `ORDER BY` columns |
| `TABLE ACCESS BY INDEX ROWID` | Index lookup + table access | Use covering index |

### Step 3: Apply Fix and Verify

```sql
-- Add index
CREATE INDEX idx_orders_customer ON orders(customer_id);

-- Flush plan cache (so new plan is used)
ALTER SYSTEM FLUSH PLAN CACHE;

-- Re-check performance
SELECT * FROM oceanbase.DBA_OB_SQL_AUDIT WHERE query LIKE '%customer_id = 123%';
```

---

## Common Slow Query Patterns

### 1. Missing Index

```sql
-- Slow: full table scan
SELECT * FROM orders WHERE customer_id = 123;

-- Fix: add index
CREATE INDEX idx_orders_customer ON orders(customer_id);
```

### 2. SELECT *

```sql
-- Slow: reads all columns + may cause table access
SELECT * FROM employees WHERE dept_id = 10;

-- Fix: select only needed columns
SELECT emp_id, emp_name FROM employees WHERE dept_id = 10;
```

### 3. N+1 Query Problem

```sql
-- Slow: N+1 queries (app code)
-- SELECT * FROM orders WHERE customer_id = 1;
-- SELECT * FROM orders WHERE customer_id = 2;
-- ... (N times)

-- Fix: single query with IN
SELECT * FROM orders WHERE customer_id IN (1, 2, 3, ..., N);
```

---

## MySQL Mode vs Oracle Mode

| Aspect | MySQL Mode | Oracle Mode |
|--------|-----------|-------------|
| Audit view (sys) | `oceanbase.GV$OB_SQL_AUDIT` | `sys.v$sql_audit` |
| EXPLAIN | `EXPLAIN sql` | `EXPLAIN PLAN FOR sql` |

---

## Common Mistakes

1. **Not enabling SQL audit**
   - ❌ `SHOW PARAMETERS LIKE 'enable_sql_audit'` = FALSE
   - ✅ `ALTER SYSTEM SET enable_sql_audit = 'TRUE';`

2. **Analyzing wrong query**
   - ❌ Single slow query ≠ always slow (check `ELAPSED_TIME` of each execution)
   - ✅ Check both `ELAPSED_TIME` (total) and `QUEUE_TIME` / `GET_PLAN_TIME` (overhead breakdown)

3. **Adding index then forgetting to flush plan cache**
   - ❌ Old slow plan still used
   - ✅ `ALTER SYSTEM FLUSH PLAN CACHE;`

4. **Not checking actual rows vs estimated rows**
   - ❌ Trusting the plan without verifying
   - ✅ Run query and check actual rows vs `EST. ROWS`

5. **Fixing SQL without testing on staging**
   - ❌ Adding index directly on production
   - ✅ Test index creation + query perf on staging first

---

## OceanBase Version Notes

### V4.2 (Baseline)
- `DBA_OB_SQL_AUDIT` view
- Plan cache flush
- EXPLAIN analysis

### V4.4+
- More detailed audit columns (wait events, I/O timings)
- Faster plan cache flush

---

## Related Skills

- `skills/monitoring/sql-diagnostics.md` — SQL 一键诊断脚本（含 hint 检测、高磁盘读、重试锁等待聚合）
- `skills/monitoring/session-monitor.md` — Active session monitoring
- `skills/performance/explain-plan.md` — Execution plan analysis
- `skills/appdev/sys-tenant-connection.md` — Sys tenant connection

---

## Sources
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/slow-query
- https://www.oceanbase.com/docs/oceanbase-database-cn/v4.2.5/slow-query
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/sql-audit
