# AI 函数服务

> 适用版本：OceanBase V4.3+
> 兼容模式：MySQL Mode / Oracle Mode

---

## 概述

OceanBase V4.3+ 提供 AI 函数服务，允许在 SQL 中直接调用大语言模型（LLM）和 Embedding 模型，实现数据库原生的 AI 能力。

| 功能 | 说明 |
|------|------|
| Embedding 函数 | 文本转向量，用于向量检索 |
| LLM 函数 | SQL 中调用大模型生成文本 |
| RAG 支持 | 数据库内完成检索增强生成 |
| 模型管理 | 支持对接外部模型服务 |

---

## 1. Embedding 函数

### 1.1 配置 Embedding 模型

```sql
-- 配置外部 Embedding 服务（如 OpenAI）
ALTER SYSTEM SET ai_embedding_endpoint = 'https://api.openai.com/v1/embeddings';
ALTER SYSTEM SET ai_embedding_model = 'text-embedding-ada-002';
ALTER SYSTEM SET ai_embedding_api_key = 'sk-xxxxxxx';
ALTER SYSTEM SET ai_embedding_dimensions = 1536;
```

### 1.2 使用 Embedding 函数

```sql
-- 文本转向量
SELECT AI_EMBEDDING('OceanBase 是分布式数据库') AS embedding;

-- 批量生成向量并插入
INSERT INTO documents (id, title, content, embedding)
SELECT id, title, content,
  AI_EMBEDDING(CONCAT(title, ' ', content))
FROM source_documents;

-- 更新已有数据的向量
UPDATE documents
SET embedding = AI_EMBEDDING(content)
WHERE embedding IS NULL;
```

### 1.3 存储过程批量处理

```sql
-- Oracle Mode
CREATE OR REPLACE PROCEDURE generate_embeddings(p_batch_size NUMBER) IS
  CURSOR c_docs IS
    SELECT id, content FROM documents WHERE embedding IS NULL FETCH NEXT p_batch_size ROWS ONLY;
BEGIN
  FOR r IN c_docs LOOP
    UPDATE documents
    SET embedding = AI_EMBEDDING(r.content)
    WHERE id = r.id;
    COMMIT;
  END LOOP;
END;
/
```

---

## 2. LLM 函数

### 2.1 配置 LLM 模型

```sql
-- 配置 LLM 服务
ALTER SYSTEM SET ai_llm_endpoint = 'https://api.openai.com/v1/chat/completions';
ALTER SYSTEM SET ai_llm_model = 'gpt-4';
ALTER SYSTEM SET ai_llm_api_key = 'sk-xxxxxxx';
ALTER SYSTEM SET ai_llm_max_tokens = 2000;
ALTER SYSTEM SET ai_llm_temperature = 0.7;
```

### 2.2 使用 LLM 函数

```sql
-- 直接调用 LLM
SELECT AI_CHAT('请解释一下什么是 Paxos 协议') AS answer;

-- 带上下文调用
SELECT AI_CHAT(
  '根据以下文档内容回答问题：',
  'OceanBase 使用 Paxos 协议保证数据一致性...',
  '问题：OceanBase 使用什么协议保证一致性？'
) AS answer;

-- 文本摘要
SELECT id, title,
  AI_CHAT(CONCAT('请用一句话总结以下内容：', content)) AS summary
FROM documents
WHERE category = 'tech';
```

### 2.3 流式输出

```sql
-- Oracle Mode：使用管道函数实现流式
CREATE OR REPLACE FUNCTION generate_answer(p_question VARCHAR2)
RETURN SYS_REFCURSOR
IS
  v_cursor SYS_REFCURSOR;
BEGIN
  OPEN v_cursor FOR
    SELECT AI_CHAT(p_question, stream => TRUE) AS chunk FROM DUAL;
  RETURN v_cursor;
END;
/
```

---

## 3. RAG（检索增强生成）

### 3.1 完整 RAG 流程

```sql
-- 步骤1：存储文档及向量
INSERT INTO knowledge_base (id, title, content, embedding)
VALUES (
  1,
  'OceanBase 备份恢复',
  'OceanBase 支持物理备份和逻辑备份...',
  AI_EMBEDDING('OceanBase 支持物理备份和逻辑备份...')
);

-- 步骤2：创建向量索引
CREATE VECTOR INDEX idx_kb_embedding
ON knowledge_base(embedding)
WITH (type = 'HNSW', M = 16, metric = 'COSINE');

-- 步骤3：RAG 查询（检索 + 生成）
WITH relevant_docs AS (
  SELECT id, title, content,
    1 - COSINE_DISTANCE(embedding, AI_EMBEDDING(:user_question)) AS similarity
  FROM knowledge_base
  ORDER BY similarity DESC
  LIMIT 5
)
SELECT
  :user_question AS question,
  AI_CHAT(
    '你是 OceanBase 数据库专家。根据以下参考资料回答用户问题。如果参考资料不足以回答，请说明。',
    (SELECT LISTAGG(content, '\n') FROM relevant_docs),
    :user_question
  ) AS answer;
```

### 3.2 自动摘要与问答

```sql
-- 自动生成文档摘要
UPDATE knowledge_base
SET summary = AI_CHAT(CONCAT('请用100字总结以下内容：', content))
WHERE summary IS NULL;

-- 基于文档的问答
SELECT AI_CHAT(
  '你是客服助手。根据以下知识库文档回答问题：',
  (SELECT content FROM knowledge_base WHERE id = 1),
  '如何进行 OceanBase 数据备份？'
) AS answer;
```

---

## 4. 支持的模型服务

| 服务商 | Embedding | LLM |
|--------|-----------|-----|
| OpenAI | text-embedding-ada-002 | GPT-4, GPT-3.5 |
| Azure OpenAI | ada-002 | GPT-4 |
| 通义千问 | text-embedding-v1 | qwen-max |
| 智谱 AI | embedding-2 | glm-4 |
| 自定义 | 兼容 OpenAI API | 兼容 OpenAI API |

```sql
-- 切换模型
ALTER SYSTEM SET ai_llm_model = 'qwen-max';
ALTER SYSTEM SET ai_llm_endpoint = 'https://dashscope.aliyuncs.com/compatible-mode/v1/chat/completions';
ALTER SYSTEM SET ai_llm_api_key = 'sk-xxxxxxx';
```

---

## 5. 性能考虑

| 考虑项 | 建议 |
|--------|------|
| Embedding 批量生成 | 分批处理，避免一次性调用量过大 |
| LLM 响应时间 | 通常 1-10 秒，不适合高频实时查询 |
| API 限流 | 注意模型服务的 Rate Limit |
| 缓存策略 | 相同问题的答案可缓存 |
| 向量维度 | 选择合适的维度平衡精度和性能 |

---

## 6. 安全注意事项

| 注意事项 | 说明 |
|---------|------|
| API Key 保护 | 系统变量存储，避免泄露 |
| 数据脱敏 | 发送给 LLM 的内容需脱敏 |
| 输入验证 | 限制 prompt 长度防止滥用 |
| 审计日志 | 记录 AI 函数调用 |

---

## 参考文档

- AI 函数：https://www.oceanbase.com/docs/oceanbase-database-cn/V430/ai-functions
- AI_EMBEDDING：https://www.oceanbase.com/docs/oceanbase-database-cn/V430/ai-embedding
- AI_CHAT：https://www.oceanbase.com/docs/oceanbase-database-cn/V430/ai-chat
