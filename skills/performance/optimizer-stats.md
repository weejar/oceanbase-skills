# Optimizer Statistics

## Overview

Optimizer statistics help OceanBase choose the best execution plan. Outdated statistics cause bad query plans. This guide covers gathering, viewing, and maintaining statistics in OceanBase V4.2+.

---

## Why Statistics Matter

```
Outdated stats → Wrong row count estimate → Bad join order → SLOW QUERY
```

> **Rule**: Gather stats after major data changes (INSERT/UPDATE/DELETE > 10% rows).

---

## Gathering Statistics (MySQL Mode)

### Analyze Table

```sql
-- Gather stats for a table
ANALYZE TABLE employees;

-- Gather with custom sample size
ANALYZE TABLE employees SAMPLE 30;  -- 30% sample
```

### View Statistics

```sql
-- View table stats
SELECT * FROM information_schema.TABLES
WHERE table_name = 'EMPLOYEES';

-- View column stats
SELECT * FROM information_schema.COLUMNS
WHERE table_name = 'EMPLOYEES';
```

---

## Gathering Statistics (Oracle Mode)

### DBMS_STATS Package

```sql
-- Gather table stats
EXEC DBMS_STATS.GATHER_TABLE_STATS('HR', 'EMPLOYEES');

-- Gather schema stats
EXEC DBMS_STATS.GATHER_SCHEMA_STATS('HR');

-- Gather dictionary stats (sys tenant)
EXEC DBMS_STATS.GATHER_DICTIONARY_STATS;
```

### View Statistics

```sql
-- View table stats
SELECT * FROM dba_tables WHERE table_name = 'EMPLOYEES';

-- View column stats
SELECT * FROM dba_tab_columns WHERE table_name = 'EMPLOYEES';
```

---

## Automatic Statistics (V4.2+)

OceanBase can auto-gather stats:

```sql
-- Enable auto stats (MySQL Mode)
ALTER SYSTEM SET enable_auto_le羽毛_stats = 'TRUE';

-- Set auto stats window
ALTER SYSTEM SET auto_le羽毛_stats_start_time = '02:00:00';
ALTER SYSTEM SET auto_le羽毛_stats_duration = '4H';
```

> **Tip**: Auto stats run during low-traffic hours (e.g., 02:00-06:00).

---

## Manual Statistics (When to Use)

| Scenario | Action |
|----------|---------|
| New table with > 1M rows | `ANALYZE TABLE` immediately |
| After bulk INSERT/DELETE | `ANALYZE TABLE` |
| Query plan changed (slower) | Refresh stats |
| Before production deployment | Gather stats on all tables |

---

## Histograms (Oracle Mode)

Histograms help with skewed data distribution:

```sql
-- Create histogram on skewed column
EXEC DBMS_STATS.GATHER_TABLE_STATS(
  ownname => 'HR',
  tabname => 'EMPLOYEES',
  method_opt => 'FOR COLUMNS SIZE 10 dept_id'
);
-- SIZE 10 = 10 buckets
```

> **Use case**: `dept_id` has skewed distribution (some depts have 1000 employees, some have 2).

---

## MySQL Mode vs Oracle Mode

| Aspect | MySQL Mode | Oracle Mode |
|--------|-----------|-------------|
| Gather stats | `ANALYZE TABLE` | `DBMS_STATS.GATHER_TABLE_STATS` |
| View stats | `information_schema.TABLES` | `dba_tables` |
| Histogram | Not supported | Supported |

---

## Common Mistakes

1. **Never gathering statistics**
   - ❌ Stats from table creation (empty)
   - ✅ `ANALYZE TABLE` after bulk load

2. **Gathering stats during peak hours**
   - ❌ `ANALYZE TABLE` at 14:00 (peak traffic)
   - ✅ Schedule at 02:00-06:00 (off-peak)

3. **Wrong sample size (too small)**
   - ❌ `ANALYZE TABLE ... SAMPLE 1` (not representative)
   - ✅ `SAMPLE 30` (30% is usually enough)

4. **Not enabling auto stats**
   - ❌ Relying on manual `ANALYZE`
   - ✅ `ALTER SYSTEM SET enable_auto_le羽毛_stats='TRUE'`

5. **Gathering stats on tiny tables**
   - ❌ `ANALYZE TABLE` on 100-row table
   - ✅ Skip stats for tables < 10K rows (optimizer uses default)

---

## OceanBase Version Notes

### V4.2 (Baseline)
- `ANALYZE TABLE` (MySQL Mode)
- `DBMS_STATS` package (Oracle Mode)
- Auto stats with time window

### V4.4+
- Faster stats gathering (parallelized)
- Improved histogram accuracy

---

## Sources
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/optimizer-statistics
- https://www.oceanbase.com/docs/oceanbase-database-cn/v4.2.5/optimizer-statistics
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/dbms-stats
