---
title: "MCPTool"
type: entity
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-17-其他工具精选.md]
related: []
---

MCPTool 是 Claude Code 与 MCP（Model Context Protocol）服务器交互的桥梁。

## 核心功能

- 连接 MCP 服务器
- 动态适配任意 MCP 工具 Schema
- 远程工具调用

## 配置

| 配置项 | 值 |
|--------|-----|
| name | 'mcp' |
| isMcp | true |
| maxResultSizeChars | 100_000 |

## Schema 设计

```typescript
inputSchema = z.object({}).passthrough()
```

`passthrough()` 允许任意额外字段，适配不同 MCP 服务器的工具 Schema。

## 动态覆盖

名称、描述、Prompt 在 `mcpClient.ts` 中根据具体 MCP 工具覆盖：
1. MCP 客户端读取工具定义
2. 动态创建 MCPTool 实例
3. 注册到工具集合

## 权限检查

```typescript
async checkPermissions() {
  return {
    behavior: 'passthrough',
    message: 'MCPTool requires permission.'
  }
}
```

## 调用流程

```
Claude 调用 MCPTool
    ↓
mcpClient.ts 接收调用
    ↓
通过 MCP 协议发送请求
    ↓
MCP 服务器执行工具
    ↓
返回结果给 Claude
```

## UI 渲染

渲染函数由 MCP 工具的具体实现动态调用：
- renderToolUseMessage()
- renderToolUseProgressMessage()
- renderToolResultMessage()

## Connections

- [MCP协议](../concepts/mcp-protocol.md)
- [外部工具集成](../concepts/external-tool-integration.md)

## Open Questions

- 如何处理 MCP 服务器不可用的情况？

## Sources

- `chapters/chapter-17-其他工具精选.md`