---
title: Agent 核心面试体系
type: knowledge
status: evergreen
source_type: 本地剪藏与链接解析
created: 2026-06-24
updated: 2026-06-24
tags:
  - AI
  - Agent
  - 面试
source:
  - "https://xiaolinnote.com/agent/concept/agent.html"
  - "https://xiaolinnote.com/agent/concept/openclaw.html"
  - "https://xiaolinnote.com/ai/agent/agent_info.html"
  - "https://xiaolinnote.com/ai/agent/1_whatisagent.html"
  - "https://xiaolinnote.com/ai/agent/2_components.html"
  - "https://xiaolinnote.com/ai/agent/3_workflow_tools.html"
  - "https://xiaolinnote.com/ai/agent/4_patterns.html"
  - "https://xiaolinnote.com/ai/agent/5_react.html"
  - "https://xiaolinnote.com/ai/agent/6_three_patterns.html"
  - "https://xiaolinnote.com/ai/agent/7_tasksplit.html"
  - "https://xiaolinnote.com/ai/agent/8_memory.html"
  - "https://xiaolinnote.com/ai/agent/9_memory_storage.html"
  - "https://xiaolinnote.com/ai/agent/10_multiagent.html"
  - "https://xiaolinnote.com/ai/agent/11_single_multi.html"
  - "https://xiaolinnote.com/ai/agent/12_memcompress.html"
  - "https://xiaolinnote.com/ai/agent/13_handcode.html"
  - "https://xiaolinnote.com/ai/agent/14_planning.html"
  - "https://xiaolinnote.com/ai/agent/15_reflection.html"
  - "https://xiaolinnote.com/ai/agent/16_collab.html"
aliases: []
---

# Agent 核心面试体系

## 一句话结论

Agent 面试的主线是：LLM 不只是回答器，而是通过工具、记忆、规划和执行循环完成目标；工程回答必须说明边界、成本、可观测性和失败兜底。

## 知识结构

### 概念与架构

- Agent = LLM + Tools + Memory + Planning + Execution Loop。
- Workflow 是代码主导流程，Agent 是模型在约束内主导下一步动作，Tools 是外部能力入口。
- 面试中要用具体业务例子说明三者边界：固定审批流适合 Workflow，开放式排障/研究任务适合 Agent。

### 设计范式

- ReAct：边推理边行动，适合探索式任务，但要防止循环失控。
- Plan-and-Execute：先规划后执行，适合较长任务，可降低盲目调用。
- Reflection：生成后自评和修正，适合代码、文档、复杂推理等高质量场景。

### 工程实践

- 任务拆分：按目标、依赖、可验证中间结果拆分。
- 记忆设计：短期上下文、长期知识、用户偏好、任务状态分层存储。
- 手搓 Agent 的理由：框架抽象不匹配、需要精细控制成本/日志/权限/状态、需要和现有系统深度集成。

### 多 Agent

- 适合跨角色协作：规划、检索、执行、审查、裁决。
- 关键问题是通信、共享状态、任务路由、冲突解决和最终责任归属。

## 来源链接

