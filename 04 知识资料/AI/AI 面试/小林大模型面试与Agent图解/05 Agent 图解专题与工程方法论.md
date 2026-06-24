---
title: Agent 图解专题与工程方法论
type: knowledge
status: evergreen
source_type: 本地剪藏与链接解析
created: 2026-06-24
updated: 2026-06-24
tags:
  - AI
  - Agent
  - GraphRAG
  - Harness Engineering
  - Loop Engineering
source:
  - "https://xiaolinnote.com/agent/"
  - "https://xiaolinnote.com/agent/engineering/harness-engineering.html"
  - "https://xiaolinnote.com/agent/engineering/loop-engineering.html"
  - "https://xiaolinnote.com/ai/"
  - "https://xiaolinnote.com/"
  - "https://xiaolinnote.com/claudecode/"
  - "https://mp.weixin.qq.com/s/FYH1I8CRsuXDSybSGY_AFA"
aliases: []
---

# Agent 图解专题与工程方法论

## 一句话结论

图解 Agent 系列把 Agent/RAG 从零散面试题提升为系统工程视角：重点不是一次调用大模型，而是构建能持续吸收上下文、工具反馈和经验的工作环境。

## OpenClaw

- OpenClaw 代表开源 Agent 生态中的端到端实现思路。
- 解析重点是架构分层、工具接入、任务状态和执行循环，而不是只记项目名称。

## GraphRAG 与 LightRAG

- GraphRAG 适合实体和关系复杂、需要跨文档全局推理的知识场景。
- LightRAG 关注更轻量的图增强检索，目标是在复杂度和效果之间取平衡。
- 选型时要考虑构建成本、更新频率、查询类型和可解释性。

## Harness Engineering

- Harness 是让 Agent 更聪明的“工作环境”：工具、记忆、反馈、测试、约束、日志都属于 Harness 的一部分。
- 它强调把上下文和操作环境工程化，让模型在更好的约束和反馈中工作。

## Loop Engineering

- Loop Engineering 从一次性 Prompt 转向“循环”：计划、执行、观察、反馈、修正、沉淀。
- 适合 AI 编程、复杂任务自动化和需要持续改进的 Agent 系统。

## 来源链接

- [图解 Agent](https://xiaolinnote.com/agent/)：站点入口或专题首页，用于确定知识体系边界和来源。
- [Harness Engineering 是什么？AI Agent 越用越聪明的秘密](https://xiaolinnote.com/agent/engineering/harness-engineering.html)：围绕 Harness/Loop Engineering，把 Agent 从一次性 Prompt 扩展为可积累、可反馈、可迭代的工程系统。
- [Loop Engineering 是什么？AI 编程从 Prompt 到 Loop 的范式转变](https://xiaolinnote.com/agent/engineering/loop-engineering.html)：围绕 Harness/Loop Engineering，把 Agent 从一次性 Prompt 扩展为可积累、可反馈、可迭代的工程系统。
- [大模型面试题](https://xiaolinnote.com/ai/)：站点入口或专题首页，用于确定知识体系边界和来源。
- [图解 Agent+RAG+LLM 大模型面试题](https://xiaolinnote.com/)：站点入口或专题首页，用于确定知识体系边界和来源。
- [图解 Claude Code](https://xiaolinnote.com/claudecode/)：站点入口或专题首页，用于确定知识体系边界和来源。
- [我又坚持一年！](https://mp.weixin.qq.com/s/FYH1I8CRsuXDSybSGY_AFA)：站点入口或专题首页，用于确定知识体系边界和来源。
