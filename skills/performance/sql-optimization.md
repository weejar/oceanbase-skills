# SQL 优化诊断建议

> **⚠️ Security Note**: 代码示例中的 `YOUR_PASSWORD` 为占位符。请通过环境变量（如 `os.getenv('OB_PASSWORD')`）或运行时传入真实密码，**不要将真实密码硬编码到文件中**。

根据 `sql-collect.md` 采集的执行计划、表/列统计、索引信息，自动分析并生成优化建议报告。

适用场景：
- 基于采集报告生成诊断结论
- 批量分析多条慢 SQL 的优化方向
- 为 DBA 输出结构化优化建议

---

## 快速执行

```python
from skills.performance.sql_optimization import sql_optimization_report

# 方式1：直接传入 SQL_ID（自动采集 + 分析）
report = sql_optimization_report(
    sql_id='ABCD1234567890ABCDEF1234567890AB',
    password=os.getenv('OB_PASSWORD')
)
print(report)

# 方式2：传入已采集的数据（避免重复查询）
report = sql_optimization_report(
    sql_id='ABCD1234567890ABCDEF1234567890AB',
    password=os.getenv('OB_PASSWORD'),
    # 以下参数由 sql_collect.py 采集后传入
    plan_info=plan_list,
    audit_stats=audit_data,
    objects=object_list,
    indexes=index_data,
    table_stats=tab_data,
    column_stats=col_data
)
```

---

## 优化建议生成逻辑

### 1. 执行计划分析规则

