# 安全审计

> 适用版本：OceanBase V4.2+
> 兼容模式：MySQL Mode / Oracle Mode

---

## 概述

OceanBase 审计功能记录数据库操作行为，满足合规要求（等保、GDPR、SOC2）和安全事件追溯。

| 特性 | MySQL Mode | Oracle Mode |
|------|-----------|-------------|
| 审计开启 | `SET GLOBAL audit_system_events` | `AUDIT` / `NOAUDIT` 语句 |
| 审计存储 | 内部审计表 | `DBA_AUDIT_TRAIL` |
| 审计范围 | 全局开关 + 对象级 | 精细到语句类型 |
| 审计分析 | `OB_AUDIT` 视图 | `DBA_COMMON_AUDIT_TRAIL` |
| 性能影响 | 低（异步写入） | 低（异步写入） |

---

## 1. MySQL Mode 审计

### 1.1 开启审计

```sql
-- 开启全量审计（生产慎用）
SET GLOBAL audit_system_events = TRUE;

-- 审计特定操作
-- DML 审计
SET GLOBAL audit_dml_events = TRUE;

-- DDL 审计
SET GLOBAL audit_ddl_events = TRUE;

-- 特殊操作审计
SET GLOBAL audit_special_events = TRUE;
```

### 1.2 查询审计记录

```sql
-- 查看审计事件
SELECT * FROM oceanbase.DBA_OB_AUDIT
ORDER BY event_time DESC
LIMIT 100;

-- 查看特定用户的审计
SELECT * FROM oceanbase.DBA_OB_AUDIT
WHERE user_name = 'app_user'
  AND event_time > DATE_SUB(NOW(), INTERVAL 1 DAY);

-- 查看 DDL 操作
SELECT * FROM oceanbase.DBA_OB_AUDIT
WHERE event_type = 'DDL'
  AND event_time > DATE_SUB(NOW(), INTERVAL 7 DAY);

-- 查看失败的登录尝试
SELECT * FROM oceanbase.DBA_OB_AUDIT
WHERE event_type = 'CONNECT'
  AND return_code != 0
ORDER BY event_time DESC;
```

### 1.3 审计过滤

```sql
-- 仅审计特定数据库
SET GLOBAL audit_system_events = TRUE;
-- 通过白名单过滤
SET GLOBAL audit_system_events = FALSE;

-- 查看审计配置
SHOW VARIABLES LIKE 'audit_%';
```

---

## 2. Oracle Mode 审计

### 2.1 标准审计

```sql
-- 审计特定语句
AUDIT SELECT TABLE BY app_user BY ACCESS;
AUDIT INSERT, UPDATE, DELETE ON hr.employees BY SESSION;
AUDIT EXECUTE PROCEDURE BY app_user;

-- 审计所有 DDL
AUDIT ALL BY sys BY ACCESS;

-- 审计特权使用
AUDIT CREATE TABLE, DROP TABLE, ALTER TABLE BY ACCESS;

-- 取消审计
NOAUDIT SELECT TABLE BY app_user;
NOAUDIT ALL ON hr.employees;
```

### 2.2 统一审计（细粒度）

```sql
-- 创建审计策略
CREATE AUDIT POLICY sensitive_data_access
  PRIVILEGES SELECT
  ACTIONS SELECT ON hr.customers, SELECT ON hr.orders;

-- 启用策略
AUDIT POLICY sensitive_data_access BY app_user;

-- 查看策略
SELECT policy_name, priv_list, audit_option
FROM dba_audit_policies;

-- 删除策略
DROP AUDIT POLICY sensitive_data_access;
```

### 2.3 查询审计记录

```sql
-- 标准审计视图
SELECT username, timestamp#, returncode, obj_name, action_name
FROM dba_audit_trail
WHERE timestamp# > SYSDATE - 1
ORDER BY timestamp# DESC;

-- 统一审计视图
SELECT db_user, event_timestamp, action_code, object_name, sql_text
FROM dba_common_audit_trail
WHERE event_timestamp > SYSDATE - 7;

-- 失败登录审计
SELECT username, timestamp#, returncode, userhost
FROM dba_audit_trail
WHERE returncode IN (1017, 28000)
ORDER BY timestamp# DESC;
```

