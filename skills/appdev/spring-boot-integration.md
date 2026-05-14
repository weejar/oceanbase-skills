# Spring Boot Integration

## Overview

Spring Boot simplifies OceanBase connectivity via `spring-boot-starter-data-jpa` or `mybatis-spring-boot-starter`. This guide covers configuration and best practices for V4.2+.

---

## Dependency (Maven)

```xml
<dependency>
    <groudId>org.springframework.boot</groudId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>

<dependency>
    <groudId>mysql</groudId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.33</version>
</dependency>

<dependency>
    <groudId>com.zaxxer</groudId>
    <artifactId>HikariCP</artifactId>
</dependency>
```

---

## Configuration (`application.yml`)

```yaml
spring:
  datasource:
    url: jdbc:mysql://10.0.0.100:2883/test_db?
      useSSL=false&
      rewriteBatchedStatements=true&
      useServerPrepStmts=true
    username: app_user@mysql_tenant
    password: password
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 5000
      idle-timeout: 600000
  jpa:
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        dialect: org.hibernate.dialect.MySQL8Dialect
        format_sql: true
    show-sql: false
```

> **Tip**: Use OBProxy (port 2883) for transparent failover.

---

## Entity Class

```java
@Entity
@Table(name = "employees")
public class Employee {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long empId;

    @Column(name = "emp_name", nullable = false, length = 100)
    private String empName;

    @Column(name = "dept_id")
    private Long deptId;

    // Getters and setters
}
```

---

## Repository (JPA)

```java
public interface EmployeeRepository extends JpaRepository<Employee, Long> {
    // Derived query
    List<Employee> findByDeptId(Long deptId);

    // JPQL
    @Query("SELECT e FROM Employee e WHERE e.empName LIKE %:name%")
    List<Employee> searchByName(@Param("name") String name);
}
```

---

## Service Layer (Transactional)

```java
@Service
public class EmployeeService {
    @Autowired
    private EmployeeRepository repo;

    @Transactional
    public Employee create(Employee emp) {
        return repo.save(emp);
    }

    @Transactional(readOnly = true)
    public List<Employee> getByDept(Long deptId) {
        return repo.findByDeptId(deptId);
    }

    @Transactional
    public void batchInsert(List<Employee> list) {
        repo.saveAll(list);  // Requires batch enable in Hibernate
    }
}
```

> **Tip**: Enable batch in JPA: `spring.jpa.properties.hibernate.jdbc.batch_size=50`.

---

## Testing (TestContainers Alternative)

```java
// Use OceanBase Docker image for integration tests
@TestConfiguration
public class OBTestConfig {
    // No TestContainers for OceanBase yet; use dedicated test OB instance
    @Bean
    public DataSource dataSource() {
        return DataSourceBuilder.create()
            .url("jdbc:mysql://test-ob:2881/test_db")
            .username("app_user@mysql_tenant")
            .password("password")
            .build();
    }
}
```

---

## Common Mistakes

1. **`ddl-auto=create` in production**
   - ❌ Drops and recreates tables
   - ✅ Use `validate` or `update`

2. **Not enabling batch for `saveAll()`**
   - ❌ One INSERT per entity
   - ✅ `hibernate.jdbc.batch_size=50`

3. **`@Transactional` on `readOnly=true` for writes**
   - ❌ Writes fail
   - ✅ Remove `readOnly=true` for write operations

4. **Using `EntityManager` directly without `@Transactional`**
   - ❌ `LazyInitializationException`
   - ✅ Use `@Transactional` on service layer

5. **Not setting `show-sql=false` in production**
   - ❌ SQL logged to console (perf hit)
   - ✅ `show-sql: false`

---

## OceanBase Oracle Mode

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        dialect: org.hibernate.dialect.Oracle12cDialect
```

> **Note**: Still use MySQL JDBC driver; Hibernate dialect is Oracle.

---

## Sources
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/spring-boot
- https://www.oceanbase.com/docs/oceanbase-database-cn/v4.2.5/spring-boot
- https://spring.io/projects/spring-boot