```python
def analyze_plan(cursor, sql_id):
    """
    从 GV$OB_SQL_PLAN 提取计划操作符，按问题类型分类。
    返回: dict，问题列表，建议列表
    """
    cursor.execute("""
        SELECT
            TENANT_ID, PLAN_ID, OPERATION, OBJECT_NAME,
            OBJECT_TYPE, OPTIONS, COST, OBJECT_INSTANCE,
            ROWS_PROCESSED, BYTES, PHYSICAL_READ_REQUESTS,
            PHYSICAL_READ_BYTES, IO_INTERCONNECT_BYTES,
            TEMP_SPACE, ACCESS_PREDICATES, FILTER_PREDICATES,
            PROJECTION, PLAN_DEPTH, POSITION
        FROM oceanbase.GV$OB_SQL_PLAN
        WHERE SQL_ID = %s
        ORDER BY PLAN_ID, POSITION
    """, (sql_id,))

    ops = cursor.fetchall()
    cols = [d[0] for d in cursor.description]
    plans = [dict(zip(cols, r)) for r in ops]

    issues = []
    suggestions = []

    # --- 规则1: 全表/全索引扫描 ---
    scan_ops = ['TABLE SCAN', 'INDEX SCAN', 'INDEX FULL SCAN']
    for p in plans:
        if p['OPERATION'] in scan_ops:
            if p.get('OBJECT_TYPE') in ('TABLE', 'BASIC TABLE'):
                obj = p['OBJECT_NAME'] or '?'
                issues.append({
                    'severity': 'HIGH',
                    'type': 'FULL_TABLE_SCAN',
                    'object': obj,
                    'operation': p['OPERATION'],
                    'cost': p.get('COST', 0),
                    'rows': p.get('ROWS_PROCESSED', 0),
                    'detail': f"表 `{obj}` 执行了 {p['OPERATION']}，处理行数 {p.get('ROWS_PROCESSED', 0)}，成本 {p.get('COST', 0)}"
                })
                suggestions.append({
                    'priority': 1,
                    'category': 'INDEX',
                    'problem': f"表 `{obj}` 存在全表扫描",
                    'action': f"检查 WHERE/JOIN/ORDER BY 字段是否有索引，为 `{obj}` 添加合适索引",
                    'example': f"CREATE INDEX idx_{obj.lower()}_xxx ON {obj}(column_name);"
                })

        # --- 规则2: NL JOIN（大数据量时慢）---
        if 'NESTED LOOP' in p['OPERATION'] or 'NESTED-LOOP' in str(p['OPERATION']):
            inner_rows = p.get('ROWS_PROCESSED', 0)
            if inner_rows > 1000:
                issues.append({
                    'severity': 'MEDIUM',
                    'type': 'NESTED_LOOP_JOIN',
                    'object': p['OBJECT_NAME'] or 'JOIN',
                    'cost': p.get('COST', 0),
                    'rows': inner_rows,
                    'detail': f"嵌套循环连接内表 `{p['OBJECT_NAME']}` 处理 {inner_rows} 行，可能产生大量随机 I/O"
                })
                suggestions.append({
                    'priority': 2,
                    'category': 'JOIN',
                    'problem': f"嵌套循环连接内表 `{p['OBJECT_NAME']}` 行数过多（{inner_rows}）",
                    'action': "考虑改用 HASH JOIN：为驱动表加索引减少内表扫描；或添加 WHERE 条件减少内表行数",
                    'example': "-- 查看驱动表信息，确认是否有合适的索引"
                })

        # --- 规则3: SORT 操作（内存排序溢出）---
        if 'SORT' in p['OPERATION'] or 'ORDER BY' in str(p.get('ACCESS_PREDICATES', '')):
            sort_rows = p.get('ROWS_PROCESSED', 0)
            if sort_rows > 10000:
                issues.append({
                    'severity': 'MEDIUM',
                    'type': 'EXPENSIVE_SORT',
                    'object': p['OBJECT_NAME'] or 'SORT',
                    'cost': p.get('COST', 0),
                    'rows': sort_rows,
                    'detail': f"SORT 操作处理 {sort_rows} 行，可能溢出到磁盘"
                })
                suggestions.append({
                    'priority': 3,
                    'category': 'INDEX',
                    'problem': f"SORT 操作行数过多（{sort_rows}）",
                    'action': "在排序列上建索引，数据库可直接使用索引顺序避免排序",
                    'example': f"CREATE INDEX idx_{p['OBJECT_NAME'] or 't'}_sort ON {p['OBJECT_NAME'] or 't'}(sort_column);"
                })

        # --- 规则4: 笛卡尔积 ---
        if 'CARTESIAN' in str(p.get('OBJECT_NAME', '')) or 'CARTESIAN' in str(p.get('OPTIONS', '')):
            issues.append({
                'severity': 'HIGH',
                'type': 'CARTESIAN_PRODUCT',
                'object': p['OBJECT_NAME'] or 'JOIN',
                'cost': p.get('COST', 0),
                'rows': p.get('ROWS_PROCESSED', 0),
                'detail': "检测到笛卡尔积，结果行数爆炸"
            })
            suggestions.append({
                'priority': 1,
                'category': 'SQL',
                'problem': "存在笛卡尔积连接",
                'action': "检查 JOIN 条件是否完整，确保所有 JOIN 都有 ON 条件",
                'example': "-- 检查 SQL 是否遗漏了 JOIN 条件"
            })

        # --- 规则5: SUBPLAN（子查询未展开）---
        if 'SUBPLAN' in p['OPERATION']:
            issues.append({
                'severity': 'LOW',
                'type': 'SUBQUERY_UNFOLD',
                'object': p.get('OBJECT_NAME', 'SUBQUERY'),
                'detail': f"子查询 `{p['OBJECT_NAME']}` 可能未展开执行"
            })
            suggestions.append({
                'priority': 4,
                'category': 'SQL',
                'problem': "子查询可能未展开",
                'action': "将相关子查询改写为 JOIN；使用 HASH JOIN 替代子查询",
                'example': "-- 相关子查询 -> JOIN 改写"
            })

    return plans, issues, suggestions
```

---

### 2. 统计信息分析

