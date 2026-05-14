# Health Check and Inspection

## Overview

Regular health checks prevent surprises. This guide covers daily/weekly health check items and automation scripts for OceanBase V4.2+.

---

## Health Check Items

| Check | SQL | Expected |
|-------|-----|----------|
| Cluster status | `SHOW PARAMETERS LIKE 'cluster_status';` | `ACTIVE` |
| Tenant status | `SELECT * FROM oceanbase.DBA_OB_TENANTS;` | All `NORMAL` |
| Server status | `SELECT * FROM oceanbase.DBA_OB_SERVERS;` | All `ACTIVE` |
| Disk usage | Check `DBA_OB_SERVERS.disk_usage` | < 80% |
| MemStore usage | `SELECT * FROM oceanbase.DBA_OB_MEMSTORE;` | < 80% |
| Log archive lag | `SELECT * FROM oceanbase.DBA_OB_ARCHIVELOG;` | < 60s |

---

## Daily Health Check Script

```bash
#!/bin/bash
# daily_health_check.sh

OB_USER="root@sys"
OB_PASS="password"
OB_HOST="127.0.0.1"
OB_PORT="2881"

echo "=== OceanBase Daily Health Check: $(date) ==="

# 1. Cluster status
echo "1. Cluster Status:"
mysql -h$OB_HOST -P$OB_PORT -u$OB_USER -p$OB_PASS -e "SHOW PARAMETERS LIKE 'cluster_status';"

# 2. Server status
echo "2. Server Status:"
mysql -h$OB_HOST -P$OB_PORT -u$OB_USER -p$OB_PASS -e "SELECT svr_ip, status FROM oceanbase.DBA_OB_SERVERS;"

# 3. Tenant status
echo "3. Tenant Status:"
mysql -h$OB_HOST -P$OB_PORT -u$OB_USER -p$OB_PASS -e "SELECT tenant_name, status FROM oceanbase.DBA_OB_TENANTS;"

# 4. Disk usage
echo "4. Disk Usage:"
mysql -h$OB_HOST -P$OB_PORT -u$OB_USER -p$OB_PASS -e "SELECT svr_ip, round(disk_usage*100,2) AS disk_pct FROM oceanbase.DBA_OB_SERVERS;"

# 5. Log archive lag
echo "5. Log Archive Lag:"
mysql -h$OB_HOST -P$OB_PORT -u$OB_USER -p$OB_PASS -e "SELECT tenant_id, round(lag_seconds,2) AS lag_sec FROM oceanbase.DBA_OB_ARCHIVELOG;"

echo "=== Check Complete ==="
```

> **Schedule**: Run daily via cron at 08:00.

---

## Key VIEWS for Health Check

| View | Purpose |
|------|---------|
| `DBA_OB_SERVERS` | Server status, disk, CPU |
| `DBA_OB_TENANTS` | Tenant status |
| `DBA_OB_ZONES` | Zone status |
| `DBA_OB_MEMSTORE` | Memory usage |
| `DBA_OB_ARCHIVELOG` | Log archive lag |
| `DBA_OB_SQL_AUDIT` | Slow SQL (top 10) |

---

## Automated Inspection (via OCP)

OCP provides built-in inspection reports:

1. **Navigate**: OCP → **Health Check** → **Inspection Report**
2. **Schedule**: Daily or weekly
3. **Report includes**: Server status, tenant status, disk, memory, slow SQL, backup status

> **Tip**: OCP auto-detects issues and sends alerts.

---

## MySQL Mode vs Oracle Mode

| Aspect | MySQL Mode | Oracle Mode |
|--------|-----------|-------------|
| System views | `oceanbase.DBA_OB_*` | `SYS.DBA_OB_*` |
| Connection | `mysql -h -P -u -p` | Same |

---

## Common Mistakes

1. **Not automating health checks**
   - ❌ Manual check once a month
   - ✅ Daily cron + OCP auto-inspection

2. **Ignoring WARN-level alerts**
   - ❌ Only check `ERROR`
   - ✅ `WARN` = future `ERROR`

3. **Not checking log archive lag**
   - ❌ Lag grows to hours undetected
   - ✅ Alert if lag > 60s

4. **Forgetting to check tenant status**
   - ❌ Tenant `STOPPED` for days
   - ✅ Daily check: all tenants = `NORMAL`

5. **Not documenting health check procedure**
   - ❌ Each DBA checks different items
   - ✅ Standardized checklist (runbook)

---

## OceanBase Version Notes

### V4.2 (Baseline)
- Health check views
- OCP inspection reports

### V4.4+
- More inspection items in OCP
- Improved health check automation

---

## Sources
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/health-check
- https://www.oceanbase.com/docs/oceanbase-database-cn/v4.2.5/health-check
- https://en.oceanbase.com/docs/ocp/health-check
