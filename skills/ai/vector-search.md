# 向量检索与向量索引

> 适用版本：OceanBase V4.3+
> 兼容模式：MySQL Mode / Oracle Mode

---

## 概述

OceanBase V4.3+ 引入向量数据类型和向量索引，支持高性能的近似最近邻（ANN）搜索，可直接在数据库中实现 AI 应用所需的向量检索能力。

| 特性 | 说明 |
|------|------|
| 向量类型 | `VECTOR(n)` 定长浮点向量 |
| 向量索引 | IVF-FLAT、HNSW |
| 距离函数 | 欧氏距离（L2）、内积（IP）、余弦相似度 |
| 语法 | SQL 标准语法，与标量查询混合 |
| 事务 | 向量操作支持 ACID 事务 |

---

## 1. 向量数据类型

### 1.1 创建向量列

```sql
-- MySQL Mode
CREATE TABLE documents (
  id         BIGINT PRIMARY KEY,
  title      VARCHAR(256),
  content    TEXT,
  embedding  VECTOR(768),        -- 768维向量
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- Oracle Mode
CREATE TABLE documents (
  id         NUMBER(20) PRIMARY KEY,
  title      VARCHAR2(256),
  content    CLOB,
  embedding  VECTOR(768),
  created_at DATE DEFAULT SYSDATE
);
```

### 1.2 插入向量数据

```sql
-- 直接插入向量
INSERT INTO documents (id, title, content, embedding) VALUES (
  1,
  'OceanBase 分布式架构',
  'OceanBase 采用 Paxos 协议...',
  VECTOR('[0.1, 0.2, 0.3, ..., 0.5]')  -- 768维
);

-- 使用字符串格式插入
INSERT INTO documents (id, title, embedding) VALUES (
  2, '数据库索引', '[0.15, 0.25, 0.35, ..., 0.55]'
);

-- 批量插入（推荐）
INSERT INTO documents (id, title, embedding) VALUES
  (3, 'SQL优化', '[0.12, 0.22, 0.32, ..., 0.52]'),
  (4, '事务管理', '[0.18, 0.28, 0.38, ..., 0.58]');
```

---

## 2. 距离函数

### 2.1 支持的距离度量

| 距离函数 | SQL 语法 | 说明 | 适用场景 |
|---------|---------|------|---------|
| L2 距离 | `L2_DISTANCE(v1, v2)` | 欧氏距离 | 通用 |
| 内积 | `INNER_PRODUCT(v1, v2)` | 点积 | 归一化向量 |
| 余弦距离 | `COSINE_DISTANCE(v1, v2)` | 1 - 余弦相似度 | 文本相似度 |

```sql
-- L2 距离（越小越相似）
SELECT id, title, L2_DISTANCE(embedding, VECTOR('[0.1, 0.2, ...]')) AS distance
FROM documents
ORDER BY distance ASC
LIMIT 10;

-- 余弦距离（越小越相似）
SELECT id, title, COSINE_DISTANCE(embedding, VECTOR('[0.1, 0.2, ...]')) AS distance
FROM documents
ORDER BY distance ASC
LIMIT 10;

-- 内积（越大越相似）
SELECT id, title, INNER_PRODUCT(embedding, VECTOR('[0.1, 0.2, ...]')) AS score
FROM documents
ORDER BY score DESC
LIMIT 10;
```

### 2.2 向量与标量混合查询

```sql
-- 向量相似度 + 分类过滤 + 日期范围
SELECT id, title,
  COSINE_DISTANCE(embedding, VECTOR('[0.1, 0.2, ...]')) AS distance
FROM documents
WHERE category = 'database'
  AND created_at >= '2024-01-01'
ORDER BY distance ASC
LIMIT 10;
```

---

## 3. 向量索引

### 3.1 IVF-FLAT 索引

```sql
-- 创建 IVF-FLAT 向量索引
CREATE VECTOR INDEX idx_documents_embedding
ON documents(embedding)
WITH (
  type = 'IVF_FLAT',
  lists = 100,
  metric = 'L2'
);

-- 参数说明：
-- lists: 聚类中心数（推荐 sqrt(行数)）
-- metric: 'L2', 'IP', 'COSINE'
```

