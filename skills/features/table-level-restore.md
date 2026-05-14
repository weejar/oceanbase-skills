# 表级恢复

> 适用版本：OceanBase V4.2+
> 兼容模式：MySQL Mode / Oracle Mode

---

## 概述

表级恢复（Table-Level Restore）允许从备份集中恢复单个表或部分表，无需恢复整个数据库，大幅缩短恢复时间。

| 特性 | 全库恢复 | 表级恢复 |
|------|---------|---------|
| 恢复范围 | 整个集群/租户 | 单个或多个表 |
| 恢复时间 | 小时级 | 分钟级 |
| 对业务影响 | 需要停机 | 可并行，影响小 |
| 磁盘需求 | 等于全库大小 | 仅恢复表大小 |
| 适用场景 | 灾难恢复 | 误删表、误改数据 |

---

## 1. 前提条件

### 1.1 备份要求

表级恢复需要以下备份条件满足：

```sql
-- 1. 确认数据备份存在
SELECT backup_set_id, tenant_id, backup_type, start_time, end_time, status
FROM oceanbase.DBA_OB_BACKUP_SET_DETAILS
WHERE status = 'SUCCESS'
ORDER BY end_time DESC
LIMIT 5;

-- 2. 确认日志归档正常（用于 PITR）
SELECT tenant_id, dest_id, round_value, status, start_time, end_time
FROM oceanbase.DBA_OB_ARCHIVE_LOG_SUMMARY
ORDER BY end_time DESC;

-- 3. 确认备份集包含目标表
-- 查看备份详情
SELECT * FROM oceanbase.DBA_OB_BACKUP_SET_DETAILS
WHERE backup_set_id = <set_id>;
```

### 1.2 系统要求

| 要求 | 说明 |
|------|------|
| 备份集完整 | 数据备份 + 日志归档 |
| 磁盘空间 | 恢复表所需空间的 1.5-2 倍 |
| 租户状态 | 目标租户在线 |
| 版本兼容 | 恢复工具版本 ≥ 备份版本 |

---

## 2. 表级恢复操作

### 2.1 使用 obadmin 工具（推荐）

```bash
# 查看可用备份
obadmin backup list --tenant_id=1001

# 恢复单个表到指定时间点
obadmin restore table \
  --tenant_id=1001 \
  --backup_set_id=20240315120000 \
  --table_list="app_db.orders,app_db.order_items" \
  --restore_scn=1700000000000000 \
  --target_tenant=1001

# 恢复到临时租户（安全方式）
obadmin restore table \
  --tenant_id=1001 \
  --backup_set_id=20240315120000 \
  --table_list="app_db.orders" \
  --target_tenant=2001 \
  --target_database="recovery_db"
```

### 2.2 使用 SQL 命令（V4.3+）

```sql
-- Oracle Mode
-- 恢复单个表
ALTER SYSTEM RESTORE TABLE app_db.orders
  FROM BACKUP SET 20240315120000
  UNTIL SCN 1700000000000000;

-- 恢复多个表
ALTER SYSTEM RESTORE TABLE app_db.orders, app_db.order_items
  FROM BACKUP SET 20240315120000
  UNTIL TIMESTAMP '2024-03-15 14:30:00';

-- 恢复到临时位置
ALTER SYSTEM RESTORE TABLE app_db.orders
  FROM BACKUP SET 20240315120000
  TO SCHEMA recovery_db;
```

---

## 3. 恢复流程

### 3.1 标准恢复流程

```
1. 确认备份集可用
     ↓
2. 确定恢复目标（表名 + 时间点）
     ↓
3. 评估恢复时间和空间
     ↓
4. 执行表级恢复（到临时库）
     ↓
5. 验证恢复数据正确性
     ↓
6. 将数据导回生产表
     ↓
7. 清理临时库
```

### 3.2 误删表恢复

```bash
# 场景：app_db.orders 表被误 DROP

# 步骤1：创建临时租户（或使用现有测试租户）
# 确保有足够的资源

# 步骤2：恢复表到临时租户
obadmin restore table \
  --tenant_id=1001 \
  --backup_set_id=<latest_backup> \
  --table_list="app_db.orders" \
  --target_tenant=2001

# 步骤3：从临时租户导出数据
obdumper -h 10.0.0.1 -P 2881 -u root -p --tenant=tenant_temp \
  --table="orders" --output-dir=/data/restore/

# 步骤4：导入到生产（如果需要重建表）
# 先重建表结构
mysql -h obhost -P 2881 -u root@prod -p -e "
  CREATE TABLE IF NOT EXISTS app_db.orders (...);
"

# 步骤5：导入数据
obloader -h 10.0.0.1 -P 2881 -u root@p --tenant=prod \
  --table="orders" --input-dir=/data/restore/

# 步骤6：验证数据
mysql -h obhost -P 2881 -u root@prod -p -e "
  SELECT COUNT(*), MIN(created_at), MAX(created_at)
  FROM app_db.orders;
"
```

### 3.3 误改数据恢复

```bash
# 场景：orders 表数据被误 UPDATE

# 步骤1：恢复到临时库
obadmin restore table \
  --tenant_id=1001 \
  --table_list="app_db.orders" \
  --restore_scn=<误操作之前的SCN> \
  --target_tenant=2001

# 步骤2：对比数据，提取需要恢复的行
mysql -h obhost -P 2881 -u root@temp -p -e "
  SELECT * FROM app_db.orders
  WHERE order_id IN (
    SELECT order_id FROM diff_ids
  );
" > /data/restore/corrected_data.sql

# 步骤3：更新回生产
mysql -h obhost -P 2881 -u root@prod -p < /data/restore/corrected_data.sql
```

---

## 4. 恢复进度监控

```sql
-- 查看恢复任务状态
SELECT task_id, tenant_id, table_name, status, progress, start_time
FROM oceanbase.DBA_OB_RESTORE_PROGRESS;

-- 查看恢复进度百分比
SELECT task_id,
  ROUND(completed_bytes / total_bytes * 100, 2) AS progress_pct,
  total_bytes / 1024 / 1024 / 1024 AS total_gb,
  estimated_remaining_time
FROM oceanbase.DBA_OB_RESTORE_PROGRESS
WHERE status = 'RUNNING';
```

---

## 5. 注意事项

| 注意事项 | 说明 |
|---------|------|
| 恢复时间 | 与表大小和日志回放量成正比 |
| 磁盘空间 | 至少预留恢复表大小的 2 倍 |
| 锁定 | 恢复过程不锁定原表 |
| 一致性 | 恢复到的 SCN 点保证事务一致性 |
| 外键 | 恢复时外键不自动校验 |

---

## 常见错误

| 问题 | 原因 | 解决 |
|------|------|------|
| 备份集不存在 | 备份已被清理 | 检查备份保留策略 |
| 日志缺失 | 归档日志不足 | 恢复到更早的时间点 |
| 空间不足 | 恢复所需空间不够 | 清理磁盘或扩容 |
| 恢复超时 | 表太大 | 调大 `ob_restore_timeout` |
| 表结构不匹配 | 恢复时表已变更 | 先恢复表结构再恢复数据 |

---

## 参考文档

- 表级恢复：https://www.oceanbase.com/docs/oceanbase-database-cn/V420/table-level-restore
- 数据备份恢复概述：https://www.oceanbase.com/docs/oceanbase-database-cn/V420/backup-and-recovery-overview
- obdumper/obloader：https://www.oceanbase.com/docs/obloader-obdumper-cn/V420
