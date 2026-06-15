---
title: andrej-karpathy-skills 内容分析
source: https://github.com/multica-ai/andrej-karpathy-skills/tree/main
selected_source: https://github.com/multica-ai/andrej-karpathy-skills/blob/main/.cursor/rules/karpathy-guidelines.mdc
repo: multica-ai/andrej-karpathy-skills
commit: 2c606141936f1eeef17fa3043a72095b4765b9c2
created: 2026-06-11
tags:
  - AI
  - Agent
  - Skill
  - Claude-Code
  - Cursor
  - Coding-Agent
status: 已整理
---

# andrej-karpathy-skills 内容分析

> GitHub 仓库：<https://github.com/multica-ai/andrej-karpathy-skills/tree/main>  
> 选中解析文件：<https://github.com/multica-ai/andrej-karpathy-skills/blob/main/.cursor/rules/karpathy-guidelines.mdc>  
> 本次解析基于仓库 commit：`2c606141936f1eeef17fa3043a72095b4765b9c2`  
> 整理时间：2026-06-11

## 一句话总结

`andrej-karpathy-skills` 是一套面向 Claude Code / Cursor / 编码 Agent 的行为约束指南。它不是教 Agent 写某种语言或框架，而是约束 Agent 在写代码、改代码、重构、评审时避免常见 LLM 问题：**擅自假设、过度设计、乱改无关代码、缺少可验证目标**。

## 仓库解决的问题

仓库的出发点来自 Andrej Karpathy 对 LLM 编码问题的观察：

1. **模型会替用户做假设**  
   当需求模糊时，LLM 往往不澄清，而是默默选一个方向直接实现。

2. **模型容易过度工程化**  
   明明一个简单函数能解决的问题，LLM 可能会引入策略模式、配置系统、抽象层、插件机制等不必要复杂度。

3. **模型会顺手修改不该碰的代码**  
   比如改相邻代码、改注释、重排格式、删除自己没理解的逻辑，导致 diff 变大且风险上升。

4. **模型缺少清晰验证闭环**  
   很多任务只停留在“我会修复”或“我会优化”，没有定义成功标准，也没有测试或检查点。

## 仓库结构解析

| 文件 / 目录 | 作用 |
|---|---|
| `README.md` | 英文说明文档，解释背景、四原则、安装方式、使用效果 |
| `README.zh.md` | 中文版 README，内容与英文 README 基本一致 |
| `CLAUDE.md` | 给 Claude Code 使用的项目级行为指南，可直接放到项目根目录 |
| `CURSOR.md` | 说明如何在 Cursor 中使用该规则，以及 Cursor 与 Claude Code 的差异 |
| `.cursor/rules/karpathy-guidelines.mdc` | Cursor 项目规则，`alwaysApply: true`，打开项目后自动生效 |
| `skills/karpathy-guidelines/SKILL.md` | Claude Code Skill 格式的技能文件，可作为可复用 skill 安装 |
| `.claude-plugin/plugin.json` | Claude Code 插件元数据，声明插件名、描述、版本、技能路径 |
| `.claude-plugin/marketplace.json` | 插件市场元数据，用于 marketplace 安装 |
| `EXAMPLES.md` | 通过真实反例/正例解释四原则如何落地 |

## 核心原则总览

| 原则 | 中文名 | 主要约束 | 解决的问题 |
|---|---|---|---|
| Think Before Coding | 编码前思考 | 先说明假设、歧义、权衡，必要时提问 | 错误假设、隐藏困惑 |
| Simplicity First | 简洁优先 | 用最小代码解决当前问题，不做 speculative design | 过度工程、臃肿抽象 |
| Surgical Changes | 精准修改 | 只改必须改的，只清理自己造成的问题 | 无关 diff、顺手重构 |
| Goal-Driven Execution | 目标驱动执行 | 把任务转为可验证目标，循环检查直到达成 | 缺少测试和成功标准 |

## 1. 编码前思考：不要替用户默默做决定

### 规则含义

在实现前，Agent 需要显式暴露自己的理解：

