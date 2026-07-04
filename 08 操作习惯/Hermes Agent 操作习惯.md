---
title: Hermes Agent 操作习惯
date: 2026-07-01
tags: [hermes, 操作习惯, 配置, workflow]
category: 操作习惯
---

# Hermes Agent 操作习惯

> 本文档记录与 Hermes Agent 交互时的操作偏好、环境约定、工作习惯和问题解决经验。
> Hermes 在执行任务前应参考本文档，确保行为一致。

---

## 1. 环境概览

| 项目 | 说明 |
|------|------|
| 系统 | Windows 10 |
| 主目录 | `C:\Users\hebin` |
| 默认 Shell | Git Bash (MSYS) — 使用 POSIX 语法 |
| 默认 Python | Miniconda Python 3.7.4（太旧），实际开发用 Hermes venv Python 3.11.15 |
| Hermes Venv | `C:\Users\hebin\AppData\Local\hermes\hermes-agent\venv\Scripts\python.exe` |
| 包管理器 | `uv`（推荐），`pip` 可能指向 Conda 的老版本 |
| Obsidian 知识库 | `C:\Users\hebin\Obsidian\obsidian_note` |
| 笔记模式 | 用户偏好在 Obsidian 中以笔记形式记录和沉淀 |

---

## 2. 核心项目

### 2.1 TUI Agent (`C:\Users\hebin\LLM\hebin3`)

- Python 3.11，Textual 8.x TUI agent
- 69 tests 全通过
- 运行命令：`C:\Users\hebin\AppData\Local\hermes\hermes-agent\venv\Scripts\python.exe -m tui_agent`
- 配置：`config.yaml` (WPS CodingPlan)
- 测试需 `monkeypatch _find_project_config` 隔离
- 架构：所有核心逻辑手写，不依赖外部 Agent SDK
- **重要坑**：Textual App 的 `__init__` 中不能调用 `self.query_one()`，DOM 未就绪。必须在 `on_mount()` 中操作 widget。
- 详见：[[TUI Agent 代码迭代日志]]

### 2.2 WeChat Gateway

- WeChat ID: `o9cq809NURAo8qL79_nXFhp0SBz8`
- Gateway 安装为 Windows 服务（通过 Startup 文件夹自启）
- 启动路径：`C:\Users\hebin\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup\Hermes_Gateway.vbs`
- `GATEWAY_ALLOW_ALL_USERS` 已启用
- **重启 Gateway 必须从外部终端操作**（不能在 Hermes 会话内）
- Gateway 独立于 TUI 客户端，关掉 TUI 不影响微信使用
- 锁屏不影响 Gateway，但休眠/睡眠会中断
- 详见：[[Hermes Gateway 运维笔记]]

---

## 3. 命令约定

| 场景 | 使用 | 避免 |
|------|------|------|
| Python | Hermes venv 的 Python (3.11.15) | Conda Python 3.7.4 |
| 安装包 | `uv pip install` | `pip install`（可能指向 Conda） |
| 文件读写 | `read_file` / `write_file` / `patch` | `cat` / `sed` / `echo` |
| 文件搜索 | `search_files` | `grep` / `find` / `rg` |
| 目录列表 | `search_files(target='files')` | `ls` |
| Shell | Git Bash (POSIX 语法) | PowerShell / cmd |
| 路径 | `/c/Users/hebin/...` 或 `C:\Users\hebin\...` | |

---

## 4. TUI 设计偏好

- 风格：Cursor/Codex 风格终端界面
- 布局：极简 — 无 Header，单一顶部状态栏，全屏聊天日志 + 底部输入框
- 权限提示：底部窄条（非模态弹窗）
- 配色：Tokyo Night 色系

---

## 5. 工作流程偏好

- **先加载 skill，再动手** — 有 skill 先用 skill，不重复造轮子
- **遇到问题先查 session_search** — 看之前有没有处理过类似问题
- **完成任务后更新 todo** — 保持 todo 状态同步
- **复杂任务结束后主动询问是否保存为 skill** — 沉淀经验
- **修改代码后必须跑测试** — 确认没破坏已有功能
- **pitfall 必须写入文档** — 踩过的坑不要再踩

