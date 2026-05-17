# SQL 信息采集（指定 SQL_ID）

> **⚠️ Security Note**: 代码示例中的 `YOUR_PASSWORD` 为占位符。请通过环境变量（如 `os.getenv('OB_PASSWORD')`）或运行时传入真实密码，**不要将真实密码硬编码到文件中**。

通过 **SQL_ID** 采集执行计划、SQL 文本、涉及对象、索引及统计信息，输出 Markdown 格式诊断报告。

适用场景：
- 已知 `SQL_ID`，需要全面了解其执行状态
- 优化前收集基线数据
- 分析慢查询涉及的表结构和索引

---

## 快速执行

```python
from skills.performance.sql_collect import sql_collect_report

# 传入 SQL_ID，生成 Markdown 报告
report = sql_collect_report(
    sql_id='ABCD1234567890ABCDEF1234567890AB',
    password=os.getenv('OB_PASSWORD')  # 运行时传入
)
print(report)
```

---

## 核心查询说明

### Step 1：执行计划缓存概览

```python
# 从 plan cache 获取该 SQL 的执行计划列表
cursor.execute("""
SELECT
    SVR_IP, SVR_PORT, TENANT_ID, TENANT_NAME,
    PLAN_ID, PLAN_HASH,
    LAST_ACTIVE_TIME,
    EXECUTIONS,
    ROUND(ROWS_PROCESSED / GREATEST(EXECUTIONS, 1))  AS avg_rows,
    ROUND(AVG_EXE_USEC / 1000)                       AS avg_exec_ms,
    ROUND(SLOWEST_EXE_USEC / 1000)                  AS slowest_ms,
    ROUND(PLAN_SIZE / 1024 / 1024, 1)             AS plan_size_mb
FROM oceanbase.GV$OB_PLAN_CACHE_PLAN_STAT a
JOIN oceanbase.DBA_OB_TENANTS t ON a.TENANT_ID = t.TENANT_ID
WHERE a.SQL_ID = %s
ORDER BY LAST_ACTIVE_TIME DESC
LIMIT 20
""", (sql_id,))
```

> **关键信号**：
> - `EXECUTIONS = 0` → 执行计划存在但近期未使用，可能已淘汰
> - `avg_exec_ms` 高 → 执行本身慢，需优化 SQL 或加索引
> - `plan_size_mb` 大 → 计划占内存多，可能存在大 SQL

---

### Step 2：SQL 文本信息

```python
# 获取完整 SQL 文本（截取前500字符预览）
cursor.execute("""
SELECT
    TENANT_NAME,
    SQL_ID,
    LENGTH(QUERY_SQL)            AS sql_length,
    LEFT(QUERY_SQL, 500)        AS sql_preview
FROM oceanbase.GV$OB_PLAN_CACHE_PLAN_STAT a
JOIN oceanbase.DBA_OB_TENANTS t ON a.TENANT_ID = t.TENANT_ID
WHERE a.SQL_ID = %s
LIMIT 5
""", (sql_id,))
```

---

### Step 3：GV$OB_SQL_AUDIT 历史执行统计

```python
# 从 audit 视图获取该 SQL 的历史执行表现
cursor.execute("""
SELECT
    TENANT_NAME,
    COUNT(*)                                        AS exec_count,
    ROUND(SUM(ELAPSED_TIME) / 1000000, 2)        AS total_sec,
    ROUND(AVG(ELAPSED_TIME) / 1000, 1)            AS avg_ms,
    ROUND(MIN(ELAPSED_TIME) / 1000, 1)            AS min_ms,
    ROUND(MAX(ELAPSED_TIME) / 1000, 1)            AS max_ms,
    ROUND(SUM(EXECUTE_TIME) / 1000000, 2)         AS exec_sec,
    ROUND(SUM(QUEUE_TIME) / 1000000, 2)           AS queue_sec,
    SUM(DISK_READS)                                AS total_disk_reads,
    SUM(TABLE_SCAN)                                AS full_scan_count,
    SUM(IS_HIT_PLAN)                              AS cache_hit_count,
    MIN(FROM_UNIXTIME(REQUEST_TIME / 1000000))    AS first_seen,
    MAX(FROM_UNIXTIME(REQUEST_TIME / 1000000))    AS last_seen
FROM oceanbase.GV$OB_SQL_AUDIT a
JOIN oceanbase.DBA_OB_TENANTS t ON a.TENANT_ID = t.TENANT_ID
WHERE a.SQL_ID = %s
  AND IS_INNER_SQL = 0
GROUP BY TENANT_NAME
ORDER BY exec_count DESC
""", (sql_id,))
```

