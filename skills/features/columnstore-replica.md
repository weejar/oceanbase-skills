# 列存副本（HTAP分析加速）

> 适用版本：OceanBase V4.2+（V4.3 增强支持）
> 兼容模式：MySQL Mode / Oracle Mode

---

## 概述

列存副本（Column Store Replica）是 OceanBase HTAP 的核心组件。在同一份数据上同时维护行存副本（OLTP）和列存副本（OLAP），实现事务处理和分析查询的融合。

| 特性 | 行存副本 | 列存副本 |
|------|---------|---------|
| 存储格式 | 行式存储 | 列式存储 |
| 擅长场景 | 点查、小范围查询 | 全表扫描、聚合分析 |
| 索引 | B+树索引 | 列级压缩 |
| 更新方式 | 实时 | 微批异步同步 |
| 适用负载 | OLTP | OLAP |

---

## 1. 架构原理

```
写入 → 行存主副本（实时写入）
         ↓ 同步
       行存从副本（Paxos）
         ↓ 异步微批
       列存副本（列式存储）
         ↑
       OLAP 分析查询
```

---

## 2. 创建列存副本

### 2.1 为表创建列存副本

```sql
-- Oracle Mode
ALTER TABLE orders MODIFY REPLICA (
  ADD COLUMN STORE REPLICA (
    locality = 'F@zone1,F@zone2,F@zone3'
  )
);

-- MySQL Mode
ALTER TABLE orders SET COLUMN STORE REPLICA
  LOCALITY = 'F@zone1,F@zone2,F@zone3';

-- 或使用资源隔离语法
ALTER TABLE orders MODIFY REPLICA (
  ADD COLUMN STORE REPLICA (LOCALITY = 'FULL, FULL, FULL')
);
```

### 2.2 为租户全局配置

```sql
-- 在租户级别开启列存
ALTER SYSTEM SET enable_column_store = TRUE;

-- 设置列存刷新间隔
ALTER SYSTEM SET column_store_refresh_interval = '30s';

-- 设置列存内存限制
ALTER SYSTEM SET column_store_memory_limit = '2G';
```

---

## 3. 列存使用场景

### 3.1 适合列存的查询模式

```sql
-- ✅ 适合：全表聚合
SELECT region, SUM(amount), COUNT(*)
FROM orders
GROUP BY region;

-- ✅ 适合：多列过滤
SELECT * FROM orders
WHERE status = 1 AND region = 'east' AND amount > 1000;

-- ✅ 适合：列数少的大范围扫描
SELECT customer_id, SUM(amount) AS total
FROM orders
WHERE order_date BETWEEN '2024-01-01' AND '2024-12-31'
GROUP BY customer_id
ORDER BY total DESC
LIMIT 100;

-- ✅ 适合：报表查询
SELECT
  DATE(created_at) AS dt,
  status,
  COUNT(*) AS cnt,
  AVG(amount) AS avg_amount
FROM orders
WHERE created_at >= '2024-01-01'
GROUP BY DATE(created_at), status;
```

### 3.2 不适合列存的查询

```sql
-- ❌ 不适合：主键点查
SELECT * FROM orders WHERE order_id = 12345;
-- 行存 B+树 索引更快

-- ❌ 不适合：返回少量行
SELECT * FROM orders WHERE user_id = 100 LIMIT 10;
-- 行存索引更高效

-- ❌ 不适合：写密集操作
INSERT INTO orders ...; -- 写入走行存
```

---

## 4. 查询路由

### 4.1 自动路由

```sql
-- OceanBase 优化器自动选择行存或列存
-- 通过 HINT 强制使用列存
SELECT /*+ READ_FROM_COLUMN_STORE(orders) */
  region, SUM(amount)
FROM orders
GROUP BY region;

-- 强制使用行存
SELECT /*+ READ_FROM_ROW_STORE(orders) */
  * FROM orders WHERE order_id = 12345;
```

### 4.2 验证执行计划

```sql
-- EXPLAIN 查看是否使用列存
EXPLAIN
SELECT region, SUM(amount) FROM orders GROUP BY region;

-- 查找 "COLUMN STORE SCAN" 字样表示使用了列存
```

---

## 5. 列存管理

### 5.1 查看列存状态

```sql
-- 查看表的列存副本状态
SELECT table_id, table_name, column_store_status, sync_progress
FROM oceanbase.DBA_OB_TABLE_REPLICAS
WHERE table_name = 'ORDERS';

-- 查看列存同步延迟
SELECT * FROM oceanbase.__all_virtual_column_store_replica
WHERE table_id = <table_id>;

-- 查看列存内存使用
SELECT * FROM oceanbase.__all_virtual_column_store_mem_stat;
```

### 5.2 列存同步

```sql
-- 手动触发列存刷新（Oracle Mode）
ALTER TABLE orders REFRESH COLUMN STORE;

-- 修改刷新间隔
ALTER SYSTEM SET column_store_refresh_interval = '10s';
```

### 5.3 删除列存

```sql
ALTER TABLE orders MODIFY REPLICA (
  DROP COLUMN STORE REPLICA
);
```

---

## 6. 性能优化

### 6.1 列存压缩

```sql
-- 配置列存压缩
ALTER SYSTEM SET column_store_compress_strategy = 'ZSTD';

-- 查看压缩率
SELECT table_name, raw_size, compressed_size,
  ROUND((1 - compressed_size / raw_size) * 100, 2) AS compress_ratio
FROM oceanbase.DBA_OB_COLUMN_STORE_STATS;
```

### 6.2 列存内存管理

```sql
-- 设置列存可用内存
ALTER SYSTEM SET column_store_memory_limit = '4G';

-- 列存缓存淘汰策略
ALTER SYSTEM SET column_store_cache_policy = 'LRU';
```

### 6.3 优化建议

| 优化项 | 建议 |
|--------|------|
| 刷新间隔 | 实时性要求高 → 10s，允许延迟 → 60s-300s |
| 内存分配 | 分析查询多的租户分配更多列存内存 |
| 选择性建列存 | 仅大表和频繁分析查询的表建列存 |
| 避免小表列存 | 小表列存收益有限，浪费资源 |

---

## 7. 与实时物化视图对比

| 特性 | 列存副本 | 实时物化视图 |
|------|---------|------------|
| 数据同步 | 微批异步 | ON COMMIT 实时 |
| 聚合预计算 | 不支持 | 支持 |
| 存储开销 | 高（列式复制） | 中（聚合后） |
| 查询灵活性 | 灵活（任意查询） | 受限于视图定义 |
| 维护成本 | 低（自动同步） | 中（需刷新策略） |
| 适用场景 | 通用分析 | 固定报表 |

---

## 常见错误

| 问题 | 原因 | 解决 |
|------|------|------|
| 查询未走列存 | 优化器选择行存 | 使用 `READ_FROM_COLUMN_STORE` HINT |
| 列存延迟大 | 刷新间隔过长 | 缩小 `column_store_refresh_interval` |
| 内存不足 | 列存内存限制太小 | 增大 `column_store_memory_limit` |
| 创建列存失败 | 资源不足 | 检查磁盘和内存 |

---

## 参考文档

- 列存副本：https://www.oceanbase.com/docs/oceanbase-database-cn/V420/column-store
- HTAP 架构：https://www.oceanbase.com/docs/oceanbase-database-cn/V420/htap-architecture
- 列存参数：https://www.oceanbase.com/docs/oceanbase-database-cn/V420/column-store-parameters
