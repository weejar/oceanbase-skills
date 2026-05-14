# OBD 部署器

> 适用版本：OBD V2.x / OceanBase V4.2+

---

## 概述

OBD（OceanBase Deployer）是 OceanBase 官方部署运维命令行工具，已独立于 OceanBase 主包发布。详见 `devops/obd-deployment.md`。本文件补充高级用法和命令参考。

---

## 1. 完整命令参考

### 1.1 集群管理

```bash
# 部署集群
obd cluster deploy <name> -c <config.yaml> [-t <tag>]
  --skip-verify    # 跳过配置检查
  --force          # 强制部署（覆盖已有）

# 启动集群
obd cluster start <name> [--servers server1] [--timeout 600]
  --without-parameter  # 不修改系统参数

# 停止集群
obd cluster stop <name> [--servers server1] [--timeout 600]

# 重启集群
obd cluster restart <name> [--servers server1]

# 销毁集群（仅删除 OBD 记录，不删除数据）
obd cluster destroy <name>
  --force-kill     # 强制杀死进程

# 显示集群
obd cluster display <name>
  --component      # 显示组件版本信息

# 编辑配置
obd cluster edit-config <name>

# 查看配置
obd cluster show-config <name>

# 列出集群
obd cluster list
```

### 1.2 扩缩容

```bash
# 扩容
obd cluster scale-out <name> -c <scale_out.yaml>

# 缩容
obd cluster scale-in <name>
```

### 1.3 升级

```bash
# 预检查
obd cluster upgrade <name> -c <upgrade.yaml> --check-only

# 执行升级
obd cluster upgrade <name> -c <upgrade.yaml>

# 滚动升级
obd cluster upgrade <name> -c <upgrade.yaml> --roll-upgrade
```

### 1.4 备份管理

```bash
# 备份配置
obd cluster backup-config <name> --output-dir ./backup/

# 恢复配置
obd cluster recover <name> -c ./backup/config.yaml
```

---

## 2. 仓库管理

```bash
# 添加镜像源
obd mirror clone https://mirrors.aliyun.com/oceanbase/OCEANBASE_CE/stable/el/7/*.rpm

# 查看已安装的包
obd mirror list

# 列出远程可用的包
obd mirror list --remote

# 删除本地包
obd mirror remove <package_name>
```

---

## 3. 高级配置

### 3.1 多 Zone 多副本部署

```yaml
# multi_zone.yaml
user: oceanbase
oceanbase-ce:
  servers:
    - name: z1_s1
      ip: 10.0.0.1
      zone: zone1
    - name: z2_s1
      ip: 10.0.0.2
      zone: zone2
    - name: z3_s1
      ip: 10.0.0.3
      zone: zone3
    - name: z1_s2  # zone1 副本
      ip: 10.0.0.4
      zone: zone1
  global:
    cluster_id: 1
    mysql_port: 2881
    rpc_port: 2882
    devname: eth0
    home_path: /home/oceanbase/observer
    data_dir: /home/oceanbase/data
    log_dir: /home/oceanbase/log
    memory_limit: 32G
    datafile_size: 200G
    cpu_count: 16
```

### 3.2 OBProxy 集群部署

```yaml
# obproxy_cluster.yaml
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

### 3.3 ODP-Proxy（新一代）

```yaml
# odp-proxy.yaml
odp-proxy:
  servers:
    - name: odp1
      ip: 10.0.0.20
  global:
    home_path: /home/oceanbase/odp-proxy
    listen_port: 2883
    rs_list: "10.0.0.1:2881;10.0.0.2:2881;10.0.0.3:2881"
```

---

## 4. 运维排障

```bash
# 查看集群日志
obd cluster display <name> --log

# 连接到集群
obd cluster login <name> --tenant sys --user root

# 检查集群健康
obd cluster check <name>

# 重置节点
obd cluster reinstall <name> --servers server1

# 清理
obd cluster cleanup <name>
```

---

## 参考文档

- OBD 文档：https://www.oceanbase.com/docs/obd-cn/V420/overview-of-obd
- OBD 命令参考：https://www.oceanbase.com/docs/obd-cn/V420/obd-command-reference
- 配置参数：https://www.oceanbase.com/docs/obd-cn/V420/configuration-reference
