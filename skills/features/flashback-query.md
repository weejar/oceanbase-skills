# 闪回查询

> 适用版本：OceanBase V4.2+
> 兼容模式：Oracle Mode（完整）/ MySQL Mode（V4.3+）

---

## 概述

闪回查询（Flashback Query）利用多版本并发控制（MVCC）机制，查询历史时间点的数据，无需从备份恢复。

| 特性 | 闪回查询 | 备份恢复 |
|------|---------|---------|
| 查询方式 | SQL 直接查询 | 需要恢复操作 |
| 时间范围 | 取决于 undo 保留时间 | 取决于备份保留策略 |
| 性能影响 | 低 | 高 |
| 使用场景 | 快速查看误操作前的数据 | 长期数据恢复 |

---

## 1. Oracle Mode

### 1.1 闪回查询

```sql
-- 查询5分钟前的数据
SELECT * FROM orders
AS OF TIMESTAMP (SYSTIMESTAMP - INTERVAL '5' MINUTE)
WHERE order_id = 1001;

-- 查询指定时间点的数据
SELECT * FROM orders
AS OF TIMESTAMP TO_TIMESTAMP('2024-03-15 14:25:00', 'YYYY-MM-DD HH24:MI:SS')
WHERE user_id = 500;

-- 查询指定 SCN 的数据
SELECT * FROM orders
AS OF SCN 1700000000000000
WHERE status = 1;
```

### 1.2 闪回版本查询

```sql
-- 查看行的变更历史
SELECT order_id, status, amount,
  VERSIONS_STARTTIME, VERSIONS_ENDTIME,
  VERSIONS_OPERATION, VERSIONS_XID
FROM orders VERSIONS BETWEEN TIMESTAMP
  TO_TIMESTAMP('2024-03-15 10:00:00', 'YYYY-MM-DD HH24:MI:SS')
  AND TO_TIMESTAMP('2024-03-15 16:00:00', 'YYYY-MM-DD HH24:MI:SS')
WHERE order_id = 1001;

-- VERSIONS_OPERATION: I=INSERT, U=UPDATE, D=DELETE
-- VERSIONS_STARTTIME/ENDTIME: 有效时间范围
-- VERSIONS_XID: 事务ID
```

### 1.3 闪回事务查询

```sql
-- 查看事务的详细变更
SELECT xid, operation, logon_user, undo_change#
FROM FLASHBACK_TRANSACTION_QUERY
WHERE xid = '0002000000000001';
```

### 1.4 恢复误删数据

```sql
-- 场景：误删 orders 中 order_id=1001 的数据

-- 步骤1：确认误删前的数据
SELECT * FROM orders
AS OF TIMESTAMP (SYSTIMESTAMP - INTERVAL '10' MINUTE)
WHERE order_id = 1001;

-- 步骤2：重新插入
INSERT INTO orders
SELECT * FROM orders
AS OF TIMESTAMP (SYSTIMESTAMP - INTERVAL '10' MINUTE)
WHERE order_id = 1001;

COMMIT;

-- 场景：误 UPDATE
-- 步骤1：查看变更前的值
SELECT order_id, status, amount FROM orders
AS OF TIMESTAMP (SYSTIMESTAMP - INTERVAL '10' MINUTE)
WHERE order_id = 1001;

-- 步骤2：恢复
UPDATE orders o SET (status, amount) = (
  SELECT status, amount FROM orders
  AS OF TIMESTAMP (SYSTIMESTAMP - INTERVAL '10' MINUTE)
  WHERE order_id = 1001
)
WHERE order_id = 1001;
```

---

## 2. MySQL Mode（V4.3+）

### 2.1 闪回查询

```sql
-- 查询历史时间点数据
SELECT * FROM orders
AS OF TIMESTAMP DATE_SUB(NOW(), INTERVAL 10 MINUTE)
WHERE order_id = 1001;

-- 查询指定时间
SELECT * FROM orders
AS OF TIMESTAMP '2024-03-15 14:25:00'
WHERE user_id = 500;
```

### 2.2 闪回表

```sql
-- 将表恢复到指定时间点（Oracle Mode）
FLASHBACK TABLE orders TO TIMESTAMP
  (SYSTIMESTAMP - INTERVAL '30' MINUTE);

-- MySQL Mode（如支持）
FLASHBACK TABLE orders TO TIMESTAMP
  DATE_SUB(NOW(), INTERVAL 30 MINUTE);
```

---

## 3. Undo 保留管理

### 3.1 配置 undo 保留时间

```sql
-- Oracle Mode：查看当前 undo 保留时间
SHOW PARAMETER undo_retention;

-- 修改 undo 保留时间（默认通常为 1800 秒 / 30分钟）
ALTER SYSTEM SET undo_retention = 3600 SCOPE = BOTH;

-- 设置较长保留（需更多磁盘空间）
ALTER SYSTEM SET undo_retention = 86400 SCOPE = BOTH; -- 24小时

-- MySQL Mode
SET GLOBAL ob_query_timeout = 3600000000;
```

### 3.2 估算 undo 空间需求

| 保留时间 | 空间估算（假设 TPS=1000） |
|---------|------------------------|
| 30分钟 | ~5-10 GB |
| 1小时 | ~10-20 GB |
| 4小时 | ~40-80 GB |
| 24小时 | ~200-400 GB |

```sql
-- 查看 undo 空间使用
SELECT * FROM oceanbase.__all_virtual_undo_stat;
```

---

## 4. 限制

| 限制 | 说明 |
|------|------|
| 时间窗口 | 不能超过 `undo_retention` 配置 |
| DDL 操作 | DDL 之后的历史版本不可用（TRUNCATE/DROP 后无法闪回） |
| 空间回收 | undo 空间不足时会自动回收旧版本 |
| 跨分区 | 闪回查询可跨分区，但性能取决于 undo 数据分布 |

---

## 5. 最佳实践

1. **设置合理保留时间**：生产建议 ≥ 1 小时
2. **定期检查 undo 空间**：监控 undo 目录磁盘使用
3. **DDL 操作前备份**：TRUNCATE/DROP 前用 `CREATE TABLE AS` 备份
4. **快速响应**：发现误操作后立即闪回，避免 undo 被回收
5. **关键操作记录时间**：记录 DML 操作时间，便于闪回定位

---

## 常见错误

| 问题 | 原因 | 解决 |
|------|------|------|
| `ORA-08180: no snapshot found` | 超出 undo 保留时间 | 减小回溯时间或增大保留 |
| `ORA-01466: unable to read data` | 表结构已变更 | 无法闪回到 DDL 之前的版本 |
| 查询很慢 | undo 数据分散在多个节点 | 缩小时间范围 |
| 空间不足 | undo_retention 设置过大 | 减小保留时间或扩容 |

---

## 参考文档

- Oracle Mode 闪回查询：https://www.oceanbase.com/docs/oceanbase-database-cn/V420/oracle-mode/flashback-query
- MySQL Mode 闪回查询：https://www.oceanbase.com/docs/oceanbase-database-cn/V420/mysql-mode/flashback-query
- Undo 管理：https://www.oceanbase.com/docs/oceanbase-database-cn/V420/undo-management
