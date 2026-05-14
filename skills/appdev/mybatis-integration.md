# MyBatis Integration

## Overview

MyBatis is a popular Java ORM framework. This guide covers MyBatis integration with OceanBase (MySQL Mode) including configuration, mapper examples, and performance tips for V4.2+.

---

## Maven Dependency

```xml
<!-- MyBatis -->
<dependency>
    <groudId>org.mybatis</groudId>
    <artifactId>mybatis</artifactId>
    <version>3.5.13</version>
</dependency>

<!-- MySQL JDBC Driver (for OceanBase) -->
<dependency>
    <groudId>mysql</groudId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.33</version>
</dependency>

<!-- HikariCP (recommended connection pool) -->
<dependency>
    <groudId>com.zaxxer</groudId>
    <artifactId>HikariCP</artifactId>
    <version>5.0.1</version>
</dependency>
```

---

## MyBatis Configuration (`mybatis-config.xml`)

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
    "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <environments default="ob_env">
        <environment id="ob_env">
            <transactionManager type="JDBC"/>
            <dataSource type="POOLED">
                <property name="driver" value="com.mysql.cj.jdbc.Driver"/>
                <property name="url" value="jdbc:mysql://10.0.0.100:2883/test_db?useSSL=false&amp;rewriteBatchedStatements=true"/>
                <property name="username" value="app_user@mysql_tenant"/>
                <property name="password" value="password"/>
            </dataSource>
        </environment>
    </environments>
    <mappers>
        <mapper resource="mappers/EmployeeMapper.xml"/>
    </mappers>
</configuration>
```

> **Tip**: Use OBProxy (port 2883) for transparent failover.

---

## Mapper XML Example

```xml
<!-- EmployeeMapper.xml -->
<mapper namespace="com.example.EmployeeMapper">
    <select id="selectById" resultType="Employee">
        SELECT emp_id, emp_name, dept_id
        FROM employees
        WHERE emp_id = #{empId}
    </select>

    <insert id="insert" parameterType="Employee">
        INSERT INTO employees (emp_id, emp_name, dept_id)
        VALUES (#{empId}, #{empName}, #{deptId})
    </insert>

    <update id="update">
        UPDATE employees
        SET emp_name = #{empName}
        WHERE emp_id = #{empId}
    </update>

    <delete id="delete">
        DELETE FROM employees WHERE emp_id = #{empId}
    </delete>
</mapper>
```

---

## Batch Insert (Performance Critical)

```xml
<!-- Batch insert for high throughput -->
<insert id="batchInsert">
    INSERT INTO employees (emp_id, emp_name, dept_id)
    VALUES
    <foreach item="emp" collection="list" separator=",">
        (#{emp.empId}, #{emp.empName}, #{emp.deptId})
    </foreach>
</insert>
```

> **Tip**: Ensure `rewriteBatchedStatements=true` in JDBC URL for batch performance.

---

## Pagination (OceanBase MySQL Mode)

```xml
<!-- Use LIMIT for pagination (MySQL syntax) -->
<select id="selectByPage" resultType="Employee">
    SELECT emp_id, emp_name, dept_id
    FROM employees
    ORDER BY emp_id
    LIMIT #{limit} OFFSET #{offset}
</select>
```

```java
// Java code: page 1, size 20
List<Employee> list = mapper.selectByPage(20, 0);   // page 1
List<Employee> list = mapper.selectByPage(20, 20);  // page 2
```

---

## Common Mistakes

1. **Not using `rewriteBatchedStatements=true`**
   - ❌ Batch insert slow (single INSERT per round-trip)
   - ✅ Add to JDBC URL

2. **Using `SELECT *` in mappers**
   - ❌ Loads all columns (slow)
   - ✅ Specify columns explicitly

3. **Not using connection pool**
   - ❌ Creating new connection per request
   - ✅ Use HikariCP

4. **Pagination with `OFFSET` for deep pages**
   - ❌ `OFFSET 100000` (slow)
   - ✅ Use keyset pagination: `WHERE id > :last_id LIMIT 20`

5. **Not enabling MyBatis cache**
   - ❌ Repeated same queries
   - ✅ `<cache/>` in mapper XML

---

## OceanBase Oracle Mode Notes

MyBatis works the same. Only SQL syntax changes (use Oracle pagination: `ROWNUM` or `FETCH FIRST`).

---

## Sources
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/mybatis
- https://www.oceanbase.com/docs/oceanbase-database-cn/v4.2.5/mybatis
- https://mybatis.org/mybatis-3/