- 当前需求有哪些假设？
- 是否存在多种解释？
- 是否有更简单方案？
- 哪些地方不清楚，是否需要用户澄清？

### 适用场景

- 需求描述模糊，例如“优化搜索”“修复认证”“加导出功能”。
- 需求涉及安全、隐私、数据范围、接口契约等高风险决策。
- 实现路径存在明显权衡，例如性能 vs. 复杂度、快速修复 vs. 结构调整。

### 不应该怎么做

不要在需求不明确时直接选一个解释并开写。例如用户说“导出用户数据”，Agent 不应直接假设：导出所有用户、导出 JSON、写到本地文件、包含所有字段。

### 应该怎么做

先列出关键问题：导出范围、导出格式、敏感字段、数据量、触发方式。对于简单且低风险的默认路径，可以说明假设后继续；对于高风险路径，应先询问。

## 2. 简洁优先：先解决今天的问题

### 规则含义

代码应以“当前需求的最小充分解”为目标，不为未来可能出现的需求提前设计复杂架构。

### 具体约束

- 不添加需求之外的功能。
- 不为一次性逻辑创建抽象。
- 不添加未要求的灵活性、可配置性、插件化。
- 不为不可能发生的场景写大量错误处理。
- 如果 200 行能写成 50 行，应主动简化。

### 判断标准

问自己一句：**资深工程师会不会觉得这个实现过度复杂？** 如果会，就应该收敛实现。

### 典型反例

用户只要求“计算折扣”，LLM 却创建：

- `DiscountStrategy` 抽象类；
- `PercentageDiscount` / `FixedDiscount` 多个策略；
- `DiscountConfig` 配置对象；
- `DiscountCalculator` 管理器。

如果当前只需要百分比折扣，一个函数就够了。

## 3. 精准修改：每一行 diff 都要能解释

### 规则含义

编辑已有代码时，只改与用户请求直接相关的内容。不要借机“改善”邻近代码，不要顺手重构，不要改风格，不要删除自己没完全理解的逻辑。

### 具体约束

- 不改无关注释、格式和命名。
- 不重构没坏的代码。
- 尽量匹配项目现有风格，即使自己偏好不同。
- 发现无关死代码时，只说明，不擅自删除。
- 只清理因本次改动导致的 unused import / unused variable / orphan function。

### 判断标准

**每一行变更都能直接追溯到用户请求。** 如果解释不了，就不要改。

### 价值

这条规则对真实工程特别重要，因为它能显著降低：

- Code review 成本；
- 回归风险；
- merge conflict；
- 用户对 Agent 的不信任感。

## 4. 目标驱动执行：把任务变成可验证目标

### 规则含义

不要只执行“做某事”的命令，而要把它转成“满足某个可验证成功标准”的目标。

### 转换方式

| 用户说法 | Agent 应转换为 |
|---|---|
| 添加验证 | 为无效输入写测试，然后让测试通过 |
| 修复 bug | 先写能复现 bug 的测试，再修复并验证通过 |
| 重构 X | 重构前后测试都通过，行为不变 |
| 优化性能 | 定义指标，例如响应时间从 500ms 降到 100ms 内 |

### 多步骤任务格式

```text
1. [步骤] → 验证：[检查方式]
2. [步骤] → 验证：[检查方式]
3. [步骤] → 验证：[检查方式]
```

### 核心洞察

LLM 擅长在明确目标下循环执行；弱目标如“让它工作”会导致反复猜测。强目标能让 Agent 自主检查、修正、再检查。

## EXAMPLES.md 的价值

`EXAMPLES.md` 是这个仓库里非常实用的文件。它不只是重复原则，而是用“LLM 常见错误写法”和“推荐写法”做对照。

### 示例覆盖点

1. **隐藏假设**  
   说明为什么“导出用户数据”不能直接开写，需要先确认范围、字段、格式和隐私边界。

2. **多种解释**  
   “Make search faster” 可能是响应时间、吞吐量或感知速度，不能默默选择。

3. **过度抽象**  
   简单折扣计算不需要策略模式。

