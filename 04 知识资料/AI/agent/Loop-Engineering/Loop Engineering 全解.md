---
title: Loop Engineering 全解
type: knowledge
status: evergreen
created: 2026-07-05
updated: 2026-07-20
tags:
  - AI
  - Loop-Engineering
  - Agent
  - 工程范式
  - 自动化
source:
  - https://mp.weixin.qq.com/s/omwt7d9BSFX7kotW9vo9bQ
  - https://mp.weixin.qq.com/s/kICrdEkPCYAiyOiwI-Gt1Q
  - https://mp.weixin.qq.com/s/0WRjtOWhnVJ4V-8CC73sdA
confidence: high
---

# Loop Engineering 全解

> 合并自：Prompt 该退环境了 + 双层架构零基础全解 + 14 步实操手册

---

## 一句话总结

Loop Engineering 是 AI 工程化第四代范式：人不写 Prompt 逐次指挥，而是定义目标 + 验证条件 + 失败处理，让 Agent 自动循环「生成→评估→修正」直到达标。核心竞争力不是技术，是**定义目标的能力**。

---

## 一、四代 AI 工程演进

| 代 | 核心能力 | 对应学科 |
|----|---------|---------|
| Prompt Engineering | 好好说话 | 语言学 |
| Context Engineering | 给 AI 足够信息 | 信息科学 |
| Harness Engineering | 设规则和约束 | 控制论 |
| **Loop Engineering** | **定义目标和管理** | **管理学** |

> Loop 的核心：你不再给 Agent 写 Prompt 完成单次任务，而是设计一个目标，让系统自动循环——发现问题 → 解决问题 → 通知人类，一条龙。

---

## 二、双层架构

```
执行层（5 组件·动态流程）
  Trigger → Orchestrator → Generator → Evaluator → Feedback
基建层（5 能力·静态底座）
  Skills · State · Memory · Tool · DoD
```

| 组件 | 作用 | 类比 |
|------|------|------|
| **Trigger** | 启动：事件触发（PR/报错）+ 定时触发 | 闹钟 |
| **Orchestrator** | 判定继续/终止/顺序/异常 | 总指挥 |
| **Generator** | Claude 干活 | 工人 |
| **Evaluator** | 按 DoD 独立质检 | QA |
| **Feedback** | 报错→优化指令，驱动下一轮 | 教练 |

---

## 三、企业级最小实现

```python
class StateManager:
    def __init__(self):
        self.history, self.loop_count, self.max_loop, self.success = [], 0, 3, False

def orchestrator(task_id):
    state = StateManager()
    while not state.success and state.loop_count < state.max_loop:
        state.loop_count += 1
        res = generator_agent(last_feedback)
        ok, reason = evaluator_agent(res)
        state.save_record(state.loop_count, res, ok, reason)
        if ok: state.success = True
        else: last_feedback = feedback_maker(reason)
```

---

## 四、目标定义框架

| 原则 | 说明 |
|------|------|
| **完成标准可机器验证** | 不能是「感觉差不多了」，必须是可执行命令 |
| **边界条件一起定义** | 不仅要有做完的标准，更要有**不能怎么做**的边界 |
| **失败降级方案** | 无限循环成本爆炸，需定义尝试上限 |
| **目标分层** | 顶层目标 + 可验证子目标，逐层收紧 |

### 古德哈特定律

> 当一个衡量指标变成目标本身时，它就不再是好的衡量指标了。

你的 loop 条件是「让测试全通过」→ Agent 可能直接删掉失败测试。所以需要 **Harness + Loop**：Harness 是护栏，Loop 是驱动力。

---

## 五、14 步路线图

**第一段：想清楚要不要做（5 步）**
1. 确认任务重复——一次性的用 Prompt 更快
2. 确认有自动判定「干砸了」的手段（测试/lint/构建）
3. 确认 token 预算扛得住浪费
4. 确认 Agent 能跑自己写的代码（日志+复现）
5. 确认你真打算 review 产出

**第二段：搭最小能跑的 Loop（8 步）**
6. 先让一次手动运行稳定
7. 把项目背景沉淀成 Skill
8. 加状态文件（做完了什么、下一步干啥）
9. 设硬闸门（测试/构建不过就拒）
10. 配 Automation（按节奏触发，`/goal` 设停止条件）
11. 多 Agent 并行上 Worktree
12. 接上 Connectors（GitHub/Slack/Sentry）
13. 拆出 Sub-agents（写和验分开）

**第三段：守住（1 步）**
14. 盯住每个被接受的改动成本，定期复审权限、读 diff、别让 loop 碰架构

---

## 相关笔记

- [[Agent 主循环终止条件深度解析]]
- [[Vibe Coding 两大基石 Prompt]]
- [[Harness Engineering 在硅谷爆火]]
