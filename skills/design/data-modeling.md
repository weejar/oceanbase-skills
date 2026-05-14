# 数据建模与表设计

> 适用版本：OceanBase V4.2+
> 兼容模式：MySQL Mode / Oracle Mode

---

## 概述

OceanBase 分布式架构下，表设计需考虑数据分布、事务局部性、热点问题和分区策略。

| 设计维度 | 传统单机 | OceanBase 分布式 |
|---------|---------|-----------------|
| 主键选择 | 自增ID即可 | 需避免热点（雪花ID/UUID） |
| 索引设计 | 尽量多建 | 全局索引有分布式开销 |
| 分区策略 | 大表分表 | 原生分区支持 |
| 数据类型 | 随意 | 建议固定长度类型 |
| 外键 | 常用 | 不推荐（分布式性能差） |

---

## 1. 建表规范

### 1.1 MySQL Mode

```sql
-- 推荐的建表模板
CREATE TABLE orders (
  id              BIGINT       NOT NULL AUTO_INCREMENT COMMENT '订单ID',
  user_id         BIGINT       NOT NULL COMMENT '用户ID',
  order_no        VARCHAR(32)  NOT NULL COMMENT '订单号',
  amount          DECIMAL(12,2) NOT NULL COMMENT '金额',
  status          TINYINT      NOT NULL DEFAULT 0 COMMENT '状态:0-待支付,1-已支付,2-已发货',
  created_at      DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  updated_at      DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  PRIMARY KEY (id),
  UNIQUE KEY uk_order_no (order_no),
  KEY idx_user_status (user_id, status),
  KEY idx_created (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
  COMMENT='订单表'
  PARTITION BY HASH(id) PARTITIONS 16;
```

### 1.2 Oracle Mode

```sql
CREATE TABLE orders (
  id              NUMBER(20)    NOT NULL,
  user_id         NUMBER(20)    NOT NULL,
  order_no        VARCHAR2(32)  NOT NULL,
  amount          NUMBER(12,2)  NOT NULL,
  status          NUMBER(1)     DEFAULT 0 NOT NULL,
  created_at      TIMESTAMP     DEFAULT SYSTIMESTAMP NOT NULL,
  updated_at      TIMESTAMP     DEFAULT SYSTIMESTAMP NOT NULL,
  CONSTRAINT pk_orders PRIMARY KEY (id),
  CONSTRAINT uk_orders_no UNIQUE (order_no)
)
TABLESPACE users
PCTFREE 10
STORAGE (INITIAL 8M NEXT 2M);

-- 创建索引
CREATE INDEX idx_orders_user_status ON orders(user_id, status);
CREATE INDEX idx_orders_created ON orders(created_at);

-- 自动生成ID
CREATE SEQUENCE seq_orders START WITH 1 INCREMENT BY 1 CACHE 1000;
```

---

## 2. 主键设计

### 2.1 避免自增ID热点

自增主键（AUTO_INCREMENT / SEQUENCE）在分布式场景下会导致写入集中在最后一个分区。

| 主键类型 | 热点风险 | 适用场景 |
|---------|---------|---------|
| AUTO_INCREMENT | **高**（顺序写入） | 小表、低并发 |
| UUID | 低 | 无范围查询需求 |
| 雪花ID | 低 | 需要有序且避免热点 |
| 业务键+时间 | 低 | 有业务语义 |

### 2.2 推荐方案

```sql
-- 方案1：雪花ID（应用生成，BIGINT）
CREATE TABLE orders (
  id BIGINT NOT NULL COMMENT '雪花ID',
  -- ...
  PRIMARY KEY (id)
) PARTITION BY HASH(id) PARTITIONS 16;

-- 方案2：UUID（MySQL Mode）
CREATE TABLE orders (
  id BINARY(16) NOT NULL COMMENT 'UUID二进制存储',
  -- ...
  PRIMARY KEY (id)
);

-- 方案3：业务前缀 + 时间（便于排序）
-- 应用生成: ORD_20240101_XXXXXX
CREATE TABLE orders (
  order_no VARCHAR(32) NOT NULL,
  -- ...
  PRIMARY KEY (order_no)
) PARTITION BY HASH(order_no) PARTITIONS 16;
```

### 2.3 自增ID优化（必须使用时）

```sql
-- MySQL Mode：使用 AUTO_INCREMENT 但配合 HASH 分区打散
CREATE TABLE orders (
  id BIGINT NOT NULL AUTO_INCREMENT,
  -- ...
  PRIMARY KEY (id)
) PARTITION BY HASH(id) PARTITIONS 32;

-- Oracle Mode：使用 Sequence + CACHE
CREATE SEQUENCE seq_orders CACHE 2000 NOORDER;
-- NOORDER 避免有序分配导致热点
-- 或者使用 ORDER 搭配多个 Sequence 分段
CREATE SEQUENCE seq_orders_1 START WITH 1 INCREMENT BY 100 CACHE 100;
CREATE SEQUENCE seq_orders_2 START WITH 2 INCREMENT BY 100 CACHE 100;
```

---

## 3. 数据类型选择

### 3.1 类型推荐

