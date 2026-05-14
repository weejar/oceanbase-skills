# Schema 版本管理

> 适用版本：OceanBase V4.2+
> 兼容模式：MySQL Mode / Oracle Mode

---

## 概述

数据库 Schema 版本管理确保表结构变更可追溯、可回滚、可协作。OceanBase 完全兼容 MySQL/Oracle DDL 语法，可使用现有工具链。

| 方案 | 工具 | 适用场景 |
|------|------|---------|
| Flyway | Java 应用集成 | Java 技术栈 |
| Liquibase | Java/XML/YAML | 复杂变更流程 |
| pt-online-schema-change | 独立工具 | MySQL Mode 大表变更 |
| gh-ost | 独立工具 | MySQL Mode 无锁变更 |
| 手动版本管理 | SQL 脚本 | 简单场景 |

---

## 1. Schema 管理原则

### 1.1 基本原则

| 原则 | 说明 |
|------|------|
| 向后兼容 | 新版本不破坏旧版本应用 |
| 不可变迁移 | 已执行的迁移脚本不可修改 |
| 可回滚 | 每个变更提供回滚脚本 |
| 测试先行 | 在测试环境验证后再上线 |
| 拆分 DDL | 每个 DDL 语句单独一个版本文件 |

### 1.2 OceanBase DDL 特点

```sql
-- OceanBase 支持大部分 Online DDL（不锁表）
-- 但以下操作仍需注意：
ALTER TABLE orders ADD COLUMN remark VARCHAR(500);     -- Online，即时
ALTER TABLE orders ADD INDEX idx_status (status);       -- Online，后台构建
ALTER TABLE orders MODIFY COLUMN amount DECIMAL(16,4); -- Online
ALTER TABLE orders DROP COLUMN old_col;                 -- Online，元数据删除
ALTER TABLE orders DROP INDEX idx_old;                  -- Online
ALTER TABLE orders ADD PARTITION (...);                 -- Online
```

---

## 2. Flyway 集成

### 2.1 配置

```yaml
# application.yml (Spring Boot)
spring:
  flyway:
    url: jdbc:mysql://obhost:2883/app_db?useSSL=false
    user: flyway_user
    password: Flyw@y2024
    locations: classpath:db/migration
    baseline-on-migrate: true
    table: flyway_schema_history
    out-of-order: false
    validate-on-migrate: true
```

### 2.2 迁移脚本命名

```
db/migration/
  V1.0.0__create_user_table.sql
  V1.0.1__create_order_table.sql
  V1.0.2__add_order_index.sql
  V1.1.0__add_payment_table.sql
  V1.1.1__create_view_order_summary.sql
  U1.0.0__rollback_user_table.sql    -- 回滚脚本
  R__refresh_order_summary_view.sql   -- 可重复脚本
```

### 2.3 迁移脚本示例

```sql
-- V1.0.1__create_order_table.sql
CREATE TABLE IF NOT EXISTS orders (
  id         BIGINT       NOT NULL AUTO_INCREMENT,
  user_id    BIGINT       NOT NULL,
  order_no   VARCHAR(32)  NOT NULL,
  amount     DECIMAL(12,2) NOT NULL DEFAULT 0,
  status     TINYINT      NOT NULL DEFAULT 0,
  created_at DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  UNIQUE KEY uk_order_no (order_no),
  KEY idx_user_id (user_id),
  KEY idx_status (status),
  KEY idx_created_at (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
  PARTITION BY HASH(id) PARTITIONS 16;
```

### 2.4 回滚脚本

```sql
-- U1.0.1__rollback_order_table.sql
DROP TABLE IF EXISTS orders;
```

---

## 3. Liquibase 集成

### 3.1 ChangeSet 示例

