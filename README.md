# OceanBase DB Skills

> 93 个 OceanBase 数据库专项参考指南，覆盖 SQL、PL/SQL、性能调优、安全、迁移、多租户架构、分布式运维、生态工具及 AI 向量特性。
> 同时作为 WorkBuddy / Claude Code Agent Skill 使用，可在 AI 助手对话中按需加载对应主题文件。

[![License](https://img.shields.io/badge/license-Apache%202.0-blue.svg)](LICENSE)
[![OceanBase](https://img.shields.io/badge/OceanBase-V4.2%20%7C%20V4.4%20%7C%20V4.5-orange.svg)](https://www.oceanbase.com)
[![Skills](https://img.shields.io/badge/skills-93%20guides-green.svg)](#目录结构)

---

## 特性

- **双模式覆盖** — MySQL Mode 与 Oracle Mode 均有说明，差异点逐项标注
- **版本标注** — 基线为 V4.2，V4.4/V4.5 新特性单独注明
- **实战导向** — 每个文件包含背景说明、SQL 示例、最佳实践与常见误区
- **按需加载** — 文件独立，按主题查阅，不需要加载全部内容
- **AI 集成** — 可作为 WorkBuddy / Claude Code Skill 使用，Agent 自动路由到对应文件

---

## 快速开始

### 作为文档查阅

直接浏览对应分类目录，找到主题文件后阅读即可：

```
skills/performance/slow-query-analysis.md   # 慢SQL诊断
skills/performance/explain-plan.md          # 执行计划解读
skills/admin/tenant-management.md           # 租户管理
skills/appdev/jdbc-connection.md            # Java JDBC 连接
```

### 作为 Agent Skill 使用

1. 将本仓库克隆到 `~/.workbuddy/skills/oceanbase-db-skills/`
2. 在 WorkBuddy 或 Claude Code 对话中 `@skill:oceanbase-db-skills` 即可激活
3. Agent 会根据问题自动路由到对应文件

---

## 目录结构

```
skills/
├── admin/           (7)  备份恢复、物理恢复、日志归档、租户管理、用户管理、集群管理
├── architecture/    (7)  分布式架构、多租户、OBProxy、Paxos副本、物理备库、共享存储
├── performance/     (9)  执行计划、索引策略、慢SQL诊断、统计信息、等待事件、内存调优、SQL采集与优化
├── monitoring/      (8)  告警日志、健康巡检、活动会话、SQL诊断、空间管理、Top SQL
├── appdev/          (12) JDBC/Python/Go/C++、连接池、事务管理、批处理优化、Spring/MyBatis/Hibernate
├── migrations/      (9)  MySQL/Oracle/PostgreSQL/DB2/SQL Server 迁移、数据类型映射、割接验证
├── sql-dev/         (5)  SQL规范、DDL/DML最佳实践、查询改写、分页优化
├── plsql/           (3)  存储过程、触发器、包（Oracle Mode）
├── security/        (5)  权限管理、行级安全、TDE加密、审计、网络安全
├── design/          (3)  数据建模、分区策略、表组设计
├── devops/          (3)  OBD部署、Schema管理、版本升级
├── features/        (5)  物化视图、DBLink、闪回查询、列存副本、表级恢复
├── tools/           (13) OCP/OMS/ODC/obdiag/obloader/obdumper/OBD/ob-operator/oblogproxy
└── ai/              (3)  向量检索、AI函数服务、混合检索
```

---

## 重点文件速查

| 场景 | 文件 |
|------|------|
| 慢SQL / 高磁盘读 / 全表扫诊断 | `performance/slow-query-analysis.md` |
| 采集指定 SQL_ID 的执行计划与索引信息 | `performance/sql-collect.md` |
| 生成 SQL 优化诊断报告 | `performance/sql-optimization.md` |
| 读懂 EXPLAIN 执行计划 | `performance/explain-plan.md` |
| 当前活动会话 / 长空闲会话 / 等待事件 | `monitoring/session-monitor.md` |
| SQL 诊断脚本（hint检测/并发/一键诊断） | `monitoring/sql-diagnostics.md` |
| Sys 租户连接 + 租户资源查询 | `appdev/sys-tenant-connection.md` |
| 租户创建、扩缩容、资源管理 | `admin/tenant-management.md` |
| Java JDBC 最佳实践 | `appdev/jdbc-connection.md` |
| 从 Oracle 迁移到 OceanBase | `migrations/oracle-to-ob.md` |
| OBD 一键部署 | `devops/obd-deployment.md` |
| SeekDB OBD 运维操作 | `tools/seekdb-obd-operations.md` |
| 向量检索 / AI 工作负载 | `ai/vector-search.md` |

---

## 版本说明

| 版本 | 说明 |
|------|------|
| **V4.2** | 主要基线，大多数文件以此为准 |
| **V4.4** | 新特性在文件内以 `[V4.4+]` 注明 |
| **V4.5** | 新特性在文件内以 `[V4.5+]` 注明 |

兼容模式：MySQL Mode 和 Oracle Mode 的差异在每个文件的 **Compatibility Notes** 章节单独说明。

---

## 参考文档

- [OceanBase 官方文档（中文）](https://www.oceanbase.com/docs/oceanbase-database-cn)
- [OceanBase 官方文档（英文）](https://en.oceanbase.com/docs/oceanbase-database)
- [OCP 管理平台文档](https://www.oceanbase.com/docs/ocp)
- [OMS 迁移服务文档](https://www.oceanbase.com/docs/oms)
- [ODC 开发中心文档](https://www.oceanbase.com/docs/odc)
- [obdiag 诊断工具](https://www.oceanbase.com/docs/obdiag)
- [OBD 部署器](https://www.oceanbase.com/docs/obd-cn)

---

## 贡献

欢迎提交 Issue 或 PR，改进已有文件或补充新主题。内容规范见 [`SKILL_GENERATION_PROMPT.md`](SKILL_GENERATION_PROMPT.md)。

---

## License

[Apache 2.0](LICENSE)
