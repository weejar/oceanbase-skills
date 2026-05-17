---
name: session-monitor
description: OceanBase 数据库活动会话监控。查看当前会话、会话统计、长空闲会话分析、等待事件等。
agent_created: true
---

# OceanBase Session Monitor (活动会话监控)

> **⚠️ Security Note**: 代码示例中的 `YOUR_PASSWORD` 为占位符。请通过环境变量（如 `os.getenv('OB_PASSWORD')`）或运行时传入真实密码，**不要将真实密码硬编码到文件中**。

> **快速命令**: 一键查看活动会话：
> ```bash
> python -c "import pymysql; conn=pymysql.connect(host='172.20.22.213',port=2883,user='root@sys#enmotest',password='YOUR_PASSWORD',database='oceanbase'); c=conn.cursor(); c.execute('SELECT ID,SVR_IP,USER,TENANT,DB,COMMAND,TIME,STATE FROM oceanbase.GV\$OB_PROCESSLIST ORDER BY TIME DESC LIMIT 50'); [print(r) for r in c.fetchall()]; c.close(); conn.close()"
> ```

## Connection Parameters

| Parameter | Value |
|-----------|-------|
| Host | `172.20.22.213` |
| Port | `2883` |
| User | `root@sys#enmotest` |
| Password | `YOUR_PASSWORD` |
| Database | `oceanbase` |

---

## Query Active Sessions

**View**: `oceanbase.GV$OB_PROCESSLIST`

```python
import pymysql

conn = pymysql.connect(
    host='172.20.22.213',
    port=2883,
    user='root@sys#enmotest',
    password='YOUR_PASSWORD',
    database='oceanbase',
    charset='utf8mb4'
)

cursor = conn.cursor()
cursor.execute('''
SELECT
    ID,
    SVR_IP,
    SVR_PORT,
    USER,
    HOST,
    DB,
    TENANT,
    COMMAND,
    TIME,
    TOTAL_TIME,
    STATE,
    SQL_ID,
    TRANS_STATE,
    USER_CLIENT_IP,
    INFO
FROM oceanbase.GV$OB_PROCESSLIST
ORDER BY TOTAL_TIME DESC
LIMIT 100
''')

rows = cursor.fetchall()
cursor.close()
conn.close()

# Print table
for r in rows:
    print(f'{r[0]:<12} {r[1]:<16} {r[3]:<20} {r[6]:<12} {r[7]:<20} {r[8]:<10} {r[10]}')
```

---

## Session Statistics

```python
import pymysql

conn = pymysql.connect(
    host='172.20.22.213', port=2883,
    user='root@sys#enmotest', password='YOUR_PASSWORD',
    database='oceanbase', charset='utf8mb4'
)

cursor = conn.cursor()

# Count by state
cursor.execute('SELECT STATE, COUNT(*) FROM oceanbase.GV$OB_PROCESSLIST GROUP BY STATE')
print('=== Session State ===')
for row in cursor.fetchall():
    print(f'  {row[0]}: {row[1]}')

# Count by tenant
cursor.execute('SELECT TENANT, COUNT(*) FROM oceanbase.GV$OB_PROCESSLIST GROUP BY TENANT')
print('\n=== By Tenant ===')
for row in cursor.fetchall():
    print(f'  {row[0]}: {row[1]}')

# Count by user
cursor.execute('SELECT USER, TENANT, COUNT(*) FROM oceanbase.GV$OB_PROCESSLIST GROUP BY USER, TENANT ORDER BY COUNT(*) DESC LIMIT 10')
print('\n=== Top 10 Users ===')
for row in cursor.fetchall():
    print(f'  {row[0]}@{row[1]}: {row[2]}')

# Long idle sessions (>1 hour)
cursor.execute('SELECT ID, USER, TENANT, DB, TIME FROM oceanbase.GV$OB_PROCESSLIST WHERE COMMAND="Sleep" AND TIME>3600 ORDER BY TIME DESC')
print('\n=== Long Idle (>1h) ===')
for row in cursor.fetchall():
    print(f'  ID={row[0]}, {row[1]}@{row[2]}, IDLE={row[4]:.0f}s')

cursor.close()
conn.close()
```

---

## Key Columns in GV$OB_PROCESSLIST

| Column | Description |
|--------|-------------|
| `ID` | Session ID |
| `SVR_IP` | Server IP |
| `SVR_PORT` | Server Port |
| `USER` | Database user |
| `TENANT` | Tenant name |
| `DB` | Current database |
| `HOST` | Client host |
| `COMMAND` | Command type (Sleep, Query, Binlog Dump, etc.) |
| `TIME` | Idle time in seconds |
| `TOTAL_TIME` | Total connection time |
| `STATE` | Session state (ACTIVE, SLEEP, SESSION_KILLED) |
| `SQL_ID` | Current SQL ID |
| `INFO` | Current SQL statement |
| `TRANS_STATE` | Transaction state |
| `USER_CLIENT_IP` | Client IP |

---

## Other Session-Related Views

| View | Description |
|------|-------------|
| `GV$OB_SESSION` | Detailed session info |
| `GV$OB_ACTIVE_SESSION_HISTORY` | Active session history (ASH) |
| `GV$SESSION_WAIT` | Current wait events |
| `GV$SESSION_EVENT` | Session wait events summary |
| `GV$SESSION_LONGOPS` | Long running operations |
| `GV$SESSION_WAIT_HISTORY` | Wait history |
| `V$OB_PROCESSLIST` | Per-server process list (single node) |
| `CDB_WR_ACTIVE_SESSION_HISTORY` | CDB ASH (requires Diagnostic Pack) |

---

## Session State Meanings

| State | Meaning |
|-------|---------|
| `ACTIVE` | Currently executing SQL |
| `SLEEP` | Idle waiting for command |
| `SESSION_KILLED` | Session marked for termination |
| `COMMITTING` | In commit transaction |
| `ROLLING BACK` | Rolling back transaction |

---

## Command Types

| Command | Meaning |
|---------|---------|
| `Sleep` | Idle connection |
| `Query` | Executing SQL |
| `Binlog Dump` | Replication binlog sync |
| `Connect` | Connection handshake |
| `Table Dump` | Table metadata exchange |

---

## Related Skills

- `skills/appdev/sys-tenant-connection.md` - Sys tenant connection, tenant listing
- `skills/monitoring/` - Health check, obdiag, Top SQL
- `skills/performance/` - Wait events, slow query analysis

---

*Version: OceanBase V4.x | Last updated: 2026-05-14*
