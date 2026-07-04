---
title: Hermes Agent 操作本地浏览器（CDP 非无头模式）
date: 2026-07-02
tags: [hermes, browser, cdp, chrome, 浏览器自动化, 工具使用]
category: AI
source: https://mp.weixin.qq.com/s/8ff8jBZC9P3tJnNu1Cdjkw
---

# Hermes Agent 操作本地浏览器（非无头模式）

> 原文: [微信文章](https://mp.weixin.qq.com/s/8ff8jBZC9P3tJnNu1Cdjkw)

---

## 应用场景

目前绝大部分企业应用都采用 Browser/Server（浏览器/服务器）结构。这意味着 AI 可以和企业应用打通，比如：

- 填写表单
- 点击链接
- 获取结果
- 喂给 AI 进行处理

原本需要人来做的事情，AI 可以直接来做，人负责审查结果即可。随着 AI 越来越强，操作会越来越精确。

---

## 一、核心原理

Hermes Agent 支持直接通过 **Chrome DevTools Protocol（CDP）** 接管本地 Chrome，而不依赖云浏览器。

```
Chrome (--remote-debugging-port=9222) → WebSocket → Hermes Agent → 操控 Chrome
```

本质：用 `--remote-debugging-port=9222` 启动 Chrome → Hermes 通过 WebSocket 连接该端口 → 直接操控你的 Chrome 实例（**复用你的 Cookie/登录态**）。

---

## 二、启动 Chrome（远程调试模式）🔑

> ⚠️ 必须单独起一个带调试端口的 Chrome 实例，指定独立用户目录，避免和日常 Chrome 冲突。

### Windows（CMD 执行）

```cmd
"C:\Program Files\Google\Chrome\Application\chrome.exe" ^
  --remote-debugging-port=9222 ^
  --user-data-dir="%USERPROFILE%\.hermes\chrome-debug" ^
  --no-first-run ^
  --no-default-browser-check
```

### macOS（终端执行）

```bash
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
  --remote-debugging-port=9222 \
  --user-data-dir="$HOME/.hermes/chrome-debug" \
  --no-first-run \
  --no-default-browser-check
```

### Linux（终端执行）

```bash
google-chrome \
  --remote-debugging-port=9222 \
  --user-data-dir="$HOME/.hermes/chrome-debug" \
  --no-first-run \
  --no-default-browser-check
```

### 参数说明

| 参数 | 作用 |
|------|------|
| `--remote-debugging-port=9222` | 开启 CDP 调试端口（默认 9222） |
| `--user-data-dir` | 独立配置目录，不污染日常 Chrome 配置 |
| `--no-first-run` | 跳过首次启动弹窗 |

### 验证是否成功

浏览器打开 `http://localhost:9222/json`，能看到 JSON 列表即正常。

---

## 三、Hermes Agent 连接本地 Chrome

### 1. 启动 Hermes CLI

正常登录 Hermes Agent，进入交互界面。

### 2. 执行连接命令

```bash
# 默认连接本地 9222 端口（最常用）
/browser connect

# 指定 CDP 地址
/browser connect ws://localhost:9222

# 查看连接状态
/browser status
```

成功提示：`Connected to local Chrome via CDP`。

### 失败常见原因

| 问题 | 解决方法 |
|------|----------|
| 端口 9222 被占用 | 换端口（如 9223）并同步命令 |
| 已开日常 Chrome 但未指定 `user-data-dir` | 关闭所有 Chrome，重试第一步命令 |

---

## 四、正常使用浏览器工具

连接后，Hermes 所有浏览器能力都会走你的本地 Chrome：

- ✅ 自动打开标签、导航、点击、输入、截图
- ✅ 直接复用你 Chrome 的登录态/Cookie（不用重新登录）
- ✅ 你能**实时看到浏览器操作过程**

常用命令：

```bash
/browser status       # 查看连接状态
/browser disconnect   # 断开本地 Chrome，切回云/无头模式
```

---

## 五、常见问题

| 问题 | 排查方法 |
|------|----------|
| 连接超时 | 确认 Chrome 是用命令启动的（不是普通双击）；访问 `http://localhost:9222/json` 测试 |
| 和日常 Chrome 冲突 | 必须用 `--user-data-dir` 单独启动一个实例 |
| Windows 路径错误 | Chrome 可能在 `C:\Program Files (x86)\Google\Chrome\Application\chrome.exe` |

---

## 六、一键总结

1. 用带 `--remote-debugging-port=9222` 和独立 `user-data-dir` 的命令启动 Chrome
2. Hermes CLI 执行 `/browser connect`
3. 直接用本地 Chrome 自动浏览 — **零云成本、复用会话、可视操作**

---

## 相关笔记

- [[Hermes Agent 操作习惯]]
- [[Hermes Gateway 运维笔记]]
