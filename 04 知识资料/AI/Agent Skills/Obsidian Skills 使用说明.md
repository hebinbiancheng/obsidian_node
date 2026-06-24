---
title: Obsidian Skills 使用说明
created: 2026-06-24
tags:
  - Agent-Skills
  - Obsidian
  - Codex
  - 知识库
source: https://github.com/kepano/obsidian-skills
status: 启用
---

# Obsidian Skills 使用说明

## 安装位置

已安装 `kepano/obsidian-skills` 中的 5 个 skills：

- `defuddle`
- `json-canvas`
- `obsidian-bases`
- `obsidian-cli`
- `obsidian-markdown`

安装位置：

- Codex 全局：`C:\Users\hebin\.codex\skills\`
- 当前 vault 项目级 Claude Skills：`.claude/skills\`

Codex 需要重启后才会自动识别新安装的 skills。

## 每个 skill 的用途

| Skill | 适用场景 |
| --- | --- |
| `obsidian-markdown` | 创建或修改 Obsidian 笔记，使用 properties、wikilinks、embeds、callouts、tags 等 Obsidian 原生语法 |
| `defuddle` | 解析普通网页正文，去掉导航、广告和页面杂质，适合新链接解析前的正文提取 |
| `json-canvas` | 创建或修改 `.canvas` 画布、知识地图、流程图、节点关系图 |
| `obsidian-bases` | 创建或修改 Obsidian Bases `.base` 文件，用于结构化视图、筛选、公式和汇总 |
| `obsidian-cli` | 使用 Obsidian CLI 做 vault、插件、主题等命令式操作 |

## 与当前知识库规则的关系

这套 skills 是工具能力，不替代本 vault 的内容规则。

当前 vault 的长期默认规则仍以 [[知识库式解析规范]] 为准：

- 新解析长文、教程、项目文档时，优先进入 `04 知识资料/` 下的主题知识库。
- 需要保留原文结构、核心概念、流程、实践、问题、面试复盘和后续行动项。
- 关键图片下载到主题目录下的 `assets/`，并用 `![[assets/文件名.png]]` 嵌入。
- 同主题多篇来源可以合并成一个主题知识库。

`obsidian-skills` 用来提升执行质量：

- 用 `defuddle` 提取网页正文。
- 用 `obsidian-markdown` 生成符合 Obsidian 语法的笔记。
- 用 `json-canvas` 在需要时补充知识地图。
- 用 `obsidian-bases` 在需要筛选或汇总时建立结构化视图。

## 如何使用

以后可以直接提出目标，不需要手动点选 skill。例如：

```text
解析这个链接，按知识库式解析规范沉淀到 vault，并保留关键图片。
```

```text
把这个主题下已有笔记整理成一个 Canvas 知识地图。
```

```text
检查这篇笔记的 Obsidian 语法、双链、frontmatter 和图片嵌入是否规范。
```

```text
为 AI/Agent Skills 目录创建一个 Bases 汇总视图。
```

在 Codex 重启后，相关任务会自动触发这些 skills。项目级 `.claude/skills` 也已经准备好，当前 vault 作为 Claude Code 工作目录时可复用同一套 skills。

## 能否优化已有内容

可以。适合优化的内容包括：

- 缺少 YAML frontmatter、tags、aliases、status 的旧笔记。
- 普通 Markdown 链接可以改成 Obsidian wikilinks 的内部引用。
- 图片外链或散落附件可以整理到主题 `assets/`。
- 单篇大笔记可以拆成主题知识库结构。
- 已有 `.canvas` 可以补齐节点、连接、分组和布局。
- 同主题笔记可以补索引页、学习路径和反向链接。

建议按主题分批处理，避免一次性重写整个 vault。

## 新解析内容是否会用到

会。后续解析网页、文档或本地材料时，应同时使用：

- [[知识库式解析规范]]：决定内容放在哪里、拆成什么结构、如何建立双链和索引。
- `obsidian-skills`：保证 Obsidian 语法、网页正文提取、Canvas/Bases 文件格式更规范。

默认落地策略：

1. 先判断主题归属。
2. 抽取正文和关键图片。
3. 在 `04 知识资料/` 下创建或更新主题知识库。
4. 写入 frontmatter、tags、wikilinks、图片嵌入和来源索引。
5. 更新相关索引页。
