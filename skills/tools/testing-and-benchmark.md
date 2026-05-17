# OBD 测试与基准测试（Testing & Benchmark）

> 适用版本：OBD V2.x+ / OceanBase V4.2+
> 工具：obd test（Sysbench、TPC-H、TPC-C、mysqltest）

---

## 概述

OBD 内置了多种性能测试和功能测试工具，可通过 `obd test` 命令一键运行。OBD 会自动下载所需的测试工具包（`ob-sysbench`、`obtpch`、`obtpcc`），无需手动安装。

| 测试类型 | 工具 | 适用场景 |
|---------|------|---------|
| 功能测试 | mysqltest | SQL 兼容性验证 |
| OLTP 性能测试 | Sysbench | 读写/只读/插入/更新等混合负载 |
| OLAP 分析测试 | TPC-H | 复杂分析查询性能 |
| OLTP 事务测试 | TPC-C | 在线事务处理能力 |

---

## 1. 重要规则

1. **禁止通过 yum 安装测试工具。** OBD 在线时会自动下载 `ob-sysbench`、`obtpch`、`obtpcc`。
2. **非交互模式**：在脚本/CI 环境中运行前，设置自动确认：
   ```bash
   obd env set IO_DEFAULT_CONFIRM 1
   ```
3. **全量测试套件**：不要在第一个失败时终止。运行全部测试用例后，输出统一的 pass/fail/skip 报告。
4. **Java 依赖**：TPC-C 可能需要单独安装 Java 环境。

---

## 2. MySQL 功能测试（mysqltest）

```bash
obd test mysqltest <deploy_name> --test-set <test_set> [options]
```

### 参数说明

| 参数 | 说明 | 示例 |
|------|------|------|
| `--test-set` | 测试套件名称 | `basic` |
| `--test-dir` | 自定义测试目录 | `/path/to/tests` |

### 示例

```bash
obd test mysqltest test-cluster --test-set basic
```

---

## 3. Sysbench OLTP 测试

```bash
obd test sysbench <deploy_name> --tenant=<tenant> --script-name=<script> [options]
```

### 常用参数

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `--tenant` | 目标租户名 | - |
| `--script-name` | 测试脚本 | `oltp_read_write.lua` |
| `--threads` | 并发线程数 | - |
| `--time` | 测试时长（秒） | - |
| `--table-size` | 每表行数 | - |
| `--tables` | 表数量 | - |

### 可用脚本

| 脚本 | 说明 |
|------|------|
| `oltp_read_write.lua` | 混合读写（默认） |
| `oltp_read_only.lua` | 纯只读 |
| `oltp_write_only.lua` | 纯写入 |
| `oltp_insert.lua` | 纯插入 |
| `oltp_update_index.lua` | 索引列更新 |
| `oltp_update_non_index.lua` | 非索引列更新 |
| `oltp_delete.lua` | 删除测试 |
| `select_random_points.lua` | 随机点查询 |
| `select_random_ranges.lua` | 随机范围查询 |

### 示例

```bash
obd test sysbench test-cluster \
  --tenant=mysql \
  --script-name=oltp_read_write.lua \
  --threads=32 \
  --time=60 \
  --tables=16 \
  --table-size=1000000
```

---

## 4. TPC-H 分析测试

```bash
obd test tpch <deploy_name> --tenant=<tenant> --remote-tbl-dir=<server_dir> [options]
```

### 关键参数

| 参数 | 说明 | 注意 |
|------|------|------|
| `--tenant` | 目标租户名 | - |
| `--remote-tbl-dir` | **（必填）** 服务器端 TPC-H 表文件目录 | 必须指定 |
| `--scale-factor` | 数据规模因子 | 默认 1（约 1GB） |

### 示例

```bash
obd test tpch test-cluster \
  --tenant=mysql \
  --remote-tbl-dir=/tmp/tpch \
  --scale-factor=1
```

### 注意事项

- TPC-H 数据生成工具 `dbgen` 需要提前在服务器端运行，生成 `.tbl` 文件到 `--remote-tbl-dir` 目录。
- `--remote-tbl-dir` 是服务器端路径，OBD 通过 SSH 将文件导入数据库。

---

## 5. TPC-C 事务测试

```bash
obd test tpcc <deploy_name> --tenant=<tenant> --warehouses=<n> --run-mins=<minutes> [options]
```

### 关键参数

| 参数 | 说明 | 注意 |
|------|------|------|
| `--tenant` | 目标租户名 | - |
| `--warehouses` | 仓库数量 | 建议不少于 10 |
| `--run-mins` | 测试时长（分钟） | 使用 `--run-mins`，不是 `--running-minutes` |
| `--threads` | 并发线程数 | - |

### 示例

```bash
obd test tpcc test-cluster \
  --tenant=mysql \
  --warehouses=10 \
  --run-mins=5 \
  --threads=32
```

### 注意事项

- TPC-C 基于 Java 实现，可能需要单独安装 JDK（建议 JDK 8 或 11）。
- `--run-mins` 参数名不要写成 `--running-minutes`（后者无效）。
- 仓库数量影响数据量，10 个仓库约需 1-2GB 空间。

---

## 6. 非交互执行

在 CI/CD 或自动化脚本中运行测试时：

```bash
# 设置自动确认（避免交互式提示）
obd env set IO_DEFAULT_CONFIRM 1

# 然后运行测试
obd test sysbench test-cluster --tenant=mysql --script-name=oltp_read_write.lua
```

---

## 7. 测试报告规范

运行完整测试套件后，应输出包含以下内容的报告：

| 项目 | 说明 |
|------|------|
| 测试名称 | Sysbench / TPC-H / TPC-C |
| 测试配置 | 线程数、数据量、时长等 |
| Pass / Fail / Skip | 每个测试用例的执行结果 |
| 性能指标 | QPS、TPS、延迟（p50/p95/p99）等 |
| 失败说明 | 对失败用例的简要分析 |

---

## 8. 常见问题

| 问题 | 原因 | 解决 |
|------|------|------|
| 测试工具下载失败 | 网络不通 | 配置代理或离线部署 |
| `--remote-tbl-dir not specified` | TPC-H 必须指定 | 提供 `.tbl` 文件所在目录 |
| Java not found | TPC-C 需要 JDK | 安装 JDK 8+ |
| 端口冲突 | 测试租户端口被占用 | 检查集群端口配置 |
| 权限不足 | 租户资源不够 | 使用 `obd cluster tenant optimize` 调整 |

---

## 参考文档

- OBD 测试命令：https://www.oceanbase.com/docs/obd-cn/V420/run-a-test
- Sysbench 官方文档：https://github.com/akopytov/sysbench
- TPC-H 规范：http://www.tpc.org/tpch/
- TPC-C 规范：http://www.tpc.org/tpcc/

---

## 相关文件

- `devops/obd-deployment.md` — OBD 部署基础
- `tools/obd-deployer.md` — OBD 命令参考
- `admin/tenant-management.md` — 租户创建与优化
