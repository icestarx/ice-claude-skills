# Agent 后端架构详解

## 目录
- [API设计模式](#api设计模式)
- [任务队列架构](#任务队列架构)
- [状态管理](#状态管理)
- [编排层](#编排层)
- [Human-in-the-Loop](#human-in-the-loop)
- [典型目录结构与代码骨架](#典型目录结构与代码骨架)

---

## API设计模式

### REST + WebSocket/SSE 混合架构

Agent系统需要两种API风格：
- **REST**：配置、部署、任务创建等确定性操作
- **WebSocket/SSE**：实时Agent事件流（逐步推理输出、工具调用进度、中间状态）

```
API设计原则：
- POST /agents/tasks          → 创建任务，返回job_id
- GET  /agents/tasks/{id}     → 查询任务状态和结果
- SSE  /agents/tasks/{id}/stream → 实时事件流
- GET  /agents/{id}/state     → 查询Agent当前状态
- POST /agents/{id}/interrupt → 人工中断干预
```

### Async Job API 模式

长时运行Agent任务（分钟到小时级）的标准模式：

1. `POST /tasks` → 返回 `job_id`，任务进入队列
2. `GET /tasks/{job_id}/status` → 查询进度（pending/running/completed/failed）
3. `GET /tasks/{job_id}/result` → 获取最终输出
4. SSE/WebSocket → 实时推送中间步骤

**关键设计点**：
- 任务创建必须是异步的，不能阻塞HTTP连接
- 状态查询要有幂等性保证
- 结果获取要有超时和重试机制

---

## 任务队列架构

### 优先级队列

Agent任务的优先级分层：
- **Critical**：推理步骤、Human审批响应（低延迟优先）
- **Normal**：工具调用、数据处理（吞吐量优先）
- **Background**：记忆压缩、评估批处理（资源空闲时执行）

### 技术选型

| 方案 | 适用场景 | 优势 | 劣势 |
|------|----------|------|------|
| **Redis + BullMQ** | Node.js生态、中等规模 | 简单、成熟、优先级支持 | 无持久化保证 |
| **Redis + Celery** | Python生态、中等规模 | Python原生、丰富功能 | 较重的依赖 |
| **Temporal** | 需要持久化保证、企业级 | 持久执行、崩溃恢复、确定性 | 学习曲线陡 |

### Temporal 持久执行模式

每个Agent步骤作为Temporal Activity：
- LLM调用、工具执行、人工审批都是独立Activity
- Workflow在进程崩溃后自动恢复
- 内置重试、超时、指数退避配置
- Signal/Query模式允许外部中断和实时状态查看

```
关键概念：
- Workflow：Agent的完整执行流程定义
- Activity：单个步骤（LLM调用、工具执行）
- Signal：外部事件注入（人工审批、取消指令）
- Query：实时状态查看（不改变状态）
```

---

## 状态管理

Agent系统的**最难问题**。三种状态需要管理：

| 状态类型 | 描述 | 存储策略 |
|----------|------|----------|
| **对话状态** | 消息历史、当前对话上下文 | Redis（快速读写）+ PostgreSQL（持久归档） |
| **执行状态** | 当前步骤、进度、checkpoint | PostgreSQL JSONB（结构化）或Temporal（内置） |
| **协调状态** | 多Agent间的共享信息 | Redis Pub/Sub + PostgreSQL（最终一致性） |

### Event Sourcing

将每个Agent动作存储为不可变事件：
- 优势：可回放、可调试、审计追踪、崩溃恢复
- 适用：需要完整历史记录的场景
- 注意：事件量可能很大，需要定期压缩

### CQRS

分离写入和读取：
- 写端：Agent执行进度更新（高频、低延迟 → Redis）
- 读端：状态查询、历史查看（低频、结构化 → PostgreSQL）

### Checkpointing

LangGraph风格的checkpoint：
- 每个Graph节点执行后保存完整状态
- 支持从任意checkpoint恢复执行
- 存储选择：PostgreSQL checkpointer、Redis checkpointer、SQLite（本地）

---

## 编排层

三种主流编排范式：

### DAG-based（LangGraph风格）
- 节点=函数，边=条件转换
- 状态是typed dict，节点间传递
- 优势：可视化、可调试、精细控制
- 劣势：需要预定义完整图结构

### Workflow引擎（Temporal风格）
- 持久执行、保证完成
- 优势：最可靠、崩溃恢复、确定性
- 劣势：学习曲线陡、部署复杂度高

### Event驱动
- 消息队列（RabbitMQ/Kafka）pub/sub
- Agent发布事件，其他Agent订阅响应
- 优势：最灵活、解耦最彻底
- 劣势：调试困难、缺乏全局视角

---

## Human-in-the-Loop

### 中断模式（LangGraph interrupt）
- Agent执行暂停，等待人工审批
- 审批后继续执行
- 状态保存，可跨session恢复

### 审批门（Approval Gates）
在高风险操作前插入审批步骤：
- 金融交易、数据删除、外部API调用
- 审批可以是即时或异步（等待数小时）

### 异步审批（Temporal Signal）
- 人工审批可以延迟到达
- Workflow在等待期间不占用计算资源
- 审批到达后Workflow自动恢复

---

## 典型目录结构与代码骨架

### Agent服务项目结构

```
agent-service/
├── api/
│   ├── main.py                # FastAPI应用入口
│   ├── routes/
│   │   ├── tasks.py           # 任务创建/查询路由
│   │   ├── stream.py          # SSE/WebSocket路由
│   │   ├── agents.py          # Agent配置路由
│   └── middleware/
│       ├── auth.py            # 认证中间件
│       ├── rate_limit.py      # 限流中间件
│       ├── token_budget.py    # Token预算中间件
├── core/
│   ├── orchestrator.py        # Agent编排器
│   ├── state_manager.py       # 状态管理器
│   ├── checkpoint.py          # Checkpoint管理
│   ├── queue.py               # 任务队列接口
├── agents/
│   ├── base.py                # Agent基类
│   ├── planner.py             # Planner实现
│   ├── executor.py            # Executor实现
├── storage/
│   ├── redis_client.py        # Redis连接
│   ├── pg_client.py           # PostgreSQL连接
│   ├── vector_client.py       # 向量数据库连接
├── config/
│   ├── settings.py            # 配置管理
│   ├── models.yaml            # 模型配置
├── tests/
├── Dockerfile
└── docker-compose.yaml
```

### FastAPI Agent Server 骨架

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
import asyncio

app = FastAPI()

# 任务创建（异步）
@app.post("/agents/tasks")
async def create_task(request: TaskRequest):
    job_id = await queue.enqueue(request)
    return {"job_id": job_id, "status": "pending"}

# 任务状态查询
@app.get("/agents/tasks/{job_id}/status")
async def get_status(job_id: str):
    state = await state_manager.get_state(job_id)
    return {"status": state.status, "progress": state.progress}

# SSE实时事件流
@app.get("/agents/tasks/{job_id}/stream")
async def stream_events(job_id: str):
    async def event_generator():
        for event in await event_bus.subscribe(job_id):
            yield f"data: {event.json()}\n\n"
    return StreamingResponse(event_generator(), media_type="text/event-stream")

# 人工中断干预
@app.post("/agents/tasks/{job_id}/interrupt")
async def interrupt_task(job_id: str, decision: HumanDecision):
    await state_manager.inject_signal(job_id, decision)
    return {"status": "interrupted", "action": decision.action}
```

### Temporal Workflow 骨架

```python
from temporalio import workflow, activity

@workflow.defn
class AgentWorkflow:
    @workflow.run
    async def run(self, task: AgentTask) -> AgentResult:
        # Planning phase
        plan = await workflow.execute_activity(
            plan_step, task, start_to_close_timeout=timedelta(minutes=5)
        )

        # Execution with human-in-the-loop
        for step in plan.steps:
            if step.requires_approval:
                # 等待人工审批Signal
                approval = await workflow.wait_for_signal(
                    "human_approval", timeout=timedelta(hours=24)
                )
                if not approval.approved:
                    return AgentResult(status="rejected")

            result = await workflow.execute_activity(
                execute_step, step, start_to_close_timeout=timedelta(minutes=10)
            )

        return AgentResult(status="completed", data=results)
```