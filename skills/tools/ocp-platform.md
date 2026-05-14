# OCP 云平台管理

> 适用版本：OCP V3.x / OceanBase V4.2+
> 官网：https://www.oceanbase.com/product/ocp

---

## 概述

OCP（OceanBase Cloud Platform）是 OceanBase 的统一管理平台，提供集群管理、监控告警、性能诊断、租户管理等 Web 界面。

| 功能模块 | 说明 |
|---------|------|
| 集群管理 | 部署、启停、扩缩容、升级 |
| 租户管理 | 创建、扩缩容、资源调配 |
| 监控告警 | 实时监控、告警规则、通知渠道 |
| 性能诊断 | SQL 诊断、锁分析、等待事件 |
| 备份恢复 | 备份策略、恢复操作 |
| 安全管理 | 用户权限、白名单、审计 |

---

## 1. 安装部署

### 1.1 系统要求

| 项目 | 最低要求 |
|------|---------|
| CPU | 8C |
| 内存 | 16G |
| 磁盘 | 100G |
| OS | CentOS 7+ / Ubuntu 18+ |
| 网络 | 可访问所有 OBServer 节点 |

### 1.2 安装步骤

```bash
# 1. 下载 OCP 安装包
wget https://mirrors.aliyun.com/oceanbase/OCP/current/ocp.tar.gz

# 2. 解压
tar -xzf ocp.tar.gz -C /opt/ocp

# 3. 安装依赖
cd /opt/ocp
bash install.sh --deps-only

# 4. 初始化配置
bash install.sh --init

# 5. 修改配置
vi conf/ocp.yaml
# 主要配置项：
#   ocp.server.port: 8080
#   ocp.metadb.url: jdbc:mysql://localhost:2881/ocp_meta
#   ocp.ob-agent.port: 8088

# 6. 启动
bash start.sh

# 7. 访问 Web 界面
# http://<ocp_host>:8080
# 默认管理员: admin / admin
```

### 1.3 MetaDB 配置

OCP 需要一个 OceanBase 实例作为元数据库：

```sql
-- 创建 OCP 元数据租户
CREATE RESOURCE UNIT unit_ocp MAX_CPU 4, MIN_CPU 2, MEMORY_SIZE '4G';
CREATE RESOURCE POOL pool_ocp UNIT = 'unit_ocp', UNIT_NUM = 1;
CREATE TENANT ocp_meta REPLICA_NUM = 1, RESOURCE_POOL_LIST = ('pool_ocp')
  SET ob_compatibility_mode = 'mysql';
```

---

## 2. 集群管理

### 2.1 注册集群

```
Web 界面路径：集群管理 → 新增集群

填写信息：
  - 集群名称: prod_cluster
  - OBServer 列表: 10.0.0.1:2881, 10.0.0.2:2881, 10.0.0.3:2881
  - sys 租户密码: ******
  - SSH 信息: oceanbase / ******
```

### 2.2 集群监控

```
监控面板 → 选择集群

关键指标：
  - QPS / TPS
  - 活跃连接数
  - CPU / 内存使用率
  - 磁盘 I/O
  - 事务延迟 (RT)
  - SQL 响应时间分布
```

### 2.3 扩缩容

```
Web 界面路径：集群管理 → 集群详情 → 扩容

添加节点：
  1. 输入新节点 IP 和 SSH 信息
  2. 选择 Zone
  3. 点击执行扩容
```

---

## 3. 租户管理

### 3.1 创建租户

```
Web 界面路径：租户管理 → 新建租户

配置：
  - 租户名称: business_tenant
  - 兼容模式: MySQL / Oracle
  - 副本数: 3
  - Unit 规格: MAX_CPU=4, MIN_CPU=2, MEMORY_SIZE=8G
  - Zone 列表: zone1, zone2, zone3
  - root 密码: ******
```

### 3.2 租户扩缩容

```
Web 界面路径：租户管理 → 租户详情 → 资源调整

修改 Unit 规格（在线生效）：
  - 调整 CPU: 4C → 8C
  - 调整内存: 8G → 16G
  - 添加/删除 Unit
```

---

## 4. 监控告警

### 4.1 告警规则配置

```
Web 界面路径：监控告警 → 告警规则 → 新增规则

常见告警规则：
  - CPU 使用率 > 80%（持续 5 分钟）
  - 内存使用率 > 85%
  - 磁盘使用率 > 90%
  - 活跃连接数 > 1000
  - SQL 平均响应时间 > 500ms
  - 副本状态异常
  - 备份任务失败
```

### 4.2 通知渠道

```
支持的通知方式：
  - 邮件 (SMTP)
  - Webhook (钉钉/飞书/企业微信)
  - 短信
  - 电话
```

---

## 5. 性能诊断

### 5.1 SQL 诊断

```
Web 界面路径：性能诊断 → SQL 诊断

功能：
  - Top SQL 排行
  - 慢 SQL 列表
  - SQL 执行计划分析
  - SQL 审计日志
```

### 5.2 等待事件分析

```
Web 界面路径：性能诊断 → 等待事件

查看当前和历史的等待事件分布：
  - 等待事件分类统计
  - 时间趋势图
  - 关联的 SQL 语句
```

---

## 6. 备份恢复管理

```
Web 界面路径：备份恢复 → 备份策略

配置自动备份：
  - 备份类型: 全量 + 增量
  - 全量频率: 每周日 02:00
  - 增量频率: 每 30 分钟
  - 保留天数: 30 天
  - 备份路径: OSS / NFS

恢复操作：
  - 选择备份集
  - 选择恢复时间点
  - 选择恢复目标
  - 执行恢复
```

---

## 7. API 接口

```bash
# OCP 提供 RESTful API

# 获取集群列表
curl -X GET "http://ocp_host:8080/api/v2/ob/clusters" \
  -H "Authorization: Bearer <token>"

# 获取租户信息
curl -X GET "http://ocp_host:8080/api/v2/ob/tenants?clusterId=1" \
  -H "Authorization: Bearer <token>"

# 查看告警
curl -X GET "http://ocp_host:8080/api/v2/alerts?status=ACTIVE" \
  -H "Authorization: Bearer <token>"
```

---

## 常见问题

| 问题 | 解决 |
|------|------|
| 无法注册集群 | 检查 SSH 免密和端口连通性 |
| 监控数据缺失 | 检查 ob-agent 状态 |
| 告警未触发 | 检查告警规则阈值和通知渠道 |
| OCP 启动失败 | 检查 MetaDB 连接和日志 |

---

## 参考文档

- OCP 官方文档：https://www.oceanbase.com/docs/ocp-cn
- OCP 安装部署：https://www.oceanbase.com/docs/ocp-cn/V320/install-ocp
- OCP API 文档：https://www.oceanbase.com/docs/ocp-cn/V320/api-reference
