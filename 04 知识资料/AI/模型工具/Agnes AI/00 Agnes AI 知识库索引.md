---
title: Agnes AI 免费模型与 Claude Code 接入知识库
source: https://mp.weixin.qq.com/s/xt4xak1oaC3sEK0QntTTAw
author: 小林coding
source_type: 微信文章
created: 2026-06-15
published: 2026-06-13
tags:
  - AI
  - Claude-Code
  - Agnes-AI
  - API
  - 模型工具
  - 多模态
status: 已整理
---

# Agnes AI 免费模型与 Claude Code 接入知识库

> 原文：[终于可以爽玩 Claude Code，免费 Token 随便烧！](https://mp.weixin.qq.com/s/xt4xak1oaC3sEK0QntTTAw)  
> 作者：小林coding  
> 整理时间：2026-06-15

## 一句话总结

这篇文章介绍了如何把 Agnes AI 的免费文本、图片、视频模型接入 Claude Code 工作流，并分享了作者对 `Agnes-2.0-Flash`、`Agnes-Image-2.1-Flash`、`Agnes-Video-2.0` 的实测体验。核心观点是：Agnes 不一定能替代 Claude Opus 做高复杂规划，但适合承担低成本、大量试错、执行型任务和多模态素材生成。

## 知识库目录

1. [[01 Agnes AI 核心概念与模型清单]]
2. [[02 Claude Code 接入 Agnes AI 操作手册]]
3. [[03 Agnes 文本模型实测与适用场景]]
4. [[04 Agnes 图像与视频模型实测]]
5. [[05 Agnes 使用策略、风险与工作流建议]]
6. [[06 原文解析要点]]

## 核心地图

```text
Claude Code
   ↓
CC Switch 本地路由
   ↓
Agnes API: https://apihub.agnes-ai.com/v1
   ↓
文本模型：agnes-2.0-flash
图片模型：Agnes-Image-2.1-Flash
视频模型：Agnes-Video-2.0
```

## 适合沉淀的问题

- 如何把非 Claude 官方模型接入 Claude Code？
- CC Switch 在 Claude Code 工作流里扮演什么角色？
- 免费模型适合承担哪些任务？不适合承担哪些任务？
- 文本模型、图像模型、视频模型分别适合哪些工作流？
- 如何设计“高质量模型规划 + 免费模型执行”的组合工作流？

## 相关链接

- Agnes AI 文档：<https://agnes-ai.com/doc/Agnes>
- Agnes API Platform：<https://platform.agnes-ai.com/>
- Agnes API 地址：<https://apihub.agnes-ai.com/v1>
- CC Switch：<https://github.com/farion1231/cc-switch/releases>

## 关联笔记

- [[04 知识资料/AI/agent/Claude Code 多 Agent 实现机制.md]]
- [[04 知识资料/AI/agent/Harness Engineering 在硅谷爆火.md]]
- [[05 网页与链接收藏/解析规则/知识库式解析规范.md]]

## 原文图片资产

> 以下图片已从原文下载到本知识库 `assets/` 目录，后续可在对应章节中复用。

- ![[assets/01-agnes-models-overview.png]]
- ![[assets/02-agnes-first-week-stats.png]]
- ![[assets/03-agnes-api-key-page.png]]
- ![[assets/04-cc-switch-claude-cli.png]]
- ![[assets/05-cc-switch-route-config.png]]
- ![[assets/06-cc-switch-provider-config.png]]
- ![[assets/07-cc-switch-model-list.png]]
- ![[assets/08-cc-switch-compatible-params.png]]
- ![[assets/09-claude-code-agnes-verify.png]]
- ![[assets/10-agnes-codebase-analysis-result.png]]
- ![[assets/11-agnes-module-doc-result.png]]
- ![[assets/12-worldcup-forum-demo.png]]
- ![[assets/13-agnes-microservice-architecture-image.png]]
- ![[assets/14-agnes-tencent-offer-image.png]]
- ![[assets/15-agnes-gaokao-gold-list-image.png]]
- ![[assets/16-agnes-final-docs-links.png]]
