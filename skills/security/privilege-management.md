# 权限管理（MySQL/Oracle双模式）

> 适用版本：OceanBase V4.2+
> 兼容模式：MySQL Mode / Oracle Mode

---

## 概述

OceanBase 支持两种兼容模式，权限体系也因此分为两套：

| 特性 | MySQL Mode | Oracle Mode |
|------|-----------|-------------|
| 系统视图 | `mysql.user` | `DBA_USERS`, `ALL_USERS` |
| 权限授予 | `GRANT ... ON *.*` | `GRANT ... TO user` |
| 角色支持 | 内置角色（V4.2+） | 完整角色支持 |
| 用户创建 | `CREATE USER` | `CREATE USER` + 默认表空间 |
| 密码策略 | `validate_password` 插件 | Profile |
| 审计 | `mysqlaudit` | `AUDIT` 语句 |

---

## 1. 用户管理

### MySQL Mode

```sql
-- 创建用户
CREATE USER 'app_user'@'%' IDENTIFIED BY 'Str0ngP@ss!';

-- 指定密码策略
CREATE USER 'secure_user'@'%' IDENTIFIED BY 'VeryStr0ng#2024'
  PASSWORD EXPIRE INTERVAL 90 DAY;

-- 锁定/解锁
ALTER USER 'app_user'@'%' ACCOUNT LOCK;
ALTER USER 'app_user'@'%' ACCOUNT UNLOCK;

-- 修改密码
ALTER USER 'app_user'@'%' IDENTIFIED BY 'N3wP@ssw0rd';

-- 删除用户
DROP USER 'app_user'@'%';

-- 查看用户
SELECT user, host, account_locked, password_expired
FROM mysql.user
WHERE user NOT IN ('root', 'proxyro');
```

### Oracle Mode

```sql
-- 创建用户
CREATE USER app_user IDENTIFIED BY Str0ngP@ss
  DEFAULT TABLESPACE users
  QUOTA 10G ON users;

-- 使用 Profile 管理密码策略
CREATE PROFILE app_profile LIMIT
  FAILED_LOGIN_ATTEMPTS 5
  PASSWORD_LIFE_TIME 90
  PASSWORD_REUSE_TIME 365
  PASSWORD_REUSE_MAX 10
  PASSWORD_LOCK_TIME 1
  PASSWORD_GRACE_TIME 7;

ALTER USER app_user PROFILE app_profile;

-- 锁定/解锁
ALTER USER app_user ACCOUNT LOCK;
ALTER USER app_user ACCOUNT UNLOCK;

-- 查看用户
SELECT username, account_status, expiry_date, profile
FROM dba_users
WHERE username NOT IN ('SYS', 'SYSTEM');
```

---

## 2. 权限模型

### MySQL Mode 权限层级

```
全局权限 (*.*)
  └── 数据库权限 (db.*)
        └── 表权限 (db.table)
              └── 列权限 (db.table.col)
```

```sql
-- 全局权限
GRANT ALL PRIVILEGES ON *.* TO 'dba_user'@'%' WITH GRANT OPTION;
GRANT SELECT, PROCESS, RELOAD ON *.* TO 'monitor_user'@'%';

-- 数据库权限
GRANT SELECT, INSERT, UPDATE, DELETE ON app_db.* TO 'app_user'@'%';

-- 表权限
GRANT SELECT, UPDATE (status) ON app_db.orders TO 'ops_user'@'%';

-- 回收权限
REVOKE DELETE ON app_db.* FROM 'app_user'@'%';

-- 查看授权
SHOW GRANTS FOR 'app_user'@'%';
SELECT * FROM mysql.user WHERE user = 'app_user'\G
```

### Oracle Mode 权限模型

系统权限（System Privilege）和对象权限（Object Privilege）分离：

```sql
-- 系统权限
GRANT CREATE SESSION TO app_user;
GRANT CREATE TABLE, CREATE VIEW, CREATE PROCEDURE TO app_user;
GRANT SELECT ANY TABLE TO read_only_user;
GRANT CREATE ANY TABLE TO dba_user WITH ADMIN OPTION;

-- 对象权限
GRANT SELECT, INSERT, UPDATE ON schema1.orders TO app_user;
GRANT SELECT ON schema1.orders TO read_only_user WITH GRANT OPTION;

-- 回收（系统权限不会级联回收）
REVOKE CREATE TABLE FROM app_user;

-- 查看授权
SELECT * FROM dba_sys_privs WHERE grantee = 'APP_USER';
SELECT * FROM dba_tab_privs WHERE grantee = 'APP_USER';
```

---

## 3. 角色管理

### MySQL Mode（V4.2+支持角色）

```sql
-- 创建角色
CREATE ROLE 'app_readonly', 'app_readwrite', 'app_admin';

-- 授予角色权限
GRANT SELECT ON app_db.* TO 'app_readonly';
GRANT SELECT, INSERT, UPDATE, DELETE ON app_db.* TO 'app_readwrite';
GRANT ALL PRIVILEGES ON app_db.* TO 'app_admin';

-- 将角色赋予用户
GRANT 'app_readonly' TO 'report_user'@'%';
GRANT 'app_readwrite' TO 'app_user'@'%';

-- 激活角色（默认不自动激活）
SET DEFAULT ROLE ALL TO 'app_user'@'%';

-- 查看角色
SELECT * FROM mysql.role_edges;
SHOW GRANTS FOR 'app_readonly';
```

