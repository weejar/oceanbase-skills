# 行级访问控制

> 适用版本：OceanBase V4.2+
> 兼容模式：Oracle Mode（完整支持）/ MySQL Mode（V4.3+部分支持）

---

## 概述

行级安全（Row-Level Security, RLS）通过安全策略限制用户只能访问表中符合条件的行，实现细粒度的数据访问控制。

| 特性 | Oracle Mode | MySQL Mode |
|------|------------|------------|
| 实现方式 | DBMS_RLS 包 / VPD Policy | View + 当前用户过滤 |
| 策略管理 | `DBMS_RLS.ADD_POLICY` | 手动创建视图 |
| 动态谓词 | 支持运行时动态生成 | 静态 WHERE 条件 |
| 列掩码 | `DBMS_RLS.ADD_POLICY` column_level | 不原生支持 |

---

## 1. Oracle Mode — VPD 策略（Virtual Private Database）

### 1.1 创建策略函数

```sql
-- 创建策略函数，返回 WHERE 子句
CREATE OR REPLACE FUNCTION dept_access_policy(
  p_schema  IN VARCHAR2,
  p_object  IN VARCHAR2
) RETURN VARCHAR2
IS
  v_dept_id NUMBER;
BEGIN
  -- 管理员可看所有数据
  IF SYS_CONTEXT('USERENV', 'SESSION_USER') = 'ADMIN' THEN
    RETURN NULL;
  END IF;

  -- 普通用户只能看自己部门的数据
  SELECT dept_id INTO v_dept_id
  FROM emp_info
  WHERE emp_name = SYS_CONTEXT('USERENV', 'SESSION_USER');

  RETURN 'dept_id = ' || v_dept_id;
EXCEPTION
  WHEN NO_DATA_FOUND THEN
    RETURN '1 = 0'; -- 无匹配则返回空结果
END dept_access_policy;
/
```

### 1.2 添加策略

```sql
BEGIN
  DBMS_RLS.ADD_POLICY(
    object_schema   => 'HR',
    object_name     => 'EMPLOYEES',
    policy_name     => 'dept_policy',
    function_schema => 'SEC',
    policy_function => 'dept_access_policy',
    statement_types => 'SELECT, INSERT, UPDATE, DELETE',
    enable          => TRUE
  );
END;
/
```

### 1.3 策略管理

```sql
-- 禁用策略
BEGIN
  DBMS_RLS.ENABLE_POLICY('HR', 'EMPLOYEES', 'dept_policy', FALSE);
END;
/

-- 删除策略
BEGIN
  DBMS_RLS.DROP_POLICY('HR', 'EMPLOYEES', 'dept_policy');
END;
/

-- 查看已有策略
SELECT object_name, policy_name, policy_function, enabled
FROM dba_policies
WHERE object_owner = 'HR';
```

### 1.4 基于会话上下文的动态策略

```sql
-- 设置应用上下文
CREATE OR REPLACE CONTEXT app_ctx USING app_ctx_pkg;

CREATE OR REPLACE PACKAGE app_ctx_pkg IS
  PROCEDURE set_dept(p_dept_id NUMBER);
END;
/

CREATE OR REPLACE PACKAGE BODY app_ctx_pkg IS
  PROCEDURE set_dept(p_dept_id NUMBER) IS
  BEGIN
    DBMS_SESSION.SET_CONTEXT('APP_CTX', 'DEPT_ID', p_dept_id);
  END;
END;
/

-- 策略函数使用上下文
CREATE OR REPLACE FUNCTION ctx_dept_policy(
  p_schema IN VARCHAR2,
  p_object IN VARCHAR2
) RETURN VARCHAR2 IS
BEGIN
  RETURN 'dept_id = SYS_CONTEXT(''APP_CTX'', ''DEPT_ID'')';
END;
/

-- 应用登录后设置上下文
EXEC app_ctx_pkg.set_dept(10);
-- 后续查询自动过滤
SELECT * FROM hr.employees; -- 只返回 dept_id=10 的数据
```

---

## 2. MySQL Mode — 视图实现

MySQL Mode 不支持 `DBMS_RLS`，通过视图 + `CURRENT_USER()` 实现类似效果。

### 2.1 基本视图过滤

```sql
-- 创建用户-部门映射表
CREATE TABLE user_dept_map (
  username VARCHAR(64) NOT NULL,
  dept_id  INT NOT NULL,
  PRIMARY KEY (username)
);

INSERT INTO user_dept_map VALUES
  ('app_user@%', 10),
  ('ops_user@%', 20);

-- 创建带过滤条件的视图
CREATE VIEW v_orders AS
SELECT * FROM orders
WHERE dept_id = (
  SELECT dept_id FROM user_dept_map
  WHERE username = CURRENT_USER()
);

-- 用户只通过视图访问
GRANT SELECT ON app_db.v_orders TO 'app_user'@'%';
REVOKE SELECT ON app_db.orders FROM 'app_user'@'%';
```

