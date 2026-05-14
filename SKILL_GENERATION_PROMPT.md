# SKILL_GENERATION_PROMPT.md — OceanBase DB Skills Content Specification

This file defines the standard format and quality requirements for every skill file in this collection. Use this as a reference when creating new skill files or updating existing ones.

---

## File Structure (Required)

Every skill file MUST follow this exact structure:

```markdown
# [Topic Title — clear, specific, action-oriented]

## Overview
[2-4 sentences: what this covers, why it matters, when to use it]
[Include version baseline: "Available since V4.2" or "V4.4+" as applicable]

## [Section 1: Core Concept]
[Explanation with runnable examples]
[MySQL Mode and Oracle Mode code examples where they differ]

## [Section 2: Practical Usage]
[Step-by-step instructions]
[SQL examples with actual OceanBase syntax]

## [Section 3: Best Practices]
[Dos and Don'ts]
[Common configuration values]

## MySQL Mode vs Oracle Mode
[Only include if there are meaningful differences]
[Use side-by-side code blocks]

## Common Mistakes
[3-5 specific pitfalls with examples of wrong vs correct approach]

## OceanBase Version Notes
### V4.2 (Baseline)
[Features available in V4.2]

### V4.4+
[New features / changed behavior]

### V4.5+ (if applicable)
[New features / changed behavior]

## Sources
- [Official OceanBase doc link — English or Chinese]
- [Secondary source if applicable]
```

---

## Content Quality Rules

### 1. SQL Examples Must Be Runnable
- All SQL examples must use valid OceanBase syntax
- Include both MySQL Mode and Oracle Mode where they differ
- Use realistic table/column names (e.g., `employees`, `dept_id`, not `foo`/`bar`)

### 2. Cover Dual Compatibility Modes
- If a feature behaves differently in MySQL Mode vs Oracle Mode, show both
- Label code blocks clearly: `sql (MySQL Mode)` or `sql (Oracle Mode)`
- If a feature only works in one mode, state it explicitly

### 3. Reference Real System Views
- Use actual OceanBase system views: `v$sql_audit`, `v$sysstat`, `v$parameter`, etc.
- Don't use Oracle view names that don't exist in OceanBase

### 4. Version Annotations
- Baseline content targets V4.2
- New features in V4.4 or V4.5 must be marked with `> **V4.4+**: ...` callout
- If a feature was changed between versions, explain the old vs new behavior

### 5. OceanBase-Specific Concepts
Always use correct OceanBase terminology:
- ✅ `tenant`, `resource unit`, `resource pool`, `zone`, `log stream`
- ❌ `tablespace`, `RMAN`, `UNDO tablespace`, `PDB`, `CDB`

---

## Style Rules

1. **Start with the most common use case** — don't lead with edge cases
2. **Use tables for comparisons** — parameters, versions, mode differences
3. **Keep code blocks short** — max 20 lines per block; break longer examples into steps
4. **Explain the "why"** — don't just show what to do, explain why it works that way
5. **Link to official docs** — every file must have at least one `Sources` link

---

## Generation Prompt Template

When asked to create a new skill file, use this prompt:

```
Create an OceanBase DB Skills file for the topic: [TOPIC NAME]

File path: skills/[category]/[filename].md

Requirements:
- Follow the file structure defined in SKILL_GENERATION_PROMPT.md
- Cover both MySQL Mode and Oracle Mode where applicable
- Use OceanBase V4.2 as baseline; annotate V4.4/V4.5 features
- Include runnable SQL examples with realistic table names
- Reference official OceanBase documentation
- Keep the file focused on one topic; 1500-3000 words is ideal
- Include a "Common Mistakes" section with 3-5 specific pitfalls

Official docs: https://en.oceanbase.com/docs/oceanbase-database
```

---

## Category-Specific Notes

### admin/
Focus on `ALTER SYSTEM` commands, tenant-level operations, and backup/restore procedures. OceanBase does NOT use RMAN.

### appdev/
Cover both MySQL Mode and Oracle Mode connection strings. OceanBase is wire-compatible with MySQL protocols.

### architecture/
Emphasize distributed nature: Zone, Paxos, log stream, partition distribution. This is the #1 concept that differentiates OceanBase from standalone databases.

### design/
OceanBase has NO tablespaces. Use tenant resource management for isolation. Tablegroups are an OceanBase-specific optimization.

### migrations/
Focus on OMS (OceanBase Migration Service) for data movement. Cover pre-migration assessment steps.

### performance/
System views: `v$sql_audit` (not `v$sql`), `v$plan_cache_stat`, `v$memory`. Execution plan format differs from Oracle.

### plsql/
OceanBase supports a SUBSET of Oracle PL/SQL. Check the Oracle Mode compatibility docs before claiming support for a specific feature.

### tools/
Cover the OceanBase ecosystem: OCP (management UI), OMS (migration), ODC (SQL IDE), obdiag (diagnostics). These are critical for production operations.

---

## Review Checklist

Before marking a skill file as complete:
- [ ] Follows the required file structure
- [ ] All SQL examples are syntactically valid for OceanBase
- [ ] Dual mode differences are covered (or explicitly noted as N/A)
- [ ] Version annotations are correct (V4.2 baseline, V4.4/V4.5 markings)
- [ ] "Common Mistakes" section has 3-5 items
- [ ] At least one official doc link in Sources
- [ ] No references to Oracle-only features (tablespace, RMAN, UNDO, etc.)
- [ ] File is in the correct `skills/[category]/` directory
