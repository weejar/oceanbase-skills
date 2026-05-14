# obdiag Diagnostic Tool

## Overview

`obdiag` is OceanBase's official diagnostic tool. It collects logs, analyzes root causes, and generates health reports. This guide covers obdiag installation, usage, and common scenarios for OceanBase V4.2+.

---

## Installing obdiag

### Method 1: OBD (Recommended)

```bash
# Install obdiag via OBD
obd mirror clone obdiag-1.6.0
obd cluter edit-config cluster_name

# Add obdiag section:
#   obdiag:
#     servers:
#       - 10.0.0.1

obd cluster reload cluster_name
```

### Method 2: Manual (RPM)

```bash
# Download
wget https://mirrors.oceanbase.com/obdiag/obdiag-1.6.0-1.el7.x86_64.rpm

# Install
rpm -ivh obdiag-1.6.0-1.el7.x86_64.rpm

# Verify
obdiag --version
```

---

## Basic Commands

| Command | Purpose |
|---------|---------|
| `obdiag gather log` | Collect logs from all servers |
| `obdiag analyze rootcause` | Analyze root cause of error |
| `obdiag check` | Health check (all items) |
| `obdiag report` | Generate health report |
| `obdiag gather sql_audit` | Collect SQL audit data |

---

## Collecting Logs

```bash
# Collect last 2 hours of logs
obdiag gather log --since 2h

# Collect logs for specific time range
obdiag gather log --from '2025-01-01 10:00:00' --to '2025-01-01 12:00:00'

# Collect logs for specific server
obdiag gather log --svr_ip 10.0.0.1
```

> **Output**: Logs packaged as `.tar.gz` in `./obdiag_gather_dir/`.

---

## Root Cause Analysis

```bash
# Analyze specific error (provide trace_id or time)
obdiag analyze rootcause --trace_id Yxxx

# Analyze slow query
obdiag analyze rootcause --sql_id abc123

# Analyze cluster-wide issues
obdiag analyze rootcause --since 1h
```

> **Output**: Root cause report in `./obdiag_analysis_dir/`.

---

## Health Check (All Items)

```bash
# Run all health check items
obdiag check

# Run specific check item
obdiag check --check_id 10  # Check disk usage
```

### Common Check Items

| Check ID | Item | Description |
|----------|------|-------------|
| 10 | Disk usage | Check if disk > 80% |
| 20 | MemStore usage | Check if MemStore > 80% |
| 30 | Log archive lag | Check if lag > 60s |
| 40 | Server status | Check all servers are active |
| 50 | Tenant status | Check all tenants are NORMAL |

---

## Health Report

```bash
# Generate comprehensive health report
obdiag report

# Report includes:
# - Server status
# - Tenant status
# - Disk/memory usage
# - Slow SQL summary
# - Backup status
```

> **Output**: HTML report in `./obdiag_report_dir/`.

---

## MySQL Mode vs Oracle Mode

obdiag works identically for both modes.

---

## Common Mistakes

1. **Not using obdiag for troubleshooting**
   - ❌ Manually grepping logs on 10 servers
   - ✅ `obdiag gather log --since 1h` (collects from all servers)

2. **Forgetting to check obdiag output path**
   - ❌ `obdiag gather log` then can't find files
   - ✅ Output in `./obdiag_gather_dir/` (current directory)

3. **Running obdiag on production during peak hours**
   - ❌ High I/O overhead
   - ✅ Schedule during maintenance window (if gathering large logs)

4. **Not automating health checks**
   - ❌ Manual `obdiag check` once a month
   - ✅ Cron job: `0 8 * * * obdiag check && send_email`

5. **Ignoring obdiag report recommendations**
   - ❌ Report says "disk > 80%" but no action
   - ✅ Act on all WARNINGs and ERRORs in report

---

## OceanBase Version Notes

### V4.2 (Baseline)
- obdiag tool available
- Log collection, root cause analysis
- Health check and report

### V4.4+
- More check items
- Faster log collection (parallelized)

---

## Sources
- https://en.oceanbase.com/docs/obdiag
- https://www.oceanbase.com/docs/obdiag
- https://en.oceanbase.com/docs/obdiag/installation
