---
title: "MCP (Model Context Protocol)"
type: entity
created: 2026-04-15
updated: 2026-04-15
sources: ["chapter-25-mcp服务集成.md", "chapter-48-可扩展性设计.md", "chapter-49-学习资源与社区.md"]
related: [../concepts/mcp-extension.md, ../entities/anthropic.md]
---

Anthropic 开发的开放协议，用于建立 AI 模型与外部工具、资源之间的标准化连接。

## 核心概念

MCP 定义了四大核心概念：

1. **工具（Tools）**: 可被调用的函数，执行特定操作
2. **资源（Resources）**: 可被读取的数据源
3. **提示（Prompts）**: 预定义的提示模板
4. **能力（Capabilities）**: 服务器声明支持的功能

## 协议基础

基于 JSON-RPC 2.0 协议，定义客户端与服务器之间的通信规范。

## 传输方式

支持多种传输类型：
- Stdio: 本地进程通信
- SSE: Server-Sent Events
- HTTP: Streamable HTTP
- WebSocket: 双向实时通信
- 进程内传输: 避免重量级子进程
- SDK 控制传输: SDK 进程桥接

## Claude Code 集成

使用 `@modelcontextprotocol/sdk` 作为 MCP 客户端实现。支持：
- 调用外部工具执行任务
- 访问远程资源和数据源
- Elicitation 交互式确认
- 动态加载插件服务

## 配置范围

多层级配置优先级：
local > user > project > dynamic > enterprise > claudeai > managed

## Connections

- [MCP协议](../concepts/mcp-protocol.md)
- [传输层](../concepts/transport-layer.md)
- [OAuth认证](../concepts/oauth-authentication.md)
- [MCP服务集成](../sources/2026-04-15-mcp服务集成.md)

## Open Questions

- MCP 规范的未来发展方向？
- 与其他 AI 工具协议的互操作性？

## Sources

- `chapters/chapter-25-mcp服务集成.md`