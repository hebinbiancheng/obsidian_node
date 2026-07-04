---
title: Bug 调试与问题解决经验
date: 2026-07-01
tags: [debug, bug, 问题解决, workflow, 经验]
category: 操作习惯
---

# Bug 调试与问题解决经验

> 记录实际开发中遇到的典型问题和解决过程，供后续参考。

---

## 1. 系统性调试流程（systematic-debugging）

加载 `systematic-debugging` skill 后遵循四阶段流程：

```
UNDERSTAND → ISOLATE → FIX → VERIFY
```

### Phase 1: Understand（理解问题）
- 收集错误信息（traceback、日志、截图）
- 复现问题
- 确认影响范围

### Phase 2: Isolate（定位根因）
- 二分法缩小范围
- 添加日志/断点
- 检查最近的改动（git diff）

### Phase 3: Fix（修复）
- 修复根因而非表象
- 保持最小改动原则
- 考虑边界情况

### Phase 4: Verify（验证）
- 跑所有测试
- 手动复测原始问题
- 确认没有引入新问题

---

## 2. TUI Agent 典型问题案例

### 2.1 输入框不显示/不能输入

**现象**：TUI 运行后输入框区域为空，无法输入文字

**排查过程**：
1. 用 `run_test` 确认 Input 组件能正常渲染并获得焦点 → 组件本身没问题
2. 检查 CSS → 发现 `dock` 样式导致布局异常

**根因**：CSS 中的 `dock` 属性与 Textual 布局冲突

**修复方法**：
- 去掉 `dock` CSS
- 如果终端尺寸太小，Textual 的 `1fr` 可能把 input 挤没 → 拉大终端窗口

**经验教训**：
> Textual 的 CSS 布局有时和直觉不同。组件不显示时先检查 CSS 而非组件代码。

---

### 2.2 权限确认按 Y/N 没反应

**现象**：Agent 请求用户确认时，显示提示但按 Y/N 没有任何效果

**排查过程**：
1. 检查事件处理 → 按键事件正常绑定
2. 检查 LLM 行为 → 发现 **deepseek-v4-pro 不主动调用工具**，而是直接文本回复

**根因**：真实 LLM 不主动使用工具，需要在 system prompt 中强制要求

**修复方法**：
- 在 system prompt 中添加明确的工具使用要求
- 修复 `perm-hint` CSS：`display:none` → `visibility:hidden`（保持占位）
- 修复 `get_api_key()` 对字符串和 SecretStr 的兼容处理

**经验教训**：
> 真实 LLM 的行为和模拟/测试环境完全不同。Agent 的 system prompt 需要足够强硬地引导 LLM 使用工具。

---

### 2.3 Textual App `__init__` 中调用 `query_one()` 失败

**现象**：在 `App.__init__` 中调用 `self.query_one()` 报错，widget 查找不到

**根因**：Textual 的 DOM 在 `__init__` 阶段尚未构建完成，`compose()` 返回的 widget 还未挂载

**修复方法**：
- 所有 widget 访问操作移到 `on_mount()` 中执行
- 如需跨异步边界传递回调，使用 `asyncio.Event`

**经验教训**：
> Textual 的生命周期：`__init__` → `compose()` → `on_mount()`。DOM 访问只能在 `on_mount()` 及之后。

---

## 3. 通用调试技巧

### 3.1 快速定位改动引入的 Bug
```bash
# 查看最近的改动
git diff HEAD~1
# 二分法定位
git bisect start
git bisect bad HEAD
git bisect good <last_known_good_commit>
```

### 3.2 测试隔离
- 使用 `monkeypatch` 隔离外部依赖
- 项目测试需要 `monkeypatch _find_project_config` 避免读取真实 config

### 3.3 Python 版本问题
- 先确认 Python 版本：`python --version`
- Conda 的 Python 3.7.4 太旧，不支持 Textual >=0.44
- 使用 Hermes venv 的 Python 3.11.15

---

## 4. 常见 Error Pattern 速查

| 错误信息 | 可能原因 | 解决方法 |
|----------|----------|----------|
| `ModuleNotFoundError: No module named 'textual'` | 用的 Conda Python 3.7 | 切换到 Hermes venv Python |
| `AttributeError: 'App' object has no attribute 'query_one'` | 在 `__init__` 中访问 DOM | 移到 `on_mount()` |
| `pip install` 找不到包 | pip 指向 Conda 的旧环境 | 使用 `uv pip install` |
| API key 验证失败 | SecretStr vs str 类型不兼容 | 兼容两种类型 |
| GitHub API rate limit | 短时间内请求过多 | 等几分钟或使用 token |
| `getaddrinfo failed` (ilinkai.weixin.qq.com) | 锁屏后网卡进入省电模式，DNS 失败 | 关闭网卡省电 + 禁用系统睡眠 → [[Hermes Gateway 运维笔记#Q 锁屏后微信连不上]] |

---

## 5. Hermes Gateway 锁屏断连修复（2026-07-02）

**现象**：锁屏后微信连不上 Hermes，回来后 Gateway 日志出现：
```
ERROR gateway.platforms.weixin: [Weixin] poll error: Cannot connect to host ilinkai.weixin.qq.com:443 ssl:default [getaddrinfo failed]
```

**根因**：Windows 锁屏触发网络节能策略 → DNS 解析失败 → Gateway 微信连接断开。

**修复**：
1. `powercfg /change standby-timeout-ac 0` — 关闭插电睡眠
2. `powercfg /change hibernate-timeout-ac 0` — 关闭插电休眠  
3. 禁用网卡省电模式（设备管理器 / PowerShell 管理员脚本）
4. 关闭 USB 选择性暂停

**验证结果**：
- STANDBYIDLE AC = `0x00000000` ✅ 永不睡眠
- HIBERNATEIDLE AC = `0x00000000` ✅ 永不休眠
- DNS `ilinkai.weixin.qq.com` 解析正常 ✅
- Gateway pythonw.exe PID 32416 运行中 ✅
