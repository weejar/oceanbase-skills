# Alert Log Analysis

## Overview

OceanBase logs (observer.log, rs.log, election.log) are critical for diagnosing issues. This guide covers log locations, analysis techniques, and common error patterns for OceanBase V4.2+.

---

## Log Locations

| Log File | Path | Purpose |
|----------|------|---------|
| `observer.log` | `log/observer.log` | Main server log (most important) |
| `rs.log` | `log/rs.log` | RootService operations |
| `election.log` | `log/election.log` | Leader election events |
| `obproxy.log` | `log/obproxy.log` | OBProxy access log |
| `obproxy_error.log` | `log/obproxy_error.log` | OBProxy errors |

```bash
# Default log path (per OBServer)
cd /home/admin/oceanbase/log

# List logs by size
ls -lht *.log | head -20
```

---

## Reading observer.log

### Basic Log Analysis

```bash
# Find errors in last 1000 lines
tail -1000 log/observer.log | grep -i error

# Find specific error code
grep "ERROR 4030" log/observer.log

# Find slow queries
grep "slow query" log/observer.log | tail -50
```

### Key Log Levels

| Level | Meaning | Action |
|-------|---------|--------|
| `INFO` | Normal operation | No action |
| `WARN` | Potential issue | Monitor |
| `ERROR` | Error occurred | Investigate |
| `FATAL` | Critical failure | Immediate action |

---

## Common Error Patterns

### 1. Disk Full

```
[ERROR] disk is full, ret_code=-4184
```
**Fix**: Free up disk space; `ALTER SYSTEM MINOR FREEZE;` to flush MemStore.

### 2. Memory Limit Exceeded

```
[ERROR] alloc memory fail, tenant_id=1002, ret=-4012
```
**Fix**: Increase `MEMORY_SIZE` for tenant's resource unit.

### 3. Replication Lag

```
[WARN] replication lag is too large, lag_sec=120
```
**Fix**: Check network; increase `log_archive_concurrency`.

### 4. Leader Switch (Normal)

```
[INFO] leader switch success, partition=xxx, new_leader=xxx
```
**No action**: Normal Paxos leader re-election.

---

## Log Rotation

OceanBase auto-rotates logs. Configure rotation:

```sql
-- Enable auto log recycle
ALTER SYSTEM SET enable_syslog_recycle = 'TRUE';

-- Keep last N log files
ALTER SYSTEM SET max_syslog_file_count = 50;

-- Log level (INFO/WARN/ERROR)
ALTER SYSTEM SET syslog_level = 'WARN';  -- Reduce log volume
```

---

## MySQL Mode vs Oracle Mode

Log analysis is identical in both modes.

---

## Common Mistakes

1. **Not monitoring logs regularly**
   - ❌ Only check logs when users complain
   - ✅ Daily log scan for `ERROR` and `WARN`

2. **Log files consuming all disk**
   - ❌ `enable_syslog_recycle = FALSE`
   - ✅ `ALTER SYSTEM SET enable_syslog_recycle = 'TRUE';`

3. **Setting `syslog_level` to `DEBUG` in production**
   - ❌ Massive log volume
   - ✅ `syslog_level = 'WARN'` (production)

4. **Not centralizing logs**
   - ❌ Logs scattered across 10 servers
   - ✅ Use OCP log collection (centralized)

5. **Ignoring `WARN` entries**
   - ❌ Only look at `ERROR`
   - ✅ `WARN` often indicates future `ERROR`

---

## OceanBase Version Notes

### V4.2 (Baseline)
- Log files: observer.log, rs.log, election.log
- Auto log rotation
- `enable_syslog_recycle` parameter

### V4.4+
- Improved log format (more structured)
- Faster log replay

---

## Sources
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/log
- https://www.oceanbase.com/docs/oceanbase-database-cn/v4.2.5/log
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/log-parameters