4. **投机性功能**  
   保存用户偏好不应自动加缓存、校验、通知、合并等未要求能力。

5. **顺手重构**  
   修一个 email 空值 bug，不应顺手修改 username 校验、注释、docstring、格式。

6. **风格漂移**  
   加日志时不应顺手加类型注解、换引号风格、重写返回逻辑。

7. **测试优先验证**  
   修复重复分数排序 bug 前，先写复现测试。

## Cursor 规则文件解析

你选中的 `.cursor/rules/karpathy-guidelines.mdc` 是 Cursor 项目规则文件。

### 文件特征

```yaml
---
description: Behavioral guidelines...
alwaysApply: true
---
```

- `description`：说明规则用途。
- `alwaysApply: true`：表示 Cursor 打开该项目时自动应用，不需要用户手动触发。
- 正文内容基本等同于 `CLAUDE.md`，但包装成 Cursor Rules 格式。

### 适合怎么用

如果你想在任意 Cursor 项目中使用这套规则：

1. 在项目根目录创建 `.cursor/rules/`。
2. 把 `karpathy-guidelines.mdc` 放进去。
3. 打开 Cursor，确认 Settings → Rules 中出现该规则。
4. 如果已有项目规则，可合并，避免互相冲突。

## Claude Code / Cursor / Skill 三种使用形态

| 使用形态 | 文件 | 适合场景 |
|---|---|---|
| Claude Code 项目指令 | `CLAUDE.md` | 单个项目使用，简单直接 |
| Cursor 项目规则 | `.cursor/rules/karpathy-guidelines.mdc` | Cursor 中自动生效 |
| Claude Code Skill / Plugin | `skills/karpathy-guidelines/SKILL.md` + `.claude-plugin` | 跨项目复用，作为个人或团队技能安装 |

## 如何判断这套规则生效了

如果生效，你应该看到：

- Agent 在动手前更愿意说明假设和歧义。
- 需求不清楚时，Agent 会先问问题，而不是盲写。
- diff 更小，只包含与任务相关的变更。
- 代码更简单，不会动不动引入抽象层。
- 多步骤任务会带验证点。
- 修 bug 时更倾向于先复现，再修复，再验证。

## 适合使用的场景

- 写代码；
- 修 bug；
- 重构；
- Code review；
- 让 AI Agent 接手中大型工程任务；
- 约束团队内 AI 编码助手行为；
- 需要减少“AI 顺手乱改”的项目。

## 不适合或需要放宽的场景

这套规则偏谨慎，可能降低极小任务的速度。以下场景可以放宽：

- 修一个拼写错误；
- 明显的一行修改；
- 用户明确要求快速草稿；
- 原型阶段，不追求精确 diff；
- 用户明确授权“可以大幅重构”。

## 我的评价

这套仓库的价值不在于内容复杂，而在于它把 LLM 编码时最容易犯的错误压缩成了四条可执行约束。

它特别适合作为：

1. `CLAUDE.md` 的基础片段；
2. Cursor 全局或项目规则；
3. 团队 AI 编码规范；
4. Code Agent 的默认行为 Skill。

如果要在真实工程中使用，我建议不要原样只放这四条，而是与项目自己的约束合并，例如：

- 技术栈；
- 测试命令；
- lint 命令；
- 分支/提交规范；
- 禁止修改的目录；
- 代码风格和架构边界。

## 本文件夹内产物

- [[04 知识资料/AI/Agent Skills/Karpathy Guidelines/andrej-karpathy-skills 内容分析.md]]：当前完整内容分析。
- [[04 知识资料/AI/Agent Skills/Karpathy Guidelines/karpathy-guidelines 中文 SKILL.md]]：可复制使用的中文 Skill。
- [[04 知识资料/AI/Agent Skills/Karpathy Guidelines/karpathy-guidelines 中文 Cursor 规则.mdc.md]]：中文 Cursor Rule 版本，使用时可改名为 `.mdc`。

## 关联笔记

- [[04 知识资料/知识库总索引|知识库总索引]]
- [[04 知识资料/AI/AI 知识库索引|AI 知识库索引]]

