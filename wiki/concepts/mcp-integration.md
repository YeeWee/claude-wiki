---
title: "MCP Integration"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources:
  - ../sources/2026-04-15-chapter-22-插件系统.md
related:
  - ../concepts/plugin-system.md
  - ../entities/plugin.md
---

# MCP Integration

## 概要

MCP（Model Context Protocol）集成是插件系统的重要功能，允许插件通过多种方式提供 MCP 服务器，扩展 Claude Code 的工具能力。

## MCP 服务器来源

插件可通过以下方式提供 MCP 服务器：

| 来源 | 优先级 | 说明 |
|------|--------|------|
| `.mcp.json` 文件 | 最低 | 插件目录中的配置文件 |
| 清单 `mcpServers` 字段 | 高 | 支持多种配置格式 |

## 清单配置格式

**字符串路径**：
```json
{
  "mcpServers": ".mcp.json"  // JSON 文件路径
}
```

**MCPB 文件**：
```json
{
  "mcpServers": "servers/my-server.mcpb"  // MCP Bundle 文件
}
```

**内联配置**：
```json
{
  "mcpServers": {
    "my-server": {
      "type": "stdio",
      "command": "node",
      "args": ["server.js"]
    }
  }
}
```

**多个配置源**：
```json
{
  "mcpServers": [".mcp.json", "servers/extra.mcpb"]
}
```

## MCPB 文件处理

MCPB（MCP Bundle）是打包格式，支持远程 URL 和本地路径。处理时检查是否需要用户配置：

```typescript
if ('status' in result && result.status === 'needs-config') {
  // 暂不加载，等待用户配置
  return null
}
```

## 环境变量解析

支持特殊变量替换：
- `CLAUDE_PLUGIN_ROOT`：插件根目录
- `CLAUDE_PLUGIN_DATA`：插件数据目录
- 用户配置变量

## 作用域添加

为避免冲突，添加插件前缀：
```typescript
const scopedName = `plugin:${pluginName}:${name}`
const scoped: ScopedMcpServerConfig = {
  ...config,
  scope: 'dynamic',
  pluginSource,
}
```

## Connections

- [Plugin System](../concepts/plugin-system.md) - 插件的核心概念
- [Plugin](../entities/plugin.md) - 插件实体

## Sources

- `src/utils/plugins/mcpPluginIntegration.ts`