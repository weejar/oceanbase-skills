---
name: oceanbase-db-skills
description: 96 OceanBase database reference guides covering SQL, PL/SQL, performance tuning, security, migrations, multi-tenant architecture, distributed O&M, OceanBase tools ecosystem, and AI vector features. Load individual skill files on demand for expert guidance on any OceanBase topic.
---

# OceanBase DB Skills

A collection of 96 standalone reference guides for OceanBase Database (V4.2/V4.4/V4.5). Each file covers one topic with explanations, practical examples, best practices, and common mistakes. Covers both MySQL Mode and Oracle Mode compatibility.

## How to Use

1. **Find the right skill** using the category routing table below.
2. **Read only the file(s)** relevant to the user's task — do not load all files at once.
3. **Apply the guidance** to answer questions, generate code, or review existing work.
4. **Note dual modes**: Many topics have MySQL Mode vs Oracle Mode differences — check the file's "Compatibility Notes" section.

## Category Routing

| User asks about… | Read from |
|------------------|-----------|
| Backup, recovery, tenant management, resource units, logs | `skills/admin/` |
| Java/Python/Go/Node.js/.NET dev, connection pooling, JSON, transactions | `skills/appdev/` |
| Distributed architecture, multi-tenant, Zone, OBProxy, Paxos, resource isolation | `skills/architecture/` |
| Data modeling, partitioning strategy, tablegroup design | `skills/design/` |
| OBD deployment, schema migration, version upgrade | `skills/devops/` |
| Materialized views, DBLinks, flashback, columnar replica | `skills/features/` |
| Migrating from MySQL/Oracle/PostgreSQL/DB2/TiDB to OceanBase | `skills/migrations/` |
| Alert log, health check, space management, Top SQL, obdiag | `skills/monitoring/` |
| EXPLAIN plan, indexes, optimizer stats, slow query, wait events, memory, plan cache | `skills/performance/` |
| Privileges, row-level security, TDE, auditing, network security | `skills/security/` |
| SQL patterns, SQL tuning, hints, dynamic SQL, injection prevention | `skills/sql-dev/` |
| PL/SQL development (Oracle Mode), stored procedures, packages | `skills/plsql/` |
| OCP, OMS, ODC, obdiag, obloader/obdumper, OBD, ob-operator, oblogproxy | `skills/tools/` |
| Vector search, AI function service, hybrid search | `skills/ai/` |
| SeekDB install, deploy, HA (switchover/failover/decouple). | skills/seekdb/ |
| Sysbench, TPC-H, TPC-C, mysqltest benchmarks. | [skills/testing-and-benchmark](testing-and-benchmark/SKILL.md) |

## Skills Directory

```
skills/
├── admin/          Database administration (backup, recovery, tenant, user, log archive)
├── appdev/         Application development (JSON, transactions, multi-language, compatibility modes)
├── architecture/   Distributed architecture (multi-tenant, Zone, OBProxy, Paxos, resource isolation)
├── design/         Schema design (data modeling, partitioning, tablegroup)
├── devops/         DevOps (OBD deployment, schema migration, version upgrade)
├── features/       OceanBase features (MVs, DBLinks, flashback, columnar replica)
├── migrations/     Migrating to/from OceanBase (MySQL, Oracle, PostgreSQL, DB2, TiDB)
├── monitoring/     Diagnostics (alert log, health check, space, Top SQL, obdiag)
├── performance/    Tuning (EXPLAIN, indexes, optimizer, wait events, memory, plan cache)
├── security/       Security (privileges, row-level security, TDE, auditing, network)
├── sql-dev/        SQL development (tuning, patterns, hints, dynamic SQL, injection)
├── plsql/          PL/SQL development (Oracle Mode only: procedures, packages)
├── tools/          OceanBase ecosystem tools (OCP, OMS, ODC, obdiag, OBD, ob-operator)
├── seekdb/         SeekDB install, deploy, HA (switchover/failover/decouple).
├── testing-and-benchmark/         Sysbench, TPC-H, TPC-C, mysqltest benchmarks.
└── ai/             AI features (vector search, AI function service, hybrid search)

```

## Key Starting Points

- **`skills/migrations/migration-assessment.md`** — start here for any database migration project
- **`skills/performance/explain-plan.md`** — foundation for all SQL performance work
- **`skills/architecture/multi-tenant.md`** — core OceanBase multi-tenant architecture
- **`skills/admin/tenant-management.md`** — tenant creation, scaling, resource management
- **`skills/admin/cluster-management.md`** — Manage OceanBase cluster lifecycle using obd. 
- **`skills/tools/obdiag-diagnostics.md`** — obdiag diagnostic tool usage
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
