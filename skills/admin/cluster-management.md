# 集群管理（Cluster Management）

> 适用版本：OceanBase V4.2+
> 执行权限：`root@sys` 租户

---

## 概述

OceanBase 集群由多个 **Zone** 组成，每个 Zone 包含一个或多个 **Server**（节点）。集群管理涵盖 Zone 管理、Server 管理、节点替换、Unit 迁移、集群扩缩容等核心运维操作。所有集群管理命令必须在 `sys` 租户下以 `root` 用户执行。

```
集群（Cluster）
├── Zone1（机房/可用区）
│   ├── Server1 (ip1:2882)
│   └── Server2 (ip2:2882)
├── Zone2（机房/可用区）
│   ├── Server3 (ip3:2882)
│   └── Server4 (ip4:2882)
└── Zone3（机房/可用区）
    ├── Server5 (ip5:2882)
    └── Server6 (ip6:2882)
```

| 命令类型 | 语法 | 说明 |
|---------|------|------|
| Zone 管理 | `ALTER SYSTEM ADD/DELETE/STOP/START ZONE` | Zone 级别操作 |
| Server 管理 | `ALTER SYSTEM STOP/START/ADD/DELETE SERVER` | 节点级别操作 |
| 节点隔离 | `STOP SERVER` / `FORCE STOP SERVER` / `ISOLATE SERVER` | 三级安全隔离 |
| Unit 迁移 | `ALTER SYSTEM MIGRATE UNIT` | 资源单元迁移 |
| Zone 属性 | `ALTER SYSTEM ALTER ZONE SET REGION/IDC` | 修改 Zone 元数据 |

---

## 1. 连接 sys 租户

```bash
# MySQL 模式
obclient -h <observer_ip> -P 2883 -uroot@sys#<cluster_name> -p -A

# Oracle 模式
obclient -h <observer_ip> -P 2883 -uroot@sys -p -A -d oracle
```

> 所有集群管理命令仅支持在 `sys` 租户中执行。

---

## 2. Zone 管理

### 2.1 查看 Zone

```sql
-- 查看所有 Zone 信息
SELECT * FROM oceanbase.DBA_OB_ZONES;

-- 查看结果示例
+-------+----------------------------+----------------------------+--------+-----+----------+-----------+
| ZONE  | CREATE_TIME                | MODIFY_TIME                | STATUS | IDC | REGION   | TYPE      |
+-------+----------------------------+----------------------------+--------+-----+----------+-----------+
| zone1 | 2026-01-15 10:00:00         | 2026-01-15 10:00:40         | ACTIVE | HZ0 | hangzhou | ReadWrite |
| zone2 | 2026-01-15 10:00:00         | 2026-01-15 10:00:40         | ACTIVE | HZ0 | hangzhou | ReadWrite |
| zone3 | 2026-01-15 10:00:00         | 2026-01-15 10:00:40         | ACTIVE | SH0 | shanghai | ReadWrite |
+-------+----------------------------+----------------------------+--------+-----+----------+-----------+
```

| 字段 | 说明 |
|------|------|
| `ZONE` | Zone 名称 |
| `STATUS` | `ACTIVE`（正常）/ `INACTIVE`（已停止） |
| `IDC` | 机房标识 |
| `REGION` | 地域标识 |
| `TYPE` | Zone 类型（当前仅 `ReadWrite`） |

### 2.2 添加 Zone

```sql
-- 添加 Zone（扩容）
ALTER SYSTEM ADD ZONE zone4
  IDC 'hz1', REGION 'hangzhou', ZONE_TYPE 'ReadWrite';

-- 添加后状态为 INACTIVE，需启动
ALTER SYSTEM START ZONE zone4;
```

### 2.3 删除 Zone

```sql
-- 前提：Zone 下已无节点，且 Zone 已停止
ALTER SYSTEM STOP ZONE zone4;
ALTER SYSTEM DELETE ZONE zone4;
```

### 2.4 修改 Zone 属性

```sql
-- 修改 Region 和 IDC（ALTER/CHANGE/MODIFY 等价）
ALTER SYSTEM ALTER ZONE zone4 SET REGION 'shanghai', IDC 'sh1';
```

