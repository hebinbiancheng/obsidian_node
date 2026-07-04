---
title: TUI Agent 代码迭代日志
date: 2026-07-01
tags: [tui-agent, hebin3, 代码迭代, changelog]
category: 操作习惯
---

# TUI Agent 代码迭代日志

> 项目：`C:\Users\hebin\LLM\hebin3`
> 记录 TUI Agent 项目的关键代码变更、版本迭代和架构决策。

---

## 环境信息

| 项目 | 说明 |
|------|------|
| 项目路径 | `C:\Users\hebin\LLM\hebin3` |
| Python 版本 | 3.11.15（Hermes venv） |
| 框架 | Textual 8.x |
| 依赖 | textual, httpx, pydantic, pyyaml, rich |
| 测试 | 69 tests，全通过 |
| 配置 | `config.yaml` (WPS CodingPlan) |
| 架构 | 纯手写，无外部 Agent SDK 依赖 |

---

## 迭代记录

### v? — 界面美化 + Bug 修复 (2026-06-26)

**修复三个问题**：

#### 1. 输入框不显示/不能输入
- 去掉 `dock` CSS
- 在 `run_test` 中确认 Input 正常渲染并获得焦点
- 注意：终端尺寸太小可能导致 `1fr` 布局把 input 挤没

#### 2. 权限确认不触发（按 Y/N 没反应）
- 根因：deepseek-v4-pro 不主动调工具，直接文本回复
- 修复：加 system prompt 强制要求使用工具
- 修复 `perm-hint` CSS：`display:none` → `visibility:hidden`
- 修复 `get_api_key()` 兼容 SecretStr

#### 3. 界面美化
- 极简 Codex 风格
- 顶栏状态 + 全屏 chat + 底栏输入

---

### v? — TUI Agent 核心功能实现 (2026-06-17~26)

基于 `docs\从零实现一个TUI终端编码agent.md` 文档实现：

- Agent Loop（消息循环）
- 聊天日志组件（ChatLog）
- 输入框组件（Input）
- 权限确认组件
- LLM 集成（deepseek-v4-pro）
- 工具调用支持
- 状态管理

---

## 已知坑位 (Pitfalls)

1. **Textual DOM 生命周期**
   - `__init__` 中不能调用 `self.query_one()`
   - 必须在 `on_mount()` 中操作 widget
   - 跨异步边界用 `asyncio.Event`

2. **Python 版本**
   - 必须用 Python >=3.9（Textual 8.x 要求）
   - Miniconda Python 3.7.4 不兼容

3. **测试隔离**
   - 测试需要 `monkeypatch _find_project_config`
   - 否则测试会读取真实 `config.yaml`

4. **LLM 行为差异**
   - 真实 LLM 可能不主动调工具
   - system prompt 需要足够明确

---

## 常用命令

```bash
# 运行 TUI Agent
C:\Users\hebin\AppData\Local\hermes\hermes-agent\venv\Scripts\python.exe -m tui_agent

# 跑测试
cd /c/Users/hebin/LLM/hebin3 && python -m pytest tests/ -q

# 安装依赖
cd /c/Users/hebin/LLM/hebin3 && uv pip install -e ".[dev]"
```

---

## 待办

- [ ] 记录更多迭代版本的变更
- [ ] 补充架构图和模块说明
- [ ] 记录 API 接口变更
