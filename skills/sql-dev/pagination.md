# Pagination Techniques

## Overview

Pagination is essential for UI and API endpoints. This guide covers pagination techniques for OceanBase (MySQL Mode & Oracle Mode) V4.2+.

---

## Pagination Methods (Comparison)

| Method | Syntax | Performance | Use Case |
|--------|---------|-------------|----------|
| OFFSET | `LIMIT n OFFSET m` | O(n) — slow for deep pages | First few pages |
| Keyset | `WHERE id > :last_id LIMIT n` | O(1) — always fast | Deep pagination |
| Cursor | Store cursor (encrypted) | O(1) | API pagination |

> **Rule**: Use OFFSET for first 100 pages; keyset for deeper.

---

## OFFSET Pagination (MySQL Mode)

```sql
-- Page 1 (rows 1-20)
SELECT emp_id, emp_name
FROM employees
ORDER BY emp_id
LIMIT 20 OFFSET 0;

-- Page 2 (rows 21-40)
SELECT emp_id, emp_name
FROM employees
ORDER BY emp_id
LIMIT 20 OFFSET 20;

-- Page 1000 (rows 20001-20020) — SLOW!
SELECT emp_id, emp_name
FROM employees
ORDER BY emp_id
LIMIT 20 OFFSET 20000;  -- Scan 20000 rows (slow)
```

---

## Keyset Pagination (Recommended)

```sql
-- Page 1
SELECT emp_id, emp_name
FROM employees
ORDER BY emp_id
LIMIT 20;

-- Page 2 (use last emp_id from page 1)
SELECT emp_id, emp_name
FROM employees
WHERE emp_id > 10020   -- last id from page 1
ORDER BY emp_id
LIMIT 20;

-- Page 1000 — FAST! (no OFFSET scan)
SELECT emp_id, emp_name
FROM employees
WHERE emp_id > 300000
ORDER BY emp_id
LIMIT 20;
```

> **Result**: Keyset pagination is O(1) — fast even for deep pages.

---

## Oracle Mode Pagination (ROWNUM / FETCH)

```sql
-- Oracle Mode (ROWNUM method)
SELECT * FROM (
    SELECT t.*, ROWNUM AS rn
    FROM (
        SELECT emp_id, emp_name
        FROM employees
        ORDER BY emp_id
    ) t
    WHERE ROWNUM <= 40
)
WHERE rn > 20;

-- Oracle Mode (FETCH method, V4.2+)
SELECT emp_id, emp_name
FROM employees
ORDER BY emp_id
OFFSET 20 ROWS FETCH NEXT 20 ROWS ONLY;
```

> **Recommendation**: Use `FETCH NEXT n ROWS ONLY` (cleaner syntax).

---

## Cursor Pagination (API)

### Encoding Cursor

```python
import base64
import json

def encode_cursor(last_id):
    cursor = {"last_id": last_id}
    return base64.b64encode(json.dumps(cursor).encode()).decode()

def decode_cursor(cursor_str):
    return json.loads(base64.b64decode(cursor_str).decode())
```

### Using Cursor in Query

```sql
-- Client passes cursor (encrypted last_id)
SELECT emp_id, emp_name
FROM employees
WHERE emp_id > :last_id
ORDER BY emp_id
LIMIT 20;
```

> **Use Case**: API pagination (hide internals from client).

---

## Common Mistakes

1. **Using `OFFSET 100000` (deep page)**
   - ❌ `LIMIT 20 OFFSET 100000` (slow)
   - ✅ Use keyset: `WHERE id > :last_id LIMIT 20`

2. **Not ordering by unique column**
   - ❌ `ORDER BY salary` (non-unique → duplicate rows across pages)
   - ✅ `ORDER BY salary, emp_id` (unique)

3. **Using `ROWNUM` without subquery (Oracle Mode)**
   - ❌ `SELECT * FROM employees WHERE ROWNUM <= 20 ORDER BY emp_id`
   - ✅ Subquery first: `SELECT * FROM (SELECT * FROM t ORDER BY ...) WHERE ROWNUM <= 20`

4. **Not indexing ORDER BY column**
   - ❌ `ORDER BY emp_name` without index (filesort)
   - ✅ Add index on `emp_name`

5. **Returning all rows to app (no pagination)**
   - ❌ `SELECT * FROM employees` (10M rows)
   - ✅ Always paginate (LIMIT 20)

---

## MySQL Mode vs Oracle Mode

| Aspect | MySQL Mode | Oracle Mode |
|--------|-----------|-------------|
| OFFSET syntax | `LIMIT n OFFSET m` | `OFFSET m ROWS FETCH NEXT n ROWS ONLY` |
| Keyset | `WHERE id > :last_id` | Same |
| ROWNUM | ❌ | ✅ (legacy) |

---

## Sources
- https://en.oceanbase.com/docs/oceanbase-database/v4.2.5/pagination
- https://www.oceanbase.com/docs/oceanbase-database-cn/v4.2.5/pagination
- https://use-the-index-luke.com/sql/partial-results/fetch-next-page
