# 版本升级策略与操作

> 适用版本：OceanBase V4.2+
> 兼容模式：MySQL Mode / Oracle Mode

---

## 概述

OceanBase 版本升级包括：补丁升级（Patch）、小版本升级（Minor）、大版本升级（Major）。升级需严格遵循流程，确保数据安全和业务连续性。

| 升级类型 | 示例 | 风险 | 停机时间 |
|---------|------|------|---------|
| 补丁升级 | 4.2.1.0 → 4.2.1.5 | 低 | 滚动，零停机 |
| 小版本升级 | 4.2.1 → 4.2.3 | 中 | 滚动或短暂停机 |
| 大版本升级 | 4.1 → 4.2 | 高 | 需要充分测试 |

---

## 1. 升级前准备

### 1.1 升级前检查清单

```sql
-- 1. 检查当前版本
SELECT version(), build_version();
SHOW VARIABLES LIKE 'version%';

-- 2. 检查集群健康
SELECT * FROM oceanbase.DBA_OB_SERVERS;
SELECT svr_ip, svr_port, status, zone, start_service_time
FROM oceanbase.__all_server;

-- 3. 检查副本状态（所有副本正常）
SELECT zone, svr_ip, role, is_alive
FROM oceanbase.__all_server_stat;

-- 4. 检查资源使用率
SELECT svr_ip,
  cpu_capacity, cpu_assigned, cpu_used_max,
  mem_capacity, mem_assigned, mem_used
FROM oceanbase.__all_virtual_server_stat;

-- 5. 检查数据冗余度
SELECT * FROM oceanbase.__all_virtual_table_mgr
WHERE tablet_status != 'NORMAL';

-- 6. 检查是否有长事务
SELECT svr_ip, session_id, tenant_id, user_id, start_time
FROM oceanbase.__all_virtual_trans_stat
WHERE start_time < DATE_SUB(NOW(), INTERVAL 1 HOUR);
```

### 1.2 预检查脚本（OBD 自带）

```bash
# OBD 预检查
obd cluster upgrade --check-only prod_cluster -c upgrade_config.yaml

# 检查输出示例：
# Check items:
#   [PASS] all servers are alive
#   [PASS] cluster status is healthy
#   [PASS] data replicas are complete
#   [WARN] there are running long transactions
#   [PASS] disk space is sufficient
```

### 1.3 备份确认

```bash
# 确认数据备份和日志归档正常
# MySQL Mode
mysql -h obhost -P 2881 -u root -p -e "
  SELECT * FROM oceanbase.DBA_OB_BACKUP_SET_DETAILS
  WHERE status = 'SUCCESS'
  ORDER BY end_time DESC LIMIT 3;
"

# 确认日志归档正常
SELECT * FROM oceanbase.DBA_OB_ARCHIVE_LOG_SUMMARY
WHERE status = 'DOING';
```

---

## 2. 滚动升级流程

### 2.1 滚动升级步骤

```
Node1(Leader) → Node2(Follower) → Node3(Follower)
     ↓
每个节点独立升级，其余节点继续服务
```

```bash
# 步骤1：下载新版本软件包
wget https://mirrors.aliyun.com/oceanbase/OCEANBASE_CE/releases/4.3.0/el/7/oceanbase-ce-4.3.0.el7.x86_64.rpm

# 步骤2：修改 OBD 配置指向新版本
obd cluster edit-config prod_cluster
# 修改 home_path 或 package_hash

# 步骤3：执行预检查
obd cluster upgrade prod_cluster --check-only

# 步骤4：执行滚动升级
obd cluster upgrade prod_cluster

# 步骤5：观察升级进度
obd cluster display prod_cluster
```

### 2.2 手动滚动升级