- [AI Agent 是什么？Agent 面试题万字图解](https://xiaolinnote.com/agent/concept/agent.html)：围绕 Agent 定义、架构组件、Workflow/Tools 区别、ReAct/规划/反思、任务拆分、记忆、多 Agent 协作等面试点展开。
- [OpenClaw 是什么？OpenClaw 面试题万字图解](https://xiaolinnote.com/agent/concept/openclaw.html)：围绕 Agent 定义、架构组件、Workflow/Tools 区别、ReAct/规划/反思、任务拆分、记忆、多 Agent 协作等面试点展开。
- [Agent 面试题介绍](https://xiaolinnote.com/ai/agent/agent_info.html)：围绕 Agent 定义、架构组件、Workflow/Tools 区别、ReAct/规划/反思、任务拆分、记忆、多 Agent 协作等面试点展开。
- [1. 什么是 Agent？与大模型有什么本质不同？](https://xiaolinnote.com/ai/agent/1_whatisagent.html)：围绕 Agent 定义、架构组件、Workflow/Tools 区别、ReAct/规划/反思、任务拆分、记忆、多 Agent 协作等面试点展开。
- [2. Agent 的基本架构由哪些核心组件构成？](https://xiaolinnote.com/ai/agent/2_components.html)：围绕 Agent 定义、架构组件、Workflow/Tools 区别、ReAct/规划/反思、任务拆分、记忆、多 Agent 协作等面试点展开。
- [3. Workflow，Agent，Tools 这三个的概念和区别介绍一下？](https://xiaolinnote.com/ai/agent/3_workflow_tools.html)：围绕 Agent 定义、架构组件、Workflow/Tools 区别、ReAct/规划/反思、任务拆分、记忆、多 Agent 协作等面试点展开。
- [4. 了解哪些其他的 Agent 设计范式？Agent 和 Workflow的区别是什么？](https://xiaolinnote.com/ai/agent/4_patterns.html)：围绕 Agent 定义、架构组件、Workflow/Tools 区别、ReAct/规划/反思、任务拆分、记忆、多 Agent 协作等面试点展开。
- [5. Agent 推理模式有哪些？ReAct 是啥？具体是怎么实现的？](https://xiaolinnote.com/ai/agent/5_react.html)：围绕 Agent 定义、架构组件、Workflow/Tools 区别、ReAct/规划/反思、任务拆分、记忆、多 Agent 协作等面试点展开。
- [6. ReAct、Plan-and-Execute、Reflection 三种范式有什么核心区别？实际项目中该如何选型？](https://xiaolinnote.com/ai/agent/6_three_patterns.html)：围绕 Agent 定义、架构组件、Workflow/Tools 区别、ReAct/规划/反思、任务拆分、记忆、多 Agent 协作等面试点展开。
- [7. 复杂任务怎么做的任务拆分？为什么要拆分？效果如何提升？](https://xiaolinnote.com/ai/agent/7_tasksplit.html)：围绕 Agent 定义、架构组件、Workflow/Tools 区别、ReAct/规划/反思、任务拆分、记忆、多 Agent 协作等面试点展开。
- [8. 请你介绍一下 AI Agent 的记忆机制，并说明在实际开发中应该如何设计记忆模块？](https://xiaolinnote.com/ai/agent/8_memory.html)：围绕 Agent 定义、架构组件、Workflow/Tools 区别、ReAct/规划/反思、任务拆分、记忆、多 Agent 协作等面试点展开。
- [9. Agent 的长短期记忆系统怎么做的？记忆是怎么存的？粒度是多少？怎么用的？](https://xiaolinnote.com/ai/agent/9_memory_storage.html)：围绕 Agent 定义、架构组件、Workflow/Tools 区别、ReAct/规划/反思、任务拆分、记忆、多 Agent 协作等面试点展开。
- [10. 什么是 Multi-Agent？](https://xiaolinnote.com/ai/agent/10_multiagent.html)：围绕 Agent 定义、架构组件、Workflow/Tools 区别、ReAct/规划/反思、任务拆分、记忆、多 Agent 协作等面试点展开。
- [11. 说说 Single-Agent 和 Multi-Agent 的设计方案？](https://xiaolinnote.com/ai/agent/11_single_multi.html)：围绕 Agent 定义、架构组件、Workflow/Tools 区别、ReAct/规划/反思、任务拆分、记忆、多 Agent 协作等面试点展开。
- [12. Agent 记忆压缩通常有哪些方法？](https://xiaolinnote.com/ai/agent/12_memcompress.html)：围绕 Agent 定义、架构组件、Workflow/Tools 区别、ReAct/规划/反思、任务拆分、记忆、多 Agent 协作等面试点展开。
- [13. 在工程实践中，为什么有时候选择「手搓」Agent，而不是直接用成熟框架？](https://xiaolinnote.com/ai/agent/13_handcode.html)：围绕 Agent 定义、架构组件、Workflow/Tools 区别、ReAct/规划/反思、任务拆分、记忆、多 Agent 协作等面试点展开。
- [14. 如何赋予 LLM 规划能力？](https://xiaolinnote.com/ai/agent/14_planning.html)：围绕 Agent 定义、架构组件、Workflow/Tools 区别、ReAct/规划/反思、任务拆分、记忆、多 Agent 协作等面试点展开。
- [15. 讲讲 Agent 的反思机制？为什么要用反思？具体怎么实现？](https://xiaolinnote.com/ai/agent/15_reflection.html)：围绕 Agent 定义、架构组件、Workflow/Tools 区别、ReAct/规划/反思、任务拆分、记忆、多 Agent 协作等面试点展开。
- [16. 如何设计多 Agent 的协作与动态切换机制？](https://xiaolinnote.com/ai/agent/16_collab.html)：围绕 Agent 定义、架构组件、Workflow/Tools 区别、ReAct/规划/反思、任务拆分、记忆、多 Agent 协作等面试点展开。
