---
title: LLM 是如何学会调用外部工具的？SFT + RLHF 全解析
type: knowledge
status: evergreen
source_type: 微信公众号
created: 2026-07-13
tags:
  - AI
  - Function Calling
  - SFT
  - RLHF
  - 工具调用
  - 面试
source: https://mp.weixin.qq.com/s?__biz=MzY4NTE2NjU5MQ==&mid=2247484440
confidence: high
---

# LLM 是如何学会调用外部工具的？

> 来源：微信公众号「后端开发技术」· 美团大模型一面
> 系列索引：[[AI Agent 与 RAG 面试题合集索引]]

---

## 一句话总结

工具调用能力不是涌现出来的——靠两个阶段训练：SFT 教会「怎么调」，RLHF 教会「该不该调」。

---

## 原始 LLM 为什么不会调工具？

预训练阶段模型只学过「根据上文预测下一个 token」，全程在文本空间。从未见过工具调用，遇到「查天气」只会用自然语言描述，不会输出可解析的 JSON。

> 工具调用 ≠ 语言涌现能力。输出结构化 JSON 是预训练语料不存在的模式。

---

## 第一阶段：SFT（监督微调）

**核心思路**：给模型看大量正确示例，学会模仿。

一条完整训练样本：
```
System: 工具说明书（JSON schema）
User: 用户提问
Assistant: {"tool_calls": [{"name":"get_weather","arguments":{"city":"北京"}}]}
Tool: 工具返回数据
Assistant: 最终自然语言答案
```

数据来源：人工标注（种子数据）+ 强模型自动生成（主流做法）。

**SFT 的短板**：学会了「调」的动作，但不知道什么时候该调、什么时候不该调。模型会过拟合「积极调用」，边界感弱。

---

## 第二阶段：RLHF（人类反馈强化学习）

**四步流程**：

1. **生成多样回答**：对同一问题生成多种处理方式
2. **人类打分**：标注哪种更合理（「1+1=2」直接答更好）
3. **训练奖励模型**：用小模型专门打分，蒸馏人类偏好
4. **强化学习优化**：用奖励模型调整主模型，产出高分回答

**RLAIF**（AI Feedback）是 RLHF 的变体，用 AI 代替人类标注，成本更低。

---

## 运行时：Function Calling

```
应用代码传 schema + 用户问题 → 模型输出 tool_calls JSON → 
代码解析执行 → 结果塞回对话 → 模型给出最终答案
```

> 关键：模型只负责**决策**（输出 JSON），代码负责**执行**。

---

## 面试要点

1. **SFT vs RLHF**：前者解决能不能调，后者解决该不该调
2. **训练数据格式**：System → User → Assistant(tool_calls) → Tool → Assistant(答案)
3. **涌现能力 ≠ 工具调用**：工具调用需要专项训练
4. **模型决策、代码执行**：核心分工原则

---

## 相关笔记

- [[Function Calling 原理全解析]] — Function Calling 运行时机制
- [[AI Agent 与 RAG 面试题合集索引]] — 全量面试题索引
