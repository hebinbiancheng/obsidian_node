---
title: Claude Code 接入 Agnes AI 操作手册
source: https://mp.weixin.qq.com/s/xt4xak1oaC3sEK0QntTTAw
created: 2026-06-15
tags:
  - Claude-Code
  - Agnes-AI
  - CC-Switch
  - API
status: 已整理
---

# Claude Code 接入 Agnes AI 操作手册

## 1. 接入链路

原文给出的接入链路是：

```text
Claude Code -> CC Switch -> Agnes API
```

需要准备：

- Agnes API Key；
- CC Switch；
- Claude Code。

## 2. 关键地址与模型名

| 项 | 值 |
|---|---|
| Agnes API Platform | <https://platform.agnes-ai.com/> |
| API 请求地址 | `https://apihub.agnes-ai.com/v1` |
| 文本模型名 | `agnes-2.0-flash` |
| CC Switch | <https://github.com/farion1231/cc-switch/releases> |

## 3. 操作步骤

### 第一步：获取 Agnes API Key

1. 访问 Agnes API Platform：<https://platform.agnes-ai.com/>；
2. 注册并登录；
3. 进入 API Key 页面；
4. 创建并复制新的 Key。

### 第二步：安装 CC Switch

1. 打开 CC Switch releases 页面：<https://github.com/farion1231/cc-switch/releases>；
2. 下载并安装；
3. 打开 CC Switch；
4. 顶部选择 `claude-cli`。

### 第三步：开启 CC Switch 路由

1. 点击 CC Switch 左上角设置；
2. 进入路由配置；
3. 选择本地路由；
4. 找到 Claude 相关路由并启用。

启用后，请求链路变为：

```text
Claude Code 请求
  -> CC Switch
  -> Agnes API
```

### 第四步：添加 Agnes 供应商

在 CC Switch 的 Claude 页面添加供应商：

| 配置项 | 值 |
|---|---|
| 供应商类型 | `claude` |
| 配置方式 | 自定义配置 |
| API Key | 粘贴 Agnes API Key |
| 请求地址 | `https://apihub.agnes-ai.com/v1` |
| API 格式 | `openai chat completions` |
| 模型 | `agnes-2.0-flash` |

### 第五步：获取模型列表

填写完成后，先点击“获取模型列表”。

如果能拉取到模型列表，说明：

- API Key 可用；
- API 地址正确；
- CC Switch 能访问 Agnes API。

然后选择 `agnes-2.0-flash` 并按 CC Switch 提示做模型映射。

### 第六步：添加兼容参数

Claude Code 真实运行时可能携带某些 Agnes 不认识的字段，例如：

- `thinking`；
- `context_management`。

原文建议在 CC Switch 中加入类似兼容配置：

```json
{
  "allowed_openai_params": [
    "thinking",
    "context_management"
  ],
  "litellm_settings": {
    "drop_params": true
  }
}
```

作用：

- 允许特定 OpenAI 参数；
- 对不兼容参数执行 drop，避免请求失败。

### 第七步：在 Claude Code 中验证

打开 Claude Code，发送简单问题：

```text
你好
```

如果能正常回复，说明基础链路打通。

> 原文提醒：界面上可能仍显示 Opus 等模型名，但底层请求已经通过 CC Switch 转到 `agnes-2.0-flash`。

## 4. 接入检查清单

- [ ] 已获取 Agnes API Key；
- [ ] 已安装 CC Switch；
- [ ] CC Switch 选择了 `claude-cli`；
- [ ] Claude 路由已启用；
- [ ] Agnes 供应商已添加；
- [ ] API 地址为 `https://apihub.agnes-ai.com/v1`；
- [ ] API 格式为 `openai chat completions`；
- [ ] 模型为 `agnes-2.0-flash`；
- [ ] 能成功获取模型列表；
- [ ] 已处理不兼容参数；
- [ ] Claude Code 中简单问答可正常返回。

## 5. 常见问题

### 1. 获取不到模型列表

可能原因：

- API Key 错误；
- API 地址错误；
- 网络无法访问 Agnes API；
- CC Switch 路由未正确开启。

### 2. 请求因参数不兼容失败

可能原因：Claude Code 携带了 Agnes 不支持的字段。

处理方式：参考原文添加 `drop_params` 配置。

### 3. 界面显示的模型名和实际模型不一致

原文提到，界面可能仍显示 Claude / Opus，但底层已经经由 CC Switch 路由到 Agnes 模型。建议以 CC Switch 日志或请求记录为准。

## 原文操作截图

### Agnes API Key 页面

![[assets/03-agnes-api-key-page.png]]

### CC Switch 选择 Claude CLI

![[assets/04-cc-switch-claude-cli.png]]

### CC Switch 路由配置

![[assets/05-cc-switch-route-config.png]]

### Agnes 供应商配置

![[assets/06-cc-switch-provider-config.png]]

### 获取模型列表

![[assets/07-cc-switch-model-list.png]]

### 兼容参数配置

![[assets/08-cc-switch-compatible-params.png]]

### Claude Code 验证

![[assets/09-claude-code-agnes-verify.png]]
