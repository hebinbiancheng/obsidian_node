---
title: Agent 核心概念全解：LLM→Agent→Workflow→ReAct→FC→MCP→Skills→A2A
type: knowledge
status: evergreen
source_type: 微信公众号
created: 2026-07-21
tags:
  - AI
  - Agent
  - LLM
  - ReAct
  - MCP
  - Skills
  - Function Calling
  - A2A
  - 面试
source: https://mp.weixin.qq.com/s/mWjhAFRnUqJR7g82Rkepww
confidence: high
---

# Agent 核心概念全解

> 原文：[微信文章](https://mp.weixin.qq.com/s/mWjhAFRnUqJR7g82Rkepww) · 2026-07-21 · 小林
> 原始资料：`^[raw/articles/wechat-agent-core-concepts-2026.html]`

---

## 一句话总结

LLM 是只会说的顾问，Agent 是能动手的项目经理。从 LLM 的四大弊端出发，逐一讲解 Agent 如何用**工具调用 + 记忆 + 规划 + 多模式**解决，最终建立 FC→MCP→Skills→A2A 四层协议栈。

---

## 一、LLM 的四大弊端

| 弊端 | 说明 | Agent 如何解决 |
|------|------|--------------|
| 只会说不会做 | 被困在对话框里，没法操作系统 | 工具调用能力 |
| 没有记忆 | 对话一结束一切归零 | 短期记忆 + 长期记忆 |
| 不会用工具 | 不能上网搜索、查数据库、调 API | MCP 等标准化接入 |
| 不会规划 | 只能一次性生成，不会拆解任务 | 任务拆解和规划能力 |

> LLM 告诉你「怎么做」，Agent 直接帮你「做完了」。

---

## 二、Agent 核心组成

```
大脑(LLM) + 规划模块 + 记忆模块 + 工具模块
```

**实际案例**：「帮我查下周三上海天气，不下雨就安排户外团建」
- LLM：告诉你怎么查天气、怎么建日历事件
- Agent：直接调天气 API → 查到多云 25°C → 自动调日历 API 创建事件

---

## 三、Agent vs Workflow

| 维度 | Workflow | Agent |
|------|----------|-------|
| **控制权** | 代码手里（流程确定） | LLM 手里（动态决策） |
| **类比** | 工厂流水线 | 自主项目经理 |
| **可靠性** | 高，行为可预测 | 中，决策路径不确定 |
| **Token 消耗** | 低（约 Agent 的 1/4） | 高（反复推理） |
| **适用** | 固定步骤、高可靠性要求 | 开放式、无法预知步骤 |

> 生产环境最常见是**混合架构**：整体 Workflow 控制，复杂环节嵌入 Agent。

---

## 四、Agent 四种工作模式

| 模式 | 核心思想 | Token | 适用 |
|------|---------|-------|------|
| **ReAct** | Thought→Action→Observe 循环 | 高 | 灵活适应、通用性强 |
| **Plan-and-Execute** | 先列计划，再逐项执行 | 低（约 ReAct 1/5） | 步骤可预见 |
| **Reflection** | 做完后自查（或双 Agent 对话审查） | 中 | 代码生成、法律文书等高质量场景 |
| **Multi-Agent** | 多专业化 Agent 协作 | 很高 | 超复杂任务 |

> 四种模式不是互斥的——实际中组合使用。一个 Multi-Agent 系统中每个 Agent 内部用 ReAct，协作用 Multi-Agent，最后加 Reflection 质量检查。

---

## 五、Function Call（能力层）

**核心流程**：

```
定义函数(JSON Schema) → LLM 判断调用 → 输出结构化 tool_calls → 代码执行 → 结果回传
```

**关键认知**：LLM 只输出调用指令，**不亲自执行**。真正执行的是宿主程序代码。

| 维度 | Function Call | Agent |
|------|-------------|-------|
| 粒度 | 单步调用 | 循环调用（可能十几次 FC） |
| 关系 | Agent 的原子操作 | FC 的高级编排 |

---

## 六、MCP（连接层）

**类比**：AI 界的 USB-C——统一所有工具连接方式。

| 角色 | 职责 |
|------|------|
| MCP Host | AI 应用（Claude Desktop/Cursor/自定义） |
| MCP Client | 住在 Host 里的翻译官 |
| MCP Server | 对外暴露工具能力（GitHub/Slack/数据库） |

> **核心价值**：把 N×M 集成问题变成 N+M。新增服务不需要改 AI 应用代码；工具可运行时动态发现。

---

## 七、Skills（知识层）

**类比**：MCP 给了厨房（工具），Skills 是菜谱（教怎么做）。

```yaml
---
name: Code_Review_Expert
description: 当用户要求代码审查时自动触发
---
# 审查工作流
1. 看结构：单一职责、超长方法
2. 查漏洞：SQL 注入、越权、空指针
3. 审性能：for 循环查数据库、流未关闭
4. 给方案：每个问题必须附修改建议和优化代码
```

Skills 在 Agent 上下文窗口内生效，指导「什么时候该调工具、用什么策略完成任务」。

---

## 八、FC / MCP / Skills 三者关系

| 维度 | Function Call | MCP | Skills |
|------|-------------|-----|--------|
| **解决的问题** | LLM 怎么调外部函数 | 怎么统一管理大量工具 | Agent 怎么获得领域知识 |
| **运行位置** | 应用程序中 | 外部 MCP Server | Agent 上下文内 |
| **技术本质** | API 协议 | 通信标准 | 提示词扩展 |
| **标准化** | 各厂商格式不统一 | 统一开放标准 | 尚无统一标准 |

> 做饭比喻：Function Call = 会用厨具，MCP = 标准化厨房，Skills = 菜谱。三者结合才做出好菜。

---

## 九、A2A 协议（协作层）

**MCP 处理 Agent↔工具（纵向），A2A 处理 Agent↔Agent（横向）**。

| 概念 | 说明 |
|------|------|
| Agent Card | JSON 名片，描述身份、能力、擅长领域 |
| Task | 任务生命周期：创建→处理→完成/失败 |
| Message & Artifact | 过程信息 + 最终结果制品 |

Google 2025.4 发布，50+ 合作伙伴，HTTP+JSON+SSE 标准。

完整协议栈：

```
A2A（Agent-Agent 协作）
  ↓
Skills（知识层）
  ↓
MCP（连接层）
  ↓
Function Call（能力层）
```

---

## 相关笔记

- [[工程实践/Agent vs Workflow 五个维度对比]]
- [[FC-MCP/Function Calling 与 MCP 全解]]
- [[Skill/Agent Skill 本质与设计面试题解析]]
- [[面试题/Agent 架构面试题-Agent核心篇]]
- [[AI Agent 四大框架深度对比]]
