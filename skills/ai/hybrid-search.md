# 混合检索

> 适用版本：OceanBase V4.3+
> 兼容模式：MySQL Mode / Oracle Mode

---

## 概述

混合检索（Hybrid Search）结合向量检索和传统关键词/标量检索，通过多路召回 + 重排序（Rerank）提升搜索质量。OceanBase 原生支持在 SQL 中实现混合检索。

| 检索方式 | 擅长 | 不擅长 |
|---------|------|--------|
| 向量检索 | 语义相似度、模糊匹配 | 精确匹配、ID 查询 |
| 关键词检索 | 精确匹配、专有名词 | 语义理解 |
| 标量过滤 | 属性过滤、范围查询 | 语义匹配 |
| **混合检索** | **综合以上优势** | — |

---

## 1. 多路召回

### 1.1 两路召回架构

```
用户查询
  ├→ 向量检索（语义相关）
  ├→ 全文检索/关键词检索（字面匹配）
  └→ 合并去重 + 重排序 → 最终结果
```

### 1.2 向量 + 全文索引联合查询

```sql
-- 创建全文索引（MySQL Mode）
ALTER TABLE documents ADD FULLTEXT INDEX ft_content(content) WITH PARSER ngram;

-- 创建向量索引
CREATE VECTOR INDEX idx_embedding ON documents(embedding)
WITH (type = 'HNSW', M = 16, metric = 'COSINE');

-- 路径1：向量召回
SELECT id, title,
  1 - COSINE_DISTANCE(embedding, AI_EMBEDDING(:query)) AS vector_score
FROM documents
WHERE MATCH(content) AGAINST(:query IN NATURAL LANGUAGE MODE)
   OR COSINE_DISTANCE(embedding, AI_EMBEDDING(:query)) < 0.3
ORDER BY vector_score DESC
LIMIT 50;
```

### 1.3 融合评分

```sql
-- RRF（Reciprocal Rank Fusion）融合
WITH vector_results AS (
  SELECT id,
    1 - COSINE_DISTANCE(embedding, AI_EMBEDDING(:query)) AS score,
    ROW_NUMBER() OVER (ORDER BY COSINE_DISTANCE(embedding, AI_EMBEDDING(:query)) ASC) AS rank_v
  FROM documents
  ORDER BY score DESC LIMIT 20
),
text_results AS (
  SELECT id,
    MATCH(content) AGAINST(:query IN NATURAL LANGUAGE MODE) AS score,
    ROW_NUMBER() OVER (ORDER BY score DESC) AS rank_t
  FROM documents
  WHERE MATCH(content) AGAINST(:query IN NATURAL LANGUAGE MODE)
  LIMIT 20
),
fused AS (
  SELECT
    COALESCE(v.id, t.id) AS id,
    COALESCE(v.score, 0) AS vector_score,
    COALESCE(t.score, 0) AS text_score,
    COALESCE(1.0/(60 + v.rank_v), 0) +
    COALESCE(1.0/(60 + t.rank_t), 0) AS rrf_score
  FROM vector_results v
  FULL OUTER JOIN text_results t ON v.id = t.id
)
SELECT f.id, d.title, d.content,
  f.vector_score, f.text_score, f.rrf_score
FROM fused f
JOIN documents d ON f.id = d.id
ORDER BY f.rrf_score DESC
LIMIT 10;
```

---

## 2. 标量过滤 + 向量检索

### 2.1 先过滤后检索（推荐）

```sql
-- 先用标量条件缩小范围，再向量排序
SELECT id, title, category, created_at,
  COSINE_DISTANCE(embedding, AI_EMBEDDING(:query)) AS distance
FROM documents
WHERE category = 'database'
  AND created_at >= '2024-01-01'
  AND status = 'published'
ORDER BY distance ASC
LIMIT 10;
```

### 2.2 元数据增强

```sql
-- 带元数据的混合查询
SELECT id, title, category,
  COSINE_DISTANCE(embedding, AI_EMBEDDING(:query)) AS distance,
  MATCH(content) AGAINST(:query IN NATURAL LANGUAGE MODE) AS text_score,
  (1 - COSINE_DISTANCE(embedding, AI_EMBEDDING(:query))) * 0.7 +
    LEAST(MATCH(content) AGAINST(:query IN NATURAL LANGUAGE MODE) / 10, 1) * 0.3
  AS combined_score
FROM documents
WHERE category IN ('database', 'distributed-system')
ORDER BY combined_score DESC
LIMIT 10;
```

---

## 3. 重排序（Rerank）

### 3.1 数据库内重排序

