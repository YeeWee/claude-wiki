---
title: "WebSocket传输"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: ["chapter-29-bridge系统架构.md", "chapter-30-repl-bridge.md"]
related: []
---

WebSocket 是 Bridge 系统传输层的核心方案之一，提供双向实时通信能力。

## v1 方案架构

HybridTransport 结合 WebSocket 读取和 HTTP POST 写作。

URL 构建：`ws(s)://host/v(1|2)/session_ingress/ws/{sessionId}`

本地开发：ws:// 协议直连 session-ingress 服务。
生产环境：wss:// 协议通过 Envoy 代理。

## 协议选择

根据 URL 判断：
- localhost/127.0.0.1 → ws 协议、v2 版本
- 其他 → wss 协议、v1 版本

## 写入机制

写入操作通过 HTTP POST 发送，避免 WebSocket 的单向限制。

## v2 方案对比

v1 vs v2:
- v1: WebSocket + HTTP POST，无序列号，无状态上报
- v2: SSE + CCR，序列号支持，状态上报功能

## Bun 环境

Bun 的 WebSocket 支持头部/代理/TLS 选项：
```typescript
new globalThis.WebSocket(url, {
  protocols: ['mcp'],
  headers: wsHeaders,
  proxy: getWebSocketProxyUrl(url),
  tls: tlsOptions,
})
```

## Node 环境

使用 createNodeWsClient 创建客户端，支持代理和 TLS 配置。

## Connections

- [传输层](../concepts/transport-layer.md)
- [Bridge系统](../concepts/bridge-system.md)
- [Bridge系统架构](../sources/2026-04-15-bridge系统架构.md)

## Open Questions

- WebSocket 连接稳定性优化？
- 与 SSE 方案的切换策略？

## Sources

- `chapters/chapter-29-bridge系统架构.md`
- `chapters/chapter-30-repl-bridge.md`