### 3.2 HNSW 索引

```sql
-- 创建 HNSW 向量索引
CREATE VECTOR INDEX idx_documents_hnsw
ON documents(embedding)
WITH (
  type = 'HNSW',
  M = 16,
  ef_construction = 200,
  metric = 'COSINE'
);

-- 参数说明：
-- M: 每层最大连接数（推荐 16-64）
-- ef_construction: 构建时搜索范围（推荐 200-500）
```

### 3.3 索引选择

| 索引类型 | 查询精度 | 查询速度 | 构建速度 | 内存占用 | 适用场景 |
|---------|---------|---------|---------|---------|---------|
| IVF-FLAT | 高 | 中 | 快 | 低 | 中等规模数据 |
| HNSW | 高 | 快 | 慢 | 高 | 大规模数据、低延迟 |

---

## 4. 使用示例

### 4.1 文档检索（RAG）

```sql
-- 步骤1：存储文档向量
INSERT INTO documents (id, title, content, embedding) VALUES (
  1,
  'OceanBase 架构',
  'OceanBase 是分布式关系型数据库...',
  VECTOR(embedding_from_openai('OceanBase 是分布式关系型数据库...'))
);

-- 步骤2：向量检索 Top-K
SELECT id, title, content,
  1 - COSINE_DISTANCE(embedding, VECTOR(:query_embedding)) AS similarity
FROM documents
ORDER BY similarity DESC
LIMIT 5;

-- 步骤3：检索结果传给 LLM 生成回答
```

### 4.2 图像搜索

```sql
-- 图像特征存储
CREATE TABLE images (
  id BIGINT PRIMARY KEY,
  url VARCHAR(512),
  feature VECTOR(512),
  category VARCHAR(64)
);

CREATE VECTOR INDEX idx_images_feature ON images(feature)
WITH (type = 'HNSW', M = 32, metric = 'L2');

-- 以图搜图
SELECT id, url, L2_DISTANCE(feature, VECTOR(:query_image_feature)) AS distance
FROM images
WHERE category = 'landscape'
ORDER BY distance ASC
LIMIT 20;
```

### 4.3 推荐系统

```sql
-- 用户向量 + 物品向量
CREATE TABLE user_profiles (
  user_id BIGINT PRIMARY KEY,
  embedding VECTOR(128)
);

CREATE TABLE item_profiles (
  item_id BIGINT PRIMARY KEY,
  embedding VECTOR(128),
  category VARCHAR(64)
);

-- 推荐最相似物品
SELECT i.item_id, i.category,
  INNER_PRODUCT(u.embedding, i.embedding) AS score
FROM user_profiles u
CROSS JOIN item_profiles i
WHERE u.user_id = 1001
  AND i.category != 'purchased'
ORDER BY score DESC
LIMIT 20;
```

---

## 5. 性能优化

| 优化项 | 建议 |
|--------|------|
| 维度选择 | 使用降维后的向量（128-768维） |
| 索引类型 | 数据量 > 100万用 HNSW |
| HNSW M 值 | 推荐 16-32，精度和速度平衡 |
| 批量插入 | 批量插入后手动 BUILD INDEX |
| 混合查询 | 标量条件先过滤，缩小向量搜索范围 |

---

## 6. 常见错误

| 问题 | 原因 | 解决 |
|------|------|------|
| `dimension mismatch` | 向量维度不一致 | 确保所有向量维度相同 |
| 索引未使用 | 查询未触发索引 | 检查距离函数与索引 metric 一致 |
| 构建慢 | 数据量大 | 增大并行度或分批构建 |
| 精度低 | lists/M 值太小 | 增大索引参数 |

---

## 参考文档

- 向量检索：https://www.oceanbase.com/docs/oceanbase-database-cn/V430/vector-search
- 向量索引：https://www.oceanbase.com/docs/oceanbase-database-cn/V430/vector-index
- VECTOR 类型：https://www.oceanbase.com/docs/oceanbase-database-cn/V430/vector-data-type