```sql
-- 第一阶段：多路召回 Top-K
WITH recall AS (
  SELECT id, title, content,
    COALESCE(1.0/(60 + rank_v), 0) + COALESCE(1.0/(60 + rank_t), 0) AS rrf_score
  FROM (
    -- 向量召回
    SELECT id, ROW_NUMBER() OVER (ORDER BY COSINE_DISTANCE(embedding, AI_EMBEDDING(:query)) ASC) AS rank_v, 0 AS rank_t
    FROM documents LIMIT 20
  ) v
  FULL OUTER JOIN (
    -- 文本召回
    SELECT id, 0 AS rank_v, ROW_NUMBER() OVER (ORDER BY MATCH(content) AGAINST(:query IN NATURAL LANGUAGE MODE) DESC) AS rank_t
    FROM documents WHERE MATCH(content) AGAINST(:query IN NATURAL LANGUAGE MODE) LIMIT 20
  ) t ON v.id = t.id
)
-- 第二阶段：用 LLM 重排序
SELECT r.id, r.title, r.rrf_score,
  AI_CHAT(
    '请评估以下文档与查询的相关性，返回0-10的分数：',
    CONCAT('查询：', :query, '\n文档：', d.content),
    '仅返回一个数字。'
  ) AS llm_score
FROM recall r
JOIN documents d ON r.id = d.id
ORDER BY CAST(llm_score AS DECIMAL(3,1)) DESC
LIMIT 5;
```

### 3.2 应用层重排序

推荐在应用层使用专用 Rerank 模型：

```python
# Python 伪代码
# 第一阶段：SQL 多路召回
recall_results = db.execute(hybrid_recall_sql)

# 第二阶段：Rerank 模型重排
from sentence_transformers import CrossEncoder
reranker = CrossEncoder('BAAI/bge-reranker-v2-m3')

query_pairs = [(query, doc.content) for doc in recall_results]
rerank_scores = reranker.predict(query_pairs)

# 合并排序
for doc, score in zip(recall_results, rerank_scores):
    doc.final_score = 0.3 * doc.rrf_score + 0.7 * score

final_results = sorted(recall_results, key=lambda x: x.final_score, reverse=True)[:10]
```

---

## 4. 完整混合检索示例

### 4.1 文档搜索系统

```sql
-- 建表
CREATE TABLE documents (
  id         BIGINT PRIMARY KEY AUTO_INCREMENT,
  title      VARCHAR(256) NOT NULL,
  content    TEXT NOT NULL,
  category   VARCHAR(64),
  author     VARCHAR(64),
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  embedding  VECTOR(768)
);

-- 创建索引
ALTER TABLE documents ADD FULLTEXT INDEX ft_content(content) WITH PARSER ngram;
CREATE VECTOR INDEX idx_emb ON documents(embedding)
  WITH (type = 'HNSW', M = 16, metric = 'COSINE');
CREATE INDEX idx_category ON documents(category);
CREATE INDEX idx_created ON documents(created_at);

-- 混合检索存储过程
CREATE PROCEDURE hybrid_search(
  IN p_query VARCHAR(512),
  IN p_category VARCHAR(64),
  IN p_limit INT
)
BEGIN
  -- 向量 + 全文 + 标量 混合
  SELECT id, title, category,
    ROUND(1 - COSINE_DISTANCE(embedding, AI_EMBEDDING(p_query)), 4) AS vec_sim,
    ROUND(MATCH(content) AGAINST(p_query IN NATURAL LANGUAGE MODE) / 10, 4) AS txt_sim
  FROM documents
  WHERE (category = p_category OR p_category IS NULL)
    AND (MATCH(content) AGAINST(p_query IN NATURAL LANGUAGE MODE)
         OR COSINE_DISTANCE(embedding, AI_EMBEDDING(p_query)) < 0.5)
  ORDER BY (1 - COSINE_DISTANCE(embedding, AI_EMBEDDING(p_query))) * 0.6
         + LEAST(MATCH(content) AGAINST(p_query IN NATURAL LANGUAGE MODE) / 10, 1) * 0.4
  DESC
  LIMIT p_limit;
END;
```

---

## 5. 性能优化

| 优化项 | 建议 |
|--------|------|
| 索引 | 全文索引 + 向量索引 + 标量索引 |
| 分阶段 | 先标量过滤 → 向量检索 → 文本检索 |
| 缓存 | 缓存热门查询的 Embedding |
| 批量 Embedding | 预计算存储，查询时直接使用 |
| 分区 | 大表按 category 分区 |
| 并发 | 向量和全文检索并行执行 |

---

## 参考文档

- 向量检索：https://www.oceanbase.com/docs/oceanbase-database-cn/V430/vector-search
- 全文索引：https://www.oceanbase.com/docs/oceanbase-database-cn/V420/fulltext-index
- AI 函数：https://www.oceanbase.com/docs/oceanbase-database-cn/V430/ai-functions