---

### Step 4：获取执行计划详情（dbms_xplan）

```python
# 对每个 PLAN_ID 调用 dbms_xplan.display_cursor 获取详细计划
cursor.execute("""
SELECT
    SVR_IP, SVR_PORT, TENANT_ID, PLAN_ID
FROM oceanbase.GV$OB_PLAN_CACHE_PLAN_STAT
WHERE SQL_ID = %s
ORDER BY LAST_ACTIVE_TIME DESC
LIMIT 20
""", (sql_id,))

for row in rows:
    svr_ip, svr_port, tenant_id, plan_id = row
    cursor.execute(
        f"SELECT dbms_xplan.display_cursor({plan_id},'ALL','{svr_ip}',{svr_port},{tenant_id})"
    )
    plan_text = cursor.fetchone()[0]
    # 追加到报告
```

> `dbms_xplan.display_cursor` 参数：
> - Format: `'ALL'` = 完整计划含统计；`'TYPICAL'` = 标准；`'BASIC'` = 简化

---

### Step 5：涉及表对象

```python
# 从执行计划中提取涉及的表和索引
cursor.execute("""
SELECT DISTINCT
    TENANT_ID,
    OBJECT_OWNER,
    OBJECT_NAME,
    OBJECT_TYPE,
    OBJECT_ID
FROM oceanbase.GV$OB_SQL_PLAN
WHERE SQL_ID = %s
  AND OBJECT_NAME IS NOT NULL
  AND OBJECT_NAME != ''
  AND OBJECT_TYPE IN ('TABLE', 'INDEX', 'BASIC TABLE')
ORDER BY TENANT_ID, OBJECT_OWNER, OBJECT_NAME
""", (sql_id,))
```

---

### Step 6：索引信息

```python
# 对每张表查索引列表
cursor.execute("""
SELECT
    s.INDEX_NAME,
    s.TABLE_NAME,
    s.UNIQUENESS,
    s.NUM_ROWS,
    s.DISTINCT_KEYS,
    ROUND(s.AVG_SPACE / 1024, 1)          AS avg_space_kb,
    ROUND(s.AVG_LEAF_BLOCKS / 1, 0)      AS avg_leaf_blocks,
    s.LEVEL                               AS btree_level,
    ic.COLUMN_NAME,
    ic.COLUMN_POSITION,
    ic.DESCEND
FROM oceanbase.DBA_INDEXES s
JOIN oceanbase.DBA_IND_COLUMNS ic
      ON s.OWNER = ic.INDEX_OWNER
     AND s.INDEX_NAME = ic.INDEX_NAME
     AND s.TABLE_NAME = ic.TABLE_NAME
WHERE s.OWNER = %s
  AND s.TABLE_NAME = %s
ORDER BY s.INDEX_NAME, ic.COLUMN_POSITION
""", (owner, table_name))
```

---

### Step 7：表和列统计信息

```python
# 表统计
cursor.execute("""
SELECT
    OWNER, TABLE_NAME,
    NUM_ROWS,
    BLOCKS,
    EMPTY_BLOCKS,
    ROUND(AVG_ROW_LEN, 1)     AS avg_row_len,
    GLOBAL_STATS,
    LAST_ANALYZED,
    PARTITIONED,
    SAMPLE_SIZE
FROM oceanbase.DBA_TABLES
WHERE OWNER = %s
  AND TABLE_NAME = %s
""", (owner, table_name))

# 列统计
cursor.execute("""
SELECT
    TABLE_NAME, COLUMN_NAME,
    DATA_TYPE,
    NUM_DISTINCT,
    DENSITY,
    NUM_NULLS,
    NUM_BUCKETS,
    LAST_ANALYZED,
    GLOBAL_STATS,
    SAMPLE_SIZE,
    AVG_COL_LEN
FROM oceanbase.DBA_TAB_COLUMNS
WHERE OWNER = %s
  AND TABLE_NAME = %s
ORDER BY NUM_DISTINCT DESC
""", (owner, table_name))

# 直方图信息
cursor.execute("""
SELECT
    TABLE_NAME, COLUMN_NAME,
    ENDPOINT_NUMBER, ENDPOINT_VALUE, ENDPOINT_ACTUAL_VALUE
FROM oceanbase.DBA_TAB_HISTOGRAMS
WHERE OWNER = %s
  AND TABLE_NAME = %s
ORDER BY TABLE_NAME, COLUMN_NAME, ENDPOINT_NUMBER
""", (owner, table_name))
```

