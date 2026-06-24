---
title: 工具调用 MCP Skill 与协议
type: knowledge
status: evergreen
source_type: 本地剪藏与链接解析
created: 2026-06-24
updated: 2026-06-24
tags:
  - AI
  - Function Calling
  - MCP
  - Skill
  - A2A
source:
  - "https://xiaolinnote.com/ai/tools/tools_info.html"
  - "https://xiaolinnote.com/ai/tools/1_function_calling.html"
  - "https://xiaolinnote.com/ai/tools/2_llm_tool_learning.html"
  - "https://xiaolinnote.com/ai/tools/3_fc_training.html"
  - "https://xiaolinnote.com/ai/tools/4_what_is_mcp.html"
  - "https://xiaolinnote.com/ai/tools/5_mcp_components.html"
  - "https://xiaolinnote.com/ai/tools/6_mcp_vs_fc.html"
  - "https://xiaolinnote.com/ai/tools/7_fc_vs_mcp_usage.html"
  - "https://xiaolinnote.com/ai/tools/8_reasoning_no_mcp.html"
  - "https://xiaolinnote.com/ai/tools/9_skill.html"
  - "https://xiaolinnote.com/ai/tools/10_mcp_vs_skill.html"
  - "https://xiaolinnote.com/ai/tools/11_fc_skill_mcp.html"
  - "https://xiaolinnote.com/ai/tools/12_a2a_protocol.html"
  - "https://xiaolinnote.com/ai/tools/13_mcp_transport.html"
  - "https://xiaolinnote.com/ai/tools/14_sse_vs_websocket.html"
  - "https://xiaolinnote.com/ai/tools/15_webrtc_vs_ws.html"
  - "https://xiaolinnote.com/ai/tools/16_llm_gateway.html"
aliases: []
---

# 工具调用 MCP Skill 与协议

## 一句话结论

工具调用体系可以按三层理解：Function Calling 是模型表达调用意图的格式，MCP 是连接工具和上下文的协议，Skill 是把任务流程和知识封装成可复用能力包。

## Function Calling

- 模型根据工具 schema 生成结构化调用参数。
- 应用层负责校验、执行工具、返回结果，再让模型继续生成回答。
- 关键风险是参数错误、越权调用、重复调用和不可观测。

## MCP

- MCP 让模型客户端以统一方式连接外部工具、资源和提示。
- 它适合工具生态复杂、需要跨客户端复用能力的场景。
- 与 Function Calling 相比，MCP 更像工具箱和上下文协议，而不是单次调用格式。

## Skill

- Skill 把稳定任务流程、参考资料、脚本和模板打包。
- 与 MCP 的关系：MCP 提供外部能力连接，Skill 提供完成任务的方法论和资源组织。
- 好 Skill 要控制上下文成本，采用元数据、SKILL.md、references/scripts 的渐进式加载。

## A2A 与通信协议

- A2A 面向 Agent 之间的协作与任务交接。
- SSE 适合服务端单向流式输出，WebSocket 适合双向实时通信，WebRTC 更适合低延迟音视频和点对点场景。
- LLM 网关负责统一模型路由、鉴权、限流、审计、降级和成本治理。

## 来源链接