```python
def analyze_stats(cursor, objects):
    """
    检查表和列统计信息是否陈旧，返回优化建议。
    """
    issues, suggestions = [], []

    for (tenant_id, owner, table_name) in objects:
        # 表统计
        cursor.execute("""
            SELECT TABLE_NAME, NUM_ROWS, BLOCKS, SAMPLE_SIZE,
                   LAST_ANALYZED, GLOBAL_STATS, STALE_STATS
            FROM oceanbase.DBA_TAB_STATISTICS
            WHERE OWNER = %s AND TABLE_NAME = %s
        """, (owner, table_name))
        row = cursor.fetchone()
        if row:
            last_analyzed = row[4]  # LAST_ANALYZED
            num_rows = row[1] or 0

            # 检查是否从未分析
            if last_analyzed is None:
                issues.append({
                    'severity': 'HIGH',
                    'type': 'NO_STATS',
                    'object': f'{owner}.{table_name}',
                    'detail': f"表 `{owner}.{table_name}` 从未收集统计信息"
                })
                suggestions.append({
                    'priority': 1,
                    'category': 'STATS',
                    'problem': f"表 `{owner}.{table_name}` 无统计信息",
                    'action': f"执行 ANALYZE TABLE `{owner}.{table_name}` COMPUTE STATISTICS;",
                    'example': f"ANALYZE TABLE {owner}.{table_name} COMPUTE STATISTICS;"
                })

            # 检查行数是否异常（大量变化后未重新分析）
            elif num_rows == 0 and last_analyzed is not None:
                issues.append({
                    'severity': 'MEDIUM',
                    'type': 'ZERO_ROWS_STATS',
                    'object': f'{owner}.{table_name}',
                    'detail': f"统计信息显示 0 行，但可能有数据（统计过期）"
                })
                suggestions.append({
                    'priority': 2,
                    'category': 'STATS',
                    'problem': f"表 `{owner}.{table_name}` 统计信息可能过期",
                    'action': "重新收集统计信息",
                    'example': f"ANALYZE TABLE {owner}.{table_name} COMPUTE STATISTICS;"
                })

        # 列统计缺失（高区分度列若无统计可能导致错选索引）
        cursor.execute("""
            SELECT COLUMN_NAME, NUM_DISTINCT, DENSITY, NUM_NULLS, NUM_BUCKETS
            FROM oceanbase.DBA_TAB_COL_STATISTICS
            WHERE OWNER = %s AND TABLE_NAME = %s
            ORDER BY NUM_DISTINCT DESC
        """, (owner, table_name))

    return issues, suggestions
```

---

### 3. 索引覆盖分析

```python
def analyze_index_coverage(cursor, plan, indexes):
    """
    分析执行计划中引用的列是否有索引覆盖（Covering Index）。
    """
    suggestions = []

    # 从计划中提取访问列和过滤列
    access_cols = set()
    filter_cols = set()
    for p in plan:
        pred = str(p.get('ACCESS_PREDICATES', ''))
        filt = str(p.get('FILTER_PREDICATES', ''))
        proj = str(p.get('PROJECTION', ''))
        for col_str in [pred, filt, proj]:
            import re
            cols = re.findall(r'[a-zA-Z_][a-zA-Z0-9_]*', col_str)
            if pred == col_str:
                access_cols.update(cols)
            else:
                filter_cols.update(cols)

    # 检查索引是否覆盖这些列
    for (idx_name, idx_cols) in indexes.items():
        covered = access_cols & set(idx_cols)
        if covered == access_cols and len(covered) > 0:
            suggestions.append({
                'priority': 3,
                'category': 'INDEX',
                'problem': f"索引 `{idx_name}` 可覆盖 WHERE 条件列",
                'action': f"确认查询使用了该索引（EXPLAIN 查看 access_type）",
                'example': f"CREATE INDEX idx_covering ON t({' ,'.join(idx_cols)}) -- 已存在，可覆盖"
            })

    return suggestions
```

---

### 4. SQL 改写建议

```python
SQL_REWRITE_RULES = [
    {
        'pattern': r'SELECT\s+\*\s+FROM',
        'problem': '使用 SELECT *',
        'suggestion': '仅查询需要的列，减少网络传输和内存占用'
    },
    {
        'pattern': r'WHERE.*IS NOT NULL',
        'problem': 'NOT NULL 过滤效率低',
        'suggestion': '确认列确实允许 NULL，或用默认值避免 NULL 判断'
    },
    {
        'pattern': r'WHERE.*!=',
        'problem': '!= 导致索引失效（部分情况）',
        'suggestion': '评估改为范围查询或确认优化器能正确处理'
    },
    {
        'pattern': r'WHERE.*LIKE\s+[\'"][\^%]',
        'problem': 'LIKE 前缀通配导致全表扫描',
        'suggestion': '改用后缀通配或考虑全文索引'
    },
    {
        'pattern': r'WHERE\s+OR\s+',
        'problem': '多个 OR 条件可能无法使用组合索引',
        'suggestion': '将 OR 改为 UNION ALL 或确保各条件有独立索引'
    },
    {
        'pattern': r'WHERE\s+RNUM\s*<=',
        'problem': '疑似分页写法使用 ROWNUM/RNUM',
        'suggestion': '使用标准 LIMIT/OFFSET 或 ROW_NUMBER() OVER()'
    },
]

def suggest_sql_rewrite(sql_text):
    """分析 SQL 文本，给出改写建议"""
    suggestions = []
    import re
    sql_upper = sql_text.upper()
    for rule in SQL_REWRITE_RULES:
        if re.search(rule['pattern'], sql_upper):
            suggestions.append({
                'priority': 5,
                'category': 'SQL',
                'problem': rule['problem'],
                'action': rule['suggestion'],
                'example': '-- ' + sql_text[:100]
            })
    return suggestions
```

