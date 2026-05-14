# OMS 迁移服务

> 适用版本：OMS V4.x / OceanBase V4.2+
> 官网：https://www.oceanbase.com/product/oms

---

## 概述

OMS（OceanBase Migration Service）是 OceanBase 官方数据迁移服务，支持多种数据源之间的数据迁移和实时同步。

| 功能 | 说明 |
|------|------|
| 结构迁移 | 自动迁移表结构（DDL） |
| 全量迁移 | 一次性全量数据迁移 |
| 增量同步 | 实时增量数据同步（CDC） |
| 数据校验 | 迁移后自动数据校验 |
| 反向同步 | 支持迁移后反向同步回源 |

---

## 1. 支持的源/目标

| 源端 | → 目标端 | 迁移方式 |
|------|---------|---------|
| MySQL | → OceanBase (MySQL Mode) | 结构+全量+增量 |
| Oracle | → OceanBase (Oracle Mode) | 结构+全量+增量 |
| PostgreSQL | → OceanBase (MySQL Mode) | 结构+全量+增量 |
| DB2 | → OceanBase | 结构+全量 |
| TiDB | → OceanBase (MySQL Mode) | 结构+全量+增量 |
| OceanBase | → MySQL | 结构+全量+增量 |
| OceanBase | → Oracle | 结构+全量 |
| Kafka | → OceanBase | 增量同步 |

---

## 2. 安装部署

### 2.1 部署架构

```
OMS Console (Web界面)
  ├── OMS Server (迁移引擎)
  │     ├── Reader (读取源端数据)
  │     ├── Writer (写入目标端)
  │     └── Validator (数据校验)
  └── OMS MetaDB (元数据库)
```

### 2.2 安装

```bash
# 1. 下载 OMS
wget https://mirrors.aliyun.com/oceanbase/OMS/current/oms.tar.gz

# 2. 解压
tar -xzf oms.tar.gz -C /opt/oms

# 3. 配置
cd /opt/oms
vi conf/oms.yaml

# 4. 安装
bash install.sh

# 5. 访问
# http://<oms_host>:8088
# 默认: admin / admin
```

---

## 3. 创建迁移项目

### 3.1 Web 界面操作

```
步骤1：创建迁移项目
  - 项目名称: mysql_to_ob
  - 源端类型: MySQL
  - 目标端类型: OceanBase (MySQL Mode)

步骤2：配置源端连接
  - 主机: source_mysql:3306
  - 用户: repl_user
  - 密码: ******
  - 数据库: app_db

步骤3：配置目标端连接
  - 主机: ob_host:2883
  - 租户: business_tenant
  - 用户: root
  - 密码: ******
  - 数据库: app_db

步骤4：选择迁移对象
  - 全部表 / 选择特定表
  - 是否迁移结构: 是
  - 是否全量迁移: 是
  - 是否增量同步: 是

步骤5：迁移参数
  - 并发度: 8
  - 批量大小: 1000
  - 冲突策略: 忽略/覆盖/报错

步骤6：预检查 & 执行
  - 执行预检查
  - 确认后启动迁移
```

---

## 4. 增量同步（CDC）

### 4.1 配置增量同步

```yaml
# 增量同步配置
source:
  type: mysql
  binlog_format: ROW  # 必须 ROW 格式
  binlog_row_image: FULL

target:
  type: oceanbase
  tenant: business

sync:
  mode: incremental
  start_position: latest  # 从最新位置开始
  # start_position: "mysql-bin.000123:4567"  # 指定位置
  filter:
    tables:
      - schema: app_db
        table: orders
      - schema: app_db
        table: order_items
```

### 4.2 监控同步状态

```
Web 界面：迁移项目 → 项目详情 → 增量同步

关键指标：
  - 同步延迟 (Lag): 目标端落后源端的时间
  - 数据量统计: 已同步行数/字节数
  - DDL 同步状态
  - 错误日志
```

---

## 5. 数据校验

### 5.1 配置校验

```
Web 界面：迁移项目 → 数据校验

校验选项：
  - 行数校验: COUNT(*) 比较
  - 哈希校验: 每行数据哈希比较
  - 采样校验: 随机采样N行比较
  - 全量校验: 逐行比较（最慢最精确）
```

### 5.2 校验结果处理

```
校验不一致时的处理：
  1. 查看差异详情（哪些行/列不一致）
  2. 确认差异原因（增量延迟？数据冲突？）
  3. 修复方案：
     - 等待增量追平后重新校验
     - 手动修复不一致数据
     - 重新全量迁移问题表
```

---

## 6. 反向同步

```yaml
# 配置反向同步（OceanBase → MySQL）
source:
  type: oceanbase
  host: ob_host:2883
  tenant: business

target:
  type: mysql
  host: mysql_host:3306

sync:
  mode: incremental
  start_position: "2024-03-15 00:00:00"
  # 只同步增量，不重新迁移全量
```

---

## 7. 性能调优

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| 并发线程 | 4-16 | 根据源端/目标端负载调整 |
| 批量大小 | 500-5000 | 全量迁移时增大 |
| 网络带宽 | ≥1Gbps | 确保迁移带宽充足 |
| 源端负载 | ≤50% | 避免影响源端业务 |
| 目标端写入 | 开启批量写入 | 提高 OMS 写入效率 |

---

## 常见问题

| 问题 | 原因 | 解决 |
|------|------|------|
| 增量延迟大 | 目标端写入慢 | 增加并发，优化目标端索引 |
| 校验不一致 | 有实时写入 | 停止写入后重新校验 |
| DDL 迁移失败 | 不兼容语法 | 手动调整 DDL 后重试 |
| 迁移中断 | 网络异常 | OMS 支持断点续传，重新启动 |
| ORA-xxxx 错误 | Oracle 模式兼容问题 | 检查数据类型映射 |

---

## 参考文档

- OMS 官方文档：https://www.oceanbase.com/docs/oms-cn
- 数据迁移指南：https://www.oceanbase.com/docs/oms-cn/V420/data-migration-guide
- 增量同步：https://www.oceanbase.com/docs/oms-cn/V420/incremental-sync