### 2.5 Zone 隔离与恢复

Zone 隔离将流量从整个 Zone 切走，用于机房级容灾或轮转升级。

```sql
-- 最安全：STOP ZONE（保证 Paxos 多数派 + 日志同步）
ALTER SYSTEM STOP ZONE zone1;

-- 降级：FORCE STOP ZONE（跳过日志同步检查）
ALTER SYSTEM FORCE STOP ZONE zone1;

-- 最宽松：ISOLATE ZONE（仅切流量，不保证安全）
ALTER SYSTEM ISOLATE ZONE zone1;

-- 恢复 Zone
ALTER SYSTEM START ZONE zone1;
```

> **三种隔离命令的对比详见第 5 节。**

---

## 3. Server（节点）管理

### 3.1 查看节点

```sql
-- 查看所有节点信息
SELECT SVR_IP, SVR_PORT, ID, ZONE, SQL_PORT,
       WITH_ROOTSERVER, STATUS, STOP_TIME,
       START_SERVICE_TIME, BUILD_VERSION
FROM oceanbase.DBA_OB_SERVERS
ORDER BY ZONE, SVR_IP;
```

| 关键字段 | 说明 |
|---------|------|
| `SVR_IP:SVR_PORT` | 节点唯一标识（端口默认 2882） |
| `ZONE` | 所属 Zone |
| `SQL_PORT` | SQL 服务端口（默认 2881） |
| `STATUS` | `ACTIVE` / `INACTIVE` / `DELETING` |
| `STOP_TIME` | NULL = 正常；有值 = 已被隔离（Stopped） |
| `WITH_ROOTSERVER` | 是否为 RootServer 节点 |

### 3.2 添加节点

```sql
-- 在已有 Zone 上添加新节点
ALTER SYSTEM ADD SERVER '10.0.1.10:2882' ZONE 'zone1';

-- 添加后查看确认
SELECT SVR_IP, SVR_PORT, ZONE, STATUS FROM oceanbase.DBA_OB_SERVERS;
```

### 3.3 删除节点

```sql
-- 前提：节点上无 Unit，节点已停止
ALTER SYSTEM STOP SERVER '10.0.1.10:2882';
-- 确认 Unit 已迁走后删除
ALTER SYSTEM DELETE SERVER '10.0.1.10:2882' ZONE 'zone1';
```

### 3.4 节点隔离

```sql
-- 最安全：STOP SERVER（推荐首选）
ALTER SYSTEM STOP SERVER '10.0.1.10:2882';

-- 降级：FORCE STOP SERVER（跳过日志同步检查）
ALTER SYSTEM FORCE STOP SERVER '10.0.1.10:2882';

-- 最宽松：ISOLATE SERVER（仅切流量）
ALTER SYSTEM ISOLATE SERVER '10.0.1.10:2882';

-- 恢复节点
ALTER SYSTEM START SERVER '10.0.1.10:2882';
```

> 支持一次隔离多个节点：`ALTER SYSTEM STOP SERVER 'ip1:port1', 'ip2:port2';`

---

## 4. 节点替换

节点替换适用于硬件故障或不同规格替换，替换后集群节点数不变。

### 操作流程

```
Step 1: 添加新节点 → Step 2: 迁移 Unit → Step 3: 删除旧节点
```

### 4.1 添加新节点

```sql
-- 在待替换节点所在 Zone 上添加新节点
ALTER SYSTEM ADD SERVER '10.0.1.20:2882' ZONE 'zone1';
```

### 4.2 迁移 Unit

```sql
-- 查询旧节点上的 Unit
SELECT unit_id, svr_ip, svr_port
FROM oceanbase.DBA_OB_UNITS
WHERE SVR_IP = '10.0.1.10';

-- 逐个迁移 Unit 到新节点
ALTER SYSTEM MIGRATE UNIT 1001 DESTINATION '10.0.1.20:2882';
ALTER SYSTEM MIGRATE UNIT 1002 DESTINATION '10.0.1.20:2882';
ALTER SYSTEM MIGRATE UNIT 1003 DESTINATION '10.0.1.20:2882';

-- 查询迁移进度（空结果 = 完成）
SELECT JOB_ID, UNIT_ID, JOB_STATUS, PROGRESS
FROM oceanbase.DBA_OB_UNIT_JOBS
WHERE JOB_TYPE = 'MIGRATE_UNIT';
```

