# OMS Single Table Migration

## Overview

OceanBase Migration Service (OMS) supports single-table migration for granular control. This guide covers OMS single-table migration steps and best practices for V4.2+.

---

## When to Use Single-Table Migration

| Use Case | Reason |
|----------|--------|
| Large table (> 1TB) | Migrate in chunks |
| Sensitive table | Migrate during maintenance window |
| Dependent table | Migrate after parent table |
| Testing | Migrate one table for validation |

---

## OMS Single-Table Migration Steps

### Step1: Create OMS Project

1. Log in to OMS Console
2. **Create Project** → Select source (MySQL/Oracle/PostgreSQL/DB2/SQL Server)
3. **Select Target** → OceanBase (MySQL Mode or Oracle Mode)
4. **Select Tables** → Choose specific table (e.g., `employees`)

### Step2: Configure Migration

```
Source: MySQL 5.7 (10.0.0.1:3306)
      ↓
    OMS (incremental replication)
      ↓
Target: OceanBase 4.2 (10.0.0.100:2881)
```

| Setting | Recommended | Description |
|---------|-------------|-------------|
| **Migration Type** | Full + Incremental | Minimal downtime |
| **Throttle** | 50 MB/s | Avoid source overload |
| **Batch Size** | 1000 rows | Balance speed vs. load |
| **Retry Attempts** | 3 | Handle transient errors |

### Step3: Start Full Migration

```bash
# Monitor full migration progress (OMS Console)
# - Full migration must complete before incremental starts
```

> **Tip**: Full migration time = table_size / throttle.

### Step4: Start Incremental Replication

```
Full Migration (baseline) → Incremental Replication (catch-up)
```

> **Key**: Incremental replication keeps target in sync with source (real-time).

### Step5: Verify Data

```sql
-- Compare row counts
SELECT COUNT(*) FROM employees;

-- Compare data samples
SELECT * FROM employees ORDER BY emp_id LIMIT 10;
```

### Step6: Cutover (Switch App to OceanBase)

1. **Stop writes to source** (maintenance window)
2. **Wait for incremental replication to catch up** (lag < 5s)
3. **Switch app JDBC URL to OceanBase**
4. **Verify app works with OceanBase**

---

## Monitoring OMS

### OMS Console Metrics

| Metric | Description |
|--------|-------------|
| **Full Migration Progress** | % of data migrated |
| **Incremental Lag** | Seconds behind source |
| **Throughput** | Rows/s or MB/s |
| **Errors** | Failed rows (with reasons) |

> **Alert**: Lag > 60s = investigate (network or target overload).

---

## Troubleshooting

| Issue | Fix |
|-------|-----|
| Full migration slow | Increase throttle (if network allows) |
| Incremental lag > 60s | Check network; increase OBServer resources |
| Data mismatch | Check OMS error log; re-migrate table |
| DDL not replicated | OMS supports DDL replication (enable in settings) |

---

## Best Practices

1. **Migrate large tables first** — they take longest.
2. **Test cutover 3 times in staging** — avoid surprises.
3. **Monitor incremental lag** — alert if > 60s.
4. **Migrate during low-traffic window** — minimize impact.
5. **Have rollback plan** — reverse replication (OceanBase → source) if needed.

---

## Common Mistakes

1. **Not testing cutover**
   - ❌ First cutover is production
   - ✅ Test in staging 3 times

2. **Migrating all tables at once (large dataset)**
   - ❌ OMS overload
   - ✅ Migrate large tables individually

3. **Not monitoring incremental lag**
   - ❌ Lag grows to hours
   - ✅ Alert if lag > 60s

4. **Forgetting to update app JDBC URL**
   - ❌ App still connects to source after cutover
   - ✅ Update app config to OceanBase

5. **No rollback plan**
   - ❌ Can't revert if migration fails
   - ✅ Configure reverse replication (OceanBase → source)

---

## OceanBase Version Notes

### V4.2 (Baseline)
- OMS supports single-table migration
- Full + incremental replication
- DDL replication

### V4.4+
- Faster incremental replication
- Improved throughput (parallel apply)

---

## Sources
- https://en.oceanbase.com/docs/oms/single-table-migration
- https://www.oceanbase.com/docs/oms-cn/single-table-migration
- https://en.oceanbase.com/docs/oms/monitoring
