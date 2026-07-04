---
title: Hermes Gateway 运维笔记
date: 2026-07-01
tags: [hermes, gateway, 运维, wechat, 微信]
category: 操作习惯
---

# Hermes Gateway 运维笔记

> 记录 Hermes Gateway（微信接入）的安装、配置和运维经验。

---

## 基本信息

| 项目 | 说明 |
|------|------|
| WeChat ID | `o9cq809NURAo8qL79_nXFhp0SBz8` |
| 安装方式 | Windows 服务（Startup 文件夹自启） |
| 启动路径 | `C:\Users\hebin\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup\Hermes_Gateway.vbs` |
| 配置 | `GATEWAY_ALLOW_ALL_USERS` 已启用 |
| 审批模式 | `approvals.mode: manual`（危险命令需确认） |

---

## 安装过程

```bash
# 初始化 Gateway 配置
hermes gateway setup

# 安装为 Windows 自启服务
hermes gateway install
```

安装支持的消息平台超过 25 个，当前仅配置了 WeChat。

---

## 运维要点

### 重启 Gateway

> ⚠️ **重要**：Gateway 重启**不能从 Hermes 会话内部执行**。
> 当 Hermes 会话运行在 gateway 进程中时，`hermes gateway restart` 会被阻止。

**正确做法**：
1. 打开一个独立的终端窗口（cmd / PowerShell / 新标签页）
2. 运行 `hermes gateway restart`
3. 可在 Hermes 会话中验证：`hermes gateway status`

### 可用性矩阵

| 场景 | 微信还能用吗？ |
|------|:--:|
|| 关掉 Hermes TUI | ✅ 可以 |
|| 退出 Hermes 桌面客户端 | ✅ 可以 |
|| 锁屏 | ✅ 可以（需关闭网卡省电） |
|| 重启电脑（没登录） | ❌ 不行 |
|| 电脑休眠/睡眠 | ❌ 不行 |

> Gateway 依赖 Startup 文件夹启动，必须登录 Windows 后才自动运行。
>
> ⚠️ **锁屏断连问题**（2026-07-02 修复）：锁屏后 Windows 可能触发网络节能策略，导致 DNS 解析失败（`getaddrinfo failed`），微信 Gateway 与 `ilinkai.weixin.qq.com` 断开。解决方案见下方「网络优化」章节。

---

## 功能说明

微信上的 Hermes 和 TUI 里的 Hermes 是**同一个 Agent 核心**，拥有完全相同的工具能力：
- 读写文件
- 执行命令
- Git 操作
- 浏览器操作
- 搜索、数据分析等

### 危险命令确认
- 默认 `approvals.mode: manual` — 执行 `rm -rf`、`git reset --hard` 等危险操作时需要用户确认
- 可改为 `smart` 模式让 AI 自动判断

### 会话隔离
- TUI 和微信是**两个独立的会话通道**，互不相通
- 微信对话不会同步到 TUI 界面

---

## 问题排查

### Q: 微信发了消息没反应？
1. 确认电脑已登录 Windows
2. 确认 Gateway 服务正在运行：`hermes gateway status`
3. 检查网络连接

### Q: 锁屏后微信连不上？🔑
**根因**：锁屏后 Windows 网络节能策略触发，DNS 解析失败，Gateway 与 `ilinkai.weixin.qq.com` 断开。

**诊断方法**：
```bash
# 查看 Gateway 日志，找 getaddrinfo failed 错误
grep "getaddrinfo failed" C:/Users/hebin/AppData/Local/hermes/logs/gateway.log

# 测试 DNS 是否正常
nslookup ilinkai.weixin.qq.com

# 检查 Gateway 进程是否存活
tasklist | findstr pythonw
```

**修复方法**（2026-07-02 已执行）：

1. **关闭系统睡眠/休眠**（无需管理员）：
   ```bash
   powercfg /change standby-timeout-ac 0
   powercfg /change hibernate-timeout-ac 0
   ```

2. **禁用网卡省电模式**（需管理员）：
   - 设备管理器 → 网络适配器 → 属性 → 电源管理 → 取消「允许计算机关闭此设备以节约电源」
   - 或运行 `C:\Users\hebin\network_optimize.ps1`（管理员 PowerShell）

3. **关闭 USB 选择性暂停**（需管理员，通过上述脚本自动完成）

**验证**：
```bash
# 确认睡眠已关闭（AC 电源设置应为 0x00000000）
powercfg /query SCHEME_CURRENT SUB_SLEEP STANDBYIDLE
powercfg /query SCHEME_CURRENT SUB_SLEEP HIBERNATEIDLE
```

### Q: 需要改配置？
```bash
hermes config set gateway.xxx value
```

---

## 相关笔记

- [[Hermes Agent 操作习惯]]
- [[常用工具与命令速查]]
