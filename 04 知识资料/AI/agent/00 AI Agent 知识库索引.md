---
title: AI Agent 知识库索引
created: 2026-06-17
updated: 2026-06-17
tags:
  - AI
  - Agent
  - 知识库索引
status: evergreen
confidence: medium
---

# AI Agent 知识库索引

> AI Agent 相关知识的统一入口，涵盖多 Agent 机制、Harness Engineering 和 Agent Skills。

## 一句话总结

AI Agent 的核心不是"换个更强的模型"，而是围绕任务拆解、状态隔离、工具权限、异步通信、验收标准和反馈闭环建立一套可控的协作系统。

## 知识库目录

1. [[04 知识资料/AI/agent/Claude Code 多 Agent 实现机制|Claude Code 多 Agent 实现机制]] — Subagent / Fork / Coordinator 三种模式
2. [[04 知识资料/AI/agent/Harness Engineering 在硅谷爆火|Harness Engineering]] — OpenAI & Anthropic 的 Agent 工程实践
3. [[04 知识资料/AI/Agent Skills/Karpathy Guidelines/andrej-karpathy-skills 内容分析|Karpathy Guidelines 分析]] — 编码 Agent 行为约束四原则
4. [[04 知识资料/AI/Agent Skills/Karpathy Guidelines/karpathy-guidelines 中文 SKILL|Karpathy Guidelines 中文 SKILL]] — 可复用的中文 Skill
5. [[04 知识资料/AI/Agent Skills/Karpathy Guidelines/karpathy-guidelines 中文 Cursor 规则.mdc|Karpathy Guidelines Cursor 规则]] — Cursor 项目规则版本

## 核心地图

```text
AI Agent 知识体系
├── 多 Agent 协作（Claude Code 源码视角）
│   ├── 常规 Subagent：父子型任务拆分
│   ├── Fork Subagent：缓存友好的分身机制
│   └── Coordinator 模式：真正的并行协作
│
├── Harness Engineering（工程化 Agent）
│   ├── 工作台设计：文件结构、工具权限、沙箱环境
│   ├── 验收标准：明确完成条件、可验证目标
│   ├── 反馈闭环：生成器 + 评估器分离
│   └── 约束系统：架构规则、Lint、CI Gate
│
└── Agent Skills（行为约束）
    ├── 编码前思考：暴露假设和歧义
    ├── 简洁优先：最小充分解
    ├── 精准修改：每行 diff 可追溯
    └── 目标驱动：可验证成功标准
```

## 精读优先级

### 入门必看
- [[04 知识资料/AI/agent/Harness Engineering 在硅谷爆火|Harness Engineering]] — 理解 Agent 工程化的核心理念
- [[04 知识资料/AI/Agent Skills/Karpathy Guidelines/andrej-karpathy-skills 内容分析|Karpathy Guidelines]] — 编码 Agent 的行为约束

### 进阶深入
- [[04 知识资料/AI/agent/Claude Code 多 Agent 实现机制|Claude Code 多 Agent 实现机制]] — 源码级多 Agent 设计

### 实战参考
- [[04 知识资料/AI/Agent Skills/Karpathy Guidelines/karpathy-guidelines 中文 SKILL|中文 SKILL]] — 可直接安装使用
- [[04 知识资料/AI/Agent Skills/Karpathy Guidelines/karpathy-guidelines 中文 Cursor 规则.mdc|Cursor 规则]] — Cursor 项目规则

## 可迁移的 5 条 Multi-Agent 设计原则

1. **上下文隔离要按字段粒度做** — 不粗暴全共享或全新建
2. **通信走消息，不走同步函数调用** — 天然异步、支持并发
3. **工具权限要分级管控** — 防止子 Agent 递归派发
4. **缓存友好是一种架构能力** — prompt 前缀复用降低成本
5. **并行优先，协调者负责合成** — 真正价值来自并行

## Harness Engineering 三个核心问题

1. **AI 在哪里干活？** — 代码仓库、文件系统、文档目录、沙箱
2. **AI 用什么干活？** — bash、测试、lint、日志、DevTools
3. **AI 怎么知道自己干得对不对？** — 测试、验收标准、评估器、反馈

## 适合沉淀的问题

- Subagent、Fork Subagent、Coordinator 三种模式分别适合什么场景？
- 如何设计子 Agent 的工具权限表？
- Harness Engineering 和传统 DevOps 有什么异同？
- 如何把 Karpathy Guidelines 四原则融入日常 AI 编码工作流？
- 为什么 AI 自评不可靠？如何设计评估器？

## 关联笔记

- [[04 知识资料/AI/AI 知识库索引|AI 知识库索引]]
- [[04 知识资料/AI/模型工具/Agnes AI/00 Agnes AI 知识库索引|Agnes AI 模型工具]]
- [[04 知识资料/知识库总索引|知识库总索引]]
