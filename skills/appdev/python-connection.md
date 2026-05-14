# Python Connection (PyMySQL/mysqlclient/JDBC)

## Overview

Python connects to OceanBase (MySQL Mode) via standard MySQL drivers. This guide covers PyMySQL, mysqlclient, and SQLAlchemy usage for OceanBase V4.2+.

---

## PyMySQL (Pure Python, Easy to Install)

### Installation

```bash
pip install PyMySQL
```

### Basic Connection

```python
import pymysql

conn = pymysql.connect(
    host='10.0.0.100',      # OBProxy IP
    port=2883,               # OBProxy port
    user='app_user@mysql_tenant',
    password='password',
    database='test_db',
    charset='utf8mb4',
    connect_timeout=5,
    read_timeout=30,
    write_timeout=30
)

cursor = conn.cursor()
cursor.execute("SELECT 1 FROM DUAL")
result = cursor.fetchone()
print(result)
conn.close()
```

---

## mysqlclient (Faster, C Extension)

### Installation

```bash
# Requires MySQL dev headers
pip install mysqlclient
```

### Basic Connection

```python
import MySQLdb

conn = MySQLdb.connect(
    host='10.0.0.100',
    port=2883,
    user='app_user@mysql_tenant',
    password='password',
    database='test_db',
    charset='utf8mb4'
)
```

> **Tip**: `mysqlclient` is faster than PyMySQL but requires C compilation.

---

## SQLAlchemy (ORM/Connection Pooling)

### Installation

```bash
pip install sqlalchemy pymysql
```

### Configuration

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

# Create engine (connection pool built-in)
engine = create_engine(
    'mysql+pymysql://app_user@mysql_tenant:password@10.0.0.100:2883/test_db',
    pool_size=10,
    pool_recycle=3600,      # Recycle connections every hour
    pool_pre_ping=True,      # Validate before use
    connect_args={
        'connect_timeout': 5,
        'charset': 'utf8mb4'
    }
)

Session = sessionmaker(bind=engine)
session = Session()

# Use session...
session.close()
```

| Parameter | Recommended | Description |
|-----------|-------------|-------------|
| `pool_size` | 10 | Connection pool size |
| `pool_recycle` | 3600 | Recycle connections (avoid stale) |
| `pool_pre_ping` | True | Validate before use |

---

## ORM (SQLAlchemy ORM)

```python
from sqlalchemy import Column, Integer, String, create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker

Base = declarative_base()

class Employee(Base):
    __tablename__ = 'employees'
    emp_id = Column(Integer, primary_key=True)
    emp_name = Column(String(100), nullable=False)
    dept_id = Column(Integer)

# Create tables
Base.metadata.create_all(engine)

# Insert
session = Session()
emp = Employee(emp_name='Alice', dept_id=10)
session.add(emp)
session.commit()
```

---

## Batch Processing (Performance)

```python
# Use executemany for batch INSERT
cursor = conn.cursor()
sql = "INSERT INTO employees (emp_name, dept_id) VALUES (%s, %s)"
values = [('Alice', 10), ('Bob', 20), ('Charlie', 30)]
cursor.executemany(sql, values)
conn.commit()
```

> **Tip**: `executemany()` is much faster than single `execute()` in a loop.

---

## Common Mistakes

1. **Not using `pool_pre_ping=True` (SQLAlchemy)**
   - ❌ Stale connections cause errors
   - ✅ `pool_pre_ping=True` (validates before use)

2. **Not closing connections**
   - ❌ Connection leak
   - ✅ Use `with` statement:
   ```python
   with pymysql.connect(...) as conn:
       with conn.cursor() as cursor:
           cursor.execute(...)
   ```

3. **Using `SELECT *`**
   - ❌ Loads all columns (slow)
   - ✅ Specify columns explicitly

4. **Not setting timeouts**
   - ❌ Query hangs forever
   - ✅ `connect_timeout=5, read_timeout=30`

5. **Not using `executemany()` for batch**
   - ❌ Single INSERT per loop iteration
   - ✅ Use `executemany()`

---

## OceanBase Oracle Mode (Python)

For Oracle mode, Python does not have a native direct driver. You need to use the jaydebeapi library via JDBC bridging to connect.

```python
import jaydebeapi

# 配置连接参数
# jdbc:oceanbase://IP:PORT
url = 'jdbc:oceanbase://172.20.2.2:2883'
#USERNAME@TANTENT_NAME#CLUSTER_NAME
user = 'anbob@orcl#enmotest'
password = 'anbob9876'
# 驱动类路径，通常不需要更改
driver = 'com.alipay.oceanbase.jdbc.Driver'
# Jar 包路径
jar_file = 'D:\code\oracle-compatibility-tool\pydrivers\oceanbase-client-2.4.1.jar' 

# 建立连接
conn = jaydebeapi.connect(
    driver, 
    url, 
    [user, password], 
    jar_file
)

curs = conn.cursor()
# 执行查询
curs.execute("SELECT * FROM test")
result = curs.fetchall()

for row in result:
    print(row)

curs.close()
conn.close()
```

---

## Sources
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/python
- https://www.oceanbase.com/docs/oceanbase-database-cn/v4.2.5/python
- https://pymysql.readthedocs.io/
- https://www.anbob.com/archives/9616.html