---

## 3. 审计事件分类

| 事件类别 | MySQL Mode | Oracle Mode | 说明 |
|---------|-----------|------------|------|
| 登录/登出 | CONNECT | LOGON/LOGOFF | 连接审计 |
| DML | audit_dml_events | INSERT/UPDATE/DELETE | 数据操作 |
| DDL | audit_ddl_events | CREATE/ALTER/DROP | 结构变更 |
| 权限变更 | audit_special_events | GRANT/REVOKE | 权限管理 |
| 特殊操作 | audit_special_events | EXPLAIN/LOCK TABLE | 特殊语句 |

---

## 4. 审计报表

### 4.1 每日安全报表

```sql
-- Oracle Mode
-- 当日操作统计
SELECT
  username,
  COUNT(*) AS total_ops,
  SUM(CASE WHEN returncode != 0 THEN 1 ELSE 0 END) AS failed_ops,
  COUNT(DISTINCT action_name) AS action_types
FROM dba_audit_trail
WHERE timestamp# >= TRUNC(SYSDATE)
GROUP BY username
ORDER BY total_ops DESC;

-- 敏感表访问统计
SELECT
  username,
  obj_name,
  COUNT(*) AS access_count
FROM dba_audit_trail
WHERE obj_name IN ('CUSTOMERS', 'ORDERS', 'EMPLOYEES', 'PAYROLL')
  AND timestamp# >= TRUNC(SYSDATE) - 7
GROUP BY username, obj_name
ORDER BY access_count DESC;
```

### 4.2 异常检测

```sql
-- 高频失败登录检测
SELECT username, userhost, COUNT(*) AS fail_count
FROM dba_audit_trail
WHERE returncode != 0
  AND timestamp# >= SYSDATE - 1
GROUP BY username, userhost
HAVING COUNT(*) > 10;

-- 非工作时间操作
SELECT username, action_name, obj_name, timestamp#
FROM dba_audit_trail
WHERE TO_CHAR(timestamp#, 'HH24') NOT BETWEEN '08' AND '20'
  AND timestamp# >= SYSDATE - 1
ORDER BY timestamp# DESC;

-- DDL 变更审计
SELECT username, action_name, obj_name, timestamp#, sql_text
FROM dba_audit_trail
WHERE action_name IN ('CREATE', 'ALTER', 'DROP', 'TRUNCATE')
  AND timestamp# >= SYSDATE - 7
ORDER BY timestamp# DESC;
```

---

## 5. 审计性能优化

| 策略 | 说明 |
|------|------|
| 异步写入 | OceanBase 审计异步写入，不阻塞业务 |
| 按需开启 | 仅审计关键操作，避免全量审计 |
| 定期清理 | 归档旧审计记录，控制表大小 |
| 分区存储 | 审计表按时间分区，便于清理 |

```sql
-- 清理30天前的审计记录（sys 租户）
DELETE FROM oceanbase.__all_audit_log
WHERE event_time < DATE_SUB(NOW(), INTERVAL 30 DAY)
LIMIT 10000;

-- 或使用分区裁剪自动管理
```

---

## 6. 常见错误

| 问题 | 原因 | 解决 |
|------|------|------|
| 审计记录为空 | 未开启审计 | 检查 `audit_system_events` |
| 审计表过大 | 长期未清理 | 定期归档清理 |
| 审计信息不全 | 仅开启了部分审计 | 补充审计配置 |
| `ORA-00942` 审计表不存在 | 审计功能未初始化 | 执行审计初始化 |

---

## 参考文档

- MySQL Mode 审计：https://www.oceanbase.com/docs/oceanbase-database-cn/V420/mysql-mode/audit
- Oracle Mode 审计：https://www.oceanbase.com/docs/oceanbase-database-cn/V420/oracle-mode/auditing
- 统一审计：https://www.oceanbase.com/docs/oceanbase-database-cn/V420/oracle-mode/unified-auditing
