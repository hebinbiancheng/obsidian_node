---
title: Agent 面试核心问题
type: knowledge
status: evergreen
source_type: 微信公众号合集
created: 2026-06-24
updated: 2026-06-24
tags:
  - AI
  - Agent
  - 面试
  - RAG
source:
  - "https://mp.weixin.qq.com/s?__biz=Mzk3NTQ2MTI2Mg==&mid=2247484076&idx=1&sn=02755305c2191bad28a4eebe57f284fc&chksm=c5ac33e65d081bd307d43df190c10e92bfb4f385b5607d1f3c9599200876793ead6e7eb9a3cf&mpshare=1&scene=24&srcid=0617hjxDVdIB0BNv01iCjFSg&sharer_shareinfo=f001f1775199f05a2d9d51b0ab4763fc&sharer_shareinfo_first=f001f1775199f05a2d9d51b0ab4763fc#rd"
aliases: []
---

# Agent 面试核心问题

## 一句话结论

Agent 面试的核心不是背 ReAct、RAG、MCP 等名词，而是能讲清楚“为什么这么设计、线上怎么兜底、成本和效果如何取舍”。

## 高频考点

### Agent 与 LLM

- LLM 主要负责生成回答；Agent 在 LLM 外增加工具、记忆、规划和循环执行能力。
- 典型回答要能说明：Agent 可以调用外部 API、读取状态、执行动作、根据观察继续决策。
- 风险点是自主循环带来的不确定性，所以必须有步数限制、超时、工具白名单、权限边界和审计日志。

### Agent 与 Workflow

- Workflow 的流程由代码控制，确定性高、成本低、适合固定链路。
- Agent 的控制权更多交给模型，灵活但成本高、可预测性弱。
- 生产系统常见做法是混合：简单任务走 Workflow，复杂任务才进入 Agent，失败后转人工或降级。

### ReAct 与工具调用

- ReAct 的基本循环是 Thought -> Action -> Observation -> Final Answer。
- 防死循环要设置最大步数、重复工具调用检测、全局超时和失败熔断。
- 工具设计要强调幂等、参数校验、最小权限、错误可解释和调用日志。

### RAG 架构

- 标准链路：文档切分、向量化、召回、重排、上下文拼接、生成、引用和反馈。
- 面试常追问召回质量：切分粒度、embedding 选择、混合检索、rerank、查询改写、权限过滤。
- 线上治理要覆盖幻觉、过期知识、引用缺失、召回污染和成本控制。

### 多 Agent

- 多 Agent 适合跨领域协作，如规划、检索、执行、审查分工。
- 不适合为了“显得高级”而拆分，拆得越多通信成本、Token 成本和失败点越多。
- 可靠设计需要角色边界、共享状态、冲突仲裁、最终裁决者和可观测性。

## 面试回答模板

1. 先定义概念。
2. 给出一个生产场景。
3. 说明架构选择。
4. 讲 trade-off：准确率、延迟、Token、稳定性、维护成本。
5. 补充线上兜底：监控、限流、回滚、人工接管。
