# OBD 部署与集群管理

> 适用版本：OceanBase V4.2+ / OBD V2.x+
> 工具：obd（OceanBase Deployer）

---

## 概述

OBD（OceanBase Deployer）是 OceanBase 官方自动化部署工具，支持集群部署、扩缩容、配置管理、升级和销毁。

| 功能 | 命令 |
|------|------|
| 部署集群 | `obd cluster deploy` |
| 启动集群 | `obd cluster start` |
| 停止集群 | `obd cluster stop` |
| 扩容 | `obd cluster scale-out` |
| 缩容 | `obd cluster scale-in` |
| 销毁集群 | `obd cluster destroy` |
| 配置修改 | `obd cluster edit-config` |
| 升级 | `obd cluster upgrade` |
| 重启 | `obd cluster restart` |

---

## 1. 安装 OBD

```bash
# 方式1：独立安装（推荐生产）
wget https://mirrors.aliyun.com/oceanbase/OBDEPLOY/current/el/7/obd.x86_64.rpm
rpm -ivh obd-*.rpm

# 方式2：OceanBase 软件包自带
tar -xzf oceanbase-ce-4.2.1.el7.x86_64.tar.gz
export PATH=$PWD/bin:$PATH

# 方式3： yum 安装
# 添加 OceanBase 官方的 YUM 仓库
yum install -y yum-utils
yum-config-manager --add-repo https://mirrors.aliyun.com/oceanbase/OceanBase.repo

# 直接安装 ob-deploy（即 OBD）
yum install -y ob-deploy

# 验证
obd --version
obd list
```

---

## 2. 环境准备

### 2.1 系统要求

| 项目 | 最低要求 | 推荐配置 |
|------|---------|---------|
| CPU | 4C | 16C+ |
| 内存 | 8G | 64G+ |
| 磁盘 | 50G SSD | 500G+ NVMe SSD |
| 文件描述符 | 65535 | 65535 |
| 内核参数 | 见下方 | 见下方 |

### 2.2 系统初始化

```bash
#!/bin/bash
# 系统初始化脚本（所有节点执行）

# 1. 关闭 swap
swapoff -a
echo "vm.swappiness = 0" >> /etc/sysctl.conf

# 2. 内核参数
cat >> /etc/sysctl.conf << 'EOF'
net.core.somaxconn = 2048
net.ipv4.tcp_max_syn_backlog = 4096
net.ipv4.tcp_tw_reuse = 1
net.ipv4.ip_local_port_range = 3500 65535
net.core.netdev_max_backlog = 10000
vm.min_free_kbytes = 2097152
EOF
sysctl -p

# 3. 文件描述符
cat >> /etc/security/limits.conf << 'EOF'
*    soft    nofile    65535
*    hard    nofile    65535
*    soft    stack     20480
*    hard    stack     20480
*    soft    nproc     65535
*    hard    nproc     65535
*    soft    memlock   unlimited
*    hard    memlock   unlimited
*    soft    core      unlimited
*    hard    core      unlimited
EOF

# 4. 关闭防火墙
systemctl stop firewalld
systemctl disable firewalld

# 5. 关闭 SELinux
setenforce 0
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config

# 6. 创建用户
useradd -m oceanbase
echo "oceanbase:OBPassw0rd" | chpasswd

# 7. 创建目录
mkdir -p /home/oceanbase/{data,log,software}
chown -R oceanbase:oceanbase /home/oceanbase
```

### 2.3 SSH 免密（部署节点到所有节点）

```bash
su - oceanbase
ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa
ssh-copy-id oceanbase@node2
ssh-copy-id oceanbase@node3
```

---

## 3. 部署集群

### 3.1 最小部署（单机1-1-1）

```bash
# 部署单节点集群（开发/测试）
obd cluster deploy my_cluster \
  --type oceanbase-ce \
  -c single_node.yaml

# single_node.yaml 示例
```

```yaml
# single_node.yaml
user: oceanbase
oceanbase-ce:
  servers:
    - name: server1
      ip: 192.168.1.10
  global:
    devname: eth0
    mysql_port: 2881
    rpc_port: 2882
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

### 3.2 生产部署（3节点1-1-1）

```yaml
# cluster.yaml
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

```bash
# 部署
obd cluster deploy prod_cluster -c cluster.yaml

# 启动
obd cluster start prod_cluster

# 查看状态
obd cluster display prod_cluster
```

### 3.3 初始化集群

```bash
# 连接集群（默认 root@sys 密码为空）
mysql -h 10.0.0.1 -P 2881 -u root -p

-- 修改 root 密码
ALTER USER root IDENTIFIED BY 'Root@Pass2024';

-- 创建业务租户
CREATE RESOURCE UNIT unit1 MAX_CPU 4, MIN_CPU 2, MEMORY_SIZE '4G', LOG_DISK_SIZE '6G';
CREATE RESOURCE POOL pool1 UNIT = 'unit1', UNIT_NUM = 1, ZONE_LIST = ('zone1','zone2','zone3');
CREATE TENANT business REPLICA_NUM = 3, RESOURCE_POOL_LIST = ('pool1')
  SET ob_compatibility_mode = 'mysql';
-- 初始密码 root@业务租户
ALTER USER root IDENTIFIED BY 'BizRoot@2024';
```

---

## 4. 集群管理

### 4.1 扩容（增加节点）

```bash
# 修改配置添加新节点
obd cluster edit-config prod_cluster
# 在编辑器中添加新 server

# 扩容
obd cluster scale-out prod_cluster

# 验证
obd cluster display prod_cluster
```

### 4.2 缩容（移除节点）

```bash
obd cluster edit-config prod_cluster
# 删除要移除的 server

obd cluster scale-in prod_cluster
```

### 4.3 修改配置

```bash
# 修改参数并重启
obd cluster edit-config prod_cluster
# 修改 global 下的参数

obd cluster restart prod_cluster

# 仅重启特定节点
obd cluster restart prod_cluster --servers server1
```

### 4.4 查看状态

```bash
# 集群状态
obd cluster display prod_cluster

# 集群列表
obd cluster list

# 查看集群配置
obd cluster show-config prod_cluster
```

---

## 5. OBProxy 部署

```yaml
# obproxy.yaml
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

```bash
obd cluster deploy prod_proxy -c obproxy.yaml
obd cluster start prod_proxy
```

---

## 6. 常见问题

| 问题 | 原因 | 解决 |
|------|------|------|
| ` ssh connect failed` | 免密配置失败 | 检查 ~/.ssh/authorized_keys |
| `port already in use` | 端口被占用 | `kill -9 $(lsof -t -i:2881)` |
| `insufficient memory` | 内存不够 | 增加 `memory_limit` 或加内存 |
| `data_dir not found` | 目录不存在 | 创建并赋权 |
| `boot strap failed` | 初始化失败 | 检查 `log/observer.log` |
| 启动超时 | 磁盘慢或内存不够 | 增大 `boot_stale_time` 或检查资源 |

---

## 参考文档

- OBD 安装：https://www.oceanbase.com/docs/obd-cn/V420/install-obd
- 部署集群：https://www.oceanbase.com/docs/obd-cn/V420/deploy-a-cluster
- 配置参考：https://www.oceanbase.com/docs/obd-cn/V420/configuration
- 扩缩容：https://www.oceanbase.com/docs/obd-cn/V420/resize-a-cluster
