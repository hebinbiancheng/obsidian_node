---
title: AI Agent 四大框架深度对比
type: knowledge
status: evergreen
source_type: 微信公众号
created: 2026-05-27
updated: 2026-07-13
tags:
  - AI
  - Agent
  - 框架
  - LangChain
  - AutoGen
  - CrewAI
  - Claude Code
  - 技术选型
source: https://mp.weixin.qq.com/s/FTdytNtVBLQXkN1I0hEE8Q
confidence: high
---

# AI Agent 四大框架深度对比

> 原文：[微信文章](https://mp.weixin.qq.com/s/FTdytNtVBLQXkN1I0hEE8Q) · 2026-05-27
> 原始资料：`^[raw/articles/wechat-agent-framework-compare-2026.html]`

---

## 一句话总结

四款框架的设计哲学决定适用场景——LangChain 追求显式可控，AutoGen 追求灵活协作，CrewAI 追求快速原型，Claude Code SDK 追求模型原生能力。每个框架对 Agent 四大核心模块（感知/规划/行动/反思）的实现方式完全不同。

---

## 一、框架总览与设计哲学

| 框架 | 设计哲学 | 核心机制 | 最适合 |
|------|---------|---------|--------|
| **LangChain/LangGraph** | 显式状态图 | 有向图状态机（StateGraph） | 企业级复杂业务流程 |
| **AutoGen** | 对话编排 | 异步 Actor 模型 + 消息传递 | 开放式多 Agent 协作 |
| **CrewAI** | 团队协作 | 角色-任务-团队三层声明式 | 快速原型、低学习成本 |
| **Claude Code SDK** | 模型优先 | 原生工具调用环路 | 代码生成、合规场景 |

---

## 二、LangChain/LangGraph

### 设计理念

LangChain 以「链」（Chain）为核心抽象，LangGraph 将其升级为**有向状态图**：节点 = 计算步骤，边 = 状态流转条件，共享状态对象 = 全局上下文。核心价值：Agent 的隐式执行逻辑 → 可可视化编排、可精准控制的显式流程图。

### 核心节点

- **LLM 节点**：接收上下文 → 生成工具调用指令
- **工具节点**：解析指令 → 执行工具 → 结果写入全局状态
- **检查点机制**：每次状态转移自动持久化 → 任意节点暂停/热重启/回溯

### 四大模块实现

| 模块 | 实现方式 |
|------|---------|
| **感知** | 全局状态对象 + 工具节点：所有外部数据标准化注入 |
| **规划** | LLM 节点 + 条件边：模型推理 → 条件边决定分支/循环/跳转 |
| **行动** | 工具节点 + 子图编排：支持串行/并行/多层级嵌套调用 |
| **反思** | 检查点 + 条件边 + 人工干预节点：预执行合规校验 + 后置结果校验 |

**LangSmith 完整记录**：所有思考步骤、工具调用参数和结果 → 审计调试。

### LangGraph 代码风格

```python
# 状态图定义：节点和边的显式编排
graph = StateGraph(AgentState)
graph.add_node("llm", llm_node)
graph.add_node("tools", tool_node)
graph.add_conditional_edges("llm", router, {"tools": "tools", "end": END})
graph.add_edge("tools", "llm")
```

---

## 三、Microsoft AutoGen

### 设计理念

「对话作为编排」——所有 Agent 间通过异步消息协作，没有中央控制器。v0.4 采用**分布式 Actor 模型 + 异步事件驱动 Runtime**。

### 双层架构

| 层 | 职责 |
|----|------|
| **Agent 团队层** | 多个独立 Agent，通过消息传递协作 |
| **Runtime 运行时层** | 管理 Agent 生命周期、工具调用、消息路由、分布式执行 |

消息路由采用**发布/订阅模式**——发送方和接收方完全解耦。

### 四大模块实现

| 模块 | 实现方式 |
|------|---------|
| **感知** | 纯执行型 Agent + 消息传递：采集外部数据 → 封装为标准消息 → Runtime 路由 |
| **规划** | 多 Agent 对话转交逻辑：每个 Agent 局部规划 → `UserTask` 消息转交给下一个 Agent |
| **行动** | Runtime 统一调度：同步/异步/并发/分布式调用，工具结果自动捕获格式化 |
| **反思** | 多 Agent 对质式校验 + Runtime 结果回传：专门校验 Agent → 失败时路由回执行 Agent |

**人工介入**：任意节点配置人工审核 Agent，确认后从断点恢复。

### AutoGen 代码示例

```python
# Agent 间通过消息转交协作
class TriageAgent(RoutedAgent):
    @message_handler
    async def handle_task(self, message, ctx):
        # 规划本地子任务 → 转交给业务 Agent
        await self.send_message(UserTask(...), business_agent_id)

class BusinessAgent(RoutedAgent):
    @message_handler
    async def handle_task(self, message, ctx):
        # 执行本地工具调用 → 结果回传或转交下一 Agent
        result = await self.call_tool(...)
        await self.send_message(result, next_agent_id)
```

---

## 四、CrewAI

### 设计理念

「团队作为编排」——多 Agent 抽象为角色-任务-团队三层声明式配置。**中心化**编排核心：Crew 层统一决定执行顺序。

### 三层抽象

| 层 | 职责 |
|----|------|
| **Agent** | 角色封装：LLM + 专属工具集 + 角色设定 + 执行上下文 |
| **Task** | 工作单元：描述 + 预期输出 + 前置依赖 + 角色绑定 |
| **Crew** | 编排顶层：统一调度、上下文传递、结果归集 |

### 四大模块实现

| 模块 | 实现方式 |
|------|---------|
| **感知** | Agent 角色感知 + 任务上下文注入：上一个任务结果 → 注入当前 Agent |
| **规划** | Crew 编排器 + 任务依赖：生成任务执行序列 → 依次分配给角色 Agent |
| **行动** | Agent 工具集 + Crew 调度：串行/并行/分层（Manager Agent 动态拆解） |
| **反思** | 后置任务反馈 + Agent 角色间互评：独立校验 Agent → 失败回流规划 |

### CrewAI 代码示例

```python
researcher = Agent(
    role="技术研究员",
    goal="检索最新技术资料",
    tools=[search_tool],
    llm="claude-sonnet-4"
)
writer = Agent(
    role="技术文档工程师",
    goal="撰写技术指南",
    tools=[file_write_tool],
    llm="claude-sonnet-4"
)
task1 = Task(description="研究 Next.js Auth", agent=researcher)
task2 = Task(description="撰写 Auth 指南", agent=writer, context=[task1])
crew = Crew(agents=[researcher, writer], tasks=[task1, task2])
crew.kickoff()
```

---

## 五、Claude Code Agent SDK

### 设计理念

「模型优先的极简主义」——不引入额外编排抽象，直接依赖 Claude 模型原生推理能力 + 工具调用环路。「最好的编排代码，就是不引入额外的编排逻辑」。

### 框架只提供三层基础能力

1. 标准化工具接入（MCP 协议，内进程集成）
2. 生命周期控制流（beforeToolCall / afterToolCall Hooks）
3. 子 Agent 协作（任务转交、并行执行、上下文传递）

### 四大模块实现

| 模块 | 实现方式 |
|------|---------|
| **感知** | 原生工具系统 + MCP：六类开箱工具，结果直接注入模型上下文 |
| **规划** | 模型复杂度分析 + 策略选择：简单任务直接答 / 中等链式执行 / 复杂子 Agent |
| **行动** | 内进程工具服务器 + 子 Agent 并行：同步执行，零网络开销 |
| **反思** | 生命周期 Hooks：beforeToolCall 合规校验 + afterToolCall 结果校验 |

### Claude Code SDK 代码示例

```python
@tool(name="search_docs", description="Search documentation")
async def search_docs(args):
    # 对接外部搜索 API
    return {"content": [{"type": "text", "text": results}]}

class CustomBeforeHook(BeforeToolCallHook):
    async def run(self, tool_name, tool_args):
        # 校验搜索关键词长度
        if len(tool_args["query"]) < 3:
            raise ValueError("Query too short")
        return tool_args

agent = Agent(
    model="claude-sonnet-4-20250514",
    system_prompt="你是专业文档检索助手",
    tools=[search_docs],
    hooks=[CustomBeforeToolCallHook(), CustomAfterToolCallHook()]
)
result = await agent.run("Set up auth with Clerk in Next.js")
```

---

## 六、框架深度对比

### 交互模式

| 框架 | 模式 | 特点 |
|------|------|------|
| LangGraph | 状态图传递 + Supervisor | 显式流程图，可精准控制 |
| AutoGen | 发布/订阅 + 主题路由 | 去中心化，最灵活 |
| CrewAI | 任务上下文单向传递 | 中心化调度，简单 |
| Claude Code SDK | 模型-工具调用环路 | 极简，靠模型推理 |

### 状态管理

| 框架 | 持久化 | 回溯 |
|------|--------|------|
| LangGraph | 原生检查点 + 多会话 | ✅ 任意状态回溯 |
| AutoGen | Runtime 集中管理 | ❌ 无内置持久化 |
| CrewAI | 任务级快照 | ❌ 无跨任务持久化 |
| Claude Code SDK | 会话管理 | ❌ 无状态回溯 |

### 控制流

| 框架 | 控制方式 |
|------|---------|
| LangGraph | 条件边显式定义（开发者完全可控） |
| AutoGen | 对话转交动态决定（模型驱动） |
| CrewAI | 任务依赖预定义（开发者声明式） |
| Claude Code SDK | 模型推理动态决定（模型驱动） |

---

## 七、选型决策

| 场景 | 推荐框架 | 原因 |
|------|---------|------|
| 企业级复杂业务流程，需要可视化 | LangGraph | 显式流程图，检查点回溯 |
| 开放式多 Agent 研究，动态任务分配 | AutoGen | 去中心化+消息路由，最灵活 |
| 3 人团队快速 PoC，明天交 demo | CrewAI | 声明式配置，零通信代码 |
| 代码生成 / 合规审计 | Claude Code SDK | 模型原生能力，严格钩子 |
| 已有 LangChain 生态 | LangGraph 渐进升级 | 复用现有代码和工具 |

---

## 相关笔记

- [[Agent vs Workflow 五个维度对比]] — 决策权、灵活性、成本五维对比
- [[Agent 路由系统设计指南]] — Orchestrator/Router/Agent 三层架构
- [[AI Agent 50道高频面试题答案合集]] — 含框架选型题目
- [[Claude Code 多 Agent 实现机制]] — Subagent / Coordinator 路由
