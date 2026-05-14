# Skills Index — OceanBase DB Skills

Progress tracking for all 96 skill files. Update the `[ ]` → `[x]` checkbox as files are completed.

---

## Metadata Files (6/6 COMPLETE ✓)

- [x] `SKILL.md` — Skill entry point
- [x] `AGENTS.md` — Agent routing table
- [x] `CLAUDE.md` — Claude Code guidance
- [x] `SKILL_GENERATION_PROMPT.md` — Content generation specification
- [x] `skills-index.md` — This file (progress tracker)
- [x] `README.md` — Quick start guide

---

## P0 — Core DBA Topics (26/26 COMPLETE ✓)

### admin/ (6/6 COMPLETE)

- [x] `skills/admin/backup-recovery.md` — 物理备份恢复概述与策略
- [x] `skills/admin/log-archive.md` — 归档日志管理
- [x] `skills/admin/data-backup.md` — 数据备份操作
- [x] `skills/admin/physical-restore.md` — 物理恢复与表级恢复
- [x] `skills/admin/user-management.md` — 用户与权限管理基础
- [x] `skills/admin/tenant-management.md` — 租户创建/扩缩容/克隆/资源池管理

### architecture/ (7/7 COMPLETE)

- [x] `skills/architecture/distributed-architecture.md` — 分布式架构概述
- [x] `skills/architecture/multi-tenant.md` — 多租户架构
- [x] `skills/architecture/obproxy.md` — OBProxy代理架构与配置
- [x] `skills/architecture/paxos-replicas.md` — Paxos协议与副本管理
- [x] `skills/architecture/physical-standby.md` — 物理备库与容灾架构
- [x] `skills/architecture/shared-storage.md` — 共享存储架构（V4.x新特性）
- [x] `skills/architecture/resource-isolation.md` — 资源隔离（资源单元/资源池/cgroup）

### performance/ (7/7 COMPLETE)

- [x] `skills/performance/explain-plan.md` — EXPLAIN执行计划分析
- [x] `skills/performance/index-strategy.md` — 索引策略与优化
- [x] `skills/performance/optimizer-stats.md` — 统计信息收集与管理
- [x] `skills/performance/slow-query-analysis.md` — 慢SQL定位与分析
- [x] `skills/performance/wait-events.md` — 等待事件诊断
- [x] `skills/performance/memory-tuning.md` — 内存调优
- [x] `skills/performance/plan-cache-management.md` — Plan Cache管理与优化

### monitoring/ (6/6 COMPLETE)

- [x] `skills/monitoring/alert-log-analysis.md` — 告警日志分析
- [x] `skills/monitoring/health-monitor.md` — 健康检查与巡检
- [x] `skills/monitoring/space-management.md` — 空间管理与容量规划
- [x] `skills/monitoring/top-sql-queries.md` — Top SQL查询与SQL审计
- [x] `skills/monitoring/obdiag-tool.md` — obdiag诊断工具使用
- [x] `skills/monitoring/daily-inspection.md` — 日常巡检脚本与自动化

---

## P1 — Development & Migration (28/28 COMPLETE ✓)

### appdev/ (11/11 COMPLETE)

- [x] `skills/appdev/jdbc-connection.md` — Java JDBC连接（原java-oceanbase）
- [x] `skills/appdev/python-connection.md` — Python连接开发
- [x] `skills/appdev/go-connection.md` — Go连接开发
- [x] `skills/appdev/spring-boot-integration.md` — Spring Boot集成
- [x] `skills/appdev/mybatis-integration.md` — MyBatis集成
- [x] `skills/appdev/hibernate-integration.md` — Hibernate集成
- [x] `skills/appdev/cpp-connection.md` — C++连接开发
- [x] `skills/appdev/connection-pool-tuning.md` — 连接池调优
- [x] `skills/appdev/transaction-management-app.md` — 应用事务管理
- [x] `skills/appdev/batch-processing.md` — 批量处理
- [x] `skills/appdev/error-handling-app.md` — 应用错误处理

### migrations/ (9/9 COMPLETE)

- [x] `skills/migrations/preparation-and-assessment.md` — 迁移评估与准备
- [x] `skills/migrations/mysql-to-ob.md` — MySQL → OceanBase 迁移
- [x] `skills/migrations/oracle-to-ob.md` — Oracle → OceanBase 迁移
- [x] `skills/migrations/postgresql-to-ob.md` — PostgreSQL → OceanBase 迁移
- [x] `skills/migrations/db2-to-ob.md` — DB2 → OceanBase 迁移
- [x] `skills/migrations/sqlserver-to-ob.md` — SQL Server → OceanBase 迁移
- [x] `skills/migrations/data-type-mapping.md` — 跨库数据类型映射
- [x] `skills/migrations/oms-single-table-migration.md` — OMS单表迁移
- [x] `skills/migrations/verification-and-cutover.md` — 迁移验证与割接

