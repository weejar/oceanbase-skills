# 分区策略

> 适用版本：OceanBase V4.2+
> 兼容模式：MySQL Mode / Oracle Mode

---

## 概述

分区将大表按规则拆分为多个物理分区，每个分区可存储在不同节点上，实现数据分布和查询裁剪。

| 分区类型 | 适用场景 | 数据分布 | 分区裁剪 |
|---------|---------|---------|---------|
| Hash | 均匀分布，无时间维度 | 均匀 | 等值查询 |
| Range | 按时间/数值范围 | 有序 | 范围查询 |
| List | 枚举值分区 | 按枚举 | 等值查询 |
| Range Columns | 按列值范围 | 有序 | 范围查询 |
| Key | 类似 Hash，不指定列 | 均匀 | 等值查询 |

---

## 1. Hash 分区

### 适用场景
- 数据均匀分布
- 按主键做等值查询
- 没有明显的时间或范围维度

```sql
-- MySQL Mode
CREATE TABLE user_profile (
  user_id   BIGINT NOT NULL,
  nickname  VARCHAR(64),
  email     VARCHAR(256),
  created_at DATETIME,
  PRIMARY KEY (user_id)
) PARTITION BY HASH(user_id) PARTITIONS 16;

-- Oracle Mode
CREATE TABLE user_profile (
  user_id   NUMBER(20) NOT NULL PRIMARY KEY,
  nickname  VARCHAR2(64),
  email     VARCHAR2(256),
  created_at DATE
)
PARTITION BY HASH(user_id)
PARTITIONS 16;
```

### 分区数选择

| 数据量 | 建议分区数 |
|--------|-----------|
| < 1000万 | 4-8 |
| 1000万-1亿 | 8-16 |
| 1亿-10亿 | 16-64 |
| > 10亿 | 64-256 |

```sql
-- 修改分区数（OceanBase 支持 Online）
ALTER TABLE user_profile COALESCE PARTITION 4;  -- 减少4个
ALTER TABLE user_profile ADD PARTITION PARTITIONS 8;  -- 增加8个
```

---

## 2. Range 分区

### 2.1 按时间范围

```sql
-- MySQL Mode：日志表按月分区
CREATE TABLE access_log (
  id        BIGINT NOT NULL AUTO_INCREMENT,
  log_time  DATETIME NOT NULL,
  user_id   BIGINT,
  action    VARCHAR(32),
  detail    TEXT,
  PRIMARY KEY (id, log_time)
) PARTITION BY RANGE (TO_DAYS(log_time)) (
  PARTITION p202401 VALUES LESS THAN (TO_DAYS('2024-02-01')),
  PARTITION p202402 VALUES LESS THAN (TO_DAYS('2024-03-01')),
  PARTITION p202403 VALUES LESS THAN (TO_DAYS('2024-04-01')),
  PARTITION p202404 VALUES LESS THAN (TO_DAYS('2024-05-01')),
  PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

### 2.2 自动分区维护

```sql
-- 新增分区（不影响在线服务）
ALTER TABLE access_log ADD PARTITION (
  PARTITION p202405 VALUES LESS THAN (TO_DAYS('2024-06-01'))
);

-- 删除分区（快速清除历史数据）
ALTER TABLE access_log DROP PARTITION p202401;

-- 查看分区信息
SELECT PARTITION_NAME, TABLE_ROWS, DATA_LENGTH
FROM information_schema.PARTITIONS
WHERE TABLE_NAME = 'access_log';
```

### 2.3 Oracle Mode

```sql
CREATE TABLE access_log (
  id        NUMBER(20) NOT NULL,
  log_time  DATE NOT NULL,
  user_id   NUMBER(20),
  action    VARCHAR2(32),
  detail    CLOB,
  CONSTRAINT pk_access_log PRIMARY KEY (id, log_time)
)
PARTITION BY RANGE (log_time) (
  PARTITION p202401 VALUES LESS THAN (TO_DATE('2024-02-01','YYYY-MM-DD')),
  PARTITION p202402 VALUES LESS THAN (TO_DATE('2024-03-01','YYYY-MM-DD')),
  PARTITION p_future VALUES LESS THAN (MAXVALUE)
);

