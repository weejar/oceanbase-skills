# OBD 监控部署（Prometheus + Grafana）

> 适用版本：OBD V2.x+ / OceanBase V4.2+
> 组件：OBAgent + Prometheus + Grafana

---

## 概述

通过 OBD 可快速为 OceanBase 集群部署完整的监控方案：OBAgent 采集指标 → Prometheus 拉取存储 → Grafana 可视化展示。

---

## 1. 场景一：未部署 OBAgent

将 `obagent`、`prometheus`、`grafana` 一起添加到集群配置文件中：

```yaml
# 在现有集群配置中追加以下内容
obagent:
  servers:
    - <server_ip>
  global:
    home_path: /home/admin/obagent

prometheus:
  servers:
    - <server_ip>
  global:
    home_path: /home/admin/prometheus

grafana:
  servers:
    - <server_ip>
  global:
    home_path: /home/admin/grafana
```

然后重新部署或启动：

```bash
obd cluster component add <deploy_name> -c monitoring-config.yaml
obd cluster start <deploy_name>
```

---

## 2. 场景二：已部署 OBAgent

如果 OBAgent 已经在集群中运行，只需添加 `prometheus` 和 `grafana`：

```yaml
prometheus:
  servers:
    - <server_ip>
  global:
    home_path: /home/admin/prometheus

grafana:
  servers:
    - <server_ip>
  global:
    home_path: /home/admin/grafana
```

添加后，Prometheus 需要手动配置抓取 OBAgent 的指标端点。

---

## 3. 认证信息

OBD 会为 Prometheus 启用 HTTP Basic Auth。生成的凭证在集群状态中查看：

```bash
obd cluster display <deploy_name>
```

输出的信息中包含 Prometheus 的用户名（默认 `admin`）和随机生成的密码。

使用凭证访问：

```bash
# Prometheus Web UI
# http://<host>:9090  (用户名: admin, 密码: 显示的密码)

# Prometheus API
curl -u admin:<password> http://<host>:9090/-/ready

# Grafana Web UI
# http://<host>:3000  (默认 admin/admin)
```

---

## 4. 访问地址

| 组件 | 默认端口 | URL |
|------|---------|-----|
| OBAgent HTTP | 8088 | `http://<host>:8088/metrics` |
| Prometheus | 9090 | `http://<host>:9090` |
| Grafana | 3000 | `http://<host>:3000` |

---

## 5. 组件删除顺序

**有依赖关系的组件必须按正确顺序删除：**

1. 先删除 `grafana` 和 `prometheus`：
   ```bash
   obd cluster component del <deploy_name> grafana prometheus
   ```

2. 再删除 `obagent`：
   ```bash
   obd cluster component del <deploy_name> obagent
   ```

**反向操作会导致 "still depends" 错误。**

---

## 6. 监控指标概览

OBAgent 采集的关键指标分类：

| 指标类别 | 说明 |
|---------|------|
| 系统指标 | CPU、内存、磁盘 IO、网络 |
| OBServer 指标 | 活跃会话、事务数、SQL 执行 |
| 存储指标 | 磁盘使用、数据文件大小、MemStore |
| 副本指标 | Paxos 同步状态、副本数、日志同步延迟 |
| 租户指标 | 各租户 CPU/内存使用率 |

详细的监控与巡检指标请参阅 `monitoring/health-monitor.md` 和 `monitoring/daily-inspection.md`。

---

## 参考文档

- OBD 监控部署：https://www.oceanbase.com/docs/obd-cn/V420/monitor
- OBAgent 文档：https://www.oceanbase.com/docs/obagent-cn
- Grafana Dashboard 导入：https://grafana.com/grafana/dashboards/

---

## 相关文件

- `tools/obd-config-deployment.md` — OBD 配置文件部署参考
- `monitoring/health-monitor.md` — 健康检查与巡检
- `monitoring/daily-inspection.md` — 日常巡检脚本
- `monitoring/space-management.md` — 空间管理与容量规划
