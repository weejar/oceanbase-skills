# User and Privilege Management Basics

## Overview

OceanBase manages users and privileges at the **tenant level**, similar to how traditional databases manage users at the instance level. This guide covers user creation, privilege grants, and role management for both MySQL Mode and Oracle Mode in OceanBase V4.2+.

---

## User Management (MySQL Mode)

### Create a User

```sql
-- Create local user
CREATE USER 'app_user'@'%' IDENTIFIED BY 'StrongPass!2024';

-- Create user with specific host (recommended for security)
CREATE USER 'app_user'@'10.0.0.%' IDENTIFIED BY 'StrongPass!2024';

-- View users
SELECT user, host, authentication_string FROM mysql.user;
```

### Alter User

```sql
-- Change password
ALTER USER 'app_user'@'%' IDENTIFIED BY 'NewPass!2024';

-- Lock/unlock user
ALTER USER 'app_user'@'%' ACCOUNT LOCK;
ALTER USER 'app_user'@'%' ACCOUNT UNLOCK;

-- Set password expiry
ALTER USER 'app_user'@'%' PASSWORD EXPIRE;
```

### Drop User

```sql
DROP USER 'app_user'@'%';
```

---

## Privilege Management (MySQL Mode)

### Grant Privileges

```sql
-- Grant all privileges on a database
GRANT ALL PRIVILEGES ON app_db.* TO 'app_user'@'%';

-- Grant specific privileges
GRANT SELECT, INSERT, UPDATE, DELETE ON app_db.* TO 'app_user'@'%';

-- Grant with GRANT OPTION (admin only)
GRANT ALL PRIVILEGES ON *.* TO 'dba_user'@'%' WITH GRANT OPTION;

-- Apply privileges
FLUSH PRIVILEGES;
```

### Revoke Privileges

```sql
REVOKE INSERT, DELETE ON app_db.* FROM 'app_user'@'%';
FLUSH PRIVILEGES;
```

### View Grants

```sql
-- Current user's privileges
SHOW GRANTS;

-- Specific user's privileges
SHOW GRANTS FOR 'app_user'@'%';
```

---

## User Management (Oracle Mode)

### Create a User

```sql
-- Oracle Mode uses CREATE USER (no @host)
CREATE USER app_user IDENTIFIED BY "StrongPass!2024";

-- View users
SELECT username FROM dba_users;
```

### Grant Privileges (Oracle Mode)

```sql
-- Grant system privileges
GRANT CREATE SESSION, CREATE TABLE, CREATE VIEW TO app_user;

-- Grant object privileges
GRANT SELECT, INSERT ON schema1.employees TO app_user;

-- Grant DBA role
GRANT DBA TO app_user;
```

---

## Built-in Roles

### MySQL Mode

| Role | Description |
|------|-------------|
| `READ_ONLY` | SELECT only on all tables |
| `DBA` | Full administrative privileges |
| `PUBLIC` | Default role, basic connect |

```sql
-- Assign role
GRANT DBA TO 'admin_user'@'%';
```

### Oracle Mode

| Role | Description |
|------|-------------|
| `DBA` | Full DBA privileges |
| `CONNECT` | CREATE SESSION |
| `RESOURCE` | CREATE TABLE, CREATE VIEW, etc. |
| `PUBLIC` | Default role |

---

## Password Policy

```sql
-- Set password expiration (MySQL Mode)
ALTER USER 'app_user'@'%' PASSWORD EXPIRE INTERVAL 90 DAY;

-- Set failed login attempts lock
ALTER USER 'app_user'@'%'
  FAILED_LOGIN_ATTEMPTS 5
  PASSWORD_LOCK_TIME 1;

-- View password policy
SHOW PARAMETERS LIKE 'default_password_lifetime';
```

---

## MySQL Mode vs Oracle Mode

| Aspect | MySQL Mode | Oracle Mode |
|--------|-----------|-------------|
| User creation | `CREATE USER 'user'@'host'` | `CREATE USER user` |
| Privilege grant | `GRANT ... ON db.*` | `GRANT ... TO user` |
| Roles | Limited (`DBA`, `READ_ONLY`) | Full role support (`DBA`, `CONNECT`, `RESOURCE`) |
| System views | `mysql.user`, `information_schema` | `dba_users`, `dba_sys_privs` |

---

## Common Mistakes

1. **Using `%` host in production**
   - ❌ `CREATE USER 'app'@'%'` (allows connection from any IP)
   - ✅ `CREATE USER 'app'@'10.0.0.%'` (restrict to app subnet)

2. **Granting `ALL PRIVILEGES ON *.*` to app users**
   - ❌ Over-privileged application user
   - ✅ `GRANT SELECT,INSERT,UPDATE,DELETE ON app_db.*`

3. **Not setting password expiration**
   - ❌ Password never expires
   - ✅ `ALTER USER ... PASSWORD EXPIRE INTERVAL 90 DAY`

4. **Forgetting `FLUSH PRIVILEGES` (MySQL Mode)**
   - ❌ Grants not applied immediately
   - ✅ Always run `FLUSH PRIVILEGES` after `GRANT`/`REVOKE`

5. **Using weak passwords**
   - ❌ `password123`
   - ✅ Enforce complexity: min 12 chars, uppercase, lowercase, digit, special char

---

## OceanBase Version Notes

### V4.2 (Baseline)
- MySQL Mode and Oracle Mode user management
- Password policy parameters
- Role-based access control

### V4.4+
- Enhanced password verification (reuse check)
- Improved audit logging for privilege changes

---

## Sources
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/user-management
- https://www.oceanbase.com/docs/oceanbase-database-cn/v4.2.5/user-management
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/privilege-management
