# 表组设计

> 适用版本：OceanBase V4.2+
> 兼容模式：MySQL Mode / Oracle Mode

---

## 概述

表组（Tablegroup）是 OceanBase 特有的物理设计概念，将多个表的分区绑定到相同的物理节点上，使跨表 JOIN 操作在本地完成，避免分布式网络开销。

| 特性 | 使用表组 | 不使用表组 |
|------|---------|-----------|
| 跨表 JOIN | 本地执行，低延迟 | 分布式执行，网络开销 |
| 数据分布 | 分区间对齐 | 独立分布 |
| 适用场景 | 频繁 JOIN 的大表 | 独立查询或小表 |
| 管理复杂度 | 中 | 低 |

---

## 1. 表组工作原理

```
未使用表组：
  orders (分区1 → Node1, 分区2 → Node2)
  order_items (分区1 → Node2, 分区2 → Node1)
  JOIN → 分布式 shuffle

使用表组：
  orders (分区1 → Node1, 分区2 → Node2)
  order_items (分区1 → Node1, 分区2 → Node2)  -- 同节点
  JOIN → 本地 JOIN
```

---

## 2. 创建表组

### 2.1 创建并绑定

```sql
-- 1. 创建表组
CREATE TABLEGROUP tg_order;

-- 2. 建表时绑定表组
CREATE TABLE orders (
  order_id   BIGINT NOT NULL,
  user_id    BIGINT NOT NULL,
  order_date DATE NOT NULL,
  amount     DECIMAL(12,2),
  PRIMARY KEY (order_id, order_date)
) PARTITION BY RANGE (TO_DAYS(order_date)) PARTITIONS 12
  TABLEGROUP = tg_order;

-- 3. 其他表也绑定同一表组
CREATE TABLE order_items (
  item_id    BIGINT NOT NULL,
  order_id   BIGINT NOT NULL,
  order_date DATE NOT NULL,  -- 必须包含相同的分区键
  product_id BIGINT NOT NULL,
  quantity   INT,
  price      DECIMAL(10,2),
  PRIMARY KEY (item_id, order_date)
) PARTITION BY RANGE (TO_DAYS(order_date)) PARTITIONS 12
  TABLEGROUP = tg_order;
```

### 2.2 修改表组

```sql
-- 将已有表加入表组
ALTER TABLE order_items TABLEGROUP = tg_order;

-- 将表移出表组
ALTER TABLE order_items TABLEGROUP = NULL;

-- 删除表组（需先移除所有表）
ALTER TABLE orders TABLEGROUP = NULL;
ALTER TABLE order_items TABLEGROUP = NULL;
DROP TABLEGROUP tg_order;
```

---

## 3. 使用要求

### 3.1 分区规则必须一致

同一表组内的表必须满足：
- **相同分区类型**（都是 Range / 都是 Hash）
- **相同分区键**（列名和类型）
- **相同分区数**

```sql
-- ✅ 正确：分区键和分区数一致
CREATE TABLE orders (...) PARTITION BY RANGE(TO_DAYS(order_date)) PARTITIONS 12 TABLEGROUP = tg;
CREATE TABLE order_items (...) PARTITION BY RANGE(TO_DAYS(order_date)) PARTITIONS 12 TABLEGROUP = tg;

-- ❌ 错误：分区数不同
CREATE TABLE orders (...) PARTITION BY RANGE(TO_DAYS(order_date)) PARTITIONS 12 TABLEGROUP = tg;
CREATE TABLE order_items (...) PARTITION BY RANGE(TO_DAYS(order_date)) PARTITIONS 8 TABLEGROUP = tg;
-- ERROR: partition count mismatch

-- ❌ 错误：分区类型不同
CREATE TABLE orders (...) PARTITION BY RANGE(TO_DAYS(order_date)) PARTITIONS 12 TABLEGROUP = tg;
CREATE TABLE users (...) PARTITION BY HASH(user_id) PARTITIONS 12 TABLEGROUP = tg;
-- ERROR: partition type mismatch
```

### 3.2 二级分区表组

```sql
CREATE TABLEGROUP tg_payment;

-- 两张表使用相同的二级分区规则
CREATE TABLE payment (
  id          BIGINT NOT NULL,
  pay_date    DATE NOT NULL,
  user_id     BIGINT NOT NULL,
  amount      DECIMAL(12,2),
  PRIMARY KEY (id, pay_date)
) PARTITION BY RANGE (TO_DAYS(pay_date))
  SUBPARTITION BY HASH(user_id) SUBPARTITIONS 4
  TABLEGROUP = tg_payment;

CREATE TABLE payment_detail (
  id          BIGINT NOT NULL,
  pay_date    DATE NOT NULL,
  user_id     BIGINT NOT NULL,
  item_name   VARCHAR(128),
  PRIMARY KEY (id, pay_date)
) PARTITION BY RANGE (TO_DAYS(pay_date))
  SUBPARTITION BY HASH(user_id) SUBPARTITIONS 4
  TABLEGROUP = tg_payment;
```

---

## 4. 查看表组信息

```sql
-- 查看所有表组
SELECT * FROM oceanbase.DBA_OB_TABLEGROUPS;

-- 查看表组中的表
SELECT tablegroup_id, table_id, table_name
FROM oceanbase.DBA_OB_TABLEGROUP_TABLES
WHERE tablegroup_name = 'tg_order';

-- 查看表所属的表组
SELECT table_name, tablegroup_name
FROM oceanbase.DBA_OB_TABLEGROUP_TABLES
WHERE table_name = 'orders';
```

---

## 5. 性能对比

```sql
-- 验证 JOIN 是否走本地
EXPLAIN SELECT o.order_id, o.amount, oi.quantity
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
WHERE o.order_date = '2024-03-15';

-- 使用表组后，EXPLAIN 应该没有远程数据交换
-- 如果看到 "PX COORDINATOR" 或 "EXCHANGE" 则说明仍是分布式
```

---

## 6. 适用场景

### ✅ 推荐使用

| 场景 | 示例 |
|------|------|
| 主表+明细表高频 JOIN | orders + order_items |
| 宽表拆分后联合查询 | payment + payment_detail |
| 多表关联聚合报表 | 多个维度表 JOIN 事实表 |
| 需要跨表事务一致性 | 转账的 debit + credit 表 |

### ❌ 不推荐使用

| 场景 | 原因 |
|------|------|
| 表间很少 JOIN | 无性能收益 |
| 表大小差异极大 | 分区对齐后大表拖慢小表 |
| 分区键不同 | 无法对齐 |
| OLAP 大范围扫描 | 列存副本更适合分析 |

---

## 7. 常见错误

| 问题 | 原因 | 解决 |
|------|------|------|
| `partition count mismatch` | 分区数不一致 | 统一分区数 |
| `partition type mismatch` | 分区类型不同 | 统一分区类型 |
| `tablegroup not found` | 表组不存在 | 先创建表组 |
| JOIN 仍是分布式的 | 分区键不完全匹配 | 确保分区键和范围一致 |
| DROP TABLEGROUP 失败 | 仍有表绑定 | 先 ALTER TABLE TABLEGROUP=NULL |

---

## 参考文档

- 表组概述：https://www.oceanbase.com/docs/oceanbase-database-cn/V420/table-group-overview
- CREATE TABLEGROUP：https://www.oceanbase.com/docs/oceanbase-database-cn/V420/create-tablegroup
- 表组与分区对齐：https://www.oceanbase.com/docs/oceanbase-database-cn/V420/partition-alignment