---

## 完整采集脚本

```python
import pymysql
import os
from datetime import datetime

def sql_collect_report(sql_id, password=None, host='172.20.22.213', port=2883):
    """
    采集指定 SQL_ID 的完整诊断信息，输出 Markdown 报告。

    Args:
        sql_id: 32位十六进制 SQL_ID
        password: 运行时传入，不硬编码
        host: OBProxy/Observer IP
        port: OBProxy SQL 端口

    Returns:
        str: Markdown 格式报告
    """
    sql_id = sql_id.upper().strip()

    if not password:
        raise ValueError("password 参数不能为空，请通过 os.getenv('OB_PASSWORD') 传入")

    conn = pymysql.connect(
        host=host, port=port,
        user='root@sys#enmotest',
        password=password,
        database='oceanbase',
        charset='utf8mb4', connect_timeout=15
    )
    cursor = conn.cursor()

    lines = []   # 报告内容

    def q(sql, params=None):
        cursor.execute(sql, params)
        cols = [d[0] for d in cursor.description]
        return [dict(zip(cols, r)) for r in cursor.fetchall()]

    def section(title):
        lines.append(f"\n## {title}\n")

    def table_row(headers, rows_data):
        """生成 Markdown 表格"""
        if not rows_data:
            lines.append("*（无数据）*\n")
            return
        lines.append("| " + " | ".join(headers) + " |")
        lines.append("| " + " | ".join(["---"] * len(headers)) + " |")
        for row in rows_data:
            vals = [str(row.get(h, '')) for h in headers]
            lines.append("| " + " | ".join(vals) + " |")
        lines.append("")

    ts = datetime.now().strftime('%Y-%m-%d %H:%M:%S')

    # ===== 报告头 =====
    lines += [
        "# SQL 信息采集报告",
        "",
        f"**SQL_ID**: `{sql_id}`",
        f"**采集时间**: {ts}",
        f"**连接**: `{host}:{port}` / `root@sys#enmotest`",
        "",
    ]

    # ===== Step 1: Plan Cache 概览 =====
    section("Step 1 - 执行计划缓存概览")
    plans = q("""
        SELECT
            a.SVR_IP, a.SVR_PORT, a.TENANT_ID, t.TENANT_NAME,
            a.PLAN_ID, a.PLAN_HASH,
            FROM_UNIXTIME(a.LAST_ACTIVE_TIME)            AS last_active,
            a.EXECUTIONS,
            ROUND(a.ROWS_PROCESSED / GREATEST(a.EXECUTIONS, 1))  AS avg_rows,
            ROUND(a.AVG_EXE_USEC / 1000, 1)                     AS avg_exec_ms,
            ROUND(a.SLOWEST_EXE_USEC / 1000, 1)                AS slowest_ms,
            ROUND(a.PLAN_SIZE / 1024 / 1024, 1)              AS plan_size_mb
        FROM oceanbase.GV$OB_PLAN_CACHE_PLAN_STAT a
        JOIN oceanbase.DBA_OB_TENANTS t ON a.TENANT_ID = t.TENANT_ID
        WHERE a.SQL_ID = %s
        ORDER BY a.LAST_ACTIVE_TIME DESC LIMIT 20
    """, (sql_id,))

    if plans:
        table_row(
            ['TENANT_NAME', 'PLAN_ID', 'EXECUTIONS', 'avg_rows', 'avg_exec_ms', 'slowest_ms', 'plan_size_mb', 'last_active'],
            plans
        )
        plan_ids = [(p['SVR_IP'], p['SVR_PORT'], p['TENANT_ID'], p['PLAN_ID']) for p in plans]
    else:
        lines.append("*未找到执行计划（可能已被淘汰或从未执行）*\n")
        plan_ids = []

    # ===== Step 2: SQL 文本 =====
    section("Step 2 - SQL 文本")
    texts = q("""
        SELECT
            t.TENANT_NAME,
            LENGTH(a.QUERY_SQL)             AS sql_length,
            LEFT(a.QUERY_SQL, 500)          AS sql_preview
        FROM oceanbase.GV$OB_PLAN_CACHE_PLAN_STAT a
        JOIN oceanbase.DBA_OB_TENANTS t ON a.TENANT_ID = t.TENANT_ID
        WHERE a.SQL_ID = %s LIMIT 5
    """, (sql_id,))

    for t in texts:
        lines += [
            f"**租户**: {t['TENANT_NAME']}  **长度**: {t['sql_length']} 字节",
            "",
            "```sql",
            t['sql_preview'],
            "```",
            "",
        ]

    # ===== Step 3: SQL Audit 历史 =====
    section("Step 3 - SQL Audit 历史执行统计")
    audit = q("""
        SELECT
            t.TENANT_NAME,
            COUNT(*)                                         AS exec_count,
            ROUND(SUM(a.ELAPSED_TIME)/1000000, 2)          AS total_sec,
            ROUND(AVG(a.ELAPSED_TIME)/1000, 1)             AS avg_ms,
            ROUND(MIN(a.ELAPSED_TIME)/1000, 1)              AS min_ms,
            ROUND(MAX(a.ELAPSED_TIME)/1000, 1)             AS max_ms,
            SUM(a.DISK_READS)                               AS total_disk_reads,
            SUM(a.TABLE_SCAN)                               AS full_scan_count,
            ROUND(SUM(a.ELAPSED_TIME - a.EXECUTE_TIME)/1000, 1) AS overhead_ms,
            MIN(FROM_UNIXTIME(a.REQUEST_TIME/1000000))     AS first_seen,
            MAX(FROM_UNIXTIME(a.REQUEST_TIME/1000000))     AS last_seen
        FROM oceanbase.GV$OB_SQL_AUDIT a
        JOIN oceanbase.DBA_OB_TENANTS t ON a.TENANT_ID = t.TENANT_ID
        WHERE a.SQL_ID = %s AND a.IS_INNER_SQL = 0
        GROUP BY t.TENANT_NAME
        ORDER BY exec_count DESC
    """, (sql_id,))

    if audit:
        table_row(
            ['TENANT_NAME', 'exec_count', 'total_sec', 'avg_ms', 'min_ms', 'max_ms',
             'total_disk_reads', 'full_scan_count', 'overhead_ms', 'first_seen', 'last_seen'],
            audit
        )
    else:
        lines.append("*Audit 窗口内无执行记录*\n")

    # ===== Step 4: 执行计划详情 =====
    section("Step 4 - 执行计划详情")
    if plan_ids:
        for idx, (svr_ip, svr_port, tenant_id, plan_id) in enumerate(plan_ids, 1):
            lines.append(f"**计划 {idx}: PLAN_ID={plan_id}  SERVER={svr_ip}:{svr_port}  TENANT_ID={tenant_id}**\n")
            cursor.execute(
                f"SELECT dbms_xplan.display_cursor({plan_id},'ALL','{svr_ip}',{svr_port},{tenant_id}) AS plan_text"
            )
            result = cursor.fetchone()
            if result and result[0]:
                lines.append("```text")
                lines.append(result[0])
                lines.append("```\n")
            else:
                lines.append("*（计划详情为空，可能已淘汰）*\n")
    else:
        lines.append("*无执行计划详情*\n")

    # ===== Step 5: 涉及表对象 =====
    section("Step 5 - 涉及表对象")
    objects = q("""
        SELECT DISTINCT
            TENANT_ID, OBJECT_OWNER, OBJECT_NAME, OBJECT_TYPE, OBJECT_ID
        FROM oceanbase.GV$OB_SQL_PLAN
        WHERE SQL_ID = %s
          AND OBJECT_NAME IS NOT NULL AND OBJECT_NAME != ''
          AND OBJECT_TYPE IN ('TABLE', 'INDEX', 'BASIC TABLE')
        ORDER BY TENANT_ID, OBJECT_OWNER, OBJECT_NAME
    """, (sql_id,))

    if objects:
        table_row(['TENANT_ID', 'OBJECT_OWNER', 'OBJECT_NAME', 'OBJECT_TYPE', 'OBJECT_ID'], objects)
    else:
        lines.append("*未从 GV$OB_SQL_PLAN 获取到对象信息（视图或计划已淘汰）*\n")

    # ===== Step 6 & 7: 每张表的索引 + 统计 =====
    # 按 OBJECT_OWNER/OBJECT_NAME 分组（去重租户）
    seen = set()
    table_list = []
    for obj in objects:
        key = (obj['TENANT_ID'], obj['OBJECT_OWNER'], obj['OBJECT_NAME'])
        if key not in seen and obj['OBJECT_TYPE'] in ('TABLE', 'BASIC TABLE'):
            seen.add(key)
            table_list.append(key)

    for (tenant_id, owner, table_name) in table_list:
        section(f"Step 6 & 7 - {owner}.{table_name} (TENANT_ID={tenant_id})")

        # 索引
        indexes = q("""
            SELECT
                s.INDEX_NAME, s.UNIQUENESS, s.NUM_ROWS, s.DISTINCT_KEYS,
                ic.COLUMN_NAME, ic.COLUMN_POSITION, ic.DESCEND
            FROM oceanbase.DBA_INDEXES s
            JOIN oceanbase.DBA_IND_COLUMNS ic
                  ON s.OWNER = ic.INDEX_OWNER
                 AND s.INDEX_NAME = ic.INDEX_NAME
                 AND s.TABLE_NAME = ic.TABLE_NAME
            WHERE s.OWNER = %s AND s.TABLE_NAME = %s
            ORDER BY s.INDEX_NAME, ic.COLUMN_POSITION
        """, (owner, table_name))

        if indexes:
            # 按索引名聚合列
            from collections import defaultdict
            idx_dict = defaultdict(list)
            for row in indexes:
                idx_dict[row['INDEX_NAME']].append(row)
            rows_data = []
            for iname, cols in idx_dict.items():
                sample = cols[0]
                rows_data.append({
                    'INDEX_NAME': iname,
                    'UNIQUENESS': sample['UNIQUENESS'],
                    'NUM_ROWS': sample['NUM_ROWS'],
                    'COLS': ', '.join(f"{c['COLUMN_NAME']}({c['COLUMN_POSITION']})" for c in cols)
                })
            table_row(['INDEX_NAME', 'UNIQUENESS', 'NUM_ROWS', 'COLUMNS'], rows_data)
        else:
            lines.append("*无索引信息*\n")

        # 表统计
        tab_stats = q("""
            SELECT OWNER, TABLE_NAME, NUM_ROWS, BLOCKS, SAMPLE_SIZE,
                   ROUND(AVG_ROW_LEN, 1) AS avg_row_len,
                   GLOBAL_STATS, LAST_ANALYZED, PARTITIONED
            FROM oceanbase.DBA_TABLES
            WHERE OWNER = %s AND TABLE_NAME = %s
        """, (owner, table_name))
        if tab_stats:
            table_row(['TABLE_NAME', 'NUM_ROWS', 'BLOCKS', 'avg_row_len', 'SAMPLE_SIZE', 'LAST_ANALYZED', 'PARTITIONED'], tab_stats)

        # 列统计
        col_stats = q("""
            SELECT COLUMN_NAME, DATA_TYPE, NUM_DISTINCT,
                   DENSITY, NUM_NULLS, NUM_BUCKETS,
                   LAST_ANALYZED, SAMPLE_SIZE, AVG_COL_LEN
            FROM oceanbase.DBA_TAB_COLUMNS
            WHERE OWNER = %s AND TABLE_NAME = %s
            ORDER BY NUM_DISTINCT DESC
        """, (owner, table_name))
        if col_stats:
            table_row(['COLUMN_NAME', 'DATA_TYPE', 'NUM_DISTINCT', 'DENSITY', 'NUM_NULLS', 'NUM_BUCKETS', 'LAST_ANALYZED'], col_stats)

    # ===== 报告尾 =====
    lines += [
        "---",
        f"*报告生成时间: {ts}*",
    ]

    cursor.close()
    conn.close()
    return '\n'.join(lines)


