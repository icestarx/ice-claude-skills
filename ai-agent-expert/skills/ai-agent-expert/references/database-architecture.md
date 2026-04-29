# Agent 数据库架构详解

## 目录
- [向量数据库对比](#向量数据库对比)
- [记忆架构](#记忆架构)
- [会话历史存储](#会话历史存储)
- [工具注册表](#工具注册表)
- [选型决策指南](#选型决策指南)
- [SQL/Schema模板](#sqlschema模板)

---

## 向量数据库对比

### pgvector + PostgreSQL（默认推荐）

**核心优势**：
- 一套基础设施同时处理关系数据+向量检索
- ACID事务保证：状态更新和嵌入插入的一致性
- 混合查询：先SQL过滤，再向量排序
- HNSW索引（生产就绪）、半精度向量、多种距离度量

**适用场景**：绝大多数Agent系统。除非你有极端规模需求（10M+向量）或严格多租户隔离要求。

**最佳实践**：
- 同一行存储文本内容（TEXT列）+嵌入向量（VECTOR列）
- 先用SQL WHERE过滤范围，再用向量相似度排序
- 创建HNSW索引：`CREATE INDEX ON documents USING hnsw (embedding vector_cosine_ops)`

**不适用场景**：纯嵌入检索不需要关系数据、超大规模向量、需要零运维本地部署。

---

### Pinecone（大规模选择）

**核心优势**：
- Namespace分区：单索引内隔离conversation_history / agent_state / tool_registry
- Sparse-dense混合检索：语义+关键词联合匹配
- Serverless层：低延迟读取、按用量计费

**适用场景**：百万级向量、大规模生产部署、需要Namespace隔离不同Agent数据域。

**不适用场景**：小规模原型、需要复杂SQL过滤、需要本地部署。

---

### Weaviate（多租户选择）

**核心优势**：
- 多租户特性：每个Agent独立数据隔离
- GraphQL风格过滤+向量检索组合
- 内置Modules：向量化器、生成模块，减少管道复杂度

**适用场景**：多Agent系统需要per-Agent数据隔离、需要丰富过滤条件、需要内嵌向量化。

**不适用场景**：单Agent场景（过于复杂）、小规模本地部署、纯关系数据场景。

---

### Chroma（本地/原型选择）

**核心优势**：
- 嵌入式、零服务器、零配置
- 2025改进：元数据过滤增强，支持复合条件
- 与LangChain/AutoGen/CrewAI深度集成

**适用场景**：原型开发、单Agent本地开发、快速验证概念。

**不适用场景**：生产部署、大规模数据、多Agent协作需要共享存储。

---

### 对比总结

| 维度 | pgvector | Pinecone | Weaviate | Chroma |
|------|----------|----------|----------|--------|
| 默认推荐度 | ★★★★★ | ★★★ | ★★★ | ★★★ |
| 规模上限 | ~1M向量 | 10M+ | 10M+ | ~100K |
| SQL+向量混合 | 天然支持 | 不支持 | GraphQL | 不支持 |
| 多租户隔离 | 手动schema | Namespace | 自动 | 手动 |
| ACID事务 | 天然支持 | 不支持 | 有限 | 不支持 |
| 运维复杂度 | 中（已有PG） | 低（托管） | 高 | 零 |
| 本地部署 | 支持 | 不支持 | 支持 | 天然 |

**选型原则**：pgvector起步，只在明确需求时才切换到专用向量库。

---

## 记忆架构

### 三层记忆模型（2025标准架构）

| 层级 | 类型 | 存储 | Token成本 | 保真度 |
|------|------|------|-----------|--------|
| **短期/情景记忆** | 最近交互原文 | 对话缓冲区（滑动窗口） | 高（原文逐条） | 最高 |
| **中期/语义记忆** | 历史压缩摘要 | 向量数据库+摘要文本 | 中（压缩后） | 中 |
| **长期/程序记忆** | 跨任务知识 | 向量数据库+关系表 | 低（检索时按需） | 低-中 |

### 记忆控制器（高级模式）

独立的"记忆管理Agent"负责：
- 决定什么存入长期记忆（基于相关度评分+时效性+频率）
- 何时压缩短期→中期（token预算触发）
- 何时丢弃过期信息（相关性衰减）
- 跨Agent共享哪些记忆（权限控制）

**实现模式**：
```python
class MemoryController:
    def should_store(self, content, context):
        """决定是否存入长期记忆"""
        relevance = self.compute_relevance(content, context)
        recency = self.compute_recency(context.timestamp)
        frequency = self.compute_frequency(content.topic)
        score = relevance * 0.6 + recency * 0.2 + frequency * 0.2
        return score > THRESHOLD

    def should_compress(self, conversation, token_budget):
        """决定是否压缩对话历史"""
        current_tokens = estimate_tokens(conversation)
        return current_tokens > token_budget * 0.8

    def retrieve(self, query, max_results=5):
        """检索相关记忆"""
        results = self.vector_store.search(query, limit=max_results)
        return self.rank_by_relevance(results, query)
```

---

## 会话历史存储

### 关系型+嵌入混合模式

每条消息同时存储：
- 结构化数据：message_id, role, content, timestamp, session_id
- 嵌入向量：content的embedding用于语义检索

**PostgreSQL实现**：
- messages表：TEXT列存原文 + VECTOR列存嵌入
- 日常查询用SQL（按session_id, 时间范围）
- 需要语义搜索时用向量检索（"找到与X相关的历史对话"）

---

## 工具注册表

### Schema设计

动态工具发现的标准Schema：

```
工具注册表字段：
- tool_id:        UUID，唯一标识
- name:           工具名称（函数名）
- description:    功能描述（用于向量化，Agent动态发现工具）
- input_schema:   JSON Schema（参数定义）
- output_schema:  JSON Schema（返回值定义）
- domain_tags:    领域标签（["finance", "data_access"]）
- tool_type:      类型（api_call | computation | data_access）
- availability:   状态（enabled | disabled | deprecated）
- required_perms: 权限要求（["read", "execute"]）
- embedding:      VECTOR（description的嵌入，用于语义匹配）
```

**动态发现机制**：Agent不需要硬编码工具列表。输入任务描述→向量检索最相关工具→自动构建可用工具集。

---

## 选型决策指南

```
存储需求分析
│
├─ 主要需求是什么？
│   ├─ 关系数据+少量向量 → PostgreSQL + pgvector（一套搞定）
│   ├─ 纯大规模向量检索 → Pinecone（托管，免运维）
│   ├─ 多租户+丰富过滤 → Weaviate（自动隔离）
│   └─ 本地原型快速验证 → Chroma（零配置）
│
├─ 需要ACID事务吗？
│   ├─ YES → pgvector（唯一选择）
│   └─ NO → 任意向量库
│
├─ 规模预期？
│   ├─ < 100K → pgvector 或 Chroma
│   ├─ 100K - 1M → pgvector（优化索引）
│   ├─ > 1M → Pinecone 或 Weaviate
│
├─ 是否需要语义+关键词混合检索？
│   ├─ YES → Pinecone（sparse-dense）或 Weaviate（过滤+向量）
│   └─ NO → pgvector（SQL+向量足够）
```

---

## SQL/Schema模板

### Agent状态表

```sql
CREATE TABLE agent_state (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agent_id        VARCHAR(64) NOT NULL,
    session_id      UUID NOT NULL,
    status          VARCHAR(16) NOT NULL DEFAULT 'idle',
                    -- idle | planning | executing | reflecting | completed | failed
    current_step    INTEGER DEFAULT 0,
    total_steps     INTEGER DEFAULT 0,
    state_data      JSONB NOT NULL DEFAULT '{}',
                    -- 执行状态：当前工具、中间结果、checkpoint数据
    token_usage     JSONB DEFAULT '{"total": 0, "by_step": []}',
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(agent_id, session_id)
);

CREATE INDEX idx_agent_state_session ON agent_state(session_id);
CREATE INDEX idx_agent_state_status ON agent_state(status);
```

### 会话历史表

```sql
CREATE TABLE conversation_messages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id      UUID NOT NULL,
    role            VARCHAR(16) NOT NULL, -- system | user | assistant | tool
    content         TEXT NOT NULL,
    embedding       VECTOR(1536),         -- OpenAI ada-002维度，按实际模型调整
    token_count     INTEGER,
    metadata        JSONB DEFAULT '{}',   -- 工具调用结果、推理步骤等附加信息
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_messages_session ON conversation_messages(session_id);
CREATE INDEX idx_messages_time ON conversation_messages(created_at);

-- pgvector HNSW索引（生产就绪）
CREATE INDEX idx_messages_embedding ON conversation_messages
    USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64);
```

### 工具注册表

```sql
CREATE TABLE tool_registry (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(64) NOT NULL UNIQUE,
    description     TEXT NOT NULL,
    embedding       VECTOR(1536),
    input_schema    JSONB NOT NULL,        -- JSON Schema定义
    output_schema   JSONB,
    domain_tags     TEXT[] DEFAULT '{}',
    tool_type       VARCHAR(16) NOT NULL,  -- api_call | computation | data_access
    availability    VARCHAR(16) DEFAULT 'enabled',
    required_perms  TEXT[] DEFAULT '{}',
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);

-- 工具语义发现索引
CREATE INDEX idx_tools_embedding ON tool_registry
    USING hnsw (embedding vector_cosine_ops);
CREATE INDEX idx_tools_type ON tool_registry(tool_type);
CREATE INDEX idx_tools_domain ON tool_registry USING gin(domain_tags);
```

### Memory Tier存储方案

```sql
-- 短期记忆：最新对话原文（Redis更适合，但PG也可用）
-- 已在conversation_messages表中

-- 中期记忆：压缩摘要
CREATE TABLE memory_summaries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id      UUID NOT NULL,
    summary_type    VARCHAR(16) NOT NULL, -- session | topic | decision
    content         TEXT NOT NULL,
    embedding       VECTOR(1536),
    source_messages UUID[],               -- 来源消息ID列表
    relevance_score FLOAT DEFAULT 0.5,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_summaries_embedding ON memory_summaries
    USING hnsw (embedding vector_cosine_ops);

-- 长期记忆：跨会话知识
CREATE TABLE long_term_memory (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agent_id        VARCHAR(64) NOT NULL,
    knowledge_type  VARCHAR(16) NOT NULL, -- fact | procedure | preference
    content         TEXT NOT NULL,
    embedding       VECTOR(1536),
    access_count    INTEGER DEFAULT 0,     -- 访问频率
    last_accessed   TIMESTAMPTZ,
    decay_factor    FLOAT DEFAULT 1.0,     -- 时效衰减
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_ltm_embedding ON long_term_memory
    USING hnsw (embedding vector_cosine_ops);
CREATE INDEX idx_ltm_agent ON long_term_memory(agent_id);
```