---

## 完整优化建议生成脚本

```python
import pymysql
import os
from datetime import datetime
from collections import defaultdict

def sql_optimization_report(
    sql_id,
    password=None,
    host='172.20.22.213',
    port=2883,
    plan_info=None,     # 可选：直接传入已采集的计划数据
    audit_stats=None,
    objects=None,
    indexes=None,
    table_stats=None,
    column_stats=None
):
    """
    生成指定 SQL_ID 的优化诊断建议 Markdown 报告。

    参数:
        sql_id:        32位十六进制 SQL_ID
        password:      密码（运行时传入）
        host/port:     连接参数
        *_info:        可选，传入已采集数据以避免重复查询
    """
    sql_id = sql_id.upper().strip()
    if not password:
        raise ValueError("password 不能为空，请通过 os.getenv('OB_PASSWORD') 传入")

    conn = pymysql.connect(
        host=host, port=port,
        user='root@sys#enmotest',
        password=password,
        database='oceanbase',
        charset='utf8mb4', connect_timeout=15
    )
    cursor = conn.cursor()

    def q(sql, params=None):
        cursor.execute(sql, params)
        cols = [d[0] for d in cursor.description]
        return [dict(zip(cols, r)) for r in cursor.fetchall()]

    lines = []
    all_issues = []
    all_suggestions = []

    def section(title):
        lines.append(f"\n## {title}\n")

    def priority_badge(p):
        badges = {1: '🔴 P1', 2: '🟠 P2', 3: '🟡 P3', 4: '🔵 P4', 5: '⚪ P5'}
        return badges.get(p, f'P{p}')

    def issue_table(rows):
        if not rows:
            return
        lines.append("| 优先级 | 类型 | 对象 | 详情 |")
        lines.append("|--------|------|------|------|")
        for r in rows:
            lines.append(f"| {priority_badge(r.get('priority', 3))} | {r['type']} | `{r.get('object','')}` | {r.get('detail','')} |")
        lines.append("")

    # ===== 报告头 =====
    ts = datetime.now().strftime('%Y-%m-%d %H:%M:%S')
    lines += [
        "# SQL 优化诊断建议报告",
        "",
        f"**SQL_ID**: `{sql_id}`",
        f"**诊断时间**: {ts}",
        f"**连接**: `{host}:{port}` / `root@sys#enmotest`",
        "",
    ]

    # ===== 1. 执行计划分析 =====
    section("1. 执行计划分析")
    plan = q("""
        SELECT PLAN_ID, OPERATION, OBJECT_NAME, OBJECT_TYPE, OPTIONS,
               COST, ROWS_PROCESSED, BYTES,
               PHYSICAL_READ_REQUESTS, PHYSICAL_READ_BYTES,
               ACCESS_PREDICATES, FILTER_PREDICATES,
               PLAN_DEPTH, POSITION
        FROM oceanbase.GV$OB_SQL_PLAN
        WHERE SQL_ID = %s
        ORDER BY PLAN_ID, POSITION
    """, (sql_id,))

    if not plan:
        lines += [
            "*未获取到执行计划（计划已淘汰或 SQL_ID 有误）*",
            "> 建议：确认 SQL_ID 正确，或检查 Plan Cache 配置"
        ]
        cursor.close()
        conn.close()
        return '\n'.join(lines)

    lines.append(f"*共 {len(plan)} 个计划步骤*\n")

    # 逐计划 ID 分析
    plan_ids = sorted(set(p['PLAN_ID'] for p in plan))
    for pid in plan_ids:
        plan_steps = [p for p in plan if p['PLAN_ID'] == pid]
        lines.append(f"\n**PLAN_ID={pid}**（{len(plan_steps)} 步）\n")

        plan_issues, plan_suggestions = [], []

        for p in plan_steps:
            op = p['OPERATION'] or ''
            obj = p['OBJECT_NAME'] or ''
            obj_type = p['OBJECT_TYPE'] or ''
            cost = p.get('COST') or 0
            rows = p.get('ROWS_PROCESSED') or 0
            pred = str(p.get('ACCESS_PREDICATES') or '')
            filt = str(p.get('FILTER_PREDICATES') or '')

            # P1: 全表扫描
            if obj_type in ('TABLE', 'BASIC TABLE') and 'SCAN' in op.upper():
                plan_issues.append({
                    'priority': 1, 'type': 'FULL_TABLE_SCAN',
                    'object': obj, 'cost': cost, 'rows': rows,
                    'detail': f'{op} 表 `{obj}`，处理 {rows} 行，成本 {cost}'
                })
                plan_suggestions.append({
                    'priority': 1, 'category': 'INDEX',
                    'problem': f'表 `{obj}` 存在全表/索引扫描',
                    'action': f'检查 WHERE 条件列是否有索引：SHOW INDEX FROM {obj}',
                    'example': f'CREATE INDEX idx_{obj.lower()}_x ON {obj}(col);'
                })

            # P2: 大数据量 NL JOIN
            if 'NESTED LOOP' in op.upper() and rows > 5000:
                plan_issues.append({
                    'priority': 2, 'type': 'NL_JOIN_LARGE',
                    'object': obj, 'rows': rows,
                    'detail': f'嵌套循环连接内表 `{obj}` 处理 {rows} 行'
                })
                plan_suggestions.append({
                    'priority': 2, 'category': 'JOIN',
                    'problem': f'NL JOIN 内表行数过多（{rows}）',
                    'action': '为内表驱动列加索引，或改用 HASH JOIN',
                    'example': '-- 查看驱动表，加索引减少内表扫描'
                })

            # P2: SORT 大数据量
            if ('SORT' in op.upper() or 'ORDER BY' in pred) and rows > 10000:
                plan_issues.append({
                    'priority': 2, 'type': 'EXPENSIVE_SORT',
                    'object': obj, 'rows': rows,
                    'detail': f'SORT 操作处理 {rows} 行，可能溢出磁盘'
                })
                plan_suggestions.append({
                    'priority': 2, 'category': 'INDEX',
                    'problem': f'SORT 行数过多（{rows}）',
                    'action': '在排序列上建索引，利用索引顺序避免排序',
                    'example': f'CREATE INDEX idx_{obj.lower()}_sort ON {obj}(sort_col);'
                })

            # P1: 笛卡尔积
            if 'CARTESIAN' in op.upper() or 'CARTESIAN' in obj.upper():
                plan_issues.append({
                    'priority': 1, 'type': 'CARTESIAN_PRODUCT',
                    'object': obj, 'rows': rows,
                    'detail': '检测到笛卡尔积，结果行数可能爆炸'
                })
                plan_suggestions.append({
                    'priority': 1, 'category': 'SQL',
                    'problem': '存在笛卡尔积连接',
                    'action': '检查所有 JOIN 是否都有 ON 条件',
                    'example': '-- 确认 JOIN 条件完整'
                })

            # P3: 物理读
            phys_reads = p.get('PHYSICAL_READ_REQUESTS') or 0
            if phys_reads > 10000:
                plan_issues.append({
                    'priority': 3, 'type': 'HIGH_PHYSICAL_READ',
                    'object': obj, 'rows': phys_reads,
                    'detail': f'{obj} 物理读 {phys_reads} 次，数据未命中缓存'
                })
                plan_suggestions.append({
                    'priority': 3, 'category': 'INDEX',
                    'problem': f'{obj} 物理读过多（{phys_reads}）',
                    'action': '考虑加索引减少全表扫描，或增加 buffer pool',
                    'example': '-- 增加数据缓存命中率'
                })

        # 输出该计划的问题
        issue_table(plan_issues)
        for s in sorted(plan_suggestions, key=lambda x: x['priority']):
            lines.append(f"**[{priority_badge(s['priority'])}] {s['problem']}**")
            lines.append(f"- 建议: {s['action']}")
            if s.get('example'):
                lines.append(f"```sql\n{s['example']}\n```")
            lines.append("")

        all_issues.extend(plan_issues)
        all_suggestions.extend(plan_suggestions)

    # ===== 2. SQL 文本分析 =====
    section("2. SQL 文本分析")
    texts = q("""
        SELECT TENANT_NAME, LEFT(QUERY_SQL, 800) AS sql_preview
        FROM oceanbase.GV$OB_PLAN_CACHE_PLAN_STAT a
        JOIN oceanbase.DBA_OB_TENANTS t ON a.TENANT_ID = t.TENANT_ID
        WHERE a.SQL_ID = %s LIMIT 1
    """, (sql_id,))

    if texts:
        sql_text = texts[0]['sql_preview'] or ''
        lines.append(f"**租户**: {texts[0]['TENANT_NAME']}\n")
        lines.append("```sql")
        lines.append(sql_text)
        lines.append("```\n")

        import re
        # SELECT *
        if re.search(r'SELECT\s+\*\s+FROM', sql_text, re.IGNORECASE):
            all_suggestions.append({
                'priority': 4, 'category': 'SQL',
                'problem': '使用 SELECT *',
                'action': '仅查询需要的列，减少网络传输和内存占用',
                'example': '-- 改为 SELECT col1, col2 FROM t WHERE ...'
            })

        # LIKE 前缀通配
        if re.search(r'LIKE\s+[\'"][\^%]', sql_text, re.IGNORECASE):
            all_suggestions.append({
                'priority': 4, 'category': 'SQL',
                'problem': 'LIKE 前缀通配导致索引失效',
                'action': '改用后缀通配 `%xxx%` 或全文索引',
                'example': '-- 考虑全文索引替代'
            })

        # OR 过多
        or_count = len(re.findall(r'\bOR\b', sql_text, re.IGNORECASE))
        if or_count >= 3:
            all_suggestions.append({
                'priority': 4, 'category': 'SQL',
                'problem': f'SQL 中有 {or_count} 个 OR，可能无法使用组合索引',
                'action': '评估改写为 UNION ALL 或确认各条件有独立索引',
                'example': '-- OR -> UNION ALL 改写'
            })

    # ===== 3. 统计信息检查 =====
    section("3. 统计信息检查")
    objs = q("""
        SELECT DISTINCT TENANT_ID, OBJECT_OWNER, OBJECT_NAME, OBJECT_TYPE
        FROM oceanbase.GV$OB_SQL_PLAN
        WHERE SQL_ID = %s AND OBJECT_TYPE IN ('TABLE', 'BASIC TABLE')
        ORDER BY OBJECT_OWNER, OBJECT_NAME
    """, (sql_id,))

    stats_issues = []
    for o in objs:
        owner, tname = o['OBJECT_OWNER'], o['OBJECT_NAME']
        cursor.execute("""
            SELECT TABLE_NAME, NUM_ROWS, BLOCKS, LAST_ANALYZED,
                   GLOBAL_STATS, STALE_STATS
            FROM oceanbase.DBA_TAB_STATISTICS
            WHERE OWNER = %s AND TABLE_NAME = %s
        """, (owner, tname))
        row = cursor.fetchone()
        if not row:
            stats_issues.append({
                'priority': 2, 'type': 'NO_STATS',
                'object': f'{owner}.{tname}',
                'detail': '无统计信息'
            })
            all_suggestions.append({
                'priority': 2, 'category': 'STATS',
                'problem': f'表 `{owner}.{tname}` 无统计信息',
                'action': f'执行 ANALYZE TABLE {owner}.{tname} COMPUTE STATISTICS;',
                'example': f'ANALYZE TABLE {owner}.{tname} COMPUTE STATISTICS;'
            })
        else:
            last_an = row[3]
            num_rows = row[1] or 0
            if last_an:
                from datetime import datetime
                if isinstance(last_an, str):
                    try:
                        last_an = datetime.fromisoformat(last_an.replace(' ', 'T'))
                    except:
                        last_an = None
                if last_an:
                    age_days = (datetime.now() - last_an).days
                    if age_days > 7:
                        stats_issues.append({
                            'priority': 3, 'type': 'STALE_STATS',
                            'object': f'{owner}.{tname}',
                            'detail': f'统计信息 {age_days} 天未更新（{last_an.strftime("%Y-%m-%d")}）'
                        })
                        all_suggestions.append({
                            'priority': 3, 'category': 'STATS',
                            'problem': f'表 `{owner}.{tname}` 统计信息过期（{age_days}天）',
                            'action': f'执行 ANALYZE TABLE {owner}.{tname} COMPUTE STATISTICS;',
                            'example': f'ANALYZE TABLE {owner}.{tname} COMPUTE STATISTICS;'
                        })

    if stats_issues:
        issue_table(stats_issues)
    else:
        lines.append("*统计信息正常*\n")

    # ===== 4. 索引覆盖分析 =====
    section("4. 索引覆盖分析")
    for o in objs:
        owner, tname = o['OBJECT_OWNER'], o['OBJECT_NAME']
        indexes = q("""
            SELECT s.INDEX_NAME, ic.COLUMN_NAME, ic.COLUMN_POSITION
            FROM oceanbase.DBA_INDEXES s
            JOIN oceanbase.DBA_IND_COLUMNS ic
                  ON s.OWNER = ic.INDEX_OWNER
                 AND s.INDEX_NAME = ic.INDEX_NAME
                 AND s.TABLE_NAME = ic.TABLE_NAME
            WHERE s.OWNER = %s AND s.TABLE_NAME = %s
            ORDER BY s.INDEX_NAME, ic.COLUMN_POSITION
        """, (owner, tname))

        if not indexes:
            all_suggestions.append({
                'priority': 2, 'category': 'INDEX',
                'problem': f'表 `{owner}.{tname}` 无索引',
                'action': '根据 WHERE/JOIN/ORDER BY 列添加索引',
                'example': f'CREATE INDEX idx_{tname.lower()}_x ON {owner}.{tname}(col1, col2);'
            })
        else:
            idx_dict = defaultdict(list)
            for r in indexes:
                idx_dict[r['INDEX_NAME']].append(r['COLUMN_NAME'])

            lines.append(f"\n**表 `{owner}.{tname}` 索引（共 {len(idx_dict)} 个）**\n")
            lines.append("| 索引名 | 列顺序 |")
            lines.append("|--------|--------|")
            for iname, cols in idx_dict.items():
                lines.append(f"| `{iname}` | {', '.join(cols)} |")
            lines.append("")

            # 检查是否可以覆盖 WHERE
            cursor.execute("""
                SELECT DISTINCT ACCESS_PREDICATES, FILTER_PREDICATES
                FROM oceanbase.GV$OB_SQL_PLAN
                WHERE SQL_ID = %s AND OBJECT_NAME = %s
                  AND (ACCESS_PREDICATES IS NOT NULL OR FILTER_PREDICATES IS NOT NULL)
            """, (sql_id, tname))
            pred_cols = set()
            for r in cursor.fetchall():
                import re
                for col in re.findall(r'[a-zA-Z_][a-zA-Z0-9_]*', str(r[0]) + str(r[1])):
                    if len(col) > 2:  # 过滤短列名误匹配
                        pred_cols.add(col)

            for iname, cols in idx_dict.items():
                covered = pred_cols & set(cols)
                if covered:
                    all_suggestions.append({
                        'priority': 3, 'category': 'INDEX',
                        'problem': f'索引 `{iname}` 可覆盖 WHERE 列 {covered}',
                        'action': '确认查询使用了该索引（EXPLAIN 查看 access_type）',
                        'example': f'-- 索引 {iname} 列: {cols}'
                    })

    # ===== 5. Audit 历史表现 =====
    section("5. Audit 历史执行表现")
    audit = q("""
        SELECT
            COUNT(*)                                         AS exec_count,
            ROUND(AVG(ELAPSED_TIME)/1000, 1)               AS avg_ms,
            ROUND(MAX(ELAPSED_TIME)/1000, 1)               AS max_ms,
            SUM(DISK_READS)                                 AS total_disk_reads,
            SUM(TABLE_SCAN)                                 AS full_scan_count,
            SUM(IS_HIT_PLAN)                               AS cache_hit_count
        FROM oceanbase.GV$OB_SQL_AUDIT
        WHERE SQL_ID = %s AND IS_INNER_SQL = 0
    """, (sql_id,))

    if audit and audit[0]['exec_count'] > 0:
        a = audit[0]
        lines += [
            f"| 指标 | 值 |",
            f"|------|----|",
            f"| 执行次数 | {a['exec_count']} |",
            f"| 平均耗时 | {a['avg_ms']} ms |",
            f"| 最大耗时 | {a['max_ms']} ms |",
            f"| 磁盘读总数 | {a['total_disk_reads']} |",
            f"| 全表扫次数 | {a['full_scan_count']} |",
            f"| 计划缓存命中 | {a['cache_hit_count']} |",
            "",
        ]

        if float(a['avg_ms']) > 1000:
            all_suggestions.append({
                'priority': 1, 'category': 'PERF',
                'problem': f"平均耗时 {a['avg_ms']}ms，超过 1s",
                'action': '综合执行计划和索引分析结果优化',
                'example': '-- 参考上方 P1/P2 建议'
            })
    else:
        lines.append("*Audit 窗口内无执行记录*\n")

    # ===== 6. 汇总建议 =====
    section("6. 优化建议汇总（按优先级）")

    if all_suggestions:
        dedup = {}
        for s in all_suggestions:
            key = s['problem']
            if key not in dedup or s['priority'] < dedup[key]['priority']:
                dedup[key] = s

        for s in sorted(dedup.values(), key=lambda x: x['priority'])[:15]:
            lines += [
                f"### {priority_badge(s['priority'])} {s['category']}: {s['problem']}",
                "",
                f"**建议**: {s['action']}",
                "",
            ]
            if s.get('example'):
                lines.append("```sql")
                lines.append(s['example'])
                lines.append("```\n")
            else:
                lines.append("")
    else:
        lines += [
            "*未发现明显性能问题*",
            "",
            "> 如仍有性能疑虑，请检查：",
            "> 1. 执行频率是否过高（考虑缓存结果）",
            "> 2. 是否存在锁等待（检查gv$ob_processlist）",
            "> 3. 网络延迟是否正常"
        ]

    # ===== 7. 一键操作命令 =====
    section("7. 一键优化操作参考")
    lines += [
        "```bash",
        "# 1. 重新收集统计信息（解决统计陈旧问题）",
        "# 在对应租户执行：",
        "ANALYZE TABLE <owner>.<table_name> COMPUTE STATISTICS;",
        "",
        "# 2. 刷新执行计划（解决计划缓存问题）",
        "ALTER SYSTEM FLUSH PLAN CACHE;",
        "",
        "# 3. 查看实时执行情况",
        f"SELECT * FROM oceanbase.GV$OB_SQL_AUDIT WHERE SQL_ID = '{sql_id}' ORDER BY REQUEST_TIME DESC LIMIT 10;",
        "",
        "# 4. 验证优化效果（对比优化前后 avg_ms）",
        "```",
        "",
    ]

    # ===== 报告尾 =====
    lines += [
        "---",
        f"*报告生成时间: {ts}*",
        "*由 sql-optimization.md skill 自动生成*",
    ]

    cursor.close()
    conn.close()
    return '\n'.join(lines)