| 场景 | 推荐类型 | 不推荐 | 原因 |
|------|---------|--------|------|
| 金额 | `DECIMAL(p,s)` | `FLOAT/DOUBLE` | 精度问题 |
| 短文本 | `VARCHAR(n)` 固定长度 | `TEXT` | TEXT 不支持索引 |
| 长文本 | `TEXT` / `CLOB` | 大 VARCHAR | 存储效率 |
| 时间 | `DATETIME` / `TIMESTAMP` | `INT` 存时间戳 | 可读性和函数支持 |
| 布尔 | `TINYINT(1)` / `NUMBER(1)` | `CHAR(1)` | 比较效率 |
| JSON | `JSON`（V4.3+） | `TEXT` 存JSON | 可验证+可索引 |
| 大对象 | `BLOB` / `LONG RAW` | 大 VARCHAR | 存储优化 |

### 3.2 字段长度建议

```sql
-- 精确控制长度，避免浪费
-- 好
name        VARCHAR(64)    -- 精确到业务需求
phone       VARCHAR(20)    -- 足够存储国际号码
email       VARCHAR(256)   -- RFC 标准最大长度
status      TINYINT        -- 枚举值用数字
amount      DECIMAL(16,4)  -- 金额精确到分

-- 不好
name        VARCHAR(500)   -- 过长浪费
description TEXT           -- 短文本用 VARCHAR
```

---

## 4. 反范式设计

分布式场景下减少跨节点 JOIN，适当反范式化：

### 4.1 冗余字段

```sql
-- 订单表冗余用户名称（避免每次 JOIN 用户表）
CREATE TABLE orders (
  id           BIGINT NOT NULL,
  user_id      BIGINT NOT NULL,
  user_name    VARCHAR(64),     -- 冗余：来自 users 表
  user_phone   VARCHAR(20),     -- 冗余：来自 users 表
  -- ...
  PRIMARY KEY (id)
);

-- 通过触发器或应用层同步冗余字段
```

### 4.2 宽表设计

```sql
-- 电商订单宽表（典型反范式）
CREATE TABLE order_detail_wide (
  order_id      BIGINT NOT NULL,
  user_id       BIGINT NOT NULL,
  user_name     VARCHAR(64),
  product_id    BIGINT NOT NULL,
  product_name  VARCHAR(256),   -- 冗余
  category_name VARCHAR(64),    -- 冗余
  shop_name     VARCHAR(128),   -- 冗余
  -- ...
  PRIMARY KEY (order_id)
) PARTITION BY HASH(order_id) PARTITIONS 16;
```

---

## 5. 外键约束

```sql
-- OceanBase 不推荐外键（分布式场景性能差）

-- ❌ 不推荐
ALTER TABLE order_items ADD CONSTRAINT fk_order
  FOREIGN KEY (order_id) REFERENCES orders(id);

-- ✅ 推荐：应用层保证引用完整性
-- 使用索引保证 JOIN 性能即可
CREATE INDEX idx_order_items_order_id ON order_items(order_id);
```

---

## 6. 大表设计

### 6.1 分区表（超过5000万行建议分区）

```sql
-- 范围分区（按时间）
CREATE TABLE log_entries (
  id          BIGINT NOT NULL,
  log_time    DATETIME NOT NULL,
  service     VARCHAR(64),
  level       VARCHAR(10),
  message     TEXT,
  PRIMARY KEY (id, log_time)
) PARTITION BY RANGE (TO_DAYS(log_time)) (
  PARTITION p202401 VALUES LESS THAN (TO_DAYS('2024-02-01')),
  PARTITION p202402 VALUES LESS THAN (TO_DAYS('2024-03-01')),
  PARTITION p202403 VALUES LESS THAN (TO_DAYS('2024-04-01')),
  PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

### 6.2 自动分区维护

```sql
-- 每月自动新增分区
CREATE TABLE monthly_data (
  id       BIGINT NOT NULL,
  biz_date DATE NOT NULL,
  data     JSON,
  PRIMARY KEY (id, biz_date)
) PARTITION BY RANGE (TO_DAYS(biz_date)) (
  PARTITION p_start VALUES LESS THAN (TO_DAYS('2024-01-01'))
);

-- 定期执行
ALTER TABLE monthly_data ADD PARTITION (
  PARTITION p202404 VALUES LESS THAN (TO_DAYS('2024-05-01'))
);
```

---

## 7. 表设计检查清单

| 检查项 | 要求 |
|--------|------|
| 主键 | 必须有主键，避免热点 |
| 分区 | 大表（>5000万行）必须分区 |
| 字段长度 | 按业务需求精确控制 |
| 索引数量 | 单表索引≤5个，联合索引≤3列 |
| 外键 | 不使用外键 |
| 数据类型 | 金额用 DECIMAL，时间用 DATETIME |
| 字符集 | 统一 utf8mb4 |
| 注释 | 表和关键字段必须有 COMMENT |
| NOT NULL | 关键字段尽量 NOT NULL |
| DEFAULT | 有合理默认值 |

---

## 参考文档

- 建表语法：https://www.oceanbase.com/docs/oceanbase-database-cn/V420/mysql-mode/create-a-table
- 分区表：https://www.oceanbase.com/docs/oceanbase-database-cn/V420/partition-table
- 数据类型：https://www.oceanbase.com/docs/oceanbase-database-cn/V420/mysql-mode/data-types
- Oracle Mode 建表：https://www.oceanbase.com/docs/oceanbase-database-cn/V420/oracle-mode/create-table
