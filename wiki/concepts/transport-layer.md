---
title: "传输层"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: ["chapter-25-mcp服务集成.md", "chapter-29-bridge系统架构.md", "chapter-30-repl-bridge.md"]
related: []
---

传输层是 Claude Code 通信基础设施的核心抽象，支持多种传输方式适配不同场景。

## MCP 传输类型

### Stdio 传输

通过标准输入/输出流与本地进程通信。清理过程包含优雅终止：SIGINT → SIGTERM → SIGKILL，总清理时间不超过 500ms。

### SSE 传输

Server-Sent Events，用于 HTTP 服务器持久连接。EventSource 连接长期存活，不应应用超时。

### HTTP 传输

Streamable HTTP，符合 MCP 规范。Accept 头部必须声明接受 JSON 和 SSE：`application/json, text/event-stream`

### WebSocket 传输

双向实时通信。Bun 环境支持头部/代理/TLS 选项。

### 进程内传输

用于同一进程内运行 MCP 服务器和客户端，避免启动重量级子进程。使用 `createLinkedTransportPair` 创建双向传输对。

### SDK 控制传输

桥接 SDK 进程与 CLI 进程，通过控制请求传递消息。

## Bridge 传输层

### ReplBridgeTransport 抽象

统一接口：
- write/writeBatch: 消息写入
- setOnData/setOnClose/setOnConnect: 回调设置
- connect: 连接建立
- getLastSequenceNum: 序列号获取
- reportState/reportMetadata: 状态上报

### v1 WebSocket 方案

HybridTransport 结合 WebSocket 读取和 HTTP POST 写入。URL 构建：`ws(s)://host/v(1|2)/session_ingress/ws/{sessionId}`

### v2 SSE 方案

SSETransport 读取 + CCRClient 写入。优势：
- 序列号支持断点续传
- 心跳机制保持活跃
- 状态上报功能

## Connections

- [WebSocket传输](../concepts/websocket-transport.md)
- [SSE传输](../concepts/sse-transport.md)
- [MCP服务集成](../sources/2026-04-15-mcp服务集成.md)
- [Bridge系统架构](../sources/2026-04-15-bridge系统架构.md)

## Open Questions

- 传输层的统一抽象如何进一步优化？
- 新传输类型的扩展机制？

## Sources

- `chapters/chapter-25-mcp服务集成.md`
- `chapters/chapter-29-bridge系统架构.md`
- `chapters/chapter-30-repl-bridge.md`