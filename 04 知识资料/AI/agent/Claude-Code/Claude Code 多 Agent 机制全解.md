---
title: Claude Code 多 Agent 机制全解
type: knowledge
status: evergreen
created: 2026-06-11
updated: 2026-07-20
tags:
  - AI
  - Claude-Code
  - Multi-Agent
  - Subagent
  - Coordinator
  - 工程设计
source:
  - https://mp.weixin.qq.com/s/SJ_d8UOR-i3xcXDNozFx6g
  - https://mp.weixin.qq.com/s/BVVkRXKgsCn7kQrStdBGcA
confidence: high
---

# Claude Code 多 Agent 机制全解

> 合并自：小林coding 源码拆解 + Kevin 隔离通信并行缓存

---

## 一句话总结

Claude Code 的多 Agent 体系：**常规 Subagent（父子任务拆分）→ Fork Subagent（缓存友好分身）→ Coordinator（并行协作）**。核心设计围绕隔离、通信、并发、权限和成本做工程化约束。

---

## 一、为什么需要 Multi-Agent

单 Agent（LLM + 工具 + 循环）在真实项目里三个问题：上下文膨胀、角色混乱、无法真正并行。Multi-Agent 三种形态：

| 形态 | 说明 | Claude Code 对应 |
|------|------|-----------------|
| 父子型 | 主 Agent 派子 Agent 处理局部任务 | 常规 Subagent / Task 工具 |
| 平级协作 | 多 Agent 通过共享状态/消息协作 | 工程落地较难 |
| Coordinator-Worker | 协调者拆任务+派工+收结果 | Coordinator 模式 |

---

## 二、Subagent 隔离

### 工具隔离：三道门

| 层级 | 规则 |
|------|------|
| 通用黑名单 | 禁止：创建 Agent、主动问用户、切换规划模式、停止其他任务 |
| 自定义 Agent | 权限更保守 |
| 后台异步 Agent | 白名单——只有明确允许的工具才能用 |

### 上下文隔离：按字段语义逐项决策

| 状态 | 处理 | 原因 |
|------|------|------|
| 文件读取缓存 | 克隆一份 | 子 Agent 后续读取不污染父 Agent |
| 全局 UI 写 | 关闭或空操作 | 防止异步 Agent 与主线程抢写全局状态 |
| 后台任务注册 | 保留通路 | 否则子 Agent 启动的后台进程无人回收 |
| Agent ID / 深度 | 新建并递增 | 追踪嵌套层级，防止递归失控 |

---

## 三、父子通信：消息驱动

父 Agent 不阻塞等——投消息进子 Agent 信箱，子 Agent 在自己循环边界读取。

**关键能力——自动后台化**：子 Agent 执行时间长 → 自动转入后台，父 Agent 继续处理其他事，完成后异步通知。

---

## 四、Fork Subagent：缓存优化

复用父 Agent 已渲染的 system prompt、上下文、工具定义、历史前缀 → **字节级前缀一致 → 命中 prompt cache**。

| 适合 Fork | 不适合 Fork |
|-----------|------------|
| 基于父上下文生成 PR 描述/总结 | 明确专业分工任务 |
| 尝试替代推理路径 | Coordinator 模式 Worker |
| 需要分身但不污染主循环 | 需要定制 System Prompt 的独立 Agent |

---

## 五、Coordinator 模式

```
1. Research：多个 Worker 并行调研不同模块
2. Synthesis：Coordinator 消化→形成方案
3. Implementation：按方案派 Worker 实现
4. Verification：新 Worker 独立验证（避免被实现阶段上下文带偏）
```

**关键设计**：Worker 不能继续创建 Worker、不能管理团队。保持扁平结构：一个 Coordinator，下面多个 Worker，Worker 间不直接互联。

---

## 六、5 条设计原则

| # | 原则 |
|---|------|
| 1 | 上下文隔离到字段粒度——不简单共享或清空 |
| 2 | 通信走消息，不走同步函数调用——否则退化串行 |
| 3 | 工具权限分级——创建 Agent/问用户/改全局状态 不下放 |
| 4 | 缓存友好是架构能力——Fork → 前缀一致 → cache 命中 |
| 5 | 并行不是终点，综合才是——Coordinator 必须理解、判断冲突、提炼共识 |

---

## 相关笔记

- [[AI Agent 四大框架深度对比]]
- [[Claude Code 百万行代码库最佳实践]]
- [[Claude Code 缓存优化四大杀手]]
