---
title: 什么是 Function Calling？原理全解析
type: knowledge
status: evergreen
source_type: 微信公众号
created: 2026-07-13
tags:
  - AI
  - Function Calling
  - 工具调用
  - 面试
source: https://mp.weixin.qq.com/s?__biz=MzY4NTE2NjU5MQ==&mid=2247484424
confidence: high
---

# 什么是 Function Calling？原理全解析

> 来源：微信公众号「后端开发技术」· 鹅厂一面
> 系列索引：[[AI Agent 与 RAG 面试题合集索引]]

---

## 一句话总结

Function Calling 是模型输出结构化 JSON（而非自然语言）来指示调用哪个工具、参数是什么——模型只做决策，代码负责执行，职责分离是核心设计。

---

## 简要回答

开发者用 JSON schema 描述工具，传给模型。模型判断需要调工具时不输出自然语言，而是输出结构化的 `tool_calls` JSON。代码拿到 JSON 去执行，结果塞回对话，模型再生成最终答案。

**两轮对话**：第一轮模型说「我要调这个工具」→ 代码执行 → 第二轮模型拿到结果说「答案是这个」。

---

## 三个角色

| 角色 | 类比 | 职责 |
|------|------|------|
| 开发者 | HR | 写 JSON schema，描述工具有哪些、能做什么、要什么参数 |
| 模型 | 经理 | 读说明书，决定调哪个工具、参数填什么，下达指令 |
| 代码 | 员工 | 真正跑函数、访问网络、查数据库，汇报结果 |

> ⚠️ 关键：模型全程只是在「下指令」，不执行任何代码，没有直接访问网络的权限。

---

## 工具定义（JSON Schema）

```python
tools = [{
    "type": "function",
    "function": {
        "name": "get_weather",
        "description": "查询指定城市的实时天气，仅支持中国大陆城市",
        "parameters": {
            "type": "object",
            "properties": {
                "city": {
                    "type": "string",
                    "description": "城市名称，如'北京'，不带省份前缀"
                },
                "unit": {
                    "type": "string",
                    "enum": ["celsius", "fahrenheit"],
                    "description": "温度单位，默认摄氏度"
                }
            },
            "required": ["city"]
        }
    }
}]
```

**最关键字段：`description`**。写得越清晰，模型选择越准确。模糊描述会导致模型「瞎猜」调用。

---

## 完整调用流程

```python
# 第一轮：传工具定义 + 用户问题
response = client.chat.completions.create(
    model="gpt-4o", messages=messages, tools=tools, tool_choice="auto"
)

if msg.finish_reason == "tool_calls":  # 模型要调工具
    tool_call = msg.tool_calls[0]
    func_args = json.loads(tool_call.function.arguments)
    
    # 中间执行：你的代码真正跑函数
    result = get_weather(func_args["city"])
    
    # 第二轮：结果塞回对话，再次调模型
    messages.append({"role": "tool", "tool_call_id": tool_call.id, "content": result})
    final = client.chat.completions.create(model="gpt-4o", messages=messages)
```

`finish_reason == "tool_calls"` 是模型明确告诉你「我需要工具帮助，还没准备好给答案」。

---

## 并行调用

模型一次可返回多个 `tool_calls`。前提：多个工具之间**没有依赖关系**。

查北京 + 上海天气 → 并行；先查订单号再查物流 → 必须串行。

**优势**：从「两轮对话 × N 个工具」压缩到「一轮对话 + 并行执行」，耗时大幅降低。

---

## 面试要点

1. **模型决策、代码执行**：强调职责分工，模型不访问网络、不执行代码
2. **结构化 JSON 替代自然语言**：这是 Function Calling 和旧方案的本质上区别
3. **两轮对话 + 中间执行**：讲清楚 `finish_reason == "tool_calls"` 的分支
4. **并行调用**：前提是无依赖，用 `asyncio.gather` 或多线程实现

---

## 相关笔记

- [[MCP 三层次架构深度解析]] — MCP vs Function Calling
- [[AI Agent 与 RAG 面试题合集索引]] — 全量面试题索引
- [[Agent 架构面试题-Agent核心篇]] — Agent 工具调用面试题
