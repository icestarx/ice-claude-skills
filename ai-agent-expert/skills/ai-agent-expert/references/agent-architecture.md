# AI Agent 架构模式详解

## 目录
- [六种核心模式](#六种核心模式)
- [模式选型决策树](#模式选型决策树)
- [模式组合策略](#模式组合策略)
- [代码模式模板](#代码模式模板)

---

## 六种核心模式

### 1. ReAct（推理+行动）

**核心机制**：Thought → Action → Observation → Thought → ... 的循环迭代。Agent先推理当前状态，选择一个工具/行动执行，观察结果，再基于新信息继续推理。

**适用场景**：
- 需要动态决策的任务（信息检索、多步骤推理）
- 工具调用密集型任务
- 不确定性高的环境（每次行动可能改变后续策略）

**不适用场景**：
- 任务步骤完全可预知（浪费推理token）
- 单次调用即可完成的简单任务
- 需要全局最优规划的任务（ReAct是贪心策略）

**Token成本预估**：中等偏高。每个循环至少1次推理+1次工具调用+1次观察解析。5步任务约10-15次LLM调用。

**典型实现**：LangGraph的`create_react_agent`，LlamaIndex的ReAct agent。

---

### 2. Chain-of-Thought / Tree-of-Thought / Graph-of-Thought

**核心机制**：
- **CoT**：逐步推理，线性链条
- **ToT**：分支探索，可回溯，树结构
- **GoT**：推理路径可合并/分裂，图结构

**适用场景**：
- 纯推理任务（数学、逻辑、策略分析）
- 不需要外部工具的复杂问题
- 需要探索多条推理路径的问题（ToT/GoT）

**不适用场景**：
- 需要与环境交互的任务（没有Action环节）
- 实时性要求高的场景（ToT/GoT探索成本高）
- 简单直接的任务（过度推理浪费token）

**Token成本预估**：CoT低（单次长推理），ToT中等（多分支并行），GoT高（复杂图操作）。

**典型实现**：HuggingFace Smolagents的CodeAgent（代码作为推理轨迹），直接Prompt工程。

---

### 3. Plan-and-Execute

**核心机制**：先由Planner生成完整步骤列表，再由Executor按序执行。两个独立的LLM调用角色。

**适用场景**：
- 结构化的多步骤任务（步骤可预知）
- 需要全局视角的规划型任务
- 成本敏感场景（总LLM调用次数少于ReAct）

**不适用场景**：
- 高度不确定的环境（计划容易失效）
- 需要实时反馈调整的任务
- 步骤间有强依赖需要动态决策

**Token成本预估**：较低。1次规划调用 + N次执行调用。比ReAct少约30-50%的总调用。

**典型实现**：LangGraph的Plan-and-Execute template，Semantic Kernel的Planner。

---

### 4. Reflection（自我纠正）

**核心机制**：
- **Reflexion**：失败后生成语言反思，存入记忆，下次尝试利用反思
- **Self-Refine**：生成→批评→改进的迭代循环
- **LATS**：结合Tree-of-Thought + Reflexion + MCTS搜索

**适用场景**：
- 需要迭代改进的任务（代码生成、文本写作、方案设计）
- 单次尝试成功率低但可通过反馈改进的场景
- 有明确质量标准的任务（反思有判断依据）

**不适用场景**：
- 一次就能做好的简单任务
- 没有明确改进标准（反思会变成空泛的自我评论）
- 实时性要求极高的场景

**Token成本预估**：高。每次迭代至少2次LLM调用（执行+反思）。3轮迭代约6-8次调用。

**典型实现**：LangChain Reflexion chain，AutoGen self-refine pattern。

---

### 5. Multi-Agent（多智能体）

四种主流拓扑结构：

| 拓扑 | 描述 | 适用场景 | 问题 |
|------|------|----------|------|
| **Supervisor** | 中心Agent分配子任务给专业Worker | 结构化分工、明确角色划分 | 中心节点瓶颈 |
| **Hierarchical** | 多层监督（战略→任务→执行） | 企业级复杂流程、多层级决策 | 层级间信息损失 |
| **Swarm** | Agent之间通过函数调用移交控制权 | 灵活切换、无固定流程 | 缺乏全局视角、协调困难 |
| **Map-Reduce** | 并行分发→收集汇总 | 批量处理、信息聚合 | 需要可分解的任务 |

**适用场景**：复杂工作流、多个专业子任务、需要不同模型/工具组合

**不适用场景**：简单单步骤任务、token预算紧张（多Agent通信成本高）、延迟敏感

**Token成本预估**：高。每个Agent独立LLM调用 + Agent间通信（handoff/context transfer）。3个Agent协作约8-12次总调用。

**典型实现**：CrewAI（角色驱动），OpenAI Agents SDK（handoff），AutoGen（对话驱动）。

---

### 6. Tool-Use / Function-Calling

**核心机制**：通过JSON Schema定义工具接口，Agent按结构化格式调用工具并接收返回值。

**适用场景**：
- 确定性操作执行（API调用、数据查询、文件操作）
- 需要与外部系统交互的Agent
- 任何需要"行动力"的Agent系统（这是几乎所有Agent的基础组件）

**不适用场景**：
- 纯推理任务（不需要工具）
- 工具不可靠或返回格式不稳定

**Token成本预估**：取决于工具数量和调用频率。工具定义本身占用系统prompt token。

**典型实现**：OpenAI function calling格式已成行业标准，所有主流框架支持。

---

## 模式选型决策树

```
任务特征分析
│
├─ 任务是否需要外部交互（API/数据库/文件）？
│   ├─ YES → 需要Tool-Use组件（基础能力）
│   │   ├─ 步骤是否可预知？
│   │   │   ├─ YES → Plan-and-Execute
│   │   │   └─ NO → ReAct（动态决策）
│   │   └─ 是否需要迭代改进？
│   │       ├─ YES → ReAct + Reflection
│   │       └─ NO → 纯ReAct
│   │
│   └─ NO → 纯推理任务
│       ├─ 单条推理路径足够？
│       │   ├─ YES → CoT
│       │   └─ NO → 是否需要回溯探索？
│       │       ├─ YES → ToT 或 LATS
│       │       └─ NO → CoT + Self-Refine
│       │
│       └─ 任务是否可分解为专业子任务？
│           ├─ YES → Multi-Agent
│           │   ├─ 有明确流程 → Supervisor
│           │   ├─ 多层级 → Hierarchical
│           │   ├─ 灵活切换 → Swarm
│           │   └─ 批量处理 → Map-Reduce
│           └─ NO → 单Agent + 上述选型
```

---

## 模式组合策略

常见有效组合：

| 组合 | 何时使用 | 注意事项 |
|------|----------|----------|
| Plan + Reflect | 结构化任务但质量需要迭代改进 | Planner用大模型，Reflector用小模型 |
| ReAct + Multi-Agent | 复杂环境下的多专业协作 | 每个Worker是独立ReAct agent |
| CoT + Tool-Use | 推理密集型任务偶尔需要工具 | 工具调用打断推理流，需恢复机制 |
| Multi-Agent + Checkpoint | 长流程多步骤需要容错 | 每个Agent步骤保存checkpoint |

---

## 代码模式模板

### ReAct Loop 伪代码

```python
def react_loop(agent, task, max_iterations=10):
    context = {"task": task, "observations": []}

    for i in range(max_iterations):
        # Thought: Agent推理当前状态
        thought = agent.think(context)

        # Action: 选择并执行工具
        action = agent.choose_action(thought)
        observation = execute_tool(action)

        # 更新上下文
        context["observations"].append({
            "thought": thought,
            "action": action,
            "observation": observation
        })

        # 检查是否完成
        if agent.is_task_complete(context):
            return agent.generate_final_answer(context)

    return agent.generate_final_answer(context)  # 达到迭代上限
```

### Plan-and-Execute 伪代码

```python
def plan_and_execute(planner, executor, task):
    # Phase 1: Planning
    plan = planner.generate_plan(task)

    # Phase 2: Execution with re-planning capability
    results = []
    for step in plan.steps:
        result = executor.execute_step(step, previous_results=results)
        results.append(result)

        # Optional: Re-plan if observation deviates
        if result.needs_replanning:
            plan = planner.replan(task, plan, results)
            continue_from_updated_plan()

    return synthesize_results(results)
```

### Multi-Agent Handoff 模式（OpenAI SDK风格）

```python
# 定义专业Agent
researcher = Agent(
    name="researcher",
    instructions="You research topics thoroughly...",
    tools=[web_search, document_lookup]
)

writer = Agent(
    name="writer",
    instructions="You write clear, engaging content...",
    tools=[grammar_check, style_analyzer]
)

# 定义Handoff转移
researcher.handoffs = [handoff(writer, "Transfer to writer when research is complete")]
writer.handoffs = [handoff(researcher, "Transfer back if more research needed")]

# Runner执行
result = Runner.run(researcher, task="Write a report on quantum computing")
```

### 典型Agent项目目录结构

```
agent-project/
├── src/
│   ├── agents/
│   │   ├── planner.py          # Planner agent definition
│   │   ├── executor.py         # Executor agent definition
│   │   └── reflector.py        # Reflector agent definition
│   ├── tools/
│   │   ├── search.py           # Web search tool
│   │   ├── database.py         # Database query tool
│   │   └── calculator.py      # Computation tool
│   ├── memory/
│   │   ├── short_term.py       # Conversation buffer
│   │   ├── long_term.py        # Vector store interface
│   │   └── checkpoint.py       # State checkpointing
│   ├── orchestration/
│   │   ├── graph.py            # LangGraph state graph
│   │   └── config.py           # Agent configuration
│   └── api/
│       ├── routes.py           # REST API endpoints
│       └── websocket.py        # SSE/WebSocket streaming
├── evals/
│   ├── golden_dataset.json     # Evaluation test cases
│   ├── metrics.py              # Custom evaluation metrics
│   └── run_evals.py            # Evaluation runner
├── config/
│   ├── models.yaml             # Model configurations
│   └── tools.yaml              # Tool registry
└── tests/
    └── test_agents.py
```