# Index Strategy and Optimization

## Overview

Indexes are critical for OLTP performance in OceanBase. This guide covers index types, selection strategy, and maintenance for OceanBase V4.2+.

---

## Index Types

| Type | Description | Use Case |
|------|-------------|----------|
| **B-tree** (default) | Balanced tree | Most OLTP queries |
| **Unique** | B-tree + uniqueness | Primary key, unique constraint |
| **Covering** | Index includes all SELECT columns | Avoid table lookup |
| **Functional** (Oracle Mode) | Index on expression | `UPPER(col)`, `col1+col2` |
| **Vector** (V4.4+) | HNSW/IVFFlat | AI vector search |

---

## Creating Indexes

### Basic Index

```sql
-- MySQL Mode
CREATE INDEX idx_emp_dept ON employees(dept_id);
CREATE INDEX idx_emp_name ON employees(last_name, first_name);

-- Oracle Mode
CREATE INDEX idx_emp_dept ON employees(dept_id);
```

### Unique Index

```sql
CREATE UNIQUE INDEX idx_email ON employees(email);
```

### Covering Index (Avoid TABLE ACCESS BY INDEX ROWID)

```sql
-- Include all SELECT columns in index
CREATE INDEX idx_orders_customer_covering
  ON orders(customer_id) INCLUDE (order_date, total_amount);

-- Now this query uses index only (no table access):
-- SELECT customer_id, order_date, total_amount FROM orders WHERE customer_id = 123;
```

---

## Index Selection Strategy

### Good Candidates for Index

| Column | Reason |
|--------|--------|
| WHERE clause predicates | `WHERE dept_id = 10` → index on `dept_id` |
| JOIN columns | `FROM orders JOIN customers ON orders.cust_id = customers.id` → index on `cust_id` |
| ORDER BY / GROUP BY | Avoids SORT operator |
| FOREIGN KEY columns | Speed up cascading operations |

### Bad Candidates for Index

| Column | Reason |
|--------|--------|
| Small table (< 1000 rows) | Full scan is faster |
| Low cardinality (M/F) | Index not selective |
| Frequently updated | Index maintenance overhead |
| TEXT/BLOB columns | Cannot index directly |

---

## Analyzing Index Usage

```sql
-- Find unused indexes (MySQL Mode)
SELECT * FROM oceanbase.DBA_OB_INDEX_USAGE
WHERE index_name = 'IDX_EMP_DEPT'
  AND accesses = 0;
```

> **Tip**: Drop unused indexes to reduce write overhead.

---

## Index Maintenance

### Rebuild Index (if fragmented)

```sql
-- Oracle Mode
ALTER INDEX idx_emp_dept REBUILD;

-- MySQL Mode (recreate)
DROP INDEX idx_emp_dept ON employees;
CREATE INDEX idx_emp_dept ON employees(dept_id);
```

### Monitor Index Size

```sql
SELECT index_name,
       round(data_size / 1024 / 1024, 2) AS size_mb
FROM oceanbase.DBA_OB_INDEXES
WHERE table_name = 'EMPLOYEES';
```

---

## MySQL Mode vs Oracle Mode

| Aspect | MySQL Mode | Oracle Mode |
|--------|-----------|-------------|
| Functional index | Not supported | Supported (`CREATE INDEX ON T(UPPER(col))`) |
| Bitmap index | Not supported | Not supported (OceanBase doesn't have bitmap) |
| Index-organized table | Not supported | Not supported |

---

## Common Mistakes

1. **Too many indexes on one table**
   - ❌ 15 indexes on `orders` table
   - ✅ 3-5 strategic indexes; each index = write overhead

2. **Indexing low-cardinality columns**
   - ❌ `CREATE INDEX ON employees(gender)`
   - ✅ Index on `department_id` (higher cardinality)

3. **Not using covering index**
   - ❌ Index lookup + table access (slow)
   - ✅ `INCLUDE` all SELECT columns

4. **Forgetting to update statistics after index creation**
   - ❌ Optimizer doesn't know about new index
   - ✅ `ANALYZE TABLE employees` or `DBMS_STATS.GATHER_TABLE_STATS`

5. **Creating index without EXPLAIN first**
   - ❌ Guessing which index is needed
   - ✅ `EXPLAIN` the query, identify `TABLE SCAN`, then add index

---

## OceanBase Version Notes

### V4.2 (Baseline)
- B-tree index
- Unique index
- Covering index (`INCLUDE`)

### V4.4+
- Vector index (HNSW, IVFFlat) for AI workloads
- Faster online index creation

---

## Sources
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/index
- https://www.oceanbase.com/docs/oceanbase-database-cn/v4.2.5/index
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/index-strategy
