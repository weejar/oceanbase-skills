# Migration Verification and Cutover

## Overview

Verification ensures data integrity after migration. Cutover is the final step to switch production to OceanBase. This guide covers verification steps and cutover best practices for V4.2+.

---

## Verification Checklist

| # | Item | Command/Action |
|---|------|-----------------|
| 1 | Row count match | `SELECT COUNT(*) FROM table` (source vs target) |
| 2 | Data sample match | `SELECT * FROM table ORDER BY pk LIMIT 10` |
| 3 | Index exists | `SHOW INDEX FROM table` |
| 4 | Stored proc works | `CALL proc_name(...)` |
| 5 | Trigger works | Test INSERT/UPDATE that fires trigger |
| 6 | View works | `SELECT * FROM view_name LIMIT 10` |
| 7 | App connects | Run app with OceanBase JDBC URL |
| 8 | Performance baseline | Compare query response time (source vs target) |

> **Rule**: Verify all 8 items before cutover.

---

## Data Verification (Sample Comparison)

```sql
-- Compare row counts
SELECT 'employees', COUNT(*) FROM employees
UNION ALL
SELECT 'departments', COUNT(*) FROM departments;

-- Compare data samples (first 10 rows)
SELECT * FROM employees ORDER BY emp_id LIMIT 10;

-- Compare checksums (if OMS provides)
-- OMS generates checksum report automatically
```

---

## Incremental Replication Lag Check

```sql
-- OceanBase: check replication lag (if source is MySQL/Oracle)
SELECT source_table,
       lag_seconds
FROM oceanbase.DBA_OB_REPLICATION_STATUS;
```

> **Requirement for cutover**: Lag < 5 seconds.

---

## Cutover Steps (Planned)

### Step1: Stop Writes to Source

```
1. Enable maintenance page (if applicable)
2. Stop application writes to source database
3. Wait for in-flight transactions to complete
```

### Step2: Final Incremental Catch-Up

```
1. OMS incremental replication catches up (lag < 5s)
2. Verify lag = 0
```

### Step3: Switch Application to OceanBase

```
1. Update application JDBC URL to OceanBase (OBProxy)
2. Restart application (or reload config)
3. Verify app connects to OceanBase
```

### Step4: Verify Application Works

```
1. Run smoke tests (key transactions)
2. Check application logs for errors
3. Monitor OceanBase performance (CPU, I/O)
```

### Step5: Decommission Source (After 7 Days)

```
1. Keep source for 7 days (rollback capability)
2. Monitor OceanBase stability
3. Decommission source after 7 days
```

---

## Rollback Plan

| Scenario | Rollback Action |
|----------|-----------------|
| App fails with OceanBase | Switch JDBC URL back to source |
| Data corruption detected | Restore from backup |
| Performance degradation | Scale OceanBase resources |

> **Tip**: Keep source database running for 7 days after cutover (rollback capability).

---

## Cutover Communication Template

```
To: All stakeholders
Subject: Database Migration Cutover - [Date]

Migration window: [start_time] to [end_time]
Expected downtime: [X minutes]

Steps:
1. Stop writes to source (T0)
2. Final incremental catch-up (T0+5min)
3. Switch app to OceanBase (T0+10min)
4. Verify app works (T0+20min)
5. Migration complete (T0+30min)

Rollback plan: Switch app back to source (5min)

Status updates: [Slack channel / email]
```

---

## Common Mistakes

1. **Not testing cutover in staging**
   - ❌ First cutover is production
   - ✅ Test cutover in staging 3 times

2. **Cutting over with lag > 60s**
   - ❌ Data loss (incremental not caught up)
   - ✅ Only cutover when lag < 5s

3. **Not having rollback plan**
   - ❌ Can't revert if OceanBase fails
   - ✅ Document rollback steps; test rollback

4. **Forgetting to update app JDBC URL**
   - ❌ App still connects to source after cutover
   - ✅ Update all app configs to OceanBase

5. **Not monitoring after cutover**
   - ❌ Issues discovered hours later
   - ✅ Monitor OceanBase closely for 48 hours

---

## Post-Cutover Checklist

| # | Item | Action |
|---|------|--------|
| 1 | Monitor performance | Check CPU, I/O, memory |
| 2 | Monitor errors | Check application logs |
| 3 | Monitor replication | If reverse replication configured |
| 4 | Validate backups | Ensure OceanBase backup is running |
| 5 | Update documentation | Record migration completion |

---

## OceanBase Version Notes

### V4.2 (Baseline)
- OMS incremental replication
- Cutover with minimal downtime
- Reverse replication (for rollback)

### V4.4+
- Faster incremental catch-up
- Improved cutover automation

---

## Sources
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/migration-verification
- https://www.oceanbase.com/docs/oceanbase-database-cn/v4.2.5/migration-verification
- https://en.oceanbase.com/docs/oms/cutover
