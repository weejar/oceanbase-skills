# SeekDB OBD 操作手册

> 适用版本：SeekDB V1.2.0+ / OBD V2.x+
> 工具：obd seekdb（SeekDB 生命周期与 HA 管理）

---

## 概述

SeekDB 是基于 OceanBase 的 AI 原生混合搜索数据库，支持向量检索、全文检索和混合检索。通过 `obd seekdb` 命令可管理 SeekDB 实例的完整生命周期，包括安装、部署、主备切换、故障恢复等操作。

> SeekDB 产品概述与功能特性请参阅 `admin/seekdb.md`。

---

## 1. 命令参考

| 命令 | 说明 |
|------|------|
| `obd seekdb install` | 交互式安装（单节点，需要 TTY） |
| `obd seekdb install --primary` | 安装为主节点（启用 RPC 用于备节点同步） |
| `obd seekdb install --standby` | 安装为备节点（从运行中的主节点选择） |
| `obd seekdb deploy <name> -c <config>` | 通过配置文件部署 |
| `obd seekdb list` | 仅列出 seekdb 部署 |
| `obd seekdb start <name>` | 启动 |
| `obd seekdb stop <name>` | 停止 |
| `obd seekdb restart <name>` | 重启 |
| `obd seekdb display <name>` | 查看信息 |
| `obd seekdb display <name> -g` | 查看拓扑图 |
| `obd seekdb destroy <name>` | 销毁（见安全规则） |
| `obd seekdb takeover <name> --home-path <path>` | 接管非 OBD 部署的实例 |
| `obd seekdb switchover <name>` | 计划内主备切换 |
| `obd seekdb failover <name>` | 紧急提升备节点为主节点 |
| `obd seekdb decouple <name>` | 解耦备节点为独立主节点 |

---

## 2. 安全规则

### 2.1 销毁带备节点的实例

销毁仍有活跃备节点的主节点需要 `--ignore-standby`：

```bash
obd seekdb destroy <name> --ignore-standby
```

不加此标志，OBD 将拒绝操作并发出孤立备节点警告。

### 2.2 Switchover vs Failover

| 操作 | 适用场景 | 主节点状态 |
|------|---------|-----------|
| `switchover` | 计划内维护 | **在线运行** |
| `failover` | 主节点宕机/不可达 | **已停止或不可达** |

**`failover` 会在主节点仍运行时报错。** 计划内切换必须使用 `switchover`。

### 2.3 同主机限制

主节点和备节点**必须在不同的 IP** 上。如果部署在同一 IP：
1. OBD 的同主机冲突避免逻辑会将 `mock_primary_rpc_info` 设为 None
2. 导致 `--role=STANDBY` 启动参数未传递
3. 备节点以主节点模式启动（`ACCESS_MODE=APPEND`），日志同步静默失败

---

## 3. 安装模式

### 3.1 交互式安装（默认）

```bash
obd seekdb install
```

- 安装单节点 SeekDB，引导式交互配置
- **需要真实的 TTY 终端**，不支持管道输入自动化
- 脚本/CI 部署请使用 `obd seekdb deploy <name> -c config.yaml`

### 3.2 主节点模式

```bash
obd seekdb install --primary
```

- 安装为主节点集群，启用 RPC 用于备节点同步
- **需要 TTY**
- 必须在备节点之前安装

### 3.3 备节点模式

```bash
obd seekdb install --standby
```

- 安装为备节点，交互式选择已部署且运行中的主节点
- **需要 TTY**
- 备节点可用磁盘空间不得低于主节点的 `log_disk_size`
- `--standby` 和 `--primary` 不可同时使用
- 备节点模式会验证主节点是否启用了 `enable_rpc_service`，未启用则需先重启主节点

### 3.4 非交互式部署的限制

```bash
obd seekdb deploy <name> -c config.yaml
```

即使在 YAML 配置中设置了 `log_restore_source`，OBD 在非交互模式下**不会**传递 `--role=STANDBY` 给 seekdb 进程。备节点以主节点模式启动，日志同步不工作。

**建立真正的主备同步，必须使用 `obd seekdb install --standby`（需要 TTY），且主备节点在不同 IP。**

---

## 4. HA 操作

### 4.1 查看拓扑

```bash
obd seekdb display <name> -g
```

显示集群拓扑图（主节点、备节点、级联关系）。

### 4.2 计划内切换（Switchover）

```bash
obd seekdb switchover <name>
```

- **用途**：计划内维护窗口期间主备互换
- **要求**：主节点必须**在线运行**
- 指定备节点提升为主节点，原主节点降为备节点
- 适用于滚动维护、计划迁移、负载重平衡

### 4.3 紧急故障恢复（Failover）

```bash
obd seekdb failover <name>
```

- **用途**：主节点宕机或不可达时的紧急提升
- **要求**：主节点必须**已停止或不可达**
- 指定备节点提升为主节点
- 故障恢复后，原主节点需手动干预才能重新加入为备节点

### 4.4 操作选择指南

| 场景 | 主节点状态 | 使用命令 |
|------|-----------|---------|
| 计划维护窗口 | 运行中 | `switchover` |
| 主节点崩溃/不可达 | 已停止/不可达 | `failover` |
| 主节点运行但缓慢 | 运行中 | `switchover`（不是 failover） |
| 测试 HA | 先停主节点 | `failover` |

### 4.5 解耦备节点

```bash
obd seekdb decouple <name>
```

- 将指定备节点从主备关系中移除
- 被解耦的备节点成为**独立主节点**，继续运行
- 适用于需要将备节点永久分离为独立数据库的场景

---

## 5. 接管现有实例

```bash
obd seekdb takeover <name> --home-path <path>
```

接管非 OBD 部署的 SeekDB 实例：
- `--home-path` 为必填参数
- 还需要主机、`mysql_port`、`root` 密码和 SSH 信息（通过配置文件或交互输入）
- OBD 自动生成配置并将实例纳入管理

---

## 6. 部署与启动示例

```bash
# 部署 SeekDB
obd seekdb deploy my-seekdb -c seekdb-config.yaml

# 启动
obd seekdb start my-seekdb

# 查看拓扑
obd seekdb display my-seekdb -g

# 接管已有实例
obd seekdb takeover my-seekdb --home-path /home/admin/seekdb
```

---

## 参考文档

- SeekDB 官方文档：https://www.oceanbase.com/docs/seekdb
- OBD 文档：https://www.oceanbase.com/docs/obd-cn/V420/overview-of-obd

---

## 相关文件

- `admin/seekdb.md` — SeekDB 产品概述与功能特性
- `tools/obd-deployer.md` — OBD 部署器命令参考
- `devops/obd-deployment.md` — OBD 部署与集群管理
