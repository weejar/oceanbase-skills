# Hibernate Integration

## Overview

Hibernate is a popular JPA ORM for Java. This guide covers Hibernate configuration with OceanBase (MySQL Mode), entity mapping, and performance tuning for V4.2+.

---

## Maven Dependency

```xml
<dependency>
    <groudId>org.hibernate</groudId>
    <artifactId>hibernate-core</artifactId>
    <version>6.2.9.Final</version>
</dependency>

<dependency>
    <groudId>mysql</groudId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.33</version>
</dependency>

<dependency>
    <groudId>com.zaxxer</groudId>
    <artifactId>HikariCP</artifactId>
    <version>5.0.1</version>
</dependency>
```

---

## Hibernate Configuration (`hibernate.cfg.xml`)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE hibernate-configuration PUBLIC
    "-//Hibernate/Hibernate Configuration DTD 3.0//EN"
    "http://hibernate.sourceforge.net/hibernate-configuration-3.0.dtd">
<hibernate-configuration>
    <session-factory>
        <property name="hibernate.connection.driver_class">com.mysql.cj.jdbc.Driver</property>
        <property name="hibernate.connection.url">jdbc:mysql://10.0.0.100:2883/test_db?useSSL=false&amp;rewriteBatchedStatements=true</property>
        <property name="hibernate.connection.username">app_user@mysql_tenant</property>
        <property name="hibernate.connection.password">password</property>
        <property name="hibernate.dialect">org.hibernate.dialect.MySQL8Dialect</property>
        <property name="hibernate.hbm2ddl.auto">validate</property>
        <property name="hibernate.show_sql">false</property>
        <property name="hibernate.format_sql">true</property>
    </session-factory>
</hibernate-configuration>
```

> **Tip**: Use `MySQL8Dialect` for OceanBase MySQL Mode.

---

## Entity Mapping Example

```java
@Entity
@Table(name = "employees")
public class Employee {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "emp_id")
    private Long empId;

    @Column(name = "emp_name", nullable = false, length = 100)
    private String empName;

    @Column(name = "dept_id")
    private Long deptId;

    // Getters and setters
}
```

---

## Spring Boot + Hibernate (JPA)

```yaml
# application.yml
spring:
  datasource:
    url: jdbc:mysql://10.0.0.100:2883/test_db?useSSL=false&rewriteBatchedStatements=true
    username: app_user@mysql_tenant
    password: password
    hikari:
      maximum-pool-size: 20
      connection-timeout: 5000
  jpa:
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        dialect: org.hibernate.dialect.MySQL8Dialect
        format_sql: true
```

---

## Batch Processing (Performance)

```java
// Enable batch processing
@BatchSize(size = 50)
public class Employee { ... }

// In code:
EntityManager em = emf.createEntityManager();
em.getTransaction().begin();
for (int i = 0; i < 1000; i++) {
    Employee e = new Employee();
    e.setEmpName("Emp" + i);
    em.persist(e);
    if (i % 50 == 0) {
        em.flush();
        em.clear();
    }
}
em.getTransaction().commit();
```

> **Tip**: `rewriteBatchedStatements=true` in JDBC URL is critical for batch performance.

---

## Common Mistakes

1. **Using `hibernate.hbm2ddl.auto=create` in production**
   - ❌ Drops and recreates tables
   - ✅ Use `validate` or `update`

2. **Not enabling batch processing**
   - ❌ `INSERT` per entity (slow)
   - ✅ `rewriteBatchedStatements=true` + `@BatchSize`

3. **Using `SELECT *` via entity load**
   - ❌ Loads all columns
   - ✅ Use `@Query` with JPQL specifying columns

4. **Not setting `show_sql=false` in production**
   - ❌ SQL logged to console (performance hit)
   - ✅ `show_sql=false`

5. **Forgetting to close `EntityManager`**
   - ❌ Connection leak
   - ✅ Use `@PersistenceContext` + `@Transactional` (Spring auto-closes)

---

## OceanBase Oracle Mode

Use `Oracle12cDialect`:
```xml
<property name="hibernate.dialect">org.hibernate.dialect.Oracle12cDialect</property>
```

> **Note**: OceanBase Oracle Mode speaks MySQL protocol, but Hibernate uses Oracle dialect for SQL generation.

---

## Sources
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/hibernate
- https://www.oceanbase.com/docs/oceanbase-database-cn/v4.2.5/hibernate
- https://hibernate.org/orm/