### 2.2 使用存储过程封装

```sql
DELIMITER //
CREATE PROCEDURE query_orders(IN p_order_id INT)
BEGIN
  DECLARE v_dept_id INT;
  SELECT dept_id INTO v_dept_id
  FROM user_dept_map WHERE username = CURRENT_USER();

  IF v_dept_id IS NULL THEN
    SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Access denied';
  END IF;

  SELECT * FROM orders
  WHERE dept_id = v_dept_id
    AND (p_order_id IS NULL OR order_id = p_order_id);
END //
DELIMITER ;

GRANT EXECUTE ON app_db.query_orders TO 'app_user'@'%';
```

---

## 3. 列级安全（Oracle Mode）

### 3.1 列掩码策略

```sql
-- 创建列掩码函数
CREATE OR REPLACE FUNCTION mask_salary(
  p_schema IN VARCHAR2,
  p_object IN VARCHAR2,
  p_column IN VARCHAR2
) RETURN VARCHAR2 IS
BEGIN
  IF SYS_CONTEXT('USERENV', 'SESSION_USER') = 'HR_ADMIN' THEN
    RETURN NULL; -- 不加掩码
  END IF;
  -- 其他用户看到掩码
  RETURN '1=0'; -- 完全隐藏
END;
/

-- 添加列级策略
BEGIN
  DBMS_RLS.ADD_POLICY(
    object_schema   => 'HR',
    object_name     => 'EMPLOYEES',
    policy_name     => 'salary_mask',
    function_schema => 'SEC',
    policy_function => 'mask_salary',
    sec_relevant_cols => 'salary',
    sec_relevant_cols_opt => DBMS_RLS.ALL_ROWS
  );
END;
/

-- HR_ADMIN 看到真实薪资
-- 其他用户 salary 列显示为 NULL
```

---

## 4. 数据脱敏（通用方案）

对无法使用 RLS 的场景，使用数据脱敏视图：

```sql
-- MySQL Mode / Oracle Mode 通用
CREATE VIEW v_customer_masked AS
SELECT
  customer_id,
  CONCAT(LEFT(phone, 3), '****', RIGHT(phone, 4)) AS phone,
  CONCAT(LEFT(id_card, 4), '**********', RIGHT(id_card, 4)) AS id_card,
  email,
  -- 其他非敏感字段...
  created_at
FROM customers;
```

---

## 5. 性能影响与优化

| 因素 | 影响 | 建议 |
|------|------|------|
| 策略函数复杂度 | 每次查询都执行 | 函数使用上下文变量，避免重复查询 |
| 过滤列无索引 | 全表扫描 | 确保策略过滤列有索引 |
| 多策略叠加 | 多函数调用 | 合并策略，减少函数数量 |
| 视图嵌套 | 执行计划复杂 | 避免视图多层嵌套 |

```sql
-- 确保过滤列有索引
CREATE INDEX idx_orders_dept_id ON orders(dept_id);

-- 使用策略组减少函数调用
-- Oracle Mode: 在单个策略函数中处理多表逻辑
```

---

## 6. 常见错误

| 错误 | 原因 | 解决 |
|------|------|------|
| `ORA-28104: invalid input value for policy type` | 策略参数错误 | 检查 `ADD_POLICY` 参数 |
| `ORA-01031: insufficient privileges` | 无 DBMS_RLS 执行权限 | `GRANT EXECUTE ON DBMS_RLS` |
| 视图查询慢 | 缺少索引 | 在过滤列上创建索引 |
| MySQL Mode 无法使用 DBMS_RLS | 兼容模式不支持 | 使用视图方案替代 |

---

## 最佳实践

1. **Oracle Mode 优先使用 VPD 策略**：声明式管理，易于维护
2. **MySQL Mode 使用视图**：创建只读视图，回收底层表权限
3. **策略函数保持简单**：避免在函数中执行复杂查询
4. **定期审计策略**：检查 `DBA_POLICIES` 确认策略正确启用
5. **测试充分**：验证管理员和普通用户的数据可见性

---

## 参考文档

- Oracle Mode 安全策略：https://www.oceanbase.com/docs/oceanbase-database-cn/V420/oracle-mode/manage-privileges-1
- MySQL Mode 权限：https://www.oceanbase.com/docs/oceanbase-database-cn/V420/mysql-mode/grant-privilege
- DBMS_RLS 包：https://www.oceanbase.com/docs/oceanbase-database-cn/V420/oracle-mode/dbms_rls
