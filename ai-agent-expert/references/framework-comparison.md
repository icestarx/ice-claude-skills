# Agent 框架对比与选型详解

## 目录
- [六大框架深度对比](#六大框架深度对比)
- [选型决策流程图](#选型决策流程图)
- [集成场景](#集成场景)

---

## 六大框架深度对比

### LangGraph

**定位**：图式状态机框架，Agent开发最成熟的通用选择。

**核心架构**：
- StateGraph：节点=函数，边=条件转换
- 状态是typed dict，节点间传递和更新
- 支持循环图（不同于DAG的严格线性）

**优势**：
- 最大的控制力和灵活性——几乎可以表达任何Agent模式
- 内置checkpointing（PostgreSQL/Redis/SQLite）
- Human-in-the-loop中断模式成熟
- 流式输出支持好
- 生产就绪度最强

**劣势**：
- 学习曲线陡峭（需要理解图概念）
- LangChain生态依赖（部分人反感LangChain的复杂性）
- 图定义代码较冗长

**适用场景**：复杂有状态工作流、需要精细控制执行路径、生产级部署

**典型代码模式**：
```python
from langgraph.graph import StateGraph, END

graph = StateGraph(AgentState)
graph.add_node("planner", plan_step)
graph.add_node("executor", execute_step)
graph.add_node("reflector", reflect_step)
graph.add_edge("planner", "executor")
graph.add_conditional_edges("executor", should_reflect, {
    True: "reflector", False: END
})
graph.add_edge("reflector", "planner")  # 循环：反思后重新规划
app = graph.compile(checkpointer=PostgresSaver(pg_conn))
```

---

### CrewAI

**定位**：角色驱动多Agent框架，最易上手的多Agent选择。

**核心架构**：
- Agent有Role、Goal、Backstory
- Task有描述、预期输出、负责Agent
- Crew编排Agent执行Tasks

**优势**：
- 极低入门门槛——几分钟搭建一个多Agent团队
- 角色隐喻直观易懂
- 支持开源和商业LLM
- CrewAI+企业版2025推出

**劣势**：
- 非角色驱动的工作流不自然
- 控制力不如LangGraph（执行流程不够透明）
- 状态管理较基础
- 生产部署案例较少

**适用场景**：快速原型多Agent团队、结构化协作任务、角色清晰的工作流

**典型代码模式**：
```python
from crewai import Agent, Task, Crew

researcher = Agent(role="研究员", goal="深度研究主题", backstory="资深研究员...")
writer = Agent(role="写作者", goal="撰写清晰报告", backstory="专业写手...")

research_task = Task(description="研究量子计算最新进展", agent=researcher)
write_task = Task(description="基于研究结果撰写报告", agent=writer)

crew = Crew(agents=[researcher, writer], tasks=[research_task, write_task])
result = crew.kickoff()
```

---

### AutoGen

**定位**：研究型多Agent对话框架，最灵活的模式实验工具。

**核心架构**：
- Agent间通过对话交互
- 支持复杂的对话模式（嵌套、群聊、递归）
- v0.4重构改善了稳定性

**优势**：
- 最灵活的Agent对话模式
- 强研究背景（Microsoft Research）
- 适合实验新颖架构

**劣势**：
- 生产就绪度有限（v0.2→v0.4迁移痛苦）
- 文档有缺口
- API变更频繁
- 社区规模较小

**适用场景**：研究实验、新颖架构原型、对话驱动的多Agent系统

---

### Claude Agent SDK

**定位**：Anthropic官方Agent SDK，Claude生态原生选择。

**核心架构**：
- 深度集成Claude能力（tool use、extended thinking、computer use）
- 内置安全guardrails
- 清洁API设计

**优势**：
- Claude能力最深度利用
- 安全特性最强（Constitutional AI、RLHF）
- API设计简洁优雅
- Anthropic官方支持

**劣势**：
- 仅支持Claude模型（不模型无关）
- 生态和社区规模较小（2025初刚推出）
- 第三方工具集成有限

**适用场景**：Claude驱动的Agent系统、安全可靠性优先、Anthropic生态

---

### OpenAI Agents SDK

**定位**：OpenAI官方Agent SDK，Swarm的正式化演进。

**核心架构**：
- Agent Loop：Runner调用LLM→执行工具/handoff→反馈→迭代
- Handoff：Agent间通过formalized `handoff()`转移控制权+完整上下文
- Guardrails：`@input_guardrail` / `@output_guardrail`装饰器

**优势**：
- 生产就绪（Swarm的工程化版本）
- Handoff模式清晰
- Guardrails内置且可组合
- Tracing仪表盘内置

**劣势**：
- 仅支持OpenAI模型
- 社区生态刚起步（2025年3月发布）
- 复杂状态管理需要自己实现

**适用场景**：OpenAI驱动的多Agent系统、需要guardrails保护、Handoff协作模式

**典型代码模式**：
```python
from openai_agents import Agent, Runner, handoff, Guardrail

triage = Agent(name="triage", instructions="分类用户请求...")
specialist = Agent(name="specialist", instructions="处理专业问题...")

triage.handoffs = [handoff(specialist)]

@input_guardrail
async def check_injection(ctx, agent, input):
    result = await classifier.classify(input)
    return GuardrailResult(tripwire_triggered=result.risk > 0.7)

result = await Runner.run(triage, input="帮我分析市场数据", guardrails=[check_injection])
```

---

### Semantic Kernel

**定位**：Microsoft企业级AI编排框架，.NET/Azure生态首选。

**核心架构**：
- Planner架构（自动规划执行步骤）
- Plugin系统（技能注册和发现）
- 多语言支持（.NET、Python、Java）
- 深Azure集成

**优势**：
- 企业级成熟度最强
- .NET生态深度集成
- Azure AI Services原生支持
- 多语言支持最完整
- Microsoft战略级产品

**劣势**：
- .NET优先文化（Python开发者体验差）
- 比较笨重
- 社区创新速度较慢
- 学习曲线不友好

**适用场景**：企业/.NET环境、Azure基础设施、需要多语言支持

---

## 对比总结矩阵

| 维度 | LangGraph | CrewAI | AutoGen | Claude SDK | OpenAI SDK | Semantic Kernel |
|------|-----------|--------|---------|------------|------------|----------------|
| 易上手 | ★★★ | ★★★★★ | ★★★ | ★★★★ | ★★★★ | ★★ |
| 控制力 | ★★★★★ | ★★★ | ★★★★ | ★★★ | ★★★ | ★★★★ |
| 多Agent | ★★★★ | ★★★★★ | ★★★★★ | ★★★ | ★★★★ | ★★★★ |
| 多模型 | ★★★★★ | ★★★★ | ★★★★ | ★★(仅Claude) | ★★(仅OpenAI) | ★★★★★ |
| 生产就绪 | ★★★★★ | ★★★ | ★★ | ★★★ | ★★★ | ★★★★★ |
| 状态管理 | ★★★★★ | ★★ | ★★ | ★★★ | ★★★ | ★★★★ |
| 生态规模 | 最大 | 中 | 小 | 小 | 小 | 大(企业) |

---

## 选型决策流程图

```
项目需求分析
│
├─ 第一步：确定Agent模式（参考 agent-architecture.md）
│   ├─ 单Agent → 考虑LangGraph/Claude SDK/OpenAI SDK
│   ├─ 多Agent团队 → 考虑CrewAI/OpenAI SDK/LangGraph
│   ├─ 研究实验 → 考虑AutoGen
│
├─ 第二步：确定模型生态约束
│   ├─ 仅用Claude → Claude Agent SDK（最深度利用）
│   ├─ 仅用OpenAI → OpenAI Agents SDK（原生Handoff+Guardrails）
│   ├─ 多模型混合 → LangGraph / Semantic Kernel
│   ├─ .NET/Azure → Semantic Kernel（几乎唯一选择）
│
├─ 第三步：确定生产要求
│   ├─ 快速原型 → CrewAI（最易上手）
│   ├─ 复杂状态流 → LangGraph（最强状态管理）
│   ├─ 企业级部署 → LangGraph 或 Semantic Kernel
│   ├─ 安全优先 → Claude SDK 或 OpenAI SDK（内置Guardrails）
│
├─ 第四步：最终确认
│   ├─ 复杂有状态工作流 → LangGraph（默认选择）
│   ├─ 简单多Agent团队 → CrewAI（快速原型）
│   ├─ Claude生态 → Claude SDK
│   ├─ OpenAI生态 → OpenAI Agents SDK
│   ├─ .NET企业 → Semantic Kernel
│   └─ 研究探索 → AutoGen
```

---

## 集成场景

### 何时组合多个框架

**一般原则**：尽量选一个框架，不要组合。组合引入复杂度。

但以下场景可以考虑组合：

| 场景 | 组合方式 | 原因 |
|------|----------|------|
| 企业环境多模型 | Semantic Kernel + LangGraph | SK做企业编排，LangGraph做复杂Agent逻辑 |
| 安全+灵活 | Claude SDK + LangGraph | Claude SDK做安全Agent，LangGraph做编排层 |
| 原型→生产 | CrewAI → LangGraph | CrewAI快速验证，LangGraph生产部署 |

**组合风险**：
- 状态管理不兼容（每个框架有自己的状态模型）
- Token消耗翻倍（框架间通信）
- 调试复杂度倍增（跨框架trace）
- 维护负担（多框架版本升级）