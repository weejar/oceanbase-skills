# 网络安全与白名单

> 适用版本：OceanBase V4.2+
> 兼容模式：MySQL Mode / Oracle Mode

---

## 概述

OceanBase 网络安全包含多层防护：白名单/IP过滤、连接加密、端口隔离、Proxy 代理等。

| 防护层 | 方式 | 说明 |
|--------|------|------|
| IP 白名单 | `ALTER TENANT` IP 白名单 | 限制来源 IP |
| OBProxy 层 | Proxy 网络隔离 | 隐藏 OBServer 直连 |
| SSL/TLS | 传输加密 | 防止中间人攻击 |
| 端口隔离 | SQL/RPC 分离端口 | 不同服务不同端口 |
| 租户隔离 | 连接属性路由 | 通过租户名隔离连接 |

---

## 1. IP 白名单

### 1.1 配置租户白名单

```sql
-- sys 租户操作

-- 设置白名单（仅允许指定IP段访问）
ALTER TENANT tenant1 SET VARIABLES = 'ob_tcp_invited_nodes = "10.0.0.0/8,172.16.0.0/12,192.168.1.0/24"';

-- 查看当前白名单配置
SHOW VARIABLES LIKE 'ob_tcp_invited_nodes';

-- 恢复允许所有IP（默认）
ALTER TENANT tenant1 SET VARIABLES = 'ob_tcp_invited_nodes = "%"';
```

### 1.2 多租户白名单策略

```sql
-- 生产租户：仅允许应用服务器
ALTER TENANT prod_tenant SET VARIABLES =
  'ob_tcp_invited_nodes = "10.10.1.0/24,10.10.2.0/24"';

-- 测试租户：允许办公网段
ALTER TENANT test_tenant SET VARIABLES =
  'ob_tcp_invited_nodes = "10.0.0.0/8,172.16.0.0/12"';

-- 管理/DBA租户：仅 DBA 办公室
ALTER TENANT dba_tenant SET VARIABLES =
  'ob_tcp_invited_nodes = "10.10.100.10,10.10.100.11"';
```

---

## 2. 端口规划

### 2.1 默认端口

| 端口 | 服务 | 说明 |
|------|------|------|
| 2881 | OBServer SQL | 客户端连接端口 |
| 2882 | OBServer RPC | 节点间通信端口 |
| 2883 | OBProxy | 代理端口（默认） |

### 2.2 端口安全策略

```bash
# 防火墙配置示例（Linux iptables）
# 仅允许应用服务器访问 SQL 端口
iptables -A INPUT -p tcp -s 10.10.1.0/24 --dport 2881 -j ACCEPT
iptables -A INPUT -p tcp -s 10.10.2.0/24 --dport 2881 -j ACCEPT

# RPC 端口仅集群内部访问
iptables -A INPUT -p tcp -s 10.0.0.0/8 --dport 2882 -j ACCEPT
iptables -A INPUT -p tcp --dport 2882 -j DROP

# OBProxy 端口仅允许应用网段
iptables -A INPUT -p tcp -s 10.10.0.0/16 --dport 2883 -j ACCEPT
iptables -A INPUT -p tcp --dport 2883 -j DROP

# 默认拒绝
iptables -P INPUT DROP
```

### 2.3 OBServer 监听地址

```bash
# obd 部署时指定监听地址
obd cluster edit-config my_cluster

# 在配置中设置
oceanbase:
  mysql_port: 2881
  rpc_port: 2882
  devname: eth0  # 指定网卡，避免监听 0.0.0.0
```

---

## 3. OBProxy 网络隔离

### 3.1 架构

```
应用服务器 → OBProxy (2883) → OBServer (2881)
                              ↑
                    应用不直连 OBServer
```

### 3.2 配置 OBProxy

```bash
# 启动 OBProxy，仅监听内网
obproxy \
  --listen_port 2883 \
  --prometheus_listen_port 2884 \
  --rs_list="obhost1:2881;obhost2:2881;obhost3:2881" \
  --skip_proxy_sys_private_check=true

# OBServer 配置：禁止应用直连
# 通过 IP 白名单限制 OBServer 仅允许 OBProxy 所在机器访问
ALTER SYSTEM SET ob_tcp_invited_nodes = "proxy_host_ip";
```

### 3.3 连接路由