### sql-dev/ (5/5 COMPLETE)

- [x] `skills/sql-dev/sql-coding-standards.md` — SQL编码规范
- [x] `skills/sql-dev/dml-best-practices.md` — DML最佳实践
- [x] `skills/sql-dev/ddl-best-practices.md` — DDL最佳实践
- [x] `skills/sql-dev/query-rewrite.md` — 查询改写
- [x] `skills/sql-dev/pagination.md` — 分页技术

### plsql/ (3/3 COMPLETE)

- [x] `skills/plsql/stored-procedures.md` — 存储过程与函数
- [x] `skills/plsql/triggers.md` — 触发器
- [x] `skills/plsql/packages.md` — PL包

---

## P2 — Security, Design, DevOps, Features (16/16 COMPLETE ✓)

### security/ (5/5 COMPLETE)

- [x] `skills/security/privilege-management.md` — 权限管理（MySQL/Oracle双模式）
- [x] `skills/security/row-level-security.md` — 行级访问控制
- [x] `skills/security/encryption.md` — 数据加密（TDE/传输加密/列加密）
- [x] `skills/security/auditing.md` — 安全审计
- [x] `skills/security/network-security.md` — 网络安全与白名单

### design/ (3/3 COMPLETE)

- [x] `skills/design/data-modeling.md` — 数据建模与表设计
- [x] `skills/design/partitioning-strategy.md` — 分区策略
- [x] `skills/design/tablegroup-design.md` — 表组设计

### devops/ (3/3 COMPLETE)

- [x] `skills/devops/obd-deployment.md` — OBD部署与集群管理
- [x] `skills/devops/schema-migrations.md` — Schema版本管理
- [x] `skills/devops/version-upgrade.md` — 版本升级策略与操作

### features/ (5/5 COMPLETE)

- [x] `skills/features/materialized-views.md` — 物化视图
- [x] `skills/features/database-links.md` — DBLink跨库/跨租户查询
- [x] `skills/features/table-level-restore.md` — 表级恢复
- [x] `skills/features/flashback-query.md` — 闪回查询
- [x] `skills/features/columnstore-replica.md` — 列存副本（HTAP分析加速）

---

## P3 — Ecosystem Tools & AI (11/11 COMPLETE ✓)

### tools/ (8/8 COMPLETE)

- [x] `skills/tools/ocp-platform.md` — OCP云平台管理
- [x] `skills/tools/oms-migration.md` — OMS迁移服务
- [x] `skills/tools/odc-development.md` — ODC开发中心
- [x] `skills/tools/obdiag-diagnostics.md` — obdiag诊断工具详解
- [x] `skills/tools/obloader-obdumper.md` — obloader & obdumper 数据导入导出
- [x] `skills/tools/obd-deployer.md` — OBD部署器
- [x] `skills/tools/ob-operator.md` — ob-operator Kubernetes部署
- [x] `skills/tools/oblogproxy.md` — oblogproxy Binlog服务

### ai/ (3/3 COMPLETE)

- [x] `skills/ai/vector-search.md` — 向量检索与向量索引
- [x] `skills/ai/ai-function-service.md` — AI函数服务
- [x] `skills/ai/hybrid-search.md` — 混合检索

---

## Progress Summary

| Category | Total | Completed | Remaining |
|----------|-------|------------|------------|
| Metadata | 6 | 6 | 0 |
| P0: admin/ | 6 | 6 | 0 |
| P0: architecture/ | 7 | 7 | 0 |
| P0: performance/ | 7 | 7 | 0 |
| P0: monitoring/ | 6 | 6 | 0 |
| P1: appdev/ | 11 | 11 | 0 |
| P1: migrations/ | 9 | 9 | 0 |
| P1: sql-dev/ | 5 | 5 | 0 |
| P1: plsql/ | 3 | 3 | 0 |
| P2: security/ | 5 | 5 | 0 |
| P2: design/ | 3 | 3 | 0 |
| P2: devops/ | 3 | 3 | 0 |
| P2: features/ | 5 | 5 | 0 |
| P3: tools/ | 8 | 8 | 0 |
| P3: ai/ | 3 | 3 | 0 |
| **Total** | **96** | **96** | **0** |

---

## Sources (Official Documentation)

- English: https://en.oceanbase.com/docs/oceanbase-database
- Chinese: https://www.oceanbase.com/docs/oceanbase-database-cn
- OCP: https://en.oceanbase.com/docs/ocp
- OMS: https://en.oceanbase.com/docs/oms
- ODC: https://en.oceanbase.com/docs/odc
- obdiag: https://en.oceanbase.com/docs/obdiag