### 4.3 删除旧节点

```sql
-- 确认 Unit 已全部迁走
SELECT unit_id FROM oceanbase.DBA_OB_UNITS WHERE SVR_IP = '10.0.1.10';

-- 隔离旧节点
ALTER SYSTEM STOP SERVER '10.0.1.10:2882';

-- 删除旧节点
ALTER SYSTEM DELETE SERVER '10.0.1.10:2882' ZONE 'zone1';

-- 确认删除成功
SELECT SVR_IP, STATUS FROM oceanbase.DBA_OB_SERVERS WHERE SVR_IP = '10.0.1.10';
```

---

## 5. 节点隔离命令对比（核心）

三种隔离命令的安全级别递减，适用场景不同。

### 对比表

| 特性 | `STOP SERVER/ZONE` | `FORCE STOP SERVER/ZONE` | `ISOLATE SERVER/ZONE` |
|------|-------------------|--------------------------|----------------------|
| **安全级别** | ⭐⭐⭐ 最高 | ⭐⭐ 中 | ⭐ 最低 |
| **流量切换** | ✅ 是 | ✅ 是 | ✅ 是 |
| **Paxos 多数派检查** | ✅ 严格 | ✅ 检查 | ❌ 不检查 |
| **日志同步检查** | ✅ 要求 5s 内同步 | ❌ 跳过 | ❌ 不检查 |
| **能否停止进程** | ✅ 安全 | ⚠️ 有风险 | ❌ 不安全 |
| **多 Zone 同时隔离** | ❌ 禁止 | ❌ 禁止 | ✅ 允许 |
| **前置条件** | 最严格 | 中等 | 最宽松 |

### 选择原则

```
STOP SERVER/ZONE
  ↓ 失败（日志不同步）
FORCE STOP SERVER/ZONE
  ↓ 失败（其他节点状态异常）
ISOLATE SERVER/ZONE
```

> **任何场景都应首选 `STOP SERVER/ZONE`**。只有在无法成功时才降级使用弱化命令，且弱化隔离后**不能停止进程**。

---

## 6. 集群扩缩容

### 6.1 集群扩容（增加节点）

```sql
-- 方式一：在已有 Zone 内添加节点（水平扩容）
ALTER SYSTEM ADD SERVER '10.0.2.10:2882' ZONE 'zone1';
ALTER SYSTEM ADD SERVER '10.0.2.11:2882' ZONE 'zone2';
ALTER SYSTEM ADD SERVER '10.0.2.12:2882' ZONE 'zone3';

-- 方式二：添加新 Zone + 节点
ALTER SYSTEM ADD ZONE zone4 IDC 'bj1', REGION 'beijing';
ALTER SYSTEM ADD SERVER '10.0.3.10:2882' ZONE 'zone4';
ALTER SYSTEM START ZONE zone4;

-- 扩容后调整租户资源池
ALTER RESOURCE POOL pool_app UNIT_NUM = 2;
```

### 6.2 集群缩容（减少节点）

```sql
-- Step 1: 迁移待删除节点上的 Unit
SELECT unit_id FROM oceanbase.DBA_OB_UNITS WHERE SVR_IP = '10.0.2.10';
ALTER SYSTEM MIGRATE UNIT <unit_id> DESTINATION '10.0.1.10:2882';

-- Step 2: 确认迁移完成
SELECT * FROM oceanbase.DBA_OB_UNIT_JOBS WHERE JOB_TYPE = 'MIGRATE_UNIT';

-- Step 3: 删除节点
ALTER SYSTEM STOP SERVER '10.0.2.10:2882';
ALTER SYSTEM DELETE SERVER '10.0.2.10:2882' ZONE 'zone1';

-- Step 4: 删除 Zone（如果需要）
ALTER SYSTEM STOP ZONE zone4;
ALTER SYSTEM DELETE ZONE zone4;
```

---

## 7. Unit 迁移

