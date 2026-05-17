# AGENTS.md — OceanBase DB Skills Routing Table

This file helps AI agents route user questions to the correct skill file.

## Routing Rules

When the user asks about any of the topics below, read the corresponding file(s) before answering.

---

## admin/ — Database Administration

| User asks about… | Read this file |
|------------------|---------------|
| Backup strategy, backup configuration, backup jobs | `skills/admin/backup-recovery.md` |
| Archive log, log archive, archive settings | `skills/admin/log-archive.md` |
| Data backup, full backup, incremental backup | `skills/admin/data-backup.md` |
| Physical restore, point-in-time recovery, table-level restore | `skills/admin/physical-restore.md` |
| User creation, password policy, roles | `skills/admin/user-management.md` |
| Tenant creation, tenant scaling, resource pool, resource unit | `skills/admin/tenant-management.md` |
| Manage OceanBase cluster lifecycle using obd | `skills/admin/cluster-management.md` |
| SeekDB product overview, AI hybrid search | `skills/admin/seekdb.md` |



---

## appdev/ — Application Development

| User asks about… | Read this file |
|------------------|---------------|
| Connection pool, HikariCP, Druid, Tomcat JDBC | `skills/appdev/connection-pooling.md` |
| Java development, JDBC, Spring Boot, MyBatis | `skills/appdev/java-oceanbase.md` |
| Python development, PyMySQL, mysqlclient, SQLAlchemy | `skills/appdev/python-oceanbase.md` |
| Go development, go-sql-driver, GORM | `skills/appdev/golang-oceanbase.md` |
| Node.js development, mysql2, Sequelize | `skills/appdev/nodejs-oceanbase.md` |
| .NET development, MySqlConnector, Entity Framework | `skills/appdev/dotnet-oceanbase.md` |
| JSON data type, JSON functions, JSON indexing | `skills/appdev/json-in-oceanbase.md` |
| Transaction isolation levels, read consistency | `skills/appdev/transaction-management.md` |
| Locking, FOR UPDATE, lock wait timeout | `skills/appdev/locking-concurrency.md` |
| Sequences, AUTO_INCREMENT, identity columns | `skills/appdev/sequences-identity.md` |
| MySQL Mode vs Oracle Mode development differences | `skills/appdev/compatibility-modes.md` |

---

## architecture/ — Distributed Architecture

| User asks about… | Read this file |
|------------------|---------------|
| Distributed architecture, Paxos, partition, log stream | `skills/architecture/distributed-architecture.md` |
| Multi-tenant, sys tenant, user tenant, meta tenant | `skills/architecture/multi-tenant.md` |
| OBProxy, reverse proxy, routing, connection mapping | `skills/architecture/obproxy.md` |
| Paxos protocol, replica management, leader/follower | `skills/architecture/paxos-replicas.md` |
| Physical standby, disaster recovery, primary-standby | `skills/architecture/physical-standby.md` |
| Shared storage, NFS, cloud storage integration | `skills/architecture/shared-storage.md` |
| Resource isolation, cgroup, resource unit, resource pool | `skills/architecture/resource-isolation.md` |

---

## design/ — Schema Design

| User asks about… | Read this file |
|------------------|---------------|
| Data modeling, table design, normalization | `skills/design/data-modeling.md` |
| Partitioning, Hash/Range/List partitions, composite partitioning | `skills/design/partitioning-strategy.md` |
| Tablegroup, co-location, local join optimization | `skills/design/tablegroup-design.md` |

---

## devops/ — DevOps

| User asks about… | Read this file |
|------------------|---------------|
| OBD deployment, cluster configuration, OBD commands | `skills/devops/obd-deployment.md` |
| Schema migration, Liquibase, Flyway, version control | `skills/devops/schema-migrations.md` |
| Version upgrade, rolling upgrade, upgrade checklist | `skills/devops/version-upgrade.md` |

---

## features/ — OceanBase Features

| User asks about… | Read this file |
|------------------|---------------|
| Materialized view, query rewriting, refresh strategy | `skills/features/materialized-views.md` |
| DBLink, cross-database query, cross-tenant query | `skills/features/database-links.md` |
| Table-level restore, flashback table | `skills/features/table-level-restore.md` |
| Flashback query, AS OF TIMESTAMP | `skills/features/flashback-query.md` |
| Columnar replica, HTAP, analytical acceleration | `skills/features/columnstore-replica.md` |

---

## migrations/ — Database Migrations

| User asks about… | Read this file |
|------------------|---------------|
| Migration assessment, method selection, pre-migration checklist | `skills/migrations/migration-assessment.md` |
| MySQL to OceanBase migration | `skills/migrations/migrate-mysql-to-oceanbase.md` |
| Oracle to OceanBase migration | `skills/migrations/migrate-oracle-to-oceanbase.md` |
| PostgreSQL to OceanBase migration | `skills/migrations/migrate-postgres-to-oceanbase.md` |
| DB2 to OceanBase migration | `skills/migrations/migrate-db2-to-oceanbase.md` |
| TiDB to OceanBase migration | `skills/migrations/migrate-tidb-to-oceanbase.md` |
| OceanBase to MySQL migration | `skills/migrations/migrate-ob-to-mysql.md` |
| OceanBase to Oracle migration | `skills/migrations/migrate-ob-to-oracle.md` |
| Data validation after migration | `skills/migrations/migration-data-validation.md` |

---

## monitoring/ — Monitoring & Diagnostics