# ---- 快速调用示例 ----
if __name__ == '__main__':
    import sys, os

    if len(sys.argv) < 2:
        print("用法: python sql_collect.py <SQL_ID>")
        print("示例: python sql_collect.py ABCD1234567890ABCDEF1234567890AB")
        sys.exit(1)

    sql_id = sys.argv[1]
    pwd = os.getenv('OB_PASSWORD', 'YOUR_PASSWORD')

    report = sql_collect_report(sql_id, password=pwd)

    # 输出到文件
    out_file = f"sql_collect_{sql_id[:8]}_{datetime.now().strftime('%Y%m%d_%H%M%S')}.md"
    with open(out_file, 'w', encoding='utf-8') as f:
        f.write(report)
    print(f"报告已写入: {out_file}")
```

---

## 相关对象采集字段说明

| 视图 / 表 | 用途 | 关键字段 |
|-----------|------|---------|
| `GV$OB_PLAN_CACHE_PLAN_STAT` | 执行计划缓存列表 | EXECUTIONS, AVG_EXE_USEC, SLOWEST_EXE_USEC, MEMORY_SIZE |
| `GV$OB_SQL_AUDIT` | SQL 执行历史 | ELAPSED_TIME, EXECUTE_TIME, DISK_READS, TABLE_SCAN |
| `GV$OB_SQL_PLAN` | 执行计划中的对象 | OBJECT_NAME, OBJECT_TYPE, OBJECT_OWNER |
| `DBA_OB_TENANTS` | 租户信息 | TENANT_NAME, TENANT_TYPE |
| `DBA_INDEXES` | 索引定义 | INDEX_NAME, UNIQUENESS, NUM_ROWS |
| `DBA_IND_COLUMNS` | 索引列顺序 | COLUMN_NAME, COLUMN_POSITION, DESCEND |
| `DBA_TABLES` | 表统计 | NUM_ROWS, BLOCKS, AVG_ROW_LEN, LAST_ANALYZED |
| `DBA_TAB_COLUMNS` | 列统计 | NUM_DISTINCT, DENSITY, NUM_NULLS, NUM_BUCKETS |
| `dbms_xplan.display_cursor` | 计划详情 | 完整执行计划含成本估算 |

---

## 常见问题

### 1. 执行计划列表为空
- 计划可能已从 Plan Cache 淘汰（检查 `ob_plan_cache_evict_high_percentage` 参数）
- SQL 从未在此集群执行过
- SQL_ID 输入有误（注意大小写，需32位十六进制）

### 2. 涉及表对象为空
- `GV$OB_SQL_PLAN` 视图在部分旧版本可能无数据
- SQL 执行计划已被淘汰

### 3. 统计信息陈旧
- `LAST_ANALYZED` 为空或过旧 → 需要 `ANALYZE TABLE`
- 旧统计信息导致优化器选择错误执行计划

### 4. 获取计划详情失败
- `dbms_xplan.display_cursor` 需要 `gv$ob_plan_cache_plan_stat` 中的 `SVR_IP`、`SVR_PORT`、`PLAN_ID`、`TENANT_ID`
- OBProxy 模式下 SERVER 为 OBProxy 地址，需替换为实际 Observer 地址

---

## 相关 Skills

- `skills/performance/sql-optimization.md` — 根据采集数据生成 SQL 优化建议
- `skills/performance/explain-plan.md` — 执行计划分析指南
- `skills/performance/slow-query-analysis.md` — 慢SQL分析与诊断
- `skills/monitoring/sql-diagnostics.md` — SQL 诊断工具箱

---

*Version: OceanBase V4.2+ | Last updated: 2026-05-15*
