# ODC 开发中心

> 适用版本：ODC V4.x / OceanBase V4.2+
> 官网：https://www.oceanbase.com/product/odc

---

## 概述

ODC（OceanBase Developer Center）是 OceanBase 的集成开发环境，提供 SQL 开发、数据库管理、数据导入导出、审批流程等功能。

| 功能 | 说明 |
|------|------|
| SQL 编辑器 | 智能补全、语法高亮、执行计划 |
| 对象管理 | 表、索引、视图、存储过程管理 |
| 数据导入导出 | CSV/Excel 导入导出 |
| 审批流程 | DDL/DML 审批上线 |
| 数据库变更 | Schema 变更工单 |
| 安全管控 | 敏感数据脱敏、操作审计 |

---

## 1. 安装部署

### 1.1 方式一：Web 版（SaaS）

```
直接访问：https://odc.oceanbase.com
  - 支持连接自建 OceanBase 和云 OceanBase
  - 免费使用基础功能
```

### 1.2 方式二：私有部署

```bash
# 1. 下载
wget https://mirrors.aliyun.com/oceanbase/ODC/current/odc.tar.gz

# 2. 解压
tar -xzf odc.tar.gz -C /opt/odc

# 3. 配置
vi /opt/odc/conf/application.yaml
# 配置 MetaDB、端口、认证方式等

# 4. 启动
cd /opt/odc && bash start.sh

# 5. 访问 http://<host>:8989
```

---

## 2. 连接数据库

### 2.1 创建连接

```
ODC Web 界面 → 新建连接

连接配置：
  - 连接名: prod_business
  - 主机: ob_proxy_host:2883
  - 租户: business@prod_cluster
  - 用户: app_user
  - 密码: ******
  - 默认数据库: app_db
  - 兼容模式: MySQL（自动检测）
```

### 2.2 多租户连接

```
可创建多个连接分别对应不同租户：
  - sys@prod (管理员)
  - business@prod (业务)
  - test@test_cluster (测试)
  - readonly@prod (只读)
```

---

## 3. SQL 开发

### 3.1 SQL 编辑器

```
ODC 功能：
  - 智能补全：表名、列名、函数自动补全
  - 语法高亮：MySQL/Oracle 语法
  - 执行计划：EXPLAIN 可视化展示
  - SQL 格式化：一键格式化
  - SQL 模板：常用 SQL 模板
  - 会话管理：多个 SQL 会话并行
```

### 3.2 执行计划分析

```
选中 SQL → 查看执行计划

可视化展示：
  - 操作类型和耗时
  - 表扫描方式（全表/索引）
  - JOIN 方式
  - 分区裁剪
  - 内存使用

支持操作：
  - 导出执行计划图片
  - 与历史执行计划对比
  - 自动优化建议
```

### 3.3 SQL 审计

```
SQL 审计面板：
  - 执行时间分布
  - Top 慢 SQL
  - 错误 SQL
  - 执行频率统计
  - 影响行数统计
```

---

## 4. 数据库对象管理

### 4.1 表管理

```
功能：
  - 可视化建表（表设计器）
  - 修改表结构（ALTER TABLE）
  - 查看表数据（分页浏览）
  - 导入/导出表数据
  - 查看表统计信息
  - 索引管理
  - 分区管理
```

### 4.2 存储过程管理

```
Oracle Mode 支持：
  - 创建/编辑存储过程
  - 调试存储过程
  - 查看编译错误
  - 包管理
  - 触发器管理
```

---

## 5. 数据导入导出

### 5.1 导入

```
操作：工具 → 导入数据

支持格式：
  - CSV
  - Excel (.xlsx)
  - SQL 脚本
  - JSON

配置选项：
  - 编码: UTF-8 / GBK
  - 分隔符: 逗号 / Tab / 自定义
  - 批量大小: 1000
  - 遇到错误: 跳过 / 终止
  - 并行度: 4
```

### 5.2 导出

```
操作：工具 → 导出数据

导出选项：
  - 格式: CSV / Excel / SQL INSERT / JSON
  - 编码: UTF-8
  - 大文件拆分: 按行数拆分
  - 仅导出查询结果
  - 定时导出任务
```

---

## 6. 变更审批

### 6.1 审批流程

```
开发人员提交 SQL 变更
     ↓
DBA 审批（检查 SQL 合规性）
     ↓
自动执行（或在指定维护窗口执行）
     ↓
执行结果通知
```

### 6.2 审批规则

```
可配置的审批规则：
  - DDL 操作必须审批
  - DELETE/UPDATE 无 WHERE 必须审批
  - 涉及大表（>1000万行）必须审批
  - 指定时间段外操作必须审批
  - 生产环境所有操作必须审批
```

---

## 7. 安全功能

| 功能 | 说明 |
|------|------|
| 操作审计 | 所有操作记录可查 |
| 敏感数据脱敏 | 自动脱敏手机号、身份证等 |
| 权限控制 | 基于角色的访问控制 |
| SQL 拦截 | 危险 SQL 自动拦截 |
| 回收站 | DROP 表进入回收站，可恢复 |

---

## 参考文档

- ODC 官方文档：https://www.oceanbase.com/docs/odc-cn
- ODC 安装：https://www.oceanbase.com/docs/odc-cn/V420/install-odc
- SQL 开发：https://www.oceanbase.com/docs/odc-cn/V420/sql-development
