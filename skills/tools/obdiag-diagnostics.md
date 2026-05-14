# obdiag 诊断工具详解



> 适用版本：obdiag V2.x / OceanBase V4.2+
> 兼容模式：MySQL Mode / Oracle Mode

---

## 概述

obdiag（OceanBase Diagnostics）是 OceanBase 官方诊断工具，提供一键诊断、日志分析、性能报告生成等功能。

| 功能 | 说明 |
|------|------|
| 一键诊断 | 自动收集集群状态、日志、配置 |
| 慢SQL分析 | 分析慢SQL根因 |
| 日志分析 | 解析 observer 日志 |
| 性能报告 | 生成性能诊断报告 |
| 场景诊断 | 死锁、RT抖动、CPU高等专项诊断 |
| 信息收集 | 收集节点信息、配置、日志打包 |

---

## 1. 安装

```bash
# 方式1：源码安装
wget https://mirrors.aliyun.com/oceanbase/OBDIAG/current/obdiag-linux-x86_64.tar.gz
tar -xzf obdiag-linux-x86_64.tar.gz
export PATH=$PWD/obdiag/bin:$PATH

# 方式2：在线yum 安装
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://mirrors.aliyun.com/oceanbase/OceanBase.repo
sudo yum install -y oceanbase-diagnostic-tool
sh /opt/oceanbase-diagnostic-tool/init.sh

# 方式3：OBD 附带
# 在线安装
obd mirror enable remote
obd obdiag deploy

# 离线安装
从https://www.oceanbase.com/softwarecenter 下载obdiag 的 RPM 包
obd mirror clone oceanbase-diagnostic-tool-*.rpm
obd obdiag deploy

# 验证
obdiag --version
obdiag --help
```

注意： 在线 YUM 方式支持 CentOS 7/8，暂不支持 CentOS 9；CentOS 9 请使用下方离线部署或源码安装。

---

## 2. 基本使用

### 2.1 配置

```bash
# 初始化配置
obdiag config init

# 编辑配置
vi ~/.obdiag/config.yaml

# 配置项
# clusters:
#   - name: prod_cluster
#     ob_cluster:
#       hosts:
#         - ip: 10.0.0.1
#           port: 2881
#         - ip: 10.0.0.2
#           port: 2881
#       tenant: sys
#       user: root
#       password: ******
#     obproxy:
#       hosts:
#         - ip: 10.0.0.10
#           port: 2883

# 使用命令行配置
obdiag config -h <db_host> -u <sys_user> [-p password] [-P port]

例子
# 密码非空例子
obdiag config -hxx.xx.xx.xx -uroot@sys -p***** -P2881

# 空密码场景
obdiag config -hxx.xx.xx.xx -uroot@sys -P2881

# 通过 obproxy 场景
obdiag config -hxx.xx.xx.xx -uroot@sys#obtest  -p***** -P2883

```

注意：使用 `config` 命令会进入交互模式按照提示填入信息，obdiag >= 2.4.0 版本支持通过 `--config` 参数来传递用户侧的配置文件，让使用者可以无配置文件即可开箱即用 obdiag

### 2.2 一键诊断

```bash
# 全场景诊断
obdiag diagnose run --cluster prod_cluster

# 指定场景诊断
obdiag diagnose run --scene=rt_jitter --cluster prod_cluster
obdiag diagnose run --scene=high_cpu --cluster prod_cluster
obdiag diagnose run --scene=deadlock --cluster prod_cluster
obdiag diagnose run --scene=memory_leak --cluster prod_cluster

# 指定时间范围
obdiag diagnose run --from="2024-03-15 14:00:00" --to="2024-03-15 15:00:00"
```

### 2.3 诊断场景

