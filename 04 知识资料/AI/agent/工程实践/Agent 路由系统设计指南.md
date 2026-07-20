---
title: Agent 路由系统设计指南：从原理到避坑
type: knowledge
status: evergreen
source_type: 微信公众号
created: 2026-06-22
updated: 2026-07-13
tags:
  - AI
  - Agent
  - 路由系统
  - 多Agent架构
  - 降级
  - 面试
source: https://mp.weixin.qq.com/s/sYNZ95GxB4v9GFawvzAgFw
confidence: high
---

# Agent 路由系统设计指南

> 原文：[微信文章](https://mp.weixin.qq.com/s/sYNZ95GxB4v9GFawvzAgFw) · 2026-06-22 · 字节二面
> 原始资料：`^[raw/articles/wechat-article-20260713-152920.html]`

---

## 一句话总结

多 Agent 路由层不是「选哪个 Agent」，而是**选错了之后怎么兜住**——关键在降级链路、拓扑模式、十个坑的可观测体系。

---

## 一、为什么路由系统会成为瓶颈

单 Agent 不存在路由。当系统拆成多个专精 Agent（代码/检索/数据分析/客服），路由层变成新单点：

- **路由错了**：下游 Agent 再强也救不回来
- **路由延迟**：叠加到整个链路的首字节时间
- **可解释性**：直接决定出问题后能否 debug

> 路由系统需要被当作独立的、可评估的、可降级的、可观测的子系统设计。

---

## 二、三层架构

### Orchestrator（协调层）

**输出 DAG，不是自然语言**。

```json
{"session_id": "sess_8821", "tasks": [
  {"id": "t1", "goal": "查询用户最近的订单", "depends_on": []},
  {"id": "t2", "goal": "生成退款建议", "depends_on": ["t1"]}
]}
```

### Router（路由层）

输入：子任务 + 上下文摘要 → 输出：`{agent: "code", confidence: 0.91}`。**必须无状态**，同一输入永远一致结果。

### Agent（执行层）

每个 Agent 对外暴露能力声明（capability schema），而非黑盒：

```json
{"agent_id": "code_agent", "description": "代码生成、调试、审查、测试",
 "tools_allowlist": ["read_file", "write_file", "run_tests"],
 "cost_tier": "medium", "avg_latency_ms": 4200}
```

**既是 Router 语义匹配依据，也是安全边界（坑10）**。

---

## 三、五种路由策略

### 1. 规则路由（关键词+正则）

```python
RULES = [
  (r"退款|退货|发票", "billing_agent"),
  (r"SQL|查询.*数据|报表", "data_agent"),
  (r"bug|报错|代码|函数", "code_agent"),
]
```

**优点**：零延迟、零成本、强可解释。**缺点**：规则冲突 → 维护成本指数上升，覆盖不到长尾和复合意图。

### 2. LLM 分类路由

Prompt 要写清楚「擅长什么 **+ 不擅长什么**」。负例和正例同样重要。建议 few-shot 而非零样本，边界 case 的示例比规则描述更稳定。

### 3. Embedding 相似度路由

Agent 能力描述向量化 → 输入 query 向量化 → 余弦相似度匹配。冷启动友好：新增 Agent 只需写描述、生成向量。缺点：对否定语义不敏感（「不要用 SQL」）。

### 4. Bandit 路由（性能反馈）

同一任务可能有多个 Agent 都能处理，用 Thompson Sampling / UCB 根据历史成功率和用户满意度动态调权——探索新 Agent vs 利用已知最优。收敛慢，通常作为前几种之上的动态调权层。

### 5. 混合路由（生产推荐）

```
规则（60-70% 流量）→ Embedding → LLM 分类 → Bandit 最终裁决 → 人工兜底
```

便宜的方法先尝试，贵的方法只在必要时兜底。

---

## 四、五种路由拓扑

| 拓扑 | 说明 | 场景 |
|------|------|------|
| **Pipeline** | 固定顺序流水线，A 输出 = B 输入 | KYC 审核、文档三段式 |
| **Supervisor** | 顶层路由做分发、收敛、升级/重试 | 客服 triage，最主流 |
| **Handoff** | Agent 间控制权**单向转移**，原 Agent 彻底退出 | 客服从通用坐席→专科 |
| **Fan-out / Fan-in** | 并行分发多 Agent → 汇总结果 | 同时检索+查DB+计算 |
| **Swarm / Debate** | 平级协作/互评 → 裁判合并 | 多视角审核、复杂决策 |

**Supervisor vs Handoff**：Supervisor 控制权收回协调层；Handoff 单向转移，移交后不回头。选错拓扑 → 不必要的中转上下文，白白增加延迟。

---

## 五、十个坑全解

| # | 坑 | 解法 |
|---|-----|------|
| 1 | **路由层做了太多业务逻辑**：Router 开始判断「这是 bug 修复还是新功能」→ 失控 | Router 只管分发，业务细分是 Agent 自己的事。单个 Agent 描述控制在 1-2 句话 |
| 2 | **Agent 之间直接调用形成隐式耦合**：A 内 hardcode 调 B → B 改接口全链路崩 | 所有跨 Agent 调用走 Orchestrator 或消息总线，Agent 只向上汇报 |
| 3 | **没有置信度降级机制**：只选最高分 → 边界 case 静默出错，不报异常不进日志 | LLM 超时退回规则 → 规则没命中走 general_agent → 连续低置信度人工干预。整套链路，不靠「换一个候选」碰运气 |
| 4 | **上下文在路由层丢失**：多轮对话中 Router 只拿到当前一句 → 「那这个呢」被错误分类 | Orchestrator 维护 session 上下文，每次路由时带摘要。Router 本身不存状态 |
| 5 | **工具调用没有幂等保护**：Agent 重试 → 重复发邮件/下订单/扣款 | 所有工具带 `request_id`，工具层幂等校验，同一 id 直接返回首次结果 |
| 6 | **同步调用阻塞整条链**：多个独立子任务串行 → 延迟线性叠加 | 画出任务依赖图：有数据依赖才串行，无依赖默认并行（Fan-out） |
| 7 | **Agent 能力描述模糊，路由长期漂移**：兜底描述写多了 → Router 倾向丢给 general_agent | 能力声明既写擅长什么也写不擅长什么，且随 Agent 能力变化同步更新 |
| 8 | **路由决策没有可观测性**：用户反馈「处理不对」→ 无法复盘 | 每次路由打日志：trace_id、输入摘要、各候选分数、最终选择、理由、是否触发降级 |
| 9 | **路由层本身是单点故障**：分类模型超时或限流 → 整个入口瘫 | 显式降级链路：LLM 超时→规则→general_agent，而非直接报错。路由层可用性目标应**高于**下游 Agent |
| 10 | **Agent 间没有权限隔离**：每个 Agent 能调全部工具 → prompt injection 影响系统级别 | 每个 Agent 只授权能力声明里列出的 tools_allowlist。路由层分发时校验「任务所需权限是否在目标 Agent 授权范围」 |

---

## 六、路由质量评估（5 个指标）

| 指标 | 做法 |
|------|------|
| **精确率+召回率** | 每个 Agent 单独计算，不只看整体。整体 90% 可能掩盖某 Agent 召回率仅 50% |
| **混淆矩阵** | 找出哪两个 Agent 最易混淆 → 能力描述边界不清 |
| **置信度校准曲线** | 模型给 0.8 置信度时，实际准确率是否 80%？系统性偏差 → 重新标定阈值 |
| **离线评测集+人工标注闭环** | 线上日志抽样 → 人工标注 → 回灌离线评测集。改 prompt 前先跑回归 |
| **延迟+成本分布** | 监控「走到了哪一层」分布。大量请求落入最贵 LLM 兜底 → 规则/Embedding 层覆盖率需提升 |

---

## 七、生产级路由 Prompt 模板

```
你是一个意图路由器。根据用户输入，从以下 Agent 中选择最合适的一个。

- code_agent：处理代码生成、调试、代码审查、单元测试。
  不处理：纯数据查询、不涉及代码的报表需求（应路由到 data_agent）。
- search_agent：处理联网查询、文档检索、事实核查。
  不处理：需要执行计算或访问内部数据库的任务。
- data_agent：处理数据分析、SQL 查询、图表生成。
  不处理：代码层面的调试，即使涉及 SQL 代码本身的语法错误。
- general_agent：处理闲聊、任务规划、以及不属于以上任何一类的请求。

参考示例：
输入："帮我看看这段 Python 为什么报 KeyError" → code_agent（涉及代码调试）
输入："上个月各地区销售额对比，画个图" → data_agent
输入："这个 SQL 语句报语法错误，帮我改一下" → code_agent（本质调试代码语法，不是数据分析）

输出 JSON：{"agent": "<name>", "confidence": <0-1>, "reason": "<一句话，为什么不是其他候选>"}

用户输入：{input}
会话摘要（如有）：{context_summary}
```

置信度 < 0.65 → Orchestrator 拦截，触发澄清流程。连续两次低置信度 → 人工或 general_agent。

---

## 八、最小落地代码（混合路由）

```python
def route(query: str, session_ctx: dict) -> dict:
    trace_id = new_trace_id()
    # 1. 规则路由优先（零成本）
    result = rule_route(query)
    if result:
        log_routing_decision(trace_id, query, result)
        return result
    # 2. Embedding 路由
    result = embedding_route(query, threshold=0.72)
    if result["confidence"] >= 0.85:
        log_routing_decision(trace_id, query, result)
        return result
    # 3. LLM 分类（带 few-shot + 会话摘要）
    try:
        result = llm_classify_route(query, context_summary=session_ctx.get("summary"))
    except TimeoutError:
        result = {"agent": "general_agent", "confidence": 0.0, "method": "fallback_timeout"}
    # 4. 置信度兜底
    if result["confidence"] < 0.65:
        result = trigger_clarification_or_human_escalation(query, session_ctx)
    log_routing_decision(trace_id, query, result)
    return result
```

---

## 总结

> 路由层要薄，Orchestrator 要聪明，Agent 要专一，工具调用要防御性编程，每一次决策都要可观测、可降级、可复盘。

三层职责靠接口契约分开，不靠「心照不宣」。路由按场景混用——规则兜底高频，LLM/Embedding 处理长尾，Bandit 动态调权。拓扑按任务依赖和控制权转移选，选错直接影响延迟和失败模式。十个坑半数与「边界不清」有关，半数与「缺乏防御性设计」有关。上线后持续用精确率/召回率/置信度校准/成本分布做回归评估。

---

## 相关笔记

- [[Skill编排的6种依赖关系]] — DAG 设计
- [[Agent vs Workflow 五个维度对比]] — Supervisor vs Pipeline 拓扑对应
- [[AI Agent 四大框架深度对比]] — 各框架的 Supervisor 模式实现
- [[Agent 主循环终止条件深度解析]] — 降级与终止策略
