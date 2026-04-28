# AI Agent Expert / AI Agent 专家

An AI Agent architecture and development expert skill for Claude Code, providing comprehensive guidance on agent pattern selection, backend design, database architecture, performance optimization, and production deployment.

一个面向 Claude Code 的 AI Agent 架构与开发专家 skill，提供全面的智能体模式选型、后端设计、数据库架构、性能优化和生产部署指导。

---

## 简介 / Overview

This skill acts as a full-stack AI Agent architect. When triggered, it follows a structured 6-step workflow (Understand → Read → Architect → Implement → Verify → Iterate) and draws from 6 deep reference files to provide expert-level guidance.

本 skill 作为全栈 AI Agent 架构专家运作。触发后遵循6步工作流（理解 → 阅读 → 架构 → 实现 → 验证 → 迭代），从6个深度参考文件中提取专业知识提供指导。

---

## 文件结构 / File Structure

```
ai-agent-expert/
├── SKILL.md                          # 核心入口文件 (Core entry file)
│   ├── Role Declaration              # 角色声明
│   ├── Workflow (6 steps)            # 工作流程
│   ├── Architecture Decision Router  # 架构决策路由表
│   ├── Core Principles (5 rules)     # 5条核心原则 + 执行动作
│   ├── Quick Reference               # 快速参考表
│   ├── Anti-Pattern Red Flags        # 反模式红旗清单
│   └── Knowledge Freshness           # 知识时效性声明
│
└── references/                       # 深度参考文件 (Deep reference files)
    ├── agent-architecture.md         # Agent架构模式详解
    │   6种模式 + 选型决策树 + 代码模式模板
    │
    ├── backend-architecture.md       # Agent后端架构详解
    │   API设计 + 任务队列 + 状态管理 + 编排层 + 代码骨架
    │
    ├── database-architecture.md      # Agent数据库架构详解
    │   向量数据库对比 + 记忆架构 + SQL/Schema模板
    │
    ├── performance-optimization.md   # Agent性能优化详解
    │   Token优化 + 缓存策略 + 流式输出 + 配置模板
    │
    ├── production-concerns.md        # Agent生产部署详解
    │   多层安全防御 + 可观测性 + 评估体系 + 防御代码示例
    │
    └── framework-comparison.md       # Agent框架对比与选型
    │   6大框架深度对比 + 选型决策流程图
```

---

## 核心原则 / Core Principles

1. **Agent-first design**: 先确定 Agent 模式再选框架，模式驱动架构。/ Determine agent pattern BEFORE choosing framework.
2. **State is the hardest problem**: 状态管理是架构瓶颈，必须早期设计。/ Agent state management is the architectural bottleneck.
3. **Token budget awareness**: 每个设计决策都有 token 成本影响。/ Every design decision has a token cost implication.
4. **Security by separation**: 隔离对话上下文和工具执行上下文。/ Isolate conversational context from tool-execution context.
5. **Eval before deploy**: 先建评估体系再上线。/ Build evaluation harness before deploying to production.

---

## 触发方式 / Triggering

本 skill 在以下场景自动触发 / Auto-triggers on:

- 用户讨论 Agent 架构设计、智能体开发 / Agent architecture design, building agents
- 提到 Agent 模式（ReAct, Plan-and-Execute, Reflection, Multi-Agent 等）
- 提到 Agent 框架（LangGraph, CrewAI, AutoGen, Claude Agent SDK, OpenAI Agents SDK, Semantic Kernel）
- 讨论 Agent 后端/数据库/性能/安全/评估问题
- 讨论 LLM 系统架构、工具编排、自主系统设计

也可通过 `/ai-agent-expert` 手动调用 / Also invocable via `/ai-agent-expert`.

**不触发场景** / Does NOT trigger on: 简单框架配置问题（如 "LangGraph怎么定义node"）

---

## 知识覆盖范围 / Knowledge Coverage

| 领域 / Domain | 内容 / Content |
|---|---|
| Agent 架构模式 | ReAct, CoT/ToT/GoT, Plan-and-Execute, Reflection, Multi-Agent (4拓扑), Tool-Use |
| Agent 框架 | LangGraph, CrewAI, AutoGen, Claude Agent SDK, OpenAI Agents SDK, Semantic Kernel |
| 后端架构 | REST+SSE API, Async Job, 任务队列, 状态管理, 编排层, Human-in-the-Loop |
| 数据库架构 | pgvector, Pinecone, Weaviate, Chroma + 三层记忆 + Schema模板 |
| 性能优化 | Token优化, 缓存(语义/前缀/精确), 流式输出, 模型路由, 成本控制 |
| 生产部署 | 6层安全防御, OpenTelemetry/LangFuse可观测, 评估体系, 断路器弹性 |

---

## 安装 / Installation

将整个 `ai-agent-expert/` 目录复制到 Claude Code 的 skills 目录：

```bash
cp -r ai-agent-expert/ ~/.claude/skills/ai-agent-expert/
```

Copy the entire `ai-agent-expert/` directory to Claude Code's skills directory.

---

## 语言 / Language

- **SKILL.md**: 核心指令英文（确保 Claude 精确执行），中文注释辅助理解
- **references/**: 全部中文（方便中文用户深入阅读）

SKILL.md core instructions in English for precise execution; references in Chinese for deep reading.

---

## 知识时效性 / Knowledge Freshness

本 skill 的知识基于 2024-2026 年 AI Agent 生态状态。Agent 框架和模式演进迅速，建议在使用时通过 Context7 或 WebSearch 验证最新文档。

Knowledge reflects the AI agent ecosystem as of 2024-2026. Agent frameworks evolve rapidly — verify current docs via Context7/WebSearch when needed.

---

## License

MIT