Unit 迁移用于负载均衡、节点替换和资源调整。

```sql
-- 迁移 Unit 到目标节点
ALTER SYSTEM MIGRATE UNIT unit_id DESTINATION 'svr_ip:svr_port';

-- 查看迁移进度
SELECT JOB_ID, UNIT_ID, JOB_STATUS, PROGRESS, START_TIME, MODIFY_TIME
FROM oceanbase.DBA_OB_UNIT_JOBS
WHERE JOB_TYPE = 'MIGRATE_UNIT';

-- 查看所有 Unit 分布
SELECT u.unit_id, u.svr_ip, u.zone, t.tenant_name,
       u.max_cpu, u.memory_size
FROM oceanbase.DBA_OB_UNITS u
LEFT JOIN oceanbase.DBA_OB_TENANTS t ON u.tenant_id = t.tenant_id
ORDER BY u.svr_ip, u.zone;
```

---

## 8. 集群信息查看

### 8.1 集群总览

```sql
-- 查看集群名称和基本信息
SHOW PARAMETERS LIKE 'cluster';

-- 查看集群版本
SELECT BUILD_VERSION FROM oceanbase.DBA_OB_SERVERS LIMIT 1;

-- 查看 RootServer 节点
SELECT SVR_IP, SVR_PORT, ZONE
FROM oceanbase.DBA_OB_SERVERS
WHERE WITH_ROOTSERVER = 'YES';
```

### 8.2 租户 Locality 与 Primary Zone

```sql
-- 查看所有租户的 Locality
SELECT tenant_name, locality, primary_zone, compatibility_mode
FROM oceanbase.DBA_OB_TENANTS;

-- 修改租户 Primary Zone
ALTER TENANT tenant_app PRIMARY_ZONE = 'zone1;zone2,zone3';

-- 修改租户 Locality（调整副本分布）
ALTER TENANT tenant_app LOCALITY = 'FULL{1}@zone1, FULL{1}@zone2, FULL{1}@zone3';
```

### 8.3 资源单元与资源池

```sql
-- 查看所有资源单元
SELECT unit_id, tenant_id, svr_ip, zone,
       max_cpu, min_cpu, memory_size, data_disk_size
FROM oceanbase.DBA_OB_UNITS;

-- 查看所有资源池
SELECT pool_id, tenant_id, unit_config_id, unit_count, zone_list
FROM oceanbase.DBA_OB_RESOURCE_POOLS;

-- 查看资源单元规格
SELECT name, max_cpu, min_cpu, memory_size, data_disk_size
FROM oceanbase.DBA_OB_UNIT_CONFIGS;
```

---

## 9. 轮转升级（Rolling Upgrade）

通过 Zone 隔离实现集群无损升级。

```
Zone1 隔离 → 升级 Zone1 → 恢复 Zone1
                                    ↓
Zone3 隔离 → 升级 Zone3 → 恢复 Zone3
                                    ↓
Zone2 隔离 → 升级 Zone2 → 恢复 Zone2
```

```sql
-- Step 1: 隔离 Zone1
ALTER SYSTEM STOP ZONE zone1;

-- Step 2: 确认 Zone1 已隔离
SELECT ZONE, STATUS FROM oceanbase.DBA_OB_ZONES WHERE ZONE = 'zone1';

-- Step 3: 执行升级（通过 OBD 或手动）
-- obd cluster upgrade <deploy_name> --version <new_version>

-- Step 4: 恢复 Zone1
ALTER SYSTEM START ZONE zone1;

-- Step 5: 重复 Step 1-4 处理 Zone3 和 Zone2
```

> **注意**：升级前务必完成预检查（`obd cluster check`），并确认已备份。详见 `devops/version-upgrade.md`。

---

## 10. 常见错误与排查

