---
title: "MCP协议"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: ["chapter-17-其他工具精选.md", "chapter-25-mcp服务集成.md"]
related: []
---

Model Context Protocol (MCP) 是 Anthropic 开发的开放协议，用于建立 AI 模型与外部系统之间的标准化通信。

## MCPTool 集成

MCPTool 使用 `passthrough()` Schema 允许任意输入对象，实际 Schema 由 MCP 服务器定义。

### 调用流程

```
Claude 调用 MCPTool
    ↓
mcpClient.ts 接收调用
    ↓
MCP 协议发送请求
    ↓
MCP 服务器执行
    ↓
返回结果
```

### 动态覆盖

MCP 客户端读取工具定义后：
1. 动态创建 MCPTool 实例
2. 覆盖名称、描述、Prompt
3. 注册到工具集合

### UI 渲染

渲染函数由 MCP 工具实现动态调用。

## 协议基础

基于 JSON-RPC 2.0，定义了客户端与服务器之间的通信规范。

## 四大核心概念

### 工具（Tools）

可被调用的函数，执行特定操作。工具名称遵循 `mcp__serverName__toolName` 格式。

工具属性：
- description: 工具描述
- readOnlyHint: 是否只读
- searchHint: 搜索提示
- alwaysLoad: 是否始终加载

### 资源（Resources）

可被读取的数据源。支持订阅机制 (`resources.subscribe`)。

### 提示（Prompts）

预定义的提示模板，可转换为 Claude Code 的 Command。

### 能力（Capabilities）

服务器声明支持的功能，包括：
- tools: 工具支持
- prompts: 提示支持
- resources: 资源支持
- resources.subscribe: 资源订阅

## Elicitation 机制

服务器向用户请求信息或确认：
- **Form 模式**: 结构化数据输入
- **URL 模式**: 引导用户访问 URL

响应动作：accept、decline、cancel

## 连接生命周期

1. 创建 Client，声明客户端能力
2. 连接 Transport
3. 发送 initialize 请求
4. 接收 InitializeResult + capabilities
5. 发送 initialized 通知
6. 注册通知处理器

## Connections

- [MCP](../entities/mcp.md)
- [传输层](../concepts/transport-layer.md)
- [MCP服务集成](../sources/2026-04-15-mcp服务集成.md)

## Open Questions

- MCP 规范版本演进计划？
- 与 LangChain/LlamaIndex 等框架的集成？

## Sources

- `chapters/chapter-25-mcp服务集成.md`