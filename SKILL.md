---
name: oceanbase-db-skills
description: 108 OceanBase database reference guides covering SQL, PL/SQL, performance tuning, security, migrations, multi-tenant architecture, distributed O&M, OceanBase tools ecosystem, and AI vector features. Load individual skill files on demand for expert guidance on any OceanBase topic.
---

# OceanBase DB Skills

A collection of 108 standalone reference guides for OceanBase Database (V4.2/V4.4/V4.5). Each file covers one topic with explanations, practical examples, best practices, and common mistakes. Covers both MySQL Mode and Oracle Mode compatibility.

## How to Use

1. **Find the right skill** using the category routing table below.
2. **Read only the file(s)** relevant to the user's task — do not load all files at once.
3. **Apply the guidance** to answer questions, generate code, or review existing work.
4. **Note dual modes**: Many topics have MySQL Mode vs Oracle Mode differences — check the file's "Compatibility Notes" section.

## Category Routing

| User asks about… | Read from |
|------------------|-----------|
| Backup, recovery, tenant management, resource units, logs | `skills/admin/` |
| Java/Python/Go/Node.js/.NET dev, connection pooling, JSON, transactions, **sys tenant connection**, **connect oceanbase** | `skills/appdev/` |
| Distributed architecture, multi-tenant, Zone, OBProxy, Paxos, resource isolation | `skills/architecture/` |
| Data modeling, partitioning strategy, tablegroup design | `skills/design/` |
| OBD deployment, schema migration, version upgrade | `skills/devops/` |
| Materialized views, DBLinks, flashback, columnar replica | `skills/features/` |
| Migrating from MySQL/Oracle/PostgreSQL/DB2/TiDB to OceanBase | `skills/migrations/` |
| Alert log, health check, space management, Top SQL, obdiag, **session monitoring**, **active session**, **sql-diagnostics**, **slow-query-analysis**, **slow SQL**, **slow-query**, **查询慢SQL**, **hint detection**, **append hint**, **PARALLEL hint**, **direct-path insert**, **batch insert**, **multi-value insert**, **rewriteBatchedStatements**, **批处理优化**, **is_batched_multi_stmt** | `skills/monitoring/` |
| EXPLAIN plan, indexes, optimizer stats, slow query, wait events, memory, plan cache, **high disk reads**, **full table scan**, **lock contention**, **retry SQL**, **sql-collect**, **sql-optimization**, **SQL采集**, **SQL优化建议**, **SQL优化诊断**, **SQL诊断报告**, **SQL优化报告** | `skills/performance/` |
| Privileges, row-level security, TDE, auditing, network security | `skills/security/` |
| SQL patterns, SQL tuning, hints, dynamic SQL, injection prevention | `skills/sql-dev/` |
| PL/SQL development (Oracle Mode), stored procedures, packages | `skills/plsql/` |
| OCP, OMS, ODC, obdiag, obloader/obdumper, OBD, ob-operator, oblogproxy, testing & benchmark, SeekDB OBD ops, OBD config, OBD monitoring, tenant OBD management | `skills/tools/` |
| Vector search, AI function service, hybrid search | `skills/ai/` |

## Skills Directory

```
skills/
├── admin/          Database administration (backup, recovery, tenant, user, log archive)
├── appdev/         Application development (JSON, transactions, multi-language, compatibility modes, **sys-tenant-connection.md**)
├── architecture/   Distributed architecture (multi-tenant, Zone, OBProxy, Paxos, resource isolation)
├── design/         Schema design (data modeling, partitioning, tablegroup)
├── devops/         DevOps (OBD deployment, schema migration, version upgrade)
├── features/       OceanBase features (MVs, DBLinks, flashback, columnar replica)
├── migrations/     Migrating to/from OceanBase (MySQL, Oracle, PostgreSQL, DB2, TiDB)
├── monitoring/     Diagnostics (alert log, health check, space, Top SQL, obdiag, **session-monitor.md**, **sql-diagnostics.md**, **slow-query-analysis.md**)
├── performance/    Tuning (EXPLAIN, indexes, optimizer, wait events, memory, plan cache, **slow-query-analysis.md**, **sql-collect.md**, **sql-optimization.md**)
├── security/       Security (privileges, row-level security, TDE, auditing, network)
├── sql-dev/        SQL development (tuning, patterns, hints, dynamic SQL, injection)
├── plsql/          PL/SQL development (Oracle Mode only: procedures, packages)
├── tools/          OceanBase ecosystem tools (OCP, OMS, ODC, obdiag, OBD, ob-operator, testing, SeekDB OBD, monitoring)
└── ai/             AI features (vector search, AI function service, hybrid search)
```

- **`skills/performance/slow-query-analysis.md`** — start here for any SQL performance issue (慢SQL、高磁盘读、全表扫、hint检测、一键诊断)
- **`skills/performance/sql-collect.md`** — 采集指定 SQL_ID 的执行计划、对象、索引和统计信息，输出 Markdown 报告
- **`skills/performance/sql-optimization.md`** — 基于采集数据生成 SQL 优化诊断建议（自动规则分析 + 汇总报告）
- **`skills/performance/explain-plan.md`** — foundation for all SQL performance work
- **`skills/performance/explain-plan.md`** — foundation for all SQL performance work
- **`skills/architecture/multi-tenant.md`** — core OceanBase multi-tenant architecture
- **`skills/admin/tenant-management.md`** — tenant creation, scaling, resource management
- **`skills/admin/cluster-management.md`** — Zone/Server 管理、Unit 迁移、集群扩缩容
- **`skills/tools/testing-and-benchmark.md`** — OBD 测试与基准测试（Sysbench/TPC-H/TPC-C）
- **`skills/tools/seekdb-obd-operations.md`** — SeekDB OBD 操作（部署/HA/切换）
- **`skills/tools/obd-config-deployment.md`** — OBD 配置文件部署参考
- **`skills/tools/obd-monitoring-setup.md`** — Prometheus + Grafana 监控部署
- **`skills/tools/obdiag-diagnostics.md`** — obdiag diagnostic tool usage
- **`skills/appdev/sys-tenant-connection.md`** - Sys租户连接与租户信息查询（连接方法/租户列表/资源详情）
- **`skills/monitoring/session-monitor.md`** - 活动会话监控（当前会话/会话统计/长空闲会话/等待事件）
- **`skills/monitoring/sql-diagnostics.md`** - SQL 诊断（慢SQL/磁盘读/hint检测/全表扫/并发等待/完整诊断脚本）
- **`skills/ai/vector-search.md`** — vector search and AI workloads on OceanBase

## Version Baseline

- **Primary baseline**: OceanBase V4.2 (MySQL Mode V4.2 / Oracle Mode V4.2)
- **New feature annotations**: V4.4 / V4.5 features are noted in "OceanBase Version Notes" section
- **Dual mode coverage**: MySQL Mode and Oracle Mode differences are noted per topic

## Sources

All content is adapted from official OceanBase documentation:
- English: https://en.oceanbase.com/docs/oceanbase-database
- Chinese: https://www.oceanbase.com/docs/oceanbase-database-cn
- OCP: https://en.oceanbase.com/docs/ocp
- OMS: https://en.oceanbase.com/docs/oms
- ODC: https://en.oceanbase.com/docs/odc