```bash
# 逐节点升级
for node in node1 node2 node3; do
  echo "Upgrading $node..."

  # 1. 在节点上停止 OBServer
  ssh oceanbase@$node "kill -TERM \$(pgrep observer)"

  # 2. 等待进程退出
  ssh oceanbase@$node "while pgrep observer > /dev/null; do sleep 2; done"

  # 3. 升级软件包
  ssh oceanbase@$node "sudo rpm -Uvh oceanbase-ce-4.3.0.el7.x86_64.rpm"

  # 4. 启动新版本
  ssh oceanbase@$node "cd ~/observer && bin/observer &"

  # 5. 等待节点恢复
  sleep 60
  mysql -h $node -P 2881 -u root -p -e "SELECT version();"

  echo "$node upgraded successfully"
done
```

---

## 3. 升级后验证

### 3.1 功能验证

```sql
-- 1. 版本确认
SELECT version();

-- 2. 集群健康
SELECT svr_ip, status, start_service_time
FROM oceanbase.__all_server;

-- 3. 租户状态
SELECT tenant_name, tenant_type, primary_zone, compatibility_mode
FROM oceanbase.DBA_OB_TENANTS;

-- 4. 副本状态
SELECT table_id, tablet_id, svr_ip, role, is_alive
FROM oceanbase.__all_virtual_tablet_replica
WHERE role = 'LEADER' AND is_alive = 0;

-- 5. 数据一致性
SELECT COUNT(*) FROM __all_server_stat WHERE status != 'ACTIVE';
-- 应返回 0
```

### 3.2 性能验证

```sql
-- 1. 检查延迟
SELECT svr_ip, zone, active_count, block_tx_cnt
FROM oceanbase.__all_virtual_server_stat;

-- 2. 检查慢查询
SELECT query_sql, elapsed_time, svr_ip
FROM oceanbase.DBA_OB_TENANT_SQL_AUDIT
WHERE elapsed_time > 1000000  -- > 1s
ORDER BY elapsed_time DESC
LIMIT 20;

-- 3. 检查索引状态
SELECT table_name, index_name, status
FROM oceanbase.DBA_OB_INDEX_STATUS
WHERE status != 'VALID';
-- 应返回 0 行
```

### 3.3 回滚方案

```bash
# 如果升级失败需要回滚
# 1. 停止所有节点
obd cluster stop prod_cluster

# 2. 修改配置回退到旧版本
obd cluster edit-config prod_cluster

# 3. 启动旧版本
obd cluster start prod_cluster

# 注意：大版本回滚可能导致数据兼容性问题
# 小版本回滚通常安全
```

---

## 4. 版本兼容性

### 4.1 升级路径

```
4.1.x → 4.2.x  （需中间步骤：4.1.x → 4.1.latest → 4.2.x）
4.2.x → 4.3.x  （直接升级）
4.2.1 → 4.2.5  （补丁升级，直接）
```

### 4.2 不兼容变更

| 版本 | 变更 | 影响 |
|------|------|------|
| 4.2→4.3 | 新增系统视图 | 无影响 |
| 4.1→4.2 | 改进执行计划 | 可能影响SQL性能 |
| 3.x→4.x | 全新架构 | 需数据迁移 |

---

## 5. 升级时间表

| 集群规模 | 升级类型 | 预估时间 |
|---------|---------|---------|
| 1节点 | 补丁 | 5-10 分钟 |
| 3节点 | 补丁 | 15-30 分钟 |
| 3节点 | 小版本 | 30-60 分钟 |
| 3节点 | 大版本 | 2-4 小时 |

---

## 6. 最佳实践

1. **先测试环境再生产**：完整执行一次升级流程
2. **低峰期操作**：选择业务低峰窗口
3. **通知应用方**：提前协调应用团队
4. **备份优先**：确认备份成功后再升级
5. **监控升级过程**：实时观察集群状态
6. **准备回滚**：明确回滚方案和判断条件
7. **逐步推进**：先升级1个观察15分钟，确认正常后继续

---

## 参考文档

- 升级指南：https://www.oceanbase.com/docs/oceanbase-database-cn/V420/upgrade-overview
- 版本发布说明：https://www.oceanbase.com/docs/release-notes
- OBD 升级：https://www.oceanbase.com/docs/obd-cn/V420/upgrade-a-cluster