```bash
# 通过租户名路由
mysql -h proxy_host -P 2883 -u root@tenant1 -p

# 通过集群名路由
mysql -h proxy_host -P 2883 -u root@tenant1#cluster_name -p
```

---

## 4. 连接认证安全

### 4.1 密码认证

```sql
-- MySQL Mode：强密码策略
ALTER USER 'app_user'@'%' IDENTIFIED BY 'C0mpl3x!P@ss2024';
ALTER USER 'app_user'@'%' PASSWORD EXPIRE INTERVAL 90 DAY;

-- Oracle Mode
ALTER USER app_user IDENTIFIED BY C0mpl3xP@ss2024;
ALTER USER app_user PROFILE strict_profile;
```

### 4.2 连接限制

```sql
-- 限制连接数（MySQL Mode）
GRANT USAGE ON *.* TO 'app_user'@'%' WITH MAX_USER_CONNECTIONS 50;
ALTER USER 'app_user'@'%' WITH MAX_USER_CONNECTIONS 50;

-- Oracle Mode（通过 Profile）
CREATE PROFILE conn_profile LIMIT
  SESSIONS_PER_USER 50
  IDLE_TIME 60
  CONNECT_TIME 480;
ALTER USER app_user PROFILE conn_profile;
```

### 4.3 空闲连接超时

```sql
-- MySQL Mode
SET GLOBAL wait_timeout = 1800;  -- 30分钟
SET GLOBAL interactive_timeout = 1800;

-- Oracle Mode
ALTER PROFILE conn_profile MODIFY IDLE_TIME 30; -- 30分钟
```

---

## 5. 网络安全检查清单

```sql
-- 1. 检查白名单配置
SHOW VARIABLES LIKE 'ob_tcp_invited_nodes';

-- 2. 检查空闲连接
-- MySQL Mode
SELECT user, host, command, time
FROM information_schema.processlist
WHERE command = 'Sleep' AND time > 3600;

-- 3. 检查异常连接来源
SELECT host, user, COUNT(*) AS conn_count
FROM information_schema.processlist
GROUP BY host, user
ORDER BY conn_count DESC;

-- 4. 检查 SSL 使用状态
SHOW VARIABLES LIKE '%ssl%';
-- Oracle Mode
SELECT name, value FROM v$parameter WHERE name LIKE '%ssl%';
```

```bash
# 5. 检查防火墙规则
iptables -L -n | grep -E "2881|2882|2883"

# 6. 检查端口监听
ss -tlnp | grep -E "2881|2882|2883"
```

---

## 6. 安全加固配置模板

```sql
-- ===================== 网络安全加固 =====================

-- 1. 白名单：仅允许应用网段
ALTER TENANT prod SET VARIABLES =
  'ob_tcp_invited_nodes = "10.10.1.0/24"';

-- 2. 密码策略
ALTER USER 'app_user'@'%' IDENTIFIED BY 'Str0ng!P@ss2024'
  PASSWORD EXPIRE INTERVAL 90 DAY;

-- 3. 连接限制
ALTER USER 'app_user'@'%' WITH MAX_USER_CONNECTIONS 50;

-- 4. 空闲超时
SET GLOBAL wait_timeout = 1800;

-- 5. 传输加密
ALTER SYSTEM SET enable_ssl = TRUE;

-- 6. 禁止危险操作
REVOKE SUPER, FILE, SHUTDOWN ON *.* FROM 'app_user'@'%';
```

---

## 常见错误

| 问题 | 原因 | 解决 |
|------|------|------|
| `Access denied` from wrong IP | 白名单限制 | 添加 IP 到白名单 |
| 连接超时 | 防火墙阻断 | 检查 iptables/安全组 |
| OBProxy 连接失败 | OBServer 白名单未含 Proxy IP | 添加 Proxy IP |
| SSL 连接失败 | 证书不匹配 | 重新生成证书链 |
| 连接数耗尽 | 无限制导致 | 设置 `MAX_USER_CONNECTIONS` |

---

## 参考文档

- 白名单配置：https://www.oceanbase.com/docs/oceanbase-database-cn/V420/configure-ip-whitelist
- OBProxy 配置：https://www.oceanbase.com/docs/oceanbase-database-cn/V420/configure-obproxy
- SSL/TLS：https://www.oceanbase.com/docs/oceanbase-database-cn/V420/configure-ssl
- 安全基线：https://www.oceanbase.com/docs/oceanbase-database-cn/V420/security-best-practices
