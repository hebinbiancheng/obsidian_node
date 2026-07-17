---
title: Loop Engineering 零基础全解：双层架构 + 四代演进
type: knowledge
status: evergreen
source_type: 微信公众号
created: 2026-06-17
updated: 2026-07-13
tags:
  - AI
  - Loop Engineering
  - Agent
  - 工程范式
  - 自动化
source: https://mp.weixin.qq.com/s/0WRjtOWhnVJ4V-8CC73sdA
confidence: high
---

# Loop Engineering 零基础全解

> 原文：[微信文章](https://mp.weixin.qq.com/s/0WRjtOWhnVJ4V-8CC73sdA) · 2026-06-17
> 原始资料：`^[raw/articles/wechat-loop-engineering-2026.html]`

---

## 一句话总结

AI 工程化四代演进：Prompt（会答）→ Context（记得住）→ Harness（能干活）→ **Loop（自己查、自己改、干到对）**。Loop = 执行层 5 组件 + 基建层 5 能力，人从「指挥官」变成「规则设计师」。

---

## 四代 AI 工程演进

| 代 | 定位 | 核心能力 | 致命局限 |
|----|------|---------|---------|
| **Prompt Engineering** | 会答 | 精细提示词、Few-shot、角色设定 | 单次生成，无记忆无修正 |
| **Context Engineering** | 记得住 | RAG、对话记忆、知识库 | 只会读信息，无自我纠错 |
| **Harness Engineering** | 能干活 | 工具调用、权限管控、沙箱 | 有执行无闭环，出错即终止 |
| **Loop Engineering** | 自己闭环 | 生成→评估→修正→迭代→达标终止 | — |

> 前三代：人是循环主体，全程指挥。  
> Loop：系统是循环主体，人只做规则设计，全程无人值守。

---

## 双层架构

```
┌──────────────────────────────────────┐
│  执行层（动态流程·5组件）             │
│  Trigger → Orchestrator → Generator  │
│     → Evaluator → Feedback →         │
├──────────────────────────────────────┤
│  基建层（静态底座·5能力）             │
│  Skills · State · Memory · Tool ·    │
│  DoD                                 │
└──────────────────────────────────────┘
```

---

## 执行层 5 组件

| 组件 | 作用 | 类比 |
|------|------|------|
| **Trigger** 触发器 | 启动条件：事件触发（PR/报错）+ 定时触发 | 闹钟 / 门铃 |
| **Orchestrator** 编排器 | 唯一大脑：判定继续/终止/顺序/异常 | 总指挥 |
| **Generator** 生成器 | Claude 干活：代码修复、文案、方案 | 工人 |
| **Evaluator** 评估器 | 独立质检：按 DoD 校验是否合格 | QA |
| **Feedback** 反馈器 | 报错→可执行优化指令，驱动下一轮 | 教练 |

---

## 基建层 5 能力

| 能力 | 定位 | 说明 |
|------|------|------|
| **Skills** | 规范底座 | 沉淀编码标准、格式、最佳实践，AI 无需每轮重学 |
| **State Management** | 状态管理 | 保存中间结果、任务进度、决策记录 |
| **Memory** | 记忆系统 | 短期（会话内）+ 长期（跨会话），驱动个性化 |
| **Tool Integration** | 工具集成 | 标准化接入外部能力，执行层按需调用 |
| **DoD** | 完成标准 | 可量化验收条件，评估器的判断依据 |

> DoD（Definition of Done）是 Loop 高质量的核心：没有量化标准，评估器就没有判断依据，循环无法终止。

---

## 演进对比：Claude 纯 Prompt 循环 vs Python 工程循环

| 维度 | Claude 纯 Prompt 循环 | Python 工程循环 |
|------|----------------------|----------------|
| 适用阶段 | 快速验证、原型 | 企业级生产 |
| 控制粒度 | 粗（依赖模型理解） | 细（代码精确控制） |
| 可观测性 | 弱 | 强（日志、指标） |
| 状态持久化 | 依赖对话 | 数据库/文件 |
| 并行能力 | 无 | 多线程/异步 |
| 部署方式 | Claude 内 | 任意服务器 |

---

## 企业级 Python 最小落地实现

以下代码包含 10 大组件全覆——技能库、状态管理、子Agent分工（Generator/Evaluator/Feedback）、编排器、连接器、触发器、工作树。可直接运行，也可替换为真实 Claude API。

### Skills 技能库

```python
SKILLS = """
标语生成技能规范：
1. 字数必须≥8个汉字
2. 内容积极向上、正能量
3. 句式工整、简洁有力，适合公开宣传
"""
```

### StateManager 状态管理/记忆持久化

```python
class StateManager:
    def __init__(self):
        self.history = []     # 迭代历史记忆
        self.loop_count = 0   # 循环次数
        self.max_loop = 3     # 最大循环上限
        self.success = False  # 是否达标终止

    def save_record(self, round_num, content, ok, reason):
        self.history.append({
            "round": round_num, "content": content,
            "ok": ok, "reason": reason
        })
```

### 子Agent 分工（Generator + Evaluator + Feedback）

```python
# 生成子Agent — Claude 干活
def generator_agent(feedback):
    # 模拟生成，实际替换为 Claude API
    if state.loop_count == 0:   return "加油"
    elif state.loop_count == 1: return "每天进步一点点"
    else:                       return "保持热爱，奔赴山海，未来可期"

# 评估子Agent — 独立质检，按 DoD 校验
def evaluator_agent(content):
    if len(content) >= 8:
        return True, "通过：字数达标、符合规范"
    else:
        return False, f"不通过：字数仅{len(content)}字，需≥8字"

# 反馈器 — 报错→优化指令
def feedback_maker(reason):
    return f"请优化问题：{reason}，重新生成工整、字数达标的正能量标语"
```

### Orchestrator 编排器 — 核心循环

```python
def orchestrator(task_id):
    global state
    state = StateManager()
    last_feedback = ""
    print("===== Loop工程化闭环启动 =====")

    while not state.success and state.loop_count < state.max_loop:
        state.loop_count += 1
        print(f"\n======== 第{state.loop_count}轮迭代 ========")

        # 1. 生成
        res = generator_agent(last_feedback)
        # 2. 评估
        ok, reason = evaluator_agent(res)
        # 3. 保存记忆
        state.save_record(state.loop_count, res, ok, reason)

        if ok:
            state.success = True
            print(f"✅ 迭代通过：{res}")
        else:
            last_feedback = feedback_maker(reason)
            print(f"❌ 迭代失败：{reason}，已生成修正指令")

    print("\n===== 迭代全部结束，状态已持久化存档 =====")

# Trigger
if __name__ == "__main__":
    orchestrator(task_id="slogan_auto_gen_001")
```

### 真实 Claude API 替换

```python
from anthropic import Anthropic
client = Anthropic(api_key="你的密钥")

def generator_agent(feedback):
    prompt = f"{SKILLS}\n历史优化反馈：{feedback}\n请生成符合规范的正能量标语"
    resp = client.messages.create(
        model="claude-3-sonnet-20240229",
        max_tokens=200,
        messages=[{"role": "user", "content": prompt}]
    )
    return resp.content[0].text
```

---

## 选型指南

✅ **Claude 纯 Prompt Loop**：快速验证、临时任务、零成本体验、个人学习

✅ **Python 工程化 Loop**：项目自动化、对接 GitHub/CI/CD、需要持久化记忆/任务隔离/多Agent分工、企业级落地

---

## 面试话术

> Loop Engineering 是 AI 工程化的第四代范式。前三代分别解决了「会答」「记得住」「能干活」，但都无法自主闭环。Loop 的核心是双层架构：执行层 5 组件负责「生成→评估→修正→迭代」的动态流程，基建层 5 能力（Skills/State/Memory/Tool/DoD）负责「凭什么稳、凭什么专业」的工程底座。最终效果是生成→自我评估→自动修正→达标终止，人从操作者变成规则设计师。

---

## 相关笔记

- [[Loop Engineering-Prompt该退环境了]] — 从 Prompt 到 Loop 的范式转变
- [[Agent 记忆管理高频面试题]] — Memory 系统
- [[Agent vs Workflow 五个维度对比]] — 决策权归属
- [[AI Agent 四大框架深度对比]] — 框架选型
