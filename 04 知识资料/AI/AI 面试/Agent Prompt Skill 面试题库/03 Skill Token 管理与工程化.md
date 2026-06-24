---
title: Skill Token 管理与工程化
type: knowledge
status: evergreen
source_type: 微信公众号合集
created: 2026-06-24
updated: 2026-06-24
tags:
  - AI
  - Skill
  - Token
  - 工程化
source:
  - "https://mp.weixin.qq.com/s?__biz=MjM5MTIyMjkzMg==&mid=2247493922&idx=1&sn=cea244459ffdfc57f24f9b8559849439&chksm=a753b21044ac330e5f3ed6f49c9f125dc14674d40ec55b7715e3af36a169ec8214126e9dd651&mpshare=1&scene=24&srcid=0622A6jaezICktGAwUN5ytFX&sharer_shareinfo=807afa4b1205ec59f99a14c3390ebb76&sharer_shareinfo_first=807afa4b1205ec59f99a14c3390ebb76#rd"
aliases: []
---

# Skill Token 管理与工程化

## 一句话结论

Skill 的 Token 管理不是“少写点”，而是用元数据、入口指令、按需参考文件和脚本形成分层加载体系。

## 三层加载结构

- 元数据：名称和描述常驻上下文，必须短、准、能帮助模型判断是否触发。
- SKILL.md：触发 Skill 后加载，应该只放核心流程、决策规则和路由说明。
- 捆绑资源：references、scripts、templates、assets 只在需要时读取，避免一次性塞入上下文。

## 控制 SKILL.md 长度

- 正文应服务于“怎么判断、怎么执行、何时读取哪个资源”。
- 复杂平台、框架、供应商差异应拆到 references。
- 大段示例、长模板和确定性转换逻辑不适合放在主说明里。

## 参考文件拆分

- 按平台拆：aws.md、gcp.md、azure.md。
- 按框架拆：react.md、vue.md、nextjs.md。
- 按阶段拆：requirements.md、implementation.md、qa.md。
- 在入口中写清楚路由规则，让模型只读当前任务相关文件。

## 用脚本替代长说明

- 文件格式转换、批量重命名、表格清洗、模板渲染等确定性任务应脚本化。
- 脚本能减少 Prompt 长度，也能提升结果稳定性。
- Skill 文档只需说明脚本用途、输入输出和调用时机。

## 图片资料

![[assets/skill-token-01.png]]
![[assets/skill-token-02.png]]
![[assets/skill-token-03.png]]
![[assets/skill-token-04.png]]
