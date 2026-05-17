# OBD 集群部署配置参考

> 适用版本：OBD V2.x+ / OceanBase V4.2+
> 工具：obd（OceanBase Deployer）

---

## 概述

本文档是 OBD 配置文件部署方式的详细参考，涵盖组件选择、配置文件结构、部署后操作（编辑配置、重载、升级、扩缩容、组件增删）。OBD 基础命令请参阅 `tools/obd-deployer.md`，环境准备请参阅 `devops/obd-deployment.md`。

---

## 1. 部署方式

### 1.1 配置文件部署

```bash
obd cluster deploy <deploy_name> -c <config_file>
obd cluster start <deploy_name>
```

### 1.2 交互式部署

```bash
obd cluster deploy <deploy_name> -i
```

引导式配置，适合不熟悉配置文件结构的用户。

### 1.3 快速 Demo

```bash
obd demo
```

部署本地单节点 Demo 集群（默认端口：2881 MySQL、2882 RPC、2886 obshell）。

---

## 2. 组件选择

OBD 配置文件中可定义的组件：

| 组件 | 用途 | 说明 |
|------|------|------|
| `oceanbase-ce` | OceanBase 数据库服务 | 核心组件 |
| `obproxy-ce` | OceanBase 代理 | 负载均衡、SQL 路由 |
| `obagent` | 监控代理 | 采集指标供 Prometheus 抓取 |
| `ocp-ce` | OCP 社区版 | 完整管理平台 |
| `prometheus` | 指标采集 | 从 OBAgent 拉取指标 |
| `grafana` | 指标可视化 | Dashboard 展示 |

**OCP 术语约定**：
- "部署 OCP" 始终指 **OCP CE**（`ocp-ce`），不是 `ocp-express`
- `ocp-express` 已被 `obshell dashboard`（端口 2886）取代，不再部署

---

## 3. 配置文件结构

### 3.1 单节点配置

```yaml
user: oceanbase
oceanbase-ce:
  servers:
    - name: server1
      ip: 192.168.1.10
  global:
    devname: eth0
    mysql_port: 2881
    rpc_port: 2882
    obshell_port: 2886
    home_path: /home/oceanbase/observer
    data_dir: /home/oceanbase/data
    log_dir: /home/oceanbase/log
    zone: zone1
    cluster_id: 1
    memory_limit: 4G
    system_memory: 1G
    datafile_size: 20G
    logfile_size: 2G
    cpu_count: 8
```

### 3.2 三节点生产配置

```yaml
user: oceanbase
oceanbase-ce:
  servers:
    - name: server1
      ip: 10.0.0.1
      zone: zone1
    - name: server2
      ip: 10.0.0.2
      zone: zone2
    - name: server3
      ip: 10.0.0.3
      zone: zone3
  global:
    devname: eth0
    mysql_port: 2881
    rpc_port: 2882
    obshell_port: 2886
    home_path: /home/oceanbase/observer
    data_dir: /home/oceanbase/data
    log_dir: /home/oceanbase/log
    cluster_id: 1
    memory_limit: 32G
    system_memory: 8G
    datafile_size: 200G
    logfile_size: 4G
    cpu_count: 32
    enable_syslog_wf: true
    enable_syslog_recycle: true
    max_syslog_file_count: 30
```

### 3.3 OBProxy 配置

```yaml
user: oceanbase
obproxy:
  servers:
    - name: proxy1
      ip: 10.0.0.10
    - name: proxy2
      ip: 10.0.0.11
  global:
    home_path: /home/oceanbase/obproxy
    listen_port: 2883
    prometheus_listen_port: 2884
    rs_list: "10.0.0.1:2881;10.0.0.2:2881;10.0.0.3:2881"
    cluster_name: prod_cluster
```

---

## 4. 部署后操作

### 4.1 编辑配置

```bash
obd cluster edit-config <deploy_name>
```

在编辑器中修改集群配置。修改后需执行 `reload` 或 `restart` 生效。

### 4.2 重载配置

```bash
obd cluster reload <deploy_name>
```

将配置变更应用到运行中的集群，无需完整重启。

### 4.3 升级组件

```bash
obd cluster upgrade <deploy_name> -c <component_name> -V <version>
```

升级指定组件到指定版本。详见 `devops/version-upgrade.md`。

### 4.4 扩容

```bash
obd cluster scale_out <deploy_name> -c <scale_out_config>
```

配置文件描述要添加的新节点。详见 `admin/cluster-management.md`。

### 4.5 增删组件

```bash
# 添加组件
obd cluster component add <deploy_name> -c <config_file>

# 删除组件（注意依赖顺序）
obd cluster component del <deploy_name> <component_name>
```

**删除顺序很重要**：有依赖关系的组件必须先删除依赖方。

正确顺序示例：
1. 先删除 `grafana` 和 `prometheus`
2. 再删除 `obagent`

反向操作会导致 "still depends" 错误。

---

## 5. 端口隔离

同一主机部署多个 OB 栈时，必须确保端口不冲突：

| 服务 | 默认端口 | 配置参数 |
|------|---------|---------|
| MySQL 协议 | 2881 | `mysql_port` |
| RPC 通信 | 2882 | `rpc_port` |
| obshell Dashboard | 2886 | `obshell_port` |
| OBProxy | 2883 | `listen_port` |
| OBProxy Prometheus | 2884 | `prometheus_listen_port` |

---

## 6. OS 环境要求

- **GLIBC**：OceanBase CE 4.3+ el8 包需要 GLIBC 2.27+（RHEL/CentOS 8+）
- CentOS 7 / AliOS 7 使用 GLIBC 2.17，需选用 **el7 包（OceanBase CE ≤4.3.x）**
- OBD 只需在控制机上运行，通过 SSH 连接远程节点
- `ocp-ce` 包默认不在公共镜像源中，需通过 `obd mirror list` 验证可用性

---

## 7. 安全提示

以下命令具有破坏性，执行前必须确认：

| 命令 | 影响 |
|------|------|
| `obd cluster destroy` | 销毁集群并删除数据 |
| `obd cluster redeploy` | 销毁并重新部署，数据丢失 |
| `obd cluster prune-config` | 删除已销毁集群的配置文件 |

**始终在执行前向用户确认。**

---

## 参考文档

- OBD 配置参考：https://www.oceanbase.com/docs/obd-cn/V420/configuration-reference
- OBD 部署集群：https://www.oceanbase.com/docs/obd-cn/V420/deploy-a-cluster
- OBD 扩缩容：https://www.oceanbase.com/docs/obd-cn/V420/resize-a-cluster

---

## 相关文件

- `devops/obd-deployment.md` — OBD 部署与集群管理（环境准备、初始化）
- `tools/obd-deployer.md` — OBD 完整命令参考
- `admin/cluster-management.md` — 集群管理（Zone/Server/Unit）
- `tools/ocp-platform.md` — OCP 云平台管理