---

## 6. 解决问题的通用流程

1. **复现问题** — 理解问题是什么，收集错误信息
2. **查历史会话** — `session_search` 搜索类似问题
3. **加载相关 skill** — 用 skill 的既定流程
4. **系统性调试** — 加载 `systematic-debugging` skill（四阶段：Understand → Isolate → Fix → Verify）
5. **根因分析** — 不只是修表象，要找到根本原因
6. **测试验证** — 修完后跑所有测试
7. **记录沉淀** — 写入操作习惯笔记

详见：[[Bug 调试与问题解决经验]]

---

## 7. Knowledge Sources（优先级从高到低）

1. 用户明确提供的 URL / 文件 / 账号 → 第一时间查看
2. Hermes 文档：`https://hermes-agent.nousresearch.com/docs`
3. `hermes-agent` skill — 配置/CLI/工具
4. `session_search` — 历史会话
5. 本项目文件夹下的笔记

**原则**：有直接来源时优先用直接来源，不要用 session_search 代替。

---

## 8. 微信文章解析 & 知识库整理规则 🔑

### 8.1 解析流程

1. `curl` 抓取微信文章 HTML
2. 提取 `og:title`（标题）和 `rich_media_content`（正文）
   - **同时提取正文中的图片** — 识别 `<img>` 标签，下载图片到知识库 `assets/` 目录，在 Markdown 中对应位置用 `![alt](assets/xxx.png)` 引用
   - 图片文件命名规则：`{文章主题}-{序号}.{扩展名}`
3. 去除 HTML 标签，格式化 Markdown，图片保留在原文对应位置
4. **验证内容正确性** — 检查是否有明显错误、格式问题、乱码。如有问题提出，无问题则继续
5. 合并或新建保存到 Obsidian 知识库

> ⚠️ 解析后必须先验证，确认内容正确无误再入库，不能直接写入。
>
> 🖼️ 图片必须保留在原位 — 不要丢弃或放到文末，确保阅读体验和原文一致。
>
> ⚠️ **微信图片时效性**：`mmbiz.qpic.cn` 的图片 URL 有时效性，延迟太久可能过期。务必在解析时立即下载，否则链接失效后无法获取。

### 8.2 合并整理原则

> ⚠️ **核心规则：相似主题合并，不要每个链接建一篇独立文档。**

| 情况 | 处理方式 |
|------|----------|
| 新主题（知识库无相关内容） | 新建文档 |
| 同类主题已有文档 | **合并到已有文档**，追加内容或作为新章节 |
| 同一系列文章 | 合并为一个文档，按章节组织 |
| 纯命令速查/参考 | 追加到已有速查文档 |

### 8.3 合并示例

| 文章 | 合并到 |
|------|--------|
| K8s Pod 调度 | `K8s Pod 调度流程面试题.md`（新建，之前无同类） |
| K8s PVC/PV 绑定 | `K8s PVC 绑定 PV 全过程.md`（新建） |
| K8s etcd 深度解析 | `etcd 如何保存 Kubernetes 状态.md`（新建） |
| K8s kubectl 命令速查 | `kubectl 常用管理命令速查.md`（新建） |
| K8s Deployment 实战 | `K8s Deployment 实战指南.md`（新建） |
| 后续 K8s 相关文章 | **合并到已有 K8s 文档**，而非新建 |
| Hermes 浏览器操作 | `Hermes Agent 操作本地浏览器.md`（新建） |
| 后续 Hermes 使用技巧 | **合并到本文档或已有 Hermes 笔记** |

### 8.4 合并判断流程

```
收到解析请求
  ↓
提取标题和主题
  ↓
搜索已有知识库是否有同类文档
  ├── 有 → 合并追加（作为新章节或补充）
  └── 无 → 新建文档
```

---

## 9. 相关笔记

- [[Bug 调试与问题解决经验]] — 常见 bug 及解决流程
- [[TUI Agent 代码迭代日志]] — 版本变更记录
- [[Hermes Gateway 运维笔记]] — Gateway 安装、配置、故障排查
- [[常用工具与命令速查]] — 常用命令和工具路径