### Oracle Mode

```sql
-- 创建角色
CREATE ROLE app_readonly;
CREATE ROLE app_readwrite;
CREATE ROLE app_admin;

-- 授予权限给角色
GRANT CREATE SESSION TO app_readonly;
GRANT SELECT ANY TABLE TO app_readonly;

GRANT CREATE SESSION, CREATE TABLE, CREATE PROCEDURE TO app_readwrite;
GRANT SELECT ANY TABLE, INSERT ANY TABLE TO app_readwrite;

-- 将角色赋予用户
GRANT app_readonly TO report_user;
GRANT app_readwrite TO app_user;

-- 默认角色
ALTER USER app_user DEFAULT ROLE app_readwrite;

-- 查看角色
SELECT * FROM dba_role_privs WHERE grantee = 'APP_USER';
SELECT role, granted_role FROM dba_role_privs
WHERE grantee IN (SELECT role FROM dba_roles);
```

---

## 4. 租户级权限（OceanBase特有）

```sql
-- 创建租户
CREATE RESOURCE UNIT unit1 MAX_CPU 4, MIN_CPU 2, MEMORY_SIZE '4G';
CREATE RESOURCE POOL pool1 UNIT = 'unit1', UNIT_NUM = 1, ZONE_LIST = ('zone1');
CREATE TENANT tenant1 REPLICA_NUM = 1, RESOURCE_POOL_LIST = ('pool1')
  SET ob_compatibility_mode = 'mysql';

-- 切换租户（sys租户）
ALTER SYSTEM SWITCH TENANT tenant1;

-- 租户内创建用户
CREATE USER t1_dba IDENTIFIED BY 'Dba@2024';
GRANT ALL PRIVILEGES ON *.* TO 't1_dba'@'%' WITH GRANT OPTION;
```

---

## 5. 密码策略

### MySQL Mode

```sql
-- 安装密码验证插件（如支持）
INSTALL PLUGIN validate_password SONAME 'validate_password.so';

-- 设置密码策略
SET GLOBAL validate_password.policy = 'MEDIUM';
SET GLOBAL validate_password.length = 12;

-- 或通过系统变量
SET GLOBAL validate_password.mixed_case_count = 1;
SET GLOBAL validate_password.number_count = 1;
SET GLOBAL validate_password.special_char_count = 1;
```

### Oracle Mode

```sql
-- 创建密码 Profile
CREATE PROFILE strict_profile LIMIT
  FAILED_LOGIN_ATTEMPTS 5
  PASSWORD_LIFE_TIME 90
  PASSWORD_REUSE_TIME 365
  PASSWORD_REUSE_MAX 12
  PASSWORD_LOCK_TIME UNLIMITED
  PASSWORD_GRACE_TIME 7
  PASSWORD_VERIFY_FUNCTION NULL;

-- 查看密码过期用户
SELECT username, expiry_date, account_status
FROM dba_users
WHERE expiry_date < SYSDATE + 30;

-- 批量续期
ALTER USER app_user PASSWORD EXPIRE NEVER;
```

---

## 6. 常见错误与排查

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| `Access denied for user` | 密码错误或权限不足 | 检查 `mysql.user` 和密码 |
| `ORA-01031: insufficient privileges` | 缺少系统权限 | `GRANT` 对应权限 |
| `ORA-01950: no privileges on tablespace` | 表空间配额不足 | `ALTER USER ... QUOTA ... ON tablespace` |
| 角色未生效 | MySQL Mode默认不激活角色 | `SET DEFAULT ROLE ALL TO user` |
| 无法跨租户访问 | 租户隔离限制 | 使用 Database Link 或 CLUSTER 操作 |

---

## 7. 最佳实践

1. **最小权限原则**：只授予必要的权限，避免 `GRANT ALL`
2. **使用角色管理**：通过角色批量管理权限，而非逐个用户授权
3. **定期审计**：定期检查 `mysql.user` 或 `DBA_USERS` 中的用户状态
4. **密码策略**：强制密码复杂度和过期策略
5. **避免应用使用 root/sys**：为每个应用创建专用账号
6. **监控权限变更**：对 `GRANT/REVOKE` 操作开启审计
7. **租户隔离**：不同业务使用不同租户，实现物理资源隔离

---

## 参考文档

- MySQL Mode 权限管理：https://www.oceanbase.com/docs/oceanbase-database-cn/V420/oracle-mode/manage-privileges-1
- Oracle Mode 权限管理：https://www.oceanbase.com/docs/oceanbase-database-cn/V420/oracle-mode/manage-privileges-1
- 创建用户：https://www.oceanbase.com/docs/oceanbase-database-cn/V420/mysql-mode/create-a-user-1
- Profile 管理：https://www.oceanbase.com/docs/oceanbase-database-cn/V420/oracle-mode/create-a-profile
