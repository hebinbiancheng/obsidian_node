---
title: MCP 深度解析：三层次架构（角色·能力·协议）
type: knowledge
status: evergreen
source_type: 微信公众号
created: 2026-05-15
updated: 2026-07-06
tags:
  - AI
  - Agent
  - MCP
  - 面试
  - JSON-RPC
  - stdio
source: https://mp.weixin.qq.com/s/8bI19Vfn-5Kqpwp0Yg1kdg
confidence: high
---

# MCP 深度解析：三层次架构

> 原文：[微信文章](https://mp.weixin.qq.com/s/8bI19Vfn-5Kqpwp0Yg1kdg) · 2026-05-15
> 原始资料：`^[raw/articles/wechat-k8s-network-troubleshooting-2026.html]`

---

## 一句话总结

MCP 不是简单的「Client + Server」，而是 **角色层（Host/Client/Server）+ 能力层（Tools/Resources/Prompts）+ 协议层（JSON-RPC 2.0 + stdio/Streamable HTTP）** 的三层架构，每层独立解耦。

---

## 简要回答

MCP 由三个维度组成：

**角色层**：Host（AI 应用）→ Client（通信模块）→ Server（独立进程），一个 Host 可连多个 Server。

**能力层**：Server 暴露三类能力：
- **Tools**：有副作用的操作（创建文件、调 API）
- **Resources**：只读数据（读文档、查数据库）
- **Prompts**：预定义提示词模板

**协议层**：JSON-RPC 2.0 消息格式 + 两种传输方式（stdio 本地 / Streamable HTTP 远程）。

---

## 第一层：角色架构 — Host / Client / Server

![MCP 架构图 1](assets/mcp-interview-1.png)

| 角色 | 职责 | 类比 |
|------|------|------|
| **Host** | AI 应用本身（Claude Desktop、Cursor） | 公司总部，决定与哪些供应商合作 |
| **Client** | Host 内部连接模块，1:1 对应 Server | 驻场联络员，负责对接某一供应商 |
| **Server** | 工具提供方独立进程 | 供应商，不关心上面是谁在调用 |

**核心关系**：Host 不直接和 Server 说话，通过 Client 中转。一个 Host 连多个 Server，模型同时拥有所有工具能力。

![MCP 架构图 2](assets/mcp-interview-2.png)

**关键设计**：Server 写一次，任何 MCP Host 都能用。GitHub 的 MCP Server 不需要分别为 Claude Desktop 和 Cursor 各写一份。

---

## 第二层：能力类型 — Tools / Resources / Prompts

![MCP 架构图 3](assets/mcp-interview-3.png)

| 能力 | 本质 | 副作用 | 触发方式 | 示例 |
|------|------|--------|----------|------|
| **Tools** | 改变世界 | ✅ 有 | 模型主动触发，需审批 | 创建文件、提交代码、发 Slack、调 API |
| **Resources** | 观察世界 | ❌ 无 | 按需暴露，宽松授权 | 读取日志、查数据库、获取文档 |
| **Prompts** | 结构化表达 | ❌ 无 | 模板化复用 | 代码审查标准、团队最佳实践模板 |

**记忆口诀**：Tools 改变世界，Resources 观察世界，Prompts 结构化表达。

![MCP 架构图 4](assets/mcp-interview-4.png)

> 💡 面试关键区分：Tools 和 Resources 的本质差异是有无副作用。说清这点比背定义更让面试官信服。

---

## 第三层：传输协议 — JSON-RPC 2.0 + 传输方式

消息格式和传输方式完全解耦：

### 消息格式：JSON-RPC 2.0

```json
// Client 查询工具列表
{"jsonrpc": "2.0", "id": 1, "method": "tools/list", "params": {}}

// Server 返回工具列表
{"jsonrpc": "2.0", "id": 1, "result": {"tools": [{"name": "read_file", ...}]}}

// Client 调用工具
{"jsonrpc": "2.0", "id": 2, "method": "tools/call",
 "params": {"name": "read_file", "arguments": {"path": "/tmp/log.txt"}}}
```

### 传输方式

| 方式 | 适用场景 | 特点 |
|------|----------|------|
| **stdio** | 本地工具 | 子进程通信，零延迟，无网络暴露 |
| **Streamable HTTP** | 远程部署 | 单端点 `/mcp`，取代旧的 HTTP+SSE 双端点 |

![MCP 架构图 5](assets/mcp-interview-5.png)

**协议演进**：MCP 早期（2024-11-05）远程方案是 HTTP+SSE 双端点（GET + POST），2025 年 3 月规范更新改为 Streamable HTTP 单端点，旧的已 deprecated。

Streamable HTTP 的设计：短请求回 JSON，长请求升级为 SSE 流——对负载均衡和 serverless 更友好。

![MCP 架构图 6](assets/mcp-interview-6.png)

---

## 🎯 面试总结

回答「MCP 由哪几部分组成」的三个要点：

1. **三层结构说清楚**：角色层 + 能力层 + 协议层，特别是 Host ≠ Client
2. **能力区分**：Tools（有副作用）vs Resources（只读）的本质差异
3. **协议解耦**：JSON-RPC 2.0 定义消息格式，stdio/Streamable HTTP 定义传输——互不耦合

---

## 相关笔记

- [[Agent 架构面试题-Agent核心篇]] — MCP 协议基础 + OAuth 2.1（第4题）
- [[AI Agent 50道高频面试题答案合集]] — MCP vs Skill 对比（Q20/Q47）
- [[16个国民级App蒸馏成Skills盘点]] — MCP/Skill 生态产品一览
- [[AI Agent 面试题与答案]] — 完整面试题索引
