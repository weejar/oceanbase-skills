# obloader & obdumper 数据导入导出

> 适用版本：obloader/obdumper V4.x / OceanBase V4.2+
> 兼容模式：MySQL Mode / Oracle Mode

---

## 概述

obloader 和 obdumper 是 OceanBase 官方的数据导入导出工具，替代传统的 MySQL dump/load，针对分布式场景优化。

| 工具 | 功能 | 方向 |
|------|------|------|
| obloader | 数据导入 | 外部文件 → OceanBase |
| obdumper | 数据导出 | OceanBase → 外部文件 |

---

## 1. 安装

```bash
# 下载
wget https://mirrors.aliyun.com/oceanbase/OBLOADER/current/obloader-linux-x86_64.tar.gz
wget https://mirrors.aliyun.com/oceanbase/OBDUMPER/current/obdumper-linux-x86_64.tar.gz

# 解压
tar -xzf obloader-linux-x86_64.tar.gz -C /opt/obloader
tar -xzf obdumper-linux-x86_64.tar.gz -C /opt/obdumper

# 配置环境变量
export PATH=$PATH:/opt/obloader/bin:/opt/obdumper/bin

# 验证
obloader --version
obdumper --version
```

---

## 2. obdumper（导出）

### 2.1 基本导出

```bash
# 导出单个表
obdumper \
  -h 10.0.0.1 -P 2883 \
  -u root@app_tenant -p 'password' \
  -D app_db -t orders \
  --output-dir /data/export/

# 导出整个数据库
obdumper \
  -h 10.0.0.1 -P 2883 \
  -u root@app_tenant -p 'password' \
  -D app_db \
  --output-dir /data/export/
```

### 2.2 导出格式

```bash
# CSV 格式（默认）
obdumper ... --file-format csv --field-delimiter ','

# SQL INSERT 格式
obdumper ... --file-format sql --batch-size 1000

# Parquet 格式（大数据场景）
obdumper ... --file-format parquet

# 控制文件大小
obdumper ... --file-size 1G  # 每个文件最大1G
```

### 2.3 导出选项

| 参数 | 说明 | 示例 |
|------|------|------|
| `-t` | 表名 | `--table orders` |
| `-q` | 过滤条件 | `--query "status=1"` |
| `-c` | 列 | `--column "id,name,status"` |
| `--threads` | 并发线程 | `--threads 8` |
| `--file-size` | 文件大小 | `--file-size 1G` |
| `--slice-by` | 按列分片 | `--slice-by id` |
| `--where` | WHERE 条件 | `--where "created_at > '2024-01-01'"` |
| `--log-level` | 日志级别 | `--log-level INFO` |

### 2.4 导出示例

```bash
# 导出大表（分片并行）
obdumper \
  -h 10.0.0.1 -P 2883 \
  -u root@app_tenant -p 'password' \
  -D app_db -t orders \
  --threads 8 \
  --slice-by id \
  --file-size 2G \
  --file-format csv \
  --output-dir /data/export/orders/

# 仅导出表结构
obdumper \
  -h 10.0.0.1 -P 2883 \
  -u root@app_tenant -p 'password' \
  -D app_db -t orders \
  --schema-only \
  --output-dir /data/export/schema/
```

---

## 3. obloader（导入）

### 3.1 基本导入

```bash
# 导入 CSV
obloader \
  -h 10.0.0.1 -P 2883 \
  -u root@app_tenant -p 'password' \
  -D app_db \
  --input-dir /data/export/orders/ \
  --table orders \
  --threads 8

# 导入 SQL 文件
obloader \
  -h 10.0.0.1 -P 2883 \
  -u root@app_tenant -p 'password' \
  -D app_db \
  --input-dir /data/export/ \
  --file-format sql \
  --threads 4
```

### 3.2 导入选项

| 参数 | 说明 | 示例 |
|------|------|------|
| `--table` | 目标表名 | `--table orders` |
| `--input-dir` | 输入目录 | `--input-dir /data/csv/` |
| `--threads` | 并发线程 | `--threads 8` |
| `--batch-size` | 批量大小 | `--batch-size 1000` |
| `--column` | 列映射 | `--column "id,name,amount"` |
| `--skip-header` | 跳过首行 | `--skip-header` |
| `--delimiter` | 分隔符 | `--delimiter ','` |
| `--enclosure` | 引号字符 | `--enclosure '"'` |
| `--encoding` | 编码 | `--encoding UTF-8` |

### 3.3 高级导入

```bash
# 导入并忽略重复（ON DUPLICATE KEY UPDATE）
obloader ... --duplicate-policy update

# 仅插入不存在的（INSERT IGNORE）
obloader ... --duplicate-policy ignore

# 遇到错误继续
obloader ... --max-errors 1000

# 导入前截断表
obloader ... --truncate-table

# 导入指定列
obloader \
  -h 10.0.0.1 -P 2883 \
  -u root@app_tenant -p 'password' \
  -D app_db \
  --table orders \
  --column "id,user_id,amount,status" \
  --input-dir /data/csv/orders/
```

---

## 4. 配置文件方式

复杂场景推荐使用配置文件：

```yaml
# export.yaml（obdumper）
db:
  host: 10.0.0.1
  port: 2883
  user: root@app_tenant
  password: ******
  database: app_db

output:
  dir: /data/export/
  format: csv
  file_size: 2G

tables:
  - name: orders
    threads: 8
    slice_by: id
  - name: order_items
    threads: 4
    where: "created_at > '2024-01-01'"
```

```yaml
# import.yaml（obloader）
db:
  host: 10.0.0.1
  port: 2883
  user: root@app_tenant
  password: ******
  database: app_db

input:
  dir: /data/export/
  format: csv
  encoding: UTF-8
  skip_header: true

tables:
  - name: orders
    threads: 8
    batch_size: 2000
    duplicate_policy: update
```

```bash
obdumper --config export.yaml
obloader --config import.yaml
```

---

## 5. 性能调优

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| `--threads` | 4-16 | 根据CPU核数调整 |
| `--batch-size` | 1000-5000 | 导入时增大提升吞吐 |
| `--slice-by` | 主键列 | 大表按主键分片并行 |
| `--file-size` | 1-4G | 避免文件过小（管理开销） |
| 目标端内存 | ≥4G | 导入时 OBServer 需要内存 |
| 目标端 redo | 足够 | 避免日志写满 |

---

## 6. 常见问题

| 问题 | 原因 | 解决 |
|------|------|------|
| 导出慢 | 表无主键分片 | 使用 `--slice-by` 指定分片列 |
| 导入报唯一键冲突 | 数据重复 | `--duplicate-policy update` |
| 中文乱码 | 编码不一致 | `--encoding GBK` |
| 内存溢出 | 批量太大 | 减小 `--batch-size` |
| 网络超时 | 数据量大 | 增大 `--socket-timeout` |

---

## 参考文档

- obloader 文档：https://www.oceanbase.com/docs/obloader-obdumper-cn/V420/obloader
- obdumper 文档：https://www.oceanbase.com/docs/obloader-obdumper-cn/V420/obdumper
- 数据迁移：https://www.oceanbase.com/docs/obloader-obdumper-cn/V420/data-migration
