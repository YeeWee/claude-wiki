---
title: "SSE传输"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: ["chapter-29-bridge系统架构.md", "chapter-30-repl-bridge.md", "chapter-25-mcp服务集成.md"]
related: []
---

Server-Sent Events (SSE) 是 Claude Code 的核心传输方案之一，用于持久连接和实时消息推送。

## Bridge v2 方案

SSETransport 读取 + CCRClient 写入，是更现代的架构。

SSE 流 URL：`{sessionUrl}/worker/events/stream`

核心优势：
- **序列号支持**: SSE 传输携带序列号，支持断点续传
- **心跳机制**: CCRClient 提供内置心跳，保持连接活跃
- **状态上报**: 支持向云端报告会话状态

## Worker 注册

v2 方案需要先注册为 session worker，获取 worker_epoch 用于后续心跳和状态上报。

## MCP SSE 传输

用于与 HTTP 服务器建立持久连接。

配置：
- authProvider: 认证提供者
- eventSourceInit: EventSource 初始化选项（长期存活，不应用超时）
- requestInit: 请求初始化选项

认证头设置：
- Authorization: Bearer token
- Accept: text/event-stream

## 连接特性

- EventSource 连接长期存活
- 服务器通过事件流推送消息
- 支持断点续传（序列号）

## Connections

- [传输层](../concepts/transport-layer.md)
- [WebSocket传输](../concepts/websocket-transport.md)
- [Bridge系统](../concepts/bridge-system.md)
- [MCP服务集成](../sources/2026-04-15-mcp服务集成.md)
- [Bridge系统架构](../sources/2026-04-15-bridge系统架构.md)

## Open Questions

- SSE 连接的断线重连优化？
- 序列号机制的可靠性？

## Sources

- `chapters/chapter-29-bridge系统架构.md`
- `chapters/chapter-30-repl-bridge.md`
- `chapters/chapter-25-mcp服务集成.md`