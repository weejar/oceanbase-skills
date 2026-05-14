# 物化视图

> 适用版本：OceanBase V4.2+
> 兼容模式：Oracle Mode（完整）/ MySQL Mode（V4.3+）

---

## 概述

物化视图（Materialized View）将查询结果物化存储为表，避免每次重复计算。适合复杂聚合查询和报表场景。

| 特性 | 普通视图 | 物化视图 |
|------|---------|---------|
| 存储 | 仅存储定义 | 存储实际数据 |
| 查询性能 | 每次重新执行 | 直接读取预计算数据 |
| 数据更新 | 实时 | 需要刷新（ON DEMAND / ON COMMIT） |
| 索引 | 不支持 | 可创建索引 |
| 使用场景 | 简化SQL | 加速复杂查询 |

---

## 1. Oracle Mode

### 1.1 创建物化视图

```sql
-- 基本物化视图（手动刷新）
CREATE MATERIALIZED VIEW mv_order_daily
BUILD IMMEDIATE
REFRESH COMPLETE ON DEMAND
ENABLE QUERY REWRITE
AS
SELECT
  order_date,
  COUNT(*) AS order_count,
  SUM(amount) AS total_amount,
  AVG(amount) AS avg_amount
FROM orders
GROUP BY order_date;

-- 快速刷新（需要物化视图日志）
CREATE MATERIALIZED VIEW LOG ON orders
  WITH PRIMARY KEY, ROWID
  INCLUDING NEW VALUES;

CREATE MATERIALIZED VIEW mv_customer_orders
BUILD IMMEDIATE
REFRESH FAST ON COMMIT
ENABLE QUERY REWRITE
AS
SELECT
  customer_id,
  COUNT(*) AS order_count,
  SUM(amount) AS total_amount,
  MAX(order_date) AS last_order_date
FROM orders
GROUP BY customer_id;
```

### 1.2 刷新策略

| 刷新方式 | 语法 | 说明 |
|---------|------|------|
| COMPLETE | `REFRESH COMPLETE` | 全量重建 |
| FAST | `REFRESH FAST` | 增量刷新（需要 MV LOG） |
| FORCE | `REFRESH FORCE` | 优先 FAST，不支持时 COMPLETE |
| ON DEMAND | `ON DEMAND` | 手动刷新 |
| ON COMMIT | `ON COMMIT` | 事务提交自动刷新 |

```sql
-- 手动刷新
BEGIN
  DBMS_MVIEW.REFRESH('mv_order_daily', 'C');  -- COMPLETE
  DBMS_MVIEW.REFRESH('mv_customer_orders', 'F'); -- FAST
END;
/

-- 查看刷新状态
SELECT mview_name, last_refresh_date, refresh_method, staleness
FROM user_mviews;

-- 查看刷新历史
SELECT mview_name, refresh_start, refresh_end, refresh_method
FROM user_mview_refresh_times;
```

### 1.3 查询改写

```sql
-- 启用查询改写
ALTER SESSION SET query_rewrite_enabled = TRUE;
ALTER SESSION SET query_rewrite_integrity = enforced;

-- 原始查询会自动改写为查物化视图
-- 下面查询等价于 SELECT * FROM mv_order_daily
SELECT
  order_date,
  COUNT(*) AS order_count,
  SUM(amount) AS total_amount
FROM orders
GROUP BY order_date;

-- 验证是否使用物化视图
EXPLAIN PLAN FOR
SELECT order_date, SUM(amount) FROM orders GROUP BY order_date;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
-- 查找 "MAT_VIEW REWRITE ACCESS" 字样
```

### 1.4 管理

```sql
-- 修改刷新方式
ALTER MATERIALIZED VIEW mv_order_daily REFRESH COMPLETE ON DEMAND;

-- 编译（失效后）
ALTER MATERIALIZED VIEW mv_order_daily COMPILE;

-- 删除
DROP MATERIALIZED VIEW mv_order_daily;

-- 查看物化视图信息
SELECT mview_name, container_table, refresh_method, last_refresh
FROM user_mviews;
```

---

## 2. MySQL Mode（V4.3+）

### 2.1 创建物化视图

```sql
-- 基本物化视图
CREATE MATERIALIZED VIEW mv_order_daily
REFRESH COMPLETE ON DEMAND
AS
SELECT
  DATE(created_at) AS order_date,
  COUNT(*) AS order_count,
  SUM(amount) AS total_amount
FROM orders
GROUP BY DATE(created_at);

-- 刷新
REFRESH MATERIALIZED VIEW mv_order_daily;

-- 查看状态
SHOW CREATE MATERIALIZED VIEW mv_order_daily\G
```

---

## 3. 性能优化

### 3.1 为物化视图建索引

```sql
-- 在物化视图上建索引加速查询
CREATE INDEX idx_mv_date ON mv_order_daily(order_date);
CREATE INDEX idx_mv_customer ON mv_customer_orders(customer_id);
```

### 3.2 刷新调度

```sql
-- Oracle Mode：使用 DBMS_SCHEDULER 定时刷新
BEGIN
  DBMS_SCHEDULER.CREATE_JOB(
    job_name        => 'refresh_mv_daily',
    job_type        => 'PLSQL_BLOCK',
    job_action      => 'BEGIN DBMS_MVIEW.REFRESH(''mv_order_daily'', ''C''); END;',
    start_date      => SYSTIMESTAMP,
    repeat_interval => 'FREQ=DAILY; BYHOUR=2; BYMINUTE=0',
    enabled         => TRUE
  );
END;
/

-- MySQL Mode：使用定时事件
CREATE EVENT refresh_mv_daily
ON SCHEDULE EVERY 1 DAY STARTS '2024-01-01 02:00:00'
DO
  REFRESH MATERIALIZED VIEW mv_order_daily;
```

---

## 4. 使用场景

| 场景 | 物化视图方案 | 刷新策略 |
|------|------------|---------|
| 日报表 | 按日聚合 | 每天凌晨全量刷新 |
| 实时大屏 | 分钟级聚合 | 每5分钟快速刷新 |
| 历史趋势 | 月/年聚合 | 月度全量刷新 |
| 跨库JOIN结果 | 预计算 JOIN | 按需手动刷新 |

---

## 5. 常见错误

| 问题 | 原因 | 解决 |
|------|------|------|
| `ORA-12014: cannot create fast refresh` | 不满足快速刷新条件 | 检查 MV LOG 或 SQL |
| 查询未改写 | `query_rewrite_integrity` 设置 | 设为 `enforced` |
| 物化视图失效 | 基表 DDL 变更 | `ALTER MVIEW COMPILE` |
| 刷新慢 | 数据量大 | 用 FAST 刷新或并行刷新 |
| MySQL Mode 不支持 | 版本过低 | 升级到 V4.3+ |

---

## 参考文档

- Oracle Mode 物化视图：https://www.oceanbase.com/docs/oceanbase-database-cn/V420/oracle-mode/materialized-view
- MySQL Mode 物化视图：https://www.oceanbase.com/docs/oceanbase-database-cn/V420/mysql-mode/materialized-view
- 查询改写：https://www.oceanbase.com/docs/oceanbase-database-cn/V420/query-rewrite