# ---- 快速调用示例 ----
if __name__ == '__main__':
    import sys
    if len(sys.argv) < 2:
        print("用法: python sql_optimization.py <SQL_ID>")
        sys.exit(1)

    sql_id = sys.argv[1]
    pwd = os.getenv('OB_PASSWORD', 'YOUR_PASSWORD')

    report = sql_optimization_report(sql_id, password=pwd)

    out_file = f"sql_optimization_{sql_id[:8]}_{datetime.now().strftime('%Y%m%d_%H%M%S')}.md"
    with open(out_file, 'w', encoding='utf-8') as f:
        f.write(report)
    print(f"报告已写入: {out_file}")
```

---

## 诊断规则优先级说明

| 优先级 | 标识 | 问题类型 | 预期影响 |
|--------|------|---------|---------|
| P1 | 🔴 | 全表扫描、笛卡尔积、无统计信息 | 高，每次执行都慢 |
| P2 | 🟠 | 大数据量NL JOIN、SORT溢出、统计过期 | 中，特定条件下慢 |
| P3 | 🟡 | 高物理读、索引可覆盖、排序多 | 中，优化收益明显 |
| P4 | 🔵 | SELECT *、OR过多、LIKE前缀通配 | 低，SQL质量问题 |
| P5 | ⚪ | 代码风格建议 | 低，运维规范 |

---

## 相关 Skills

- `skills/performance/sql-collect.md` — SQL 信息采集（采集是优化的前置步骤）
- `skills/performance/explain-plan.md` — 执行计划解读指南
- `skills/performance/slow-query-analysis.md` — 慢SQL分析与诊断
- `skills/monitoring/sql-diagnostics.md` — SQL 诊断工具箱

---

*Version: OceanBase V4.2+ | Last updated: 2026-05-15*