| 错误码 | 错误信息 | 原因 | 解决方案 |
|--------|---------|------|---------|
| `-4660` | `cannot stop server or stop zone in multiple zones` | 已有其他 Zone/节点被 Stop | 先恢复已隔离的 Zone/Server，或使用 `ISOLATE` |
| `-4179` | `Tenant(xxx) LS(x) has no leader` | 日志流无 Leader | 检查 `GV$OB_LOG_STAT` 排查 Leader 异常 |
| `-4179` | `Tenant(xxx) locality is changing` | Locality 变更中 | 等待变更完成后再操作 |
| `-4179` | `no enough valid paxos member` | 剩余副本不满足多数派 | 检查副本分布，无法 Stop 则降级用 `ISOLATE` |
| `-4179` | `log not sync` | 副本日志不同步（>5s） | 等待同步或降级用 `FORCE STOP` |
| `-4179` | `has no enough valid paxos member after stop` | Stop 后破坏多数派 | 检查 `GV$OB_LOG_STAT` 的 END_SCN |
| 节点删除失败 | 节点上仍有 Unit | Unit 未迁完 | 先 `MIGRATE UNIT` 再删除 |
| Zone 删除失败 | Zone 下仍有节点 | 节点未删除 | 先删除 Zone 下所有节点 |
| Zone 状态不变 | `START ZONE` 无效果 | Zone 下无节点 | 先添加节点再启动 Zone |

### 日志流同步检查

```sql
-- 检查指定租户的日志流 Leader 和同步状态
SELECT TENANT_ID, LS_ID, ROLE, IS_LEADER, END_SCN
FROM GV$OB_LOG_STAT
WHERE TENANT_ID = 1001;

-- 将 SCN 转换为时间戳查看延迟
SELECT SCN_TO_TIMESTAMP(END_SCN) AS sync_time
FROM GV$OB_LOG_STAT
WHERE TENANT_ID = 1001 AND LS_ID = 1;

-- 检查租户 Locality 变更状态
SELECT tenant_id, tenant_name, locality, status
FROM oceanbase.DBA_OB_TENANTS
WHERE tenant_name = 'tenant_app';
```

---

## 11. 最佳实践

1. **生产环境至少 3 Zone**：确保 Paxos 多数派容灾能力
2. **首选 `STOP SERVER/ZONE`**：任何场景都先尝试最安全的隔离命令
3. **隔离后限时恢复**：节点/Zone 隔离是临时操作，长期下线应走替换流程
4. **升级前预检查**：确认所有日志流有 Leader、日志同步正常
5. **扩容后调整资源池**：添加节点后需修改 `UNIT_NUM` 才能利用新资源
6. **删除前确认 Unit 迁移完成**：`DBA_OB_UNIT_JOBS` 查询为空再操作
7. **Region/IDC 规划**：添加 Zone 时合理设置 Region/IDC，便于容灾管理
8. **Primary Zone 合理设置**：确保读写服务就近访问，避免跨城延迟
9. **监控隔离状态**：通过 `DBA_OB_SERVERS.STOP_TIME` 和 `DBA_OB_ZONES.STATUS` 持续监控
10. **二次故障预案**：Zone 隔离期间若再发生故障可能影响多数派，需提前制定应急预案

---

## 12. 版本差异

| 版本 | 新特性 |
|------|-------|
| V4.2 | `FORCE STOP SERVER/ZONE` 命令增强、`ISOLATE SERVER/ZONE` 新增 |
| V4.3 | Unit 迁移优化、资源池在线调整 |
| V4.4+ | Zone 级别 Locality 管理、多 Region 容灾增强 |

---

## 参考文档

- 查看节点：https://www.oceanbase.com/docs/common-oceanbase-database-cn-1000000002013112
- 添加节点：https://www.oceanbase.com/docs/common-oceanbase-database-cn-1000000002013119
- 替换节点：https://www.oceanbase.com/docs/common-oceanbase-database-cn-1000000002013110
- 停止/启动节点：https://www.oceanbase.com/docs/common-oceanbase-database-cn-10000000001701226
- Zone 管理：https://www.oceanbase.com/docs/common-oceanbase-database-cn-10000000001699675
- Unit 迁移：https://www.oceanbase.com/docs/common-oceanbase-database-cn-1000000002013571

---

> **关联技能**：`tenant-management.md`（租户管理）、`architecture/distributed-architecture.md`（分布式架构）、`architecture/paxos-replicas.md`（Paxos 协议）、`devops/version-upgrade.md`（版本升级）、`devops/obd-deployment.md`（OBD 部署）
