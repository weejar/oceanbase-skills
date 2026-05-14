# DBLink 跨库/跨租户查询

> 适用版本：OceanBase V4.2+
> 兼容模式：Oracle Mode（完整）/ MySQL Mode（V4.3+部分支持）

---

## 概述

Database Link（DBLink）允许在一个租户中直接查询另一个租户或外部数据库的数据。

| 特性 | Oracle Mode | MySQL Mode |
|------|------------|------------|
| 创建语法 | `CREATE DATABASE LINK` | `CREATE DATABASE LINK`（V4.3+） |
| 使用方式 | `SELECT * FROM table@link` | `SELECT * FROM table@link` |
| 跨租户 | 支持 | 支持（V4.3+） |
| 跨 OceanBase | 支持 | 支持 |
| 跨数据库类型 | 支持（Oracle Mode 连接 MySQL） | 有限支持 |
| 事务 | 分布式事务 | 分布式事务 |

---

## 1. Oracle Mode

### 1.1 创建 DBLink 到其他租户

```sql
-- 连接到租户A，创建到租户B的 DBLink
CREATE DATABASE LINK link_to_tenant_b
  CONNECT TO biz_user IDENTIFIED BY "BizPass@2024"
  USING '(
    DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCP)(HOST = 10.0.0.1)(PORT = 2881))
      (CONNECT_DATA = (SERVICE_NAME = tenant_b))
  )';

-- 使用 DBLink 查询
SELECT * FROM orders@link_to_tenant_b WHERE amount > 1000;

-- 跨租户 JOIN
SELECT a.user_name, b.order_id, b.amount
FROM users a
JOIN orders@link_to_tenant_b b ON a.user_id = b.user_id;
```

### 1.2 创建 DBLink 到 MySQL 实例

```sql
CREATE DATABASE LINK link_to_mysql
  CONNECT TO app_user IDENTIFIED BY "MySql@2024"
  USING '(
    DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCP)(HOST = 10.0.0.100)(PORT = 3306))
      (CONNECT_DATA = (SERVICE_NAME = app_db))
  )';

SELECT * FROM products@link_to_mysql;
```

### 1.3 创建 DBLink 到另一个 OceanBase 集群

```sql
-- 连接远程 OceanBase 的 MySQL Mode 租户
CREATE DATABASE LINK link_to_remote_ob
  CONNECT TO root IDENTIFIED BY "Remote@2024"
  USING '(
    DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCP)(HOST = remote-ob.example.com)(PORT = 2881))
      (CONNECT_DATA = (SERVICE_NAME = remote_tenant))
  )';
```

### 1.4 管理 DBLink

```sql
-- 查看已有 DBLink
SELECT db_link, owner, username, host, created
FROM dba_db_links;

SELECT * FROM user_db_links;

-- 删除 DBLink
DROP DATABASE LINK link_to_tenant_b;

-- 公共 DBLink（sys租户创建，所有租户可用）
CREATE PUBLIC DATABASE LINK public_link ...
```

---

## 2. MySQL Mode（V4.3+）

### 2.1 创建 DBLink

```sql
-- 创建到其他租户的 DBLink
CREATE DATABASE LINK link_to_tenant_b
  WITH CONNECT USER 'biz_user' IDENTIFIED BY 'BizPass2024'
  USING '10.0.0.1:2881/tenant_b';

-- 使用
SELECT * FROM orders@link_to_tenant_b WHERE status = 1;
```

### 2.2 管理

```sql
-- 查看
SHOW DATABASE LINKS;
SELECT * FROM information_schema.database_links;

-- 删除
DROP DATABASE LINK link_to_tenant_b;
```

---

## 3. 性能优化

### 3.1 推送下推

```sql
-- ✅ 好：WHERE 条件会被推送到远端执行
SELECT * FROM orders@link WHERE order_date = '2024-03-15';

-- ❌ 差：函数阻止下推
SELECT * FROM orders@link WHERE DATE(created_at) = '2024-03-15';
-- 改为：
SELECT * FROM orders@link
WHERE created_at >= '2024-03-15 00:00:00'
  AND created_at < '2024-03-16 00:00:00';
```

### 3.2 减少数据传输

```sql
-- ❌ 全表传输
SELECT * FROM large_table@link;

-- ✅ 只查需要的列
SELECT id, name, status FROM large_table@link WHERE status = 1;

-- ✅ 使用 LIMIT
SELECT * FROM large_table@link WHERE status = 1 LIMIT 1000;
```

### 3.3 DRCP 连接池

```sql
-- Oracle Mode：配置远端使用 DRCP 减少连接开销
-- 在远端租户配置
ALTER SYSTEM SET __min_sstable_cnt = 1;
```

---

## 4. 分布式事务

```sql
-- 跨 DBLink 操作参与分布式事务
-- Oracle Mode
BEGIN
  -- 本地操作
  INSERT INTO local_audit_log (action, detail)
  VALUES ('SYNC', 'Start sync');

  -- 远端操作
  INSERT INTO remote_data@link (id, value)
  VALUES (1, 'test');

  COMMIT;  -- 两阶段提交
END;
/

-- 查看分布式事务状态
SELECT * FROM dba_2pc_pending;
-- 强制提交/回滚
-- EXEC DBMS_TRANSACTION.PURGE_LOST_DB_ENTRY('tx_id');
```

---

## 5. 使用场景

| 场景 | 说明 | 注意事项 |
|------|------|---------|
| 跨租户报表 | 聚合多租户数据 | 使用 WHERE 过滤减少传输 |
| 数据迁移 | 从源库拉取数据 | 批量拉取，避免大事务 |
| 混合部署 | OB + 外部 MySQL | 兼容性需验证 |
| 微服务共享数据 | 服务间通过 DBLink 共享 | 高频访问建议数据冗余 |

---

## 6. 安全配置

```sql
-- 1. 限制 DBLink 访问（仅特定用户可创建）
REVOKE CREATE DATABASE LINK FROM PUBLIC;
GRANT CREATE DATABASE LINK TO dba_user;

-- 2. 使用最小权限账号连接
-- 远端只授予 SELECT 权限
GRANT SELECT ON schema.orders TO dblink_user;

-- 3. 密码保护
-- 定期更换远端密码，及时更新 DBLink
```

---

## 常见错误

| 问题 | 原因 | 解决 |
|------|------|------|
| `ORA-02019: connection description not found` | DBLink 不存在 | 检查 DBLink 拼写 |
| `ORA-12154: TNS:could not resolve` | 网络不通 | 检查 IP/端口/防火墙 |
| `ORA-01017: invalid credentials` | 密码错误 | 检查远端用户密码 |
| 查询极慢 | 未下推/全表扫描 | 添加 WHERE 条件 |
| `ORA-02020: too many database links` | DBLink 嵌套过深 | 避免多层嵌套 |

---

## 参考文档

- Oracle Mode DBLink：https://www.oceanbase.com/docs/oceanbase-database-cn/V420/oracle-mode/database-link
- MySQL Mode DBLink：https://www.oceanbase.com/docs/oceanbase-database-cn/V420/mysql-mode/database-link
- 分布式事务：https://www.oceanbase.com/docs/oceanbase-database-cn/V420/distributed-transaction
