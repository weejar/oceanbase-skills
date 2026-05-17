# OBD 租户管理命令参考

> 适用版本：OBD V2.x+ / OceanBase V4.2+
> 工具：obd cluster tenant（OBD 租户生命周期与备份恢复）

---

## 概述

通过 `obd cluster tenant` 命令可对 OceanBase 集群中的租户进行管理，包括创建、删除、查看、优化，以及配置备份路径和执行备份恢复操作。

> 租户的 SQL 级别管理（资源单元、资源池、租户创建的完整参数）请参阅 `admin/tenant-management.md`。
> 备份恢复的详细策略请参阅 `admin/backup-recovery.md`、`admin/data-backup.md`、`admin/physical-restore.md`。

---

## 1. 租户 CRUD 命令

### 1.1 创建租户

```bash
obd cluster tenant create <deploy_name> -n <tenant_name> [options]
```

### 1.2 删除租户

```bash
obd cluster tenant drop <deploy_name> -n <tenant_name>
```

### 1.3 查看租户

```bash
obd cluster tenant show <deploy_name>
```

### 1.4 优化租户

```bash
obd cluster tenant optimize <deploy_name> <tenant_name> -o <workload>
```

支持的负载类型：
- `express_oltp` — 轻量级 OLTP
- `olap` — 分析型负载
- 其他按 OBD 版本支持

---

## 2. 备份配置

### 2.1 设置备份路径

```bash
obd cluster tenant set-backup-config <deploy_name> <tenant_name> \
  -d <data_uri> \
  -a <log_uri>
```

| 参数 | 说明 | 示例 |
|------|------|------|
| `-d` | 数据备份路径（URI 格式） | `file:///backup/data` |
| `-a` | 归档日志路径（URI 格式） | `file:///backup/log` |

支持的 URI 格式：
- 本地路径：`file:///backup/data`
- NFS 路径：`nfs://server:/backup/data`
- OSS：`oss://bucket/path`（需配置凭证）

### 2.2 执行备份

```bash
obd cluster tenant backup <deploy_name> <tenant_name>
```

启动指定租户的全量数据备份。

---

## 3. 恢复操作

```bash
obd cluster tenant restore <deploy_name> <tenant_name> <data_uri> <log_uri>
```

| 参数 | 说明 |
|------|------|
| `data_uri` | 数据备份路径 |
| `log_uri` | 归档日志路径 |

恢复操作会使用备份数据创建或覆盖目标租户。

---

## 4. 使用示例

### 4.1 创建租户

```bash
obd cluster tenant create test-cluster -n mysql
```

### 4.2 配置并执行备份

```bash
# 设置备份路径
obd cluster tenant set-backup-config test-cluster mysql \
  -d file:///backup/data \
  -a file:///backup/log

# 执行备份
obd cluster tenant backup test-cluster mysql
```

### 4.3 优化租户（OLTP 负载）

```bash
obd cluster tenant optimize test-cluster mysql -o express_oltp
```

---

## 5. 注意事项

- 备份路径必须能被集群中所有 OBServer 节点访问
- NFS 备份需确保所有节点挂载相同的 NFS 路径
- 归档日志提供时间点恢复（PITR）能力
- 备份前请确保磁盘空间充足，建议预留 2 倍数据量

---

## 参考文档

- OBD 租户管理：https://www.oceanbase.com/docs/obd-cn/V420/manage-tenants
- 备份恢复概述：https://www.oceanbase.com/docs/oceanbase-database-cn/V420/backup-and-recovery-overview

---

## 相关文件

- `admin/tenant-management.md` — 租户 SQL 级别管理（创建/扩缩容/克隆/资源池）
- `admin/backup-recovery.md` — 物理备份恢复概述与策略
- `admin/data-backup.md` — 数据备份操作详解
- `admin/physical-restore.md` — 物理恢复与表级恢复
- `admin/log-archive.md` — 归档日志管理
- `tools/obd-config-deployment.md` — OBD 配置文件部署参考