- [LLM工具调用面试题介绍](https://xiaolinnote.com/ai/tools/tools_info.html)：围绕 Function Calling、MCP、Skill、A2A、SSE/WebSocket/WebRTC 和 LLM 网关等工具调用工程问题展开。
- [1. 什么是 Function Calling ？原理是什么？](https://xiaolinnote.com/ai/tools/1_function_calling.html)：围绕 Function Calling、MCP、Skill、A2A、SSE/WebSocket/WebRTC 和 LLM 网关等工具调用工程问题展开。
- [2. LLM 是如何学会调用外部工具的？](https://xiaolinnote.com/ai/tools/2_llm_tool_learning.html)：围绕 Function Calling、MCP、Skill、A2A、SSE/WebSocket/WebRTC 和 LLM 网关等工具调用工程问题展开。
- [3. 大模型的 Function Call 能力是怎么训练出来的？](https://xiaolinnote.com/ai/tools/3_fc_training.html)：围绕 Function Calling、MCP、Skill、A2A、SSE/WebSocket/WebRTC 和 LLM 网关等工具调用工程问题展开。
- [4. 什么是 MCP（模型上下文协议）？讲讲它的核心内容？](https://xiaolinnote.com/ai/tools/4_what_is_mcp.html)：围绕 Function Calling、MCP、Skill、A2A、SSE/WebSocket/WebRTC 和 LLM 网关等工具调用工程问题展开。
- [5. MCP 由哪几部分组成？](https://xiaolinnote.com/ai/tools/5_mcp_components.html)：围绕 Function Calling、MCP、Skill、A2A、SSE/WebSocket/WebRTC 和 LLM 网关等工具调用工程问题展开。
- [6. MCP 和 Function Calling 有什么区别？有没有实际跑过 MCP？](https://xiaolinnote.com/ai/tools/6_mcp_vs_fc.html)：围绕 Function Calling、MCP、Skill、A2A、SSE/WebSocket/WebRTC 和 LLM 网关等工具调用工程问题展开。
- [7. Function Calling 也属于工具调用，请问什么场景下使用 Function Calling，什么场景下使用 MCP？](https://xiaolinnote.com/ai/tools/7_fc_vs_mcp_usage.html)：围绕 Function Calling、MCP、Skill、A2A、SSE/WebSocket/WebRTC 和 LLM 网关等工具调用工程问题展开。
- [8. 为什么有些特定的推理模型不支持 MCP 协议？](https://xiaolinnote.com/ai/tools/8_reasoning_no_mcp.html)：围绕 Function Calling、MCP、Skill、A2A、SSE/WebSocket/WebRTC 和 LLM 网关等工具调用工程问题展开。
- [9. Skill 是什么？](https://xiaolinnote.com/ai/tools/9_skill.html)：围绕 Function Calling、MCP、Skill、A2A、SSE/WebSocket/WebRTC 和 LLM 网关等工具调用工程问题展开。
- [10. MCP 和 Agent Skill 的区别是什么？](https://xiaolinnote.com/ai/tools/10_mcp_vs_skill.html)：围绕 Function Calling、MCP、Skill、A2A、SSE/WebSocket/WebRTC 和 LLM 网关等工具调用工程问题展开。
- [11. Function Calling、Skill、MCP 这三个有什么区别？](https://xiaolinnote.com/ai/tools/11_fc_skill_mcp.html)：围绕 Function Calling、MCP、Skill、A2A、SSE/WebSocket/WebRTC 和 LLM 网关等工具调用工程问题展开。
- [12. 什么是 A2A 协议？它和 MCP 协议的区别是什么？](https://xiaolinnote.com/ai/tools/12_a2a_protocol.html)：围绕 Function Calling、MCP、Skill、A2A、SSE/WebSocket/WebRTC 和 LLM 网关等工具调用工程问题展开。
- [13. MCP 协议通常采用什么通信方式？](https://xiaolinnote.com/ai/tools/13_mcp_transport.html)：围绕 Function Calling、MCP、Skill、A2A、SSE/WebSocket/WebRTC 和 LLM 网关等工具调用工程问题展开。
- [14. 说说 WebSocket 和 SSE 通信的区别及局限性？](https://xiaolinnote.com/ai/tools/14_sse_vs_websocket.html)：围绕 Function Calling、MCP、Skill、A2A、SSE/WebSocket/WebRTC 和 LLM 网关等工具调用工程问题展开。
- [15. 为什么要用 WebRTC 协议？它和 WebSocket（WS）在 AI 对话流中的核心差异是什么？](https://xiaolinnote.com/ai/tools/15_webrtc_vs_ws.html)：围绕 Function Calling、MCP、Skill、A2A、SSE/WebSocket/WebRTC 和 LLM 网关等工具调用工程问题展开。
- [16. 有没有用过大模型的网关框架？网关层解决了什么问题？](https://xiaolinnote.com/ai/tools/16_llm_gateway.html)：围绕 Function Calling、MCP、Skill、A2A、SSE/WebSocket/WebRTC 和 LLM 网关等工具调用工程问题展开。
