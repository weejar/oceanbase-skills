# Wait Events Diagnosis

## Overview

Wait events help diagnose performance bottlenecks in OceanBase. By identifying what sessions are waiting on, you can pinpoint I/O, lock, or CPU issues. This guide covers wait event analysis in OceanBase V4.2+.

---

## Key Concept

```
Session → Executes SQL → Waits for resource → Wait Event recorded
```

> **Use Case**: Database is slow but CPU is < 50%. Check wait events.

---

## Viewing Wait Events

### MySQL Mode

```sql
-- Top wait events
SELECT event,
       count(*) AS wait_count,
       round(avg(time_waited) / 1000000, 2) AS avg_sec
FROM oceanbase.DBA_OB_SYS_EVENT_HISTORY
GROUP BY event
ORDER BY wait_count DESC
LIMIT 20;
```

### Oracle Mode

```sql
-- Same data, different view
SELECT event,
       total_waits,
       round(time_waited / 1000000, 2) AS time_sec
FROM v$system_event
ORDER BY time_waited DESC;
```

---

## Common Wait Events

| Wait Event | Meaning | Fix |
|------------|---------|-----|
| `db file sequential read` | Index scan I/O | Add cache, faster disk |
| `db file scattered read` | Full scan I/O | Add index, partitioning |
| `log file sync` | Commit latency | Faster disk for redo log |
| `buffer busy waits` | Hot block contention | Increase `MEMORY_SIZE` |
| `enq: TX - row lock contention` | Row lock wait | Fix app (shorter transactions) |
| `CPU + Wait for CPU` | CPU overload | Add CPU or optimize SQL |

---

## Diagnosing Lock Waits

```sql
-- Find blocking sessions (MySQL Mode)
SELECT blocking_session_id,
       blocked_session_id,
       wait_event
FROM oceanbase.DBA_OB_LOCK_WAITS;

-- Kill blocking session (if safe)
ALTER SYSTEM KILL SESSION 'session_id,serial#';
```

> **Tip**: Long-running transactions (uncommitted) cause lock waits.

---

## Diagnosing I/O Bottlenecks

```sql
-- Check I/O wait time
SELECT event,
       round(sum(time_waited) / 1000000, 2) AS total_sec
FROM oceanbase.DBA_OB_SYS_EVENT_HISTORY
WHERE event LIKE '%db file%'
GROUP BY event;
```

> **Fix**: Use SSD (not HDD), increase `MEMORY_SIZE` (more buffer pool = less I/O).

---

## MySQL Mode vs Oracle Mode

| Aspect | MySQL Mode | Oracle Mode |
|--------|-----------|-------------|
| System view | `oceanbase.DBA_OB_SYS_EVENT_HISTORY` | `v$system_event`, `v$session_wait` |
| Lock wait view | `DBA_OB_LOCK_WAITS` | `v$lock` |

---

## Common Mistakes

1. **Not enabling wait event collection**
   - ❌ `enable_sys_event_history = FALSE`
   - ✅ `ALTER SYSTEM SET enable_sys_event_history = 'TRUE';`

2. **Fixing I/O when problem is locks**
   - ❌ Adding SSD when wait event is `enq: TX - row lock contention`
   - ✅ Fix application (shorter transactions, commit frequently)

3. **Not monitoring wait events regularly**
   - ❌ Only check when database is already slow
   - ✅ OCP alert on `db file sequential read` > 100ms avg

4. **Kill session without checking**
   - ❌ `KILL SESSION` on production transaction
   - ✅ Check `DBA_OB_LOCK_WAITS` first, kill only if safe

5. **Confusing `CPU + Wait for CPU` with I/O wait**
   - ❌ Adding SSD when problem is CPU
   - ✅ Check CPU usage; if 100%, add CPU or optimize SQL

---

## OceanBase Version Notes

### V4.2 (Baseline)
- Wait event collection
- `DBA_OB_SYS_EVENT_HISTORY` view
- Lock wait diagnosis

### V4.4+
- More granular wait event categories
- Improved lock wait diagnostics

---

## Sources
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/wait-events
- https://www.oceanbase.com/docs/oceanbase-database-cn/v4.2.5/wait-events
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/diagnostics
