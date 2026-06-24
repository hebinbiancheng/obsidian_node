---
title: Prompt 工程与 Skill 面试题
type: knowledge
status: evergreen
source_type: 微信公众号合集
created: 2026-06-24
updated: 2026-06-24
tags:
  - AI
  - Prompt
  - Skill
  - 面试
source:
  - "https://mp.weixin.qq.com/s?__biz=MzUxOTAwNTM2MQ==&mid=2247488527&idx=1&sn=1f02f17ce8fd5d0b54016207774edb8c&chksm=f8450f153c77de3f033cae6b409c7884f7488f953517fc328965cc3ce38bfa701418c2e976dd&mpshare=1&scene=24&srcid=0617sVvQFHfBuvrRIpJZUlLf&sharer_shareinfo=2122cbdd8d19bddde76ffb9b2aaeca84&sharer_shareinfo_first=2122cbdd8d19bddde76ffb9b2aaeca84#rd"
aliases: []
---

# Prompt 工程与 Skill 面试题

## 一句话结论

Prompt 工程是低成本调优入口，Skill 是把稳定经验封装成可复用能力；面试中要把两者都讲成工程闭环，而不是“写提示词技巧”。

## Prompt 工程

### 好 Prompt 的组成

- 任务目标：明确要模型完成什么。
- 上下文：提供必要背景、角色、输入数据和约束。
- 输出格式：指定 Markdown、JSON、表格或字段 schema。
- 示例：用少量高质量 few-shot 示例覆盖边界情况。
- 评估：用测试集或模型评审批量比较准确率、一致性和格式合规率。

### Zero-shot 与 Few-shot

- Zero-shot 适合常见、简单、模型熟悉的任务。
- Few-shot 适合格式严格、领域特定或容易误解的任务。
- 示例数量不是越多越好，通常从 1 到 5 个开始，根据评估结果迭代。

### CoT 与结构化推理

- CoT 可帮助复杂推理，但会增加 Token 和泄露中间思路风险。
- 生产中更常用可验证的中间结构，如步骤计划、检查清单、工具调用轨迹和最终 JSON。
- 对高风险输出，要增加自检、规则校验或二次模型评审。

## AI Skill

- Prompt 是一次性输入，Skill 是可复用能力包。
- Skill 的入口描述决定是否触发，SKILL.md 放核心流程，references 放按需资料，scripts 放确定性处理。
- 评估 Skill 要看触发准确率、任务完成率、输出一致性、Token 消耗和维护成本。
