---
title: Function Calling 与工具调用面试题合集
type: knowledge
status: evergreen
source_type: 微信公众号合集
created: 2026-07-13
tags:
  - AI
  - Function Calling
  - MCP
  - 工具调用
  - 面试
source: https://mp.weixin.qq.com/s/Iji8g6yxS7wPeGweOyV5Aw
confidence: high
---

# Function Calling 与工具调用面试题合集

> 来源：微信公众号「后端开发技术」· Function Calling / MCP 面试系列

---

## 已深度解析

| 题目 | 文档 |
|------|------|
| Function Calling 原理 | [[Function Calling 原理全解析]] |
| LLM 工具调用训练 SFT+RLHF | [[LLM 工具调用训练 SFT RLHF 全解析]] |
| MCP 三层次架构 | [[MCP 三层次架构深度解析]] |
| MCP vs Function Calling | 见下方对照表 |
| Agent Skill 本质 | [[Agent Skill 本质与设计面试题解析]] |
| Agent 路由系统 | [[Agent 路由系统设计指南]] |
| Agent 主循环终止 | [[Agent 主循环终止条件深度解析]] |

---

## MCP vs Function Calling

| 维度 | Function Calling | MCP |
|------|-----------------|-----|
| 定位 | 模型能力（运行时调用） | 协议标准（连接规范） |
| 推出方 | OpenAI 2023 | Anthropic 2024 |
| 工具发现 | 每次请求传 tools 参数 | 启动时能力协商 |
| 复用 | 每个应用自行定义 | 一次实现，所有 Host 可用 |
| 传输 | HTTP/API 原生 | stdio / Streamable HTTP |

---

## Agent 范 式对比

| 范式 | 核心循环 | 规划方式 | 适合场景 |
|------|---------|---------|---------|
| ReAct | Thought→Action→Observe | 即时规划 | 需要与外部交互 |
| Plan-and-Execute | Plan→Execute | 全局规划 | 目标明确可预见 |
| Reflection | ReAct + Self-Critique | 反思后重规划 | 复杂需要复盘 |

---

## 相关笔记
- [[AI Agent 与 RAG 面试题合集索引]] — 全量 47 篇题目索引
