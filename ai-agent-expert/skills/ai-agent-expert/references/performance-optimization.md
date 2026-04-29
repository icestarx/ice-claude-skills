# Agent 性能优化详解

## 目录
- [Token优化](#token优化)
- [缓存策略](#缓存策略)
- [流式输出](#流式输出)
- [成本控制](#成本控制)
- [Rate Limiting](#rate-limiting)
- [优化检查清单](#优化检查清单)
- [配置模板](#配置模板)

---

## Token优化

### 滑动窗口+摘要压缩

**原理**：维护最近N条消息原文，更早的消息压缩为摘要。

```
策略：
- 最近10条消息：保留原文（最高保真度）
- 10-30条消息：压缩为3-5条摘要（中等保真度）
- 30+条消息：仅保留关键决策和结论（低保真度）
```

**实现要点**：
- 压缩触发条件：token数量超过预算80%
- 摘要质量检查：压缩后验证关键信息保留率
- 渐进压缩：不要一次压缩太多，逐层渐进

### Token预算管理

**per-Agent per-Task预算**：
- 推理Agent：高预算（2000-4000 tokens/调用）
- 工具Agent：低预算（500-1000 tokens/调用）
- 总结Agent：中预算（1000-2000 tokens/调用）

**全局Token池**：
- 设置总token预算上限
- Controller Agent分配per-task预算
- 优先级分配：关键推理>常规执行>辅助任务

### 升级链（Escalation Chain）

当Agent达到预算上限时：
- 摘要当前状态
- 移交给更高级别的Supervisor Agent（更高预算上限）
- Supervisor基于摘要做最终决策

### 预估中间件

在发送LLM请求前预估token数：
- 超预算→触发压缩或拒绝
- 接近预算→警告并建议优化
- 远低于预算→正常执行

### 模型路由

根据任务复杂度路由到不同模型：

| 任务类型 | 推荐模型 | 相对成本 |
|----------|----------|----------|
| 简单格式化/提取 | Haiku / GPT-4o-mini | ~1/10 |
| 常规推理/工具调用 | Sonnet / GPT-4o | ~1/3 |
| 复杂规划/反思 | Opus / o1 | ~1 |

---

## 缓存策略

### 四层缓存体系

| 层级 | 策略 | 成本节省 | 适用场景 |
|------|------|----------|----------|
| **Prompt前缀缓存** | Anthropic/Gemini原生缓存共享系统prompt | ~90%读取成本 | 多轮对话、重复工作流 |
| **语义缓存** | 基于嵌入相似度缓存相似查询的回答 | 30-60% | FAQ类、重复任务模式 |
| **精确匹配缓存** | 基于prompt hash缓存完全相同请求 | 50-80% | 模板化调用、批处理 |
| **计算缓存** | 预计算常用工具结果和推理链 | 20-50% | 高频工具调用结果 |

### Anthropic Prompt Caching 配置要点

- Cache TTL：5分钟
- 标记cache_control: {"type": "ephemeral"}在需要缓存的prompt部分
- 最适合：长系统prompt（工具定义）、多轮对话的固定上下文
- 最低缓存token数：1024 tokens（低于此不缓存）
- 成本：写入+25%，读取-90%

---

## 流式输出

### 提前终止（Early Termination）

监控流式输出，当满足条件时提前终止：
- 已获得足够信息（答案明确）
- 输出质量达到阈值
- 用户中断信号

### 渐进质量评估

对流式输出逐chunk评估：
- 每个chunk检查是否有实质性新信息
- 连续N个chunk无新信息→终止
- 质量下降趋势→提醒或降级模型

### 并发控制

- 控制并行流式请求数量
- 不同Agent类型不同并发上限
- 突发请求排队而非直接拒绝

---

## 成本控制

### Agent池化

不再per-request创建Agent，而是：
- 预创建Agent池（共享记忆存储）
- 按需分配池中Agent
- 任务完成后归还池（而非销毁）

**节省**：减少Agent初始化成本、记忆加载成本、模型冷启动成本。

### 语义缓存（Trace级别）

在trace级别缓存整个Agent执行轨迹：
- 相似任务的完整轨迹可复用
- 节省40-60%的重复任务成本
- 需要质量验证：缓存轨迹是否仍适用当前任务

### 成本监控仪表盘

关键指标：
- per-Agent per-Task token消耗
- 缓存命中率
- 模型路由分布（各模型调用占比）
- 平均每任务成本
- 成本/质量比率

---

## Rate Limiting

### per-Agent差异化限流

不同Agent类型不同的限流策略：

| Agent类型 | 限流策略 | 原因 |
|-----------|----------|------|
| 推理Agent | 高并发（10 req/s） | 快速推理、用户体验优先 |
| 工具Agent | 中并发（5 req/s） | 外部API有自身限流 |
| 反思Agent | 低并发（2 req/s） | 长耗时、可后台执行 |
| 评估Agent | 极低（1 req/min） | 批处理、非实时 |

---

## 优化检查清单

Token优化10项检查：

1. 系统prompt是否使用prompt caching？
2. 是否有滑动窗口+摘要压缩机制？
3. 是否设置了per-Agent token预算？
4. 是否有模型路由（简单→小模型）？
5. 是否有语义缓存层？
6. 工具定义是否精简（避免过长JSON Schema）？
7. 对话历史是否按token预算截断？
8. 是否有token预估中间件？
9. 是否有升级链（budget→supervisor）？
10. 是否监控per-task token消耗？

---

## 配置模板

### Redis 语义缓存配置

```python
import redis
import hashlib
from sentence_transformers import SentenceTransformer

class SemanticCache:
    def __init__(self, redis_client, embedder, similarity_threshold=0.92):
        self.redis = redis_client
        self.embedder = embedder
        self.threshold = similarity_threshold

    def get(self, prompt: str):
        """检索语义相似的缓存结果"""
        embedding = self.embedder.encode(prompt)
        # 在Redis中搜索最近邻向量
        results = self.redis.ft("cache_index").search(
            f"@embedding:[VECTOR_RANGE $radius $query_vec]",
            query_params={"radius": 1 - self.threshold, "query_vec": embedding}
        )
        if results.total > 0:
            return results.docs[0].response  # 返回缓存回答
        return None

    def set(self, prompt: str, response: str):
        """缓存新的prompt-response对"""
        embedding = self.embedder.encode(prompt)
        key = hashlib.md5(prompt.encode()).hexdigest()
        self.redis.hset(f"cache:{key}", mapping={
            "prompt": prompt,
            "response": response,
            "embedding": embedding.tolist(),
            "timestamp": time.time()
        })
```

### Anthropic Prompt Caching 配置

```python
import anthropic

client = anthropic.Anthropic()

# 系统prompt + 工具定义标记为缓存
system_prompt = """你是Agent系统...（长系统prompt）"""

response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=4096,
    system=[
        {
            "type": "text",
            "text": system_prompt,
            "cache_control": {"type": "ephemeral"}  # 标记缓存
        }
    ],
    tools=[
        {
            "name": "search",
            "description": "搜索工具",
            "input_schema": {...},
            "cache_control": {"type": "ephemeral"}  # 工具定义也缓存
        }
    ],
    messages=[...]
)
```

### Model Router 决策逻辑

```python
class ModelRouter:
    def route(self, task: AgentTask) -> str:
        """根据任务特征路由到最优模型"""

        # 1. 估算任务复杂度
        complexity = self.estimate_complexity(task)

        # 2. 检查缓存
        if self.semantic_cache.has_match(task.prompt):
            return "cache"  # 直接返回缓存结果，无需LLM调用

        # 3. 基于复杂度路由
        if complexity <= 0.3:
            # 简单任务：提取、格式化、简单QA
            return "claude-haiku-4-5-20251001"
        elif complexity <= 0.7:
            # 中等任务：推理、工具调用、多步骤
            return "claude-sonnet-4-20250514"
        else:
            # 复杂任务：规划、反思、多Agent协调
            return "claude-opus-4-6"

    def estimate_complexity(self, task):
        """估算任务复杂度（0-1分数）"""
        score = 0.0
        score += len(task.tools) * 0.1          # 工具数量
        score += task.expected_steps * 0.05      # 预期步骤数
        score += task.requires_reasoning * 0.3   # 需要推理
        score += task.requires_planning * 0.2    # 需要规划
        score += task.is_multi_agent * 0.2       # 多Agent
        return min(score, 1.0)
```