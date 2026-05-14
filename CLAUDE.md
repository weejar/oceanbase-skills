# CLAUDE.md — Guidance for Claude Code (and other AI coding agents)

## Skill Set Overview

This directory contains 96 reference guides for OceanBase Database. Each file is a self-contained reference on one specific topic. Load only the file(s) relevant to the user's question.

## How to Use These Skills

1. **Identify the category** from the user's question using `AGENTS.md` routing table
2. **Read only the needed file(s)** — do not load entire directories
3. **Apply the content** to generate SQL, code, configurations, or explanations
4. **Check dual-mode notes** — many topics differ between MySQL Mode and Oracle Mode

## OceanBase Version Baseline

- **Baseline**: OceanBase V4.2 (both MySQL Mode and Oracle Mode)
- **V4.4 features**: marked with `> **V4.4+**: ...` callout blocks
- **V4.5 features**: marked with `> **V4.5+**: ...` callout blocks
- If the user does not specify a version, assume V4.2 and mention V4.4/V4.5 features as optional upgrades

## Dual Compatibility Modes

OceanBase supports two compatibility modes that affect SQL syntax, data types, PL/SQL, and system views:

| Aspect | MySQL Mode | Oracle Mode |
|--------|-----------|-------------|
| SQL dialect | MySQL-compatible | Oracle-compatible |
| PL/SQL | Not supported | Supported |
| Date type | DATE (no time) | DATE (with time) |
| Dual table | Not needed | `SELECT 1 FROM dual` |
| Empty string | `=''` is true | `=''` is NULL |
| System views | `information_schema` | `DBA_*`, `USER_*`, `ALL_*` |

When generating code or SQL, **always clarify which mode the user is using** if it's not obvious from context.

## Key Concepts Unique to OceanBase

- **Tenant**: Isolation unit (like a database instance). Each tenant has its own resource units.
- **Resource Unit (Unit)**: Hardware resource definition (CPU, memory, disk). Assigned to zones.
- **Resource Pool**: Collection of resource units distributed across zones.
- **Zone**: Failure domain (like a data center rack). Replicas are distributed across zones.
- **Paxos**: Consensus protocol for log replication and leader election.
- **Log Stream**: Replication pipeline for a partition's redo logs.
- **Tablegroup**: Groups tables so that partitions with the same mapping end up on the same server (enables local joins).
- **LS (Log Stream)**: The unit of log replication in OceanBase's distributed architecture.

## File Format

Each skill file follows this structure:

```markdown
# [Topic Title]

## Overview
[What this covers and why it matters]

## [Section 1]
[Explanation, code examples, best practices]

## [Section 2]
[...]

## MySQL Mode vs Oracle Mode
[Differences when applicable]

## Common Mistakes
[Pitfalls to avoid]

## OceanBase Version Notes
[V4.2 baseline, V4.4/V4.5 new features]

## Sources
- [Official doc links]
```

## When Answering User Questions

1. Read the relevant skill file(s) from the list above
2. If the question involves SQL generation, check the MySQL/Oracle mode context
3. Include actual runnable SQL examples from the skill file
4. Reference version-specific features with clear version labels
5. Point the user to the official OceanBase docs for the latest details

## Important Differences from Oracle

- OceanBase has **no tablespaces** — use tenant resource management instead
- OceanBase has **no UNDO tablespace** — uses multi-version consistency (MVCC)
- OceanBase has **no RMAN** — uses built-in `ALTER SYSTEM BACKUP` commands
- OceanBase PL/SQL is **a subset** of Oracle PL/SQL — check `skills/plsql/` for supported features
- OceanBase supports **HTAP** natively — row store for TP, columnar replica for AP

## Official Documentation Links

- English docs: https://en.oceanbase.com/docs/oceanbase-database
- Chinese docs: https://www.oceanbase.com/docs/oceanbase-database-cn
- OCP docs: https://en.oceanbase.com/docs/ocp
- OMS docs: https://en.oceanbase.com/docs/oms
- ODC docs: https://en.oceanbase.com/docs/odc
- obdiag docs: https://en.oceanbase.com/docs/obdiag