| User asks about… | Read this file |
|------------------|---------------|
| Alert log, observer.log, rs.log, log analysis | `skills/monitoring/alert-log-analysis.md` |
| Health check, daily inspection, automation scripts | `skills/monitoring/health-monitor.md` |
| Space management, capacity planning, data compression | `skills/monitoring/space-management.md` |
| Top SQL, SQL auditing, v$sql_audit | `skills/monitoring/top-sql-queries.md` |
| obdiag, diagnostics, root cause analysis | `skills/monitoring/obdiag-tool.md` |
| Daily inspection scripts, automated巡检 | `skills/monitoring/daily-inspection.md` |

---

## performance/ — Performance Tuning

| User asks about… | Read this file |
|------------------|---------------|
| EXPLAIN plan, execution plan analysis, reading plans | `skills/performance/explain-plan.md` |
| Index strategy, index selection, covering index | `skills/performance/index-strategy.md` |
| Optimizer statistics, gather_table_stats, auto stats | `skills/performance/optimizer-stats.md` |
| Slow query analysis, identifying slow SQL | `skills/performance/slow-query-analysis.md` |
| Wait events, v$system_event, performance profiling | `skills/performance/wait-events.md` |
| Memory tuning, memory parameters, memstore management | `skills/performance/memory-tuning.md` |
| Plan cache, SQL plan management, outline binding | `skills/performance/plan-cache-management.md` |

---

## security/ — Security

| User asks about… | Read this file |
|------------------|---------------|
| Privilege management, GRANT/REVOKE, system privileges | `skills/security/privilege-management.md` |
| Row-level security, VPD, fine-grained access | `skills/security/row-level-security.md` |
| TDE encryption, transparent data encryption, AES | `skills/security/encryption.md` |
| Auditing, audit logs, compliance | `skills/security/auditing.md` |
| Network security, whitelist, SSL/TLS, firewall | `skills/security/network-security.md` |

---

## sql-dev/ — SQL Development

| User asks about… | Read this file |
|------------------|---------------|
| SQL best practices, writing efficient SQL | `skills/sql-dev/sql-best-practices.md` |
| SQL tuning, rewriting queries, performance | `skills/sql-dev/sql-tuning.md` |
| Hints, Outline, plan binding | `skills/sql-dev/hints-and-outlines.md` |
| Dynamic SQL, EXECUTE IMMEDIATE (Oracle Mode) | `skills/sql-dev/dynamic-sql.md` |
| SQL injection prevention, parameterized queries | `skills/sql-dev/sql-injection-avoidance.md` |

---

## plsql/ — PL/SQL (Oracle Mode Only)

| User asks about… | Read this file |
|------------------|---------------|
| PL/SQL basics, block structure, Oracle Mode compatibility | `skills/plsql/plsql-basics.md` |
| Stored procedures, functions, CREATE PROCEDURE | `skills/plsql/stored-procedures.md` |
| Package design, package specification, package body | `skills/plsql/plsql-package-design.md` |

---

## tools/ — OceanBase Ecosystem Tools

| User asks about… | Read this file |
|------------------|---------------|
| OCP, OceanBase Cloud Platform, web management | `skills/tools/ocp-platform.md` |
| OMS, OceanBase Migration Service, data migration | `skills/tools/oms-migration.md` |
| ODC, OceanBase Developer Center, SQL development IDE | `skills/tools/odc-development.md` |
| obdiag, diagnostics, log collection, root cause analysis | `skills/tools/obdiag-diagnostics.md` |
| obloader, obdumper, data import/export | `skills/tools/obloader-obdumper.md` |
| OBD, OceanBase Deployer, cluster deployment | `skills/tools/obd-deployer.md` |
| ob-operator, Kubernetes operator, K8s deployment | `skills/tools/ob-operator.md` |
| oblogproxy, Binlog service, CDC, data replication | `skills/tools/oblogproxy.md` |
| Sysbench, TPC-H, TPC-C, mysqltest, obd test | `skills/tools/testing-and-benchmark.md` |
| SeekDB obd commands, HA, switchover, failover | `skills/tools/seekdb-obd-operations.md` |
| OBD config file deployment, YAML configuration | `skills/tools/obd-config-deployment.md` |
| Prometheus, Grafana, OBAgent monitoring setup | `skills/tools/obd-monitoring-setup.md` |
| obd tenant commands, tenant backup/restore | `skills/tools/tenant-obd-management.md` |
| Sys tenant connection, tenant listing, tenant resource query | `skills/tools/sys-tenant-connection.md` |

---

## ai/ — AI & Vector Features

| User asks about… | Read this file |
|------------------|---------------|
| Vector search, vector indexing, HNSW, IVFFlat | `skills/ai/vector-search.md` |
| AI function service, embedding generation, LLM integration | `skills/ai/ai-function-service.md` |
| Hybrid search, vector + full-text, re-ranking | `skills/ai/hybrid-search.md` |

---

## File Count Summary

| Category | File Count |
|----------|-----------|
| admin/ | 9 |
| appdev/ | 11 |
| architecture/ | 7 |
| design/ | 3 |
| devops/ | 3 |
| features/ | 5 |
| migrations/ | 9 |
| monitoring/ | 6 |
| performance/ | 7 |
| security/ | 5 |
| sql-dev/ | 5 |
| plsql/ | 3 |
| tools/ | 14 |
| ai/ | 3 |
| **Total** | **105** |