-- Interval 自动分区（V4.3+）
CREATE TABLE access_log_auto (
  id        NUMBER(20) NOT NULL,
  log_time  DATE NOT NULL,
  PRIMARY KEY (id, log_time)
)
PARTITION BY RANGE (log_time)
INTERVAL (INTERVAL '1' MONTH)
(
  PARTITION p0 VALUES LESS THAN (TO_DATE('2024-01-01','YYYY-MM-DD'))
);
```

---

## 3. Range Columns 分区

直接按列值分区，无需转换函数，支持多列：

```sql
CREATE TABLE orders (
  order_id   BIGINT NOT NULL,
  order_date DATE NOT NULL,
  region     VARCHAR(32) NOT NULL,
  amount     DECIMAL(12,2),
  PRIMARY KEY (order_id, order_date, region)
) PARTITION BY RANGE COLUMNS(order_date, region) (
  PARTITION p2024_east VALUES LESS THAN ('2024-01-01', 'south'),
  PARTITION p2024_west VALUES LESS THAN ('2024-06-01', 'south'),
  PARTITION p_future    VALUES LESS THAN (MAXVALUE, MAXVALUE)
);
```

---

## 4. List 分区

按枚举值分区：

```sql
CREATE TABLE orders (
  order_id  BIGINT NOT NULL,
  region    VARCHAR(32) NOT NULL,
  status    VARCHAR(16) NOT NULL,
  amount    DECIMAL(12,2),
  PRIMARY KEY (order_id, region)
) PARTITION BY LIST COLUMNS(region) (
  PARTITION p_east  VALUES IN ('beijing', 'shanghai', 'nanjing'),
  PARTITION p_south VALUES IN ('guangzhou', 'shenzhen', 'dongguan'),
  PARTITION p_west  VALUES IN ('chengdu', 'chongqing', 'kunming'),
  PARTITION p_other VALUES IN (DEFAULT)
);
```

---

## 5. 二级分区（子分区）

Range + Hash 组合：

```sql
-- 一级按时间Range，二级按ID Hash
CREATE TABLE payment_log (
  id          BIGINT NOT NULL,
  pay_time    DATETIME NOT NULL,
  user_id     BIGINT NOT NULL,
  amount      DECIMAL(12,2),
  PRIMARY KEY (id, pay_time)
) PARTITION BY RANGE (TO_DAYS(pay_time))
  SUBPARTITION BY HASH(user_id) SUBPARTITIONS 4 (
  PARTITION p202401 VALUES LESS THAN (TO_DAYS('2024-02-01')),
  PARTITION p202402 VALUES LESS THAN (TO_DAYS('2024-03-01')),
  PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

---

## 6. 分区裁剪优化

查询必须包含分区键才能裁剪：

```sql
-- ✅ 能裁剪（分区键 log_time 在 WHERE 中）
SELECT * FROM access_log
WHERE log_time BETWEEN '2024-03-01' AND '2024-03-31'
  AND user_id = 123;

-- ❌ 无法裁剪（缺少分区键条件）
SELECT * FROM access_log
WHERE user_id = 123;

-- EXPLAIN 验证
EXPLAIN SELECT * FROM access_log
WHERE log_time = '2024-03-15';
-- 查看是否出现 partition pruning
```

---

## 7. 分区策略选择决策

```
是否有时间维度？
  ├─ 是 → 数据量是否极大？
  │        ├─ 是 → Range + Hash 二级分区
  │        └─ 否 → Range 分区
  └─ 否 → 是否有枚举维度？
           ├─ 是 → List 分区
           └─ 否 → Hash 分区
```

---

## 8. 常见错误

| 问题 | 原因 | 解决 |
|------|------|------|
| `MAXVALUE` 已存在 | 不能有多个 MAXVALUE | 只保留一个 future 分区 |
| 主键不含分区键 | 分区表要求分区键是主键一部分 | 将分区键加入主键 |
| 分区裁剪不生效 | WHERE 条件不含分区键 | 重写查询 |
| 分区不均匀 | Hash 分区数不够 | 增加分区数 |

---

## 参考文档

- 分区表概述：https://www.oceanbase.com/docs/oceanbase-database-cn/V420/partition-table-overview
- Hash 分区：https://www.oceanbase.com/docs/oceanbase-database-cn/V420/hash-partitioning
- Range 分区：https://www.oceanbase.com/docs/oceanbase-database-cn/V420/range-partitioning
- 二级分区：https://www.oceanbase.com/docs/oceanbase-database-cn/V420/subpartitioning