| 场景 | 命令参数 | 说明 |
|------|---------|------|
| RT 抖动 | `--scene=rt_jitter` | 响应时间波动诊断 |
| CPU 飙高 | `--scene=high_cpu` | CPU 使用率异常诊断 |
| 死锁 | `--scene=deadlock` | 死锁分析 |
| 内存泄漏 | `--scene=memory_leak` | 内存持续增长 |
| 磁盘慢 | `--scene=slow_disk` | IO 延迟高 |
| 连接数耗尽 | `--scene=conn_exhausted` | 连接数异常 |
| 租户资源 | `--scene=tenant_resource` | 租户资源使用诊断 |
| 备份恢复 | `--scene=backup` | 备份/恢复问题 |
| 全量 | （默认） | 所有场景综合诊断 |

---

## 3. 日志分析

### 3.1 收集日志

```bash
# 收集所有节点日志
obdiag gather log --cluster prod_cluster

# 收集指定时间范围
obdiag gather log --from="2024-03-15 14:00:00" --to="2024-03-15 15:00:00"

# 收集指定类型
obdiag gather log --scope=observer,election,rootservice

# 收集到指定目录
obdiag gather log --output_dir=/data/diag_logs/
```

### 3.2 分析日志

```bash
# 分析错误日志
obdiag analyze log --level=ERROR --from="2024-03-15 00:00:00"

# 分析警告日志
obdiag analyze log --level=WARN --scope=observer

# 搜索关键字
obdiag analyze log --keyword="timeout,ret=-"

# 统计日志频率
obdiag analyze log --top_n=20 --from="2024-03-15 00:00:00"
```

### 3.3 日志内容

| 日志类型 | 路径 | 说明 |
|---------|------|------|
| observer 日志 | `log/observer.log` | 主日志 |
| election 日志 | `log/election.log` | 选举日志 |
| rootservice 日志 | `log/rootservice.log` | RS 日志 |
| WAL 日志 | `log/` | 事务日志 |

---

## 4. 信息收集

```bash
# 收集集群信息
obdiag gather system_info --cluster prod_cluster

# 收集内容包括：
#   - 操作系统信息
#   - 内核参数
#   - 磁盘信息
#   - 网络配置
#   - OBServer 配置
#   - 租户信息
#   - SQL 审计数据
#   - 表统计信息

# 打包收集结果
obdiag gather scene --cluster prod_cluster --output_dir=/data/ob_collect/
```

---

## 5. 性能报告

```bash
# 生成性能报告
obdiag report run --cluster prod_cluster

# 指定报告范围
obdiag report run --from="2024-03-15 00:00:00" --to="2024-03-15 23:59:59"

# 报告内容包括：
#   - 集群概览
#   - QPS/TPS 趋势
#   - SQL 响应时间分布
#   - Top 慢SQL
#   - 等待事件分布
#   - CPU/内存/IO 使用
#   - 连接数趋势
#   - 死锁信息
#   - 错误统计
```

---

## 6. 慢SQL诊断

```bash
# 慢SQL分析
obdiag analyze slow_sql --cluster prod_cluster --top_n=20

# 指定时间范围
obdiag analyze slow_sql --from="2024-03-15 00:00:00" --to="2024-03-15 23:59:59"

# 输出示例：
# Top Slow SQL #1:
#   SQL: SELECT * FROM orders WHERE status = 1 ORDER BY created_at DESC LIMIT 100
#   Avg RT: 5.23s
#   Exec Count: 1,234
#   Plan: TABLE SCAN (no index used)
#   Suggestion: Add index on (status, created_at)
```

---

## 7. 常用命令速查

```bash
obdiag config init              # 初始化配置
obdiag diagnose run              # 一键诊断
obdiag gather log               # 收集日志
obdiag analyze log              # 分析日志
obdiag gather system_info       # 收集系统信息
obdiag report run               # 生成报告
obdiag analyze slow_sql         # 分析慢SQL
obdiag list scene               # 列出诊断场景
obdiag display                  # 显示集群状态
```

---

## 参考文档

- obdiag 官方文档：https://www.oceanbase.com/docs/obdiag-cn
- obdiag 安装：https://www.oceanbase.com/docs/obdiag-cn/V420/install-obdiag
- 诊断场景：https://www.oceanbase.com/docs/obdiag-cn/V420/diagnostic-scenes
