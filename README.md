# ice-claude-skills

Claude Code skills & plugins collection.

## Plugins

| Plugin | Description | Install |
|--------|-------------|---------|
| [ai-agent-expert](./ai-agent-expert/) | AI Agent development expert - architecture, performance, backend & database design for building production-grade AI agents | `claude plugin install git@github.com:icestarx/ice-claude-skills.git --subdir ai-agent-expert` |

---

## ai-agent-expert

AI Agent 全栈架构专家插件，覆盖智能体开发全生命周期：

- **Agent 架构模式** — ReAct, Plan-and-Execute, Multi-Agent, Reflection, Tool-Use 等 6 种模式 + 选型决策树 + 代码模板
- **后端架构** — REST+SSE API、任务队列(Celery/Temporal)、状态管理(Redis+PG)、编排层
- **数据库架构** — pgvector/Pinecone/Weaviate/Chroma 选型、三层记忆架构、SQL Schema 模板
- **性能优化** — Token 优化、缓存策略、流式输出、成本控制、模型路由
- **生产部署** — 6 层安全防御、可观测性、评估体系、断路器弹性

### 安装 / Installation

```bash
# Plugin install (recommended)
claude plugin install git@github.com:icestarx/ice-claude-skills.git --subdir ai-agent-expert

# Or manual install
cp -r ai-agent-expert/skills/ai-agent-expert ~/.claude/skills/ai-agent-expert
```

### 使用 / Usage

安装后自动触发，也可手动调用 `/ai-agent-expert`。

触发场景：agent 架构设计、智能体开发、多 agent 协作、框架选型、性能优化、生产部署等。

### 文件结构 / Structure

```
ai-agent-expert/
├── .claude-plugin/plugin.json    # Plugin manifest
├── README.md
├── LICENSE
└── skills/ai-agent-expert/
    ├── SKILL.md                  # Core skill (workflow + principles + quick ref)
    └── references/
        ├── agent-architecture.md  # 6 patterns + decision tree + code templates
        ├── backend-architecture.md # API + queues + state + orchestration
        ├── database-architecture.md # Vector DBs + memory + SQL templates
        ├── performance-optimization.md # Token + caching + cost control
        ├── production-concerns.md # Security + observability + evals
        └── framework-comparison.md # 6 frameworks + selection guide
```

---

## License

MIT