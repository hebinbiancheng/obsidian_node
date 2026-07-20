---
title: Function Calling 与 MCP 全解
type: knowledge
status: evergreen
created: 2026-07-13
updated: 2026-07-20
tags:
  - AI
  - Function Calling
  - MCP
  - SFT
  - RLHF
  - 工具调用
  - 面试
source:
  - https://mp.weixin.qq.com/s?__biz=MzY4NTE2NjU5MQ==&mid=2247484424
  - https://mp.weixin.qq.com/s?__biz=MzY4NTE2NjU5MQ==&mid=2247484440
  - https://mp.weixin.qq.com/s/Iji8g6yxS7wPeGweOyV5Aw
confidence: high
---

# Function Calling 与 MCP 全解

> 合并自：Function Calling 原理 + LLM 工具调用训练 + FC/MCP 面试题合集

---

## 一句话总结

Function Calling 是模型输出结构化 JSON 指示工具调用——**模型只决策，代码负责执行**。工具调用能力不是涌现的，靠 SFT 教会「怎么调」+ RLHF 教会「该不该调」。MCP 是工具与 AI 应用之间的标准化通信协议。

---

## 一、Function Calling 核心机制

### 三个角色

| 角色 | 职责 |
|------|------|
| 开发者 | 写 JSON schema（工具说明书） |
| 模型 | 读说明书，决定调哪个工具、参数填什么——只下指令，不执行 |
| 代码 | 真正跑函数、访问网络、查数据库 |

### JSON Schema 定义

```python
tools = [{
    "type": "function",
    "function": {
        "name": "get_weather",
        "description": "查询指定城市的实时天气，仅支持中国大陆城市",
        "parameters": {
            "type": "object",
            "properties": {
                "city": {"type": "string", "description": "城市名称，如'北京'，不带省份前缀"},
                "unit": {"type": "string", "enum": ["celsius", "fahrenheit"]}
            },
            "required": ["city"]
        }
    }
}]
```

> `description` 是最关键字段——写得越清晰，模型选择越准确。

### 完整调用流程（两轮对话 + 中间执行）

```python
# 第一轮：传工具定义 + 用户问题
response = client.chat.completions.create(model="gpt-4o", messages=messages, tools=tools, tool_choice="auto")

if msg.finish_reason == "tool_calls":
    tool_call = msg.tool_calls[0]
    func_args = json.loads(tool_call.function.arguments)
    result = get_weather(func_args["city"])  # 中间执行
    # 第二轮：结果塞回对话
    messages.append({"role": "tool", "tool_call_id": tool_call.id, "content": result})
    final = client.chat.completions.create(model="gpt-4o", messages=messages)
```

### 并行调用

模型一次可返回多个 `tool_calls`。前提：工具之间无依赖关系。从「两轮对话 × N」压缩为「一轮 + 并行执行」。
- 查北京 + 上海天气 → 并行
- 先查订单号再查物流 → 必须串行

---

## 二、LLM 如何学会调用工具

> 工具调用 ≠ 涌现能力——输出结构化 JSON 是预训练语料不存在的模式。

| 阶段 | 方法 | 解决什么问题 |
|------|------|------------|
| **SFT** | 监督微调：数十万条工具调用对话样本 | 学会「能不能调」——识别工具定义、判断要不要调、输出规范 JSON |
| **RLHF** | 人类反馈强化学习：人类打分→训练奖励模型→PPO 优化 | 学会「该不该调」——建立边界感，「1+1=2」就不调计算器 |

SFT 训练样本格式：`System(工具schema) → User(问题) → Assistant(tool_calls JSON) → Tool(结果) → Assistant(答案)`

数据来源：人工标注（种子数据）+ GPT-4 自动生成（主流做法）。

**RLAIF**（AI Feedback）是 RLHF 的低成本替代方案——用 AI 代替人类标注打分。

---

## 三、MCP vs Function Calling

| 维度 | Function Calling | MCP |
|------|-----------------|-----|
| 定位 | 模型能力（运行时调用） | 协议标准（连接规范） |
| 推出方 | OpenAI 2023 | Anthropic 2024 |
| 工具发现 | 每次请求传 tools 参数 | 启动时能力协商、动态发现 |
| 复用 | 每个应用自行定义 | 一次实现，所有 Host 可用 |
| 传输 | HTTP/API 原生 | stdio / Streamable HTTP |

> Function Calling 是「模型会输出什么格式」，MCP 是「工具系统怎么标准化连接」。两者互补而非替代。

### 选型决策

| 场景 | 推荐 |
|------|------|
| 单个应用、3-5 个工具 | Function Calling |
| 多应用共享工具、需要动态发现、需要进程隔离/OAuth | MCP |

---

## 四、为什么推理模型不支持 MCP

推理模型（o1/R1/Claude Extended Thinking）的思维链是一次性连续生成的，不能中途打断。而工具调用天然需要「暂停等外部执行」。

三层冲突：KV Cache 显存占用、训练目标相悖（RL 奖励完整推理 vs FC 鼓励主动打断）、一致性断裂。

**折中方案**：工具调用在思考阶段结束后触发（o3/Extended Thinking），保证推理质量但思维阶段感知不到工具结果。

---

## 相关笔记

- [[MCP 三层次架构深度解析]]
- [[Agent 路由系统设计指南]]
- [[推理模型不支持MCP 生成范式冲突]]
