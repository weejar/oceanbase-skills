# OBProxy Architecture and Configuration

## Overview

OBProxy is OceanBase's **reverse proxy** that routes SQL requests to the correct OBServer. It handles connection mapping, partition location lookup, and transparent failover. This guide covers OBProxy architecture, configuration, and troubleshooting for OceanBase V4.2+.

---

## Architecture

```
Client App
    ↓ (MySQL protocol)
OBProxy (Stateless, can have multiple instances)
    ↓ (Lookup partition location)
OBServer (Leader replica)
```

> **Key Point**: OBProxy is **stateless**. You can run multiple OBProxy instances behind a load balancer (HAProxy, F5, etc.).

---

## OBProxy Functions

| Function | Description |
|----------|-------------|
| **SQL Routing** | Routes SQL to the OBServer hosting the leader partition |
| **Connection Mapping** | Maps client connection → OBServer connection |
| **Transparent Failover** | Redirects on OBServer failure (client doesn't notice) |
| **Load Balancing** | Distributes connections across OBServers (configurable) |
| **Weak Consistency Read** | Routes to follower replicas for `ob_read_consistency='WEAK'` |

---

## Installing OBProxy

### Via OBD (Recommended)

```bash
# OBD deploy OBProxy
obd mirror clone obproxy-ce-4.2.0.0
obd cluster edit-config cluster_name

# Add obproxy section:
#   obproxy:
#     servers:
#       - 10.0.0.1
#       - 10.0.0.2
#     properties:
#       listen_port: 2883

obd cluster reload cluster_name
```

### Manual Installation

```bash
# Download OBProxy
wget https://mirrors.oceanbase.com/oceanbase/community/el/7/x86_64/obproxy-ce-4.2.0.0-1.el7.x86_64.rpm

# Install
rpm -ivh obproxy-ce-4.2.0.0-1.el7.x86_64.rpm

# Start OBProxy
cd /opt/taobao/install/obproxy && bin/start_obproxy.sh -p 2883 -r "10.0.0.1:2881;10.0.0.2:2881;10.0.0.3:2881" -n obcluster
```

---

## Configuring OBProxy

### Key Parameters

| Parameter | Default | Recommended | Description |
|-----------|---------|-------------|-------------|
| `listen_port` | 2883 | 2883 | Client connect port |
| `enable_strict_kernel_release` | True | False (for non-Linux) | Bypass kernel check |
| `obproxy_sys_password` | (empty) | (set) | Sys user password for OBProxy |
| `observer_sys_password` | (empty) | (set) | Proxyro user password |

### Connecting Through OBProxy

```bash
# Connect to a tenant via OBProxy
mysql -h10.0.0.1 -P2883 -uapp_user@tenant_mysql#obcluster -p

# Format: user@tenant#cluster
```

> **Tip**: The `#cluster` part is the cluster name (from `SHOW PARAMETERS LIKE 'cluster';`).

---

## OBProxy High Availability

```
Load Balancer (HAProxy/F5)
       ↓
┌──────────┬──────────┬──────────┐
│ OBProxy1 │ OBProxy2 │ OBProxy3 │  (all stateless)
└─────┬────┴─────┬────┴─────┬────┘
      │           │           │
    OBServer1  OBServer2  OBServer3
```

### HAProxy Config Example

```
frontend ob_front
    bind *:2883
    default_backend ob_back

backend ob_back
    balance roundrobin
    server obp1 10.0.0.10:2883 check
    server obp2 10.0.0.11:2883 check
    server obp3 10.0.0.12:2883 check
```

---

## Monitoring OBProxy

### Check OBProxy Status

```sql
-- From OBProxy, check backend OBServers
SHOW PROXYSESSION;  -- MySQL Mode

-- Check OBProxy process
ps aux | grep obproxy
```

### OBProxy Log Locations

| Log Type | Path |
|----------|------|
| Error log | `log/obproxy_error.log` |
| Slow SQL log | `log/obproxy_slow.log` |
| Statistics log | `log/obproxy_stat.log` |

---

## Troubleshooting

### Connection Refused

```bash
# Check OBProxy is running
ps aux | grep obproxy

# Check port is listening
netstat -tlnp | grep 2883
```

### Wrong Partition Location (Stale Cache)

```bash
# Connect to OBProxy admin port (default: 2884)
mysql -h10.0.0.1 -P2884 -uadmin -p

-- Flush location cache
SHOW PROXYCC RESET CONFIG;
```

### Slow Queries via OBProxy

Check `log/obproxy_slow.log` for queries > `slow_query_time` threshold.

---

## Common Mistakes

1. **Single OBProxy instance in production**
   - ❌ One OBProxy = single point of failure
   - ✅ Deploy 3 OBProxy instances behind HAProxy

2. **OBProxy and OBServer on same server without resource isolation**
   - ❌ CPU contention
   - ✅ Separate servers, or use cgroup limits

3. **Not monitoring OBProxy logs**
   - ❌ Missing slow query visibility
   - ✅ Check `obproxy_slow.log` daily

4. **Wrong cluster name in connection string**
   - ❌ `user@tenant@cluster` (wrong separator)
   - ✅ `user@tenant#cluster` (correct separator)

5. **Firewall blocking OBProxy port**
   - ❌ Clients can't connect
   - ✅ Open port 2883 (and 2884 for admin)

---

## OceanBase Version Notes

### V4.2 (Baseline)
- OBProxy supports MySQL protocol
- Transparent failover
- Weak consistency read routing

### V4.4+
- Faster location cache refresh
- Improved routing for partitioned tables

---

## Sources
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/obproxy
- https://www.oceanbase.com/docs/oceanbase-database-cn/v4.2.5/obproxy
- https://en.oceanbase.com/docs/ocp/obproxy-deployment