```xml
<!-- db/changelog/db.changelog-master.yaml -->
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
    http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.0.xsd">

  <changeSet id="1" author="dba">
    <createTable tableName="orders">
      <column name="id" type="BIGINT" autoIncrement="true">
        <constraints primaryKey="true" nullable="false"/>
      </column>
      <column name="user_id" type="BIGINT">
        <constraints nullable="false"/>
      </column>
      <column name="amount" type="DECIMAL(12,2)"/>
      <column name="status" type="TINYINT"/>
      <column name="created_at" type="DATETIME"/>
    </createTable>
    <rollback>
      <dropTable tableName="orders"/>
    </rollback>
  </changeSet>

  <changeSet id="2" author="dba">
    <createIndex indexName="idx_orders_user"
                 tableName="orders" unique="false">
      <column name="user_id"/>
    </createIndex>
  </changeSet>
</databaseChangeLog>
```

---

## 4. 大表变更策略

### 4.1 OceanBase Online DDL

```sql
-- 添加列（即时完成）
ALTER TABLE big_table ADD COLUMN new_col VARCHAR(256) DEFAULT '';

-- 添加索引（后台构建，不阻塞读写）
ALTER TABLE big_table ADD INDEX idx_new_col (new_col);

-- 查看在线DDL进度
SELECT * FROM oceanbase.DBA_OB_LONG_OPS
WHERE operation_type LIKE '%INDEX%';

-- 取消在线DDL
-- OceanBase 不直接支持取消，可通过 KILL 会话终止
```

### 4.2 pt-online-schema-change

```bash
# 在副本/从库上操作更安全
pt-online-schema-change \
  --host=obhost --port=2881 \
  --user=dba --password=Passw0rd \
  --alter "ADD COLUMN new_col VARCHAR(256) DEFAULT ''" \
  D=app_db,t=big_table \
  --chunk-size=1000 \
  --max-lag=5s \
  --dry-run   # 先试运行
```

### 4.3 列类型变更注意事项

```sql
-- 安全：扩大范围
ALTER TABLE orders MODIFY amount DECIMAL(16,4);  -- ✅ 扩大

-- 风险：缩小范围（需检查数据）
-- 先验证
SELECT COUNT(*) FROM orders WHERE amount > 9999999999.9999;
-- 确认无数据后再变更
ALTER TABLE orders MODIFY amount DECIMAL(10,2);  -- ⚠️ 可能丢数据
```

---

## 5. Schema 变更流程

```
1. 开发者提交变更请求
     ↓
2. DBA 审核变更脚本
     ↓
3. 测试环境执行（Flyway migrate）
     ↓
4. 应用适配测试
     ↓
5. 制定上线计划（维护窗口）
     ↓
6. 预生产验证（如可用）
     ↓
7. 生产执行
     ↓
8. 验证 + 记录
```

---

## 6. 变更检查清单

| 检查项 | 说明 |
|--------|------|
| 向后兼容 | 新增列有默认值或允许 NULL |
| 索引影响 | 新索引不影响写入性能 |
| 分区影响 | DDL 不影响分区裁剪 |
| 回滚方案 | 有对应的回滚脚本 |
| 数据量评估 | 大表变更预估耗时 |
| 应用停机 | 是否需要应用配合 |

---

## 常见错误

| 问题 | 原因 | 解决 |
|------|------|------|
| Flyway 校验失败 | 已执行脚本被修改 | 不修改已执行的脚本 |
| DDL 超时 | 大表变更未完成 | 增加 `ob_ddl_timeout` |
| `duplicate column` | 列已存在 | 加 `IF NOT EXISTS` |
| 索引构建慢 | 大表数据量大 | 使用 `CREATE INDEX ... ONLINE` |

---

## 参考文档

- DDL 语法：https://www.oceanbase.com/docs/oceanbase-database-cn/V420/mysql-mode/alter-table
- Online DDL：https://www.oceanbase.com/docs/oceanbase-database-cn/V420/online-ddl
- Flyway：https://documentation.red-gate.com/flyway
- Liquibase：https://www.liquibase.org/get-started
