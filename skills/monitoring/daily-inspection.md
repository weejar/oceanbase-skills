# Daily Inspection Scripts and Automation

## Overview

Automated daily inspection catches issues before users notice. This guide covers inspection scripts, OCP automation, and alerting for OceanBase V4.2+.

---

## Daily Inspection Checklist

| # | Item | SQL | Expected |
|---|------|-----|----------|
| 1 | Cluster status | `SHOW PARAMETERS LIKE 'cluster_status';` | `ACTIVE` |
| 2 | Server status | `SELECT * FROM DBA_OB_SERVERS;` | All `ACTIVE` |
| 3 | Tenant status | `SELECT * FROM DBA_OB_TENANTS;` | All `NORMAL` |
| 4 | Disk usage | `SELECT svr_ip, disk_used/disk_total FROM DBA_OB_SERVERS;` | < 80% |
| 5 | MemStore usage | `SELECT * FROM DBA_OB_MEMSTORE;` | < 80% |
| 6 | Log archive lag | `SELECT lag_seconds FROM DBA_OB_ARCHIVELOG;` | < 60s |
| 7 | Top slow SQL | `SELECT * FROM DBA_OB_SQL_AUDIT ORDER BY sum_elapsed DESC LIMIT 10;` | < 1s avg |
| 8 | Failed logins | Check `observer.log` for `login fail` | 0 |

---

## Automated Inspection Script

```bash
#!/bin/bash
# daily_inspection.sh

OB_USER="root@sys"
OB_PASS="password"
OB_HOST="127.0.0.1"
OB_PORT="2881"
ALERT_LOG="/var/log/ob_daily_alert.log"

echo "=== Daily Inspection: $(date) ===" | tee $ALERT_LOG

# 1. Cluster status
status=$(mysql -h$OB_HOST -P$OB_PORT -u$OB_USER -p$OB_PASS -e "SHOW PARAMETERS LIKE 'cluster_status';" | grep cluster_status | awk '{print $4}')
if [ "$status" != "ACTIVE" ]; then
  echo "ALERT: Cluster status = $status" | tee -a $ALERT_LOG
fi

# 2. Server status
mysql -h$OB_HOST -P$OB_PORT -u$OB_USER -p$OB_PASS -e "SELECT svr_ip FROM oceanbase.DBA_OB_SERVERS WHERE status != 'ACTIVE';" | grep -v svr_ip | while read ip; do
  echo "ALERT: Server $ip is not ACTIVE" | tee -a $ALERT_LOG
done

# 3. Disk usage > 80%
mysql -h$OB_HOST -P$OB_PORT -u$OB_USER -p$OB_PASS -e "SELECT svr_ip, disk_used*100/disk_total AS pct FROM oceanbase.DBA_OB_SERVERS;" | grep -v svr_ip | while read ip pct; do
  if (( $(echo "$pct > 80" | bc -l) )); then
    echo "ALERT: Server $ip disk usage = $pct%" | tee -a $ALERT_LOG
  fi
done

# 4. Log archive lag > 60s
lag=$(mysql -h$OB_HOST -P$OB_PORT -u$OB_USER -p$OB_PASS -e "SELECT lag_seconds FROM oceanbase.DBA_OB_ARCHIVELOG;" | grep -v lag | sort -rn | head -1)
if (( $(echo "$lag > 60" | bc -l) )); then
  echo "ALERT: Log archive lag = $lag seconds" | tee -a $ALERT_LOG
fi

echo "=== Inspection Complete ===" | tee -a $ALERT_LOG
```

### Schedule via Cron

```bash
# Run daily at 08:00
crontab -e
# Add:
0 8 * * * /path/to/daily_inspection.sh
```

---

## OCP Automated Inspection

OCP provides built-in daily inspection:

1. **Navigate**: OCP → **Health Check** → **Inspection Report**
2. **Schedule**: Daily at 08:00
3. **Report includes**: All 8 items in checklist above
4. **Alert**: OCP sends email/Slack notification if any item fails

> **Tip**: OCP inspection is easier than maintaining custom scripts.

---

## Alerting Thresholds

| Item | Warning | Critical |
|------|---------|----------|
| Disk usage | 80% | 90% |
| MemStore usage | 80% | 90% |
| Log archive lag | 60s | 300s |
| SQL avg latency | 500ms | 1000ms |
| Server down | 1 server | 2+ servers |

---

## MySQL Mode vs Oracle Mode

Inspection SQL is nearly identical. Only system view names differ.

| Aspect | MySQL Mode | Oracle Mode |
|--------|-----------|-------------|
| System views | `oceanbase.DBA_OB_*` | `SYS.DBA_OB_*` |

---

## Common Mistakes

1. **Not automating inspection**
   - ❌ Manual check once a month
   - ✅ Daily cron + OCP auto-inspection

2. **Inspection script doesn't send alerts**
   - ❌ Script runs but no one reads output
   - ✅ Email/Slack alert on any WARNING/CRITICAL

3. **Ignoring WARNING-level alerts**
   - ❌ Only act on CRITICAL
   - ✅ WARNING = future CRITICAL (fix proactively)

4. **Not testing inspection script**
   - ❌ Script has syntax error, no one notices
   - ✅ Test script monthly with simulated failure

5. **Over-alerting (alert fatigue)**
   - ❌ Alert on every minor event
   - ✅ Only alert on actionable items

---

## OceanBase Version Notes

### V4.2 (Baseline)
- OCP inspection reports
- Custom inspection scripts supported

### V4.4+
- More inspection items in OCP
- Improved alerting integration (PagerDuty, Slack)

---

## Sources
- https://en.oceanbase.com/docs/ocp/health-check
- https://www.oceanbase.com/docs/ocp-cn/health-check
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/monitoring
