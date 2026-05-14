# Plan Cache Management

## Overview

OceanBase caches execution plans in **Plan Cache** to avoid re-parsing SQL. Managing plan cache effectively improves OLTP performance. This guide covers plan cache management in OceanBase V4.2+.

---

## Plan Cache Architecture

```
SQL Request
    ↓
Plan Cache Lookup (by SQL ID)
    ↓ (hit) → Use cached plan → Execute
    ↓ (miss) → Parse → Optimize → Cache plan → Execute
```

> **Benefit**: Plan cache hit = skip parse + optimize (faster response).

---

## Viewing Plan Cache

### MySQL Mode

```sql
-- Plan cache statistics
SELECT * FROM oceanbase.DBA_OB_PLAN_CACHE_STAT;

-- Plan cache content (actual plans)
SELECT * FROM oceanbase.DBA_OB_PLAN_CACHE_PLAN_EXPLAIN
WHERE query = 'SELECT * FROM employees WHERE dept_id = ?';
```

### Oracle Mode

```sql
-- Same data, different view names
SELECT * FROM sys.DBA_OB_PLAN_CACHE_STAT;
SELECT * FROM sys.DBA_OB_PLAN_CACHE_PLAN_EXPLAIN;
```

---

## Plan Cache Key Parameters

| Parameter | Default | Recommended | Description |
|-----------|---------|-------------|-------------|
| `plan_cache_evict_interval` | 5s | 5s | How often to evict old plans |
| `plan_cache_high_water_mark` | 90% | 90% | Evict when cache is 90% full |
| `ob_enable_plan_cache` | TRUE | TRUE | Enable/disable plan cache |

```sql
-- Check settings
SHOW PARAMETERS LIKE '%plan_cache%';
```

---

## Flushing Plan Cache

After adding indexes or gathering stats, **flush plan cache** so new plans are generated.

```sql
-- Flush plan cache for entire cluster
ALTER SYSTEM FLUSH PLAN CACHE;

-- Flush plan cache for specific tenant
ALTER SYSTEM FLUSH PLAN CACHE TENANT = 'app_tenant';
```

> **Important**: Plan cache is per-tenant. Flushing affects only that tenant.

---

## Bindging a Fixed Plan (Outline)

Force OceanBase to use a specific plan (avoid bad plan choice).

```sql
-- Create outline (Oracle Mode)
CREATE OUTLINE outline_emp_dept
  ON SELECT * FROM employees WHERE dept_id = 10
  USING INDEX idx_emp_dept;

-- Enable outline
ALTER SYSTEM SET enable_outline = 'TRUE';
```

> **Use Case**: Optimizer chooses full scan but index exists.

---

## Plan Cache Sizing

If plan cache hit rate is low (< 80%), increase cache size:

```sql
-- Increase plan cache memory (per tenant)
ALTER SYSTEM SET plan_cache_percentage = 10;  -- was 5
```

Check hit rate:

```sql
SELECT hit_rate
FROM oceanbase.DBA_OB_PLAN_CACHE_STAT
WHERE tenant_id = (SELECT tenant_id FROM oceanbase.DBA_OB_TENANTS WHERE tenant_name='app_tenant');
```

> **Target**: Plan cache hit rate > 90%.

---

## MySQL Mode vs Oracle Mode

Plan cache works the same in both modes. Only system view names differ.

| Aspect | MySQL Mode | Oracle Mode |
|--------|-----------|-------------|
| Plan cache view | `oceanbase.DBA_OB_PLAN_CACHE_STAT` | `sys.DBA_OB_PLAN_CACHE_STAT` |
| Outline | Not supported | `CREATE OUTLINE` |

---

## Common Mistakes

1. **Not flushing plan cache after adding index**
   - ❌ Old slow plan still used
   - ✅ `ALTER SYSTEM FLUSH PLAN CACHE;` after index creation

2. **Plan cache too small (low hit rate)**
   - ❌ Hit rate < 50%
   - ✅ Increase `plan_cache_percentage`

3. **Outlines not enabled (Oracle Mode)**
   - ❌ Created outline but `enable_outline = FALSE`
   - ✅ `ALTER SYSTEM SET enable_outline = 'TRUE';`

4. **Flushing plan cache during peak hours**
   - ❌ All SQL re-parses simultaneously (CPU spike)
   - ✅ Flush during maintenance window

5. **Using literals instead of bind variables**
   - ❌ `WHERE dept_id = 10`, `WHERE dept_id = 20` (different plans)
   - ✅ `WHERE dept_id = ?` (one plan cached, reused)

---

## OceanBase Version Notes

### V4.2 (Baseline)
- Plan cache per tenant
- Plan cache statistics views
- Outline support (Oracle Mode)

### V4.4+
- Faster plan cache lookup
- Improved hit rate tracking

---

## Sources
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/plan-cache
- https://www.oceanbase.com/docs/oceanbase-database-cn/v4.2.5/plan-cache
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/outline
