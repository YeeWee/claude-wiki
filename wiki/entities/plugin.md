---
title: "Plugin"
type: entity
created: 2026-04-15
updated: 2026-04-15
sources:
  - ../sources/2026-04-15-chapter-22-插件系统.md
related:
  - ../concepts/plugin-system.md
  - ../concepts/mcp-integration.md
---

# Plugin

## 概要

Plugin（插件）是 Claude Code 的扩展单元，通过安装插件来增强 CLI 的功能。插件可以提供自定义命令、AI 代理、技能、钩子、输出样式以及 MCP 服务器集成。

## 插件标识符

使用 `{name}@{marketplace}` 格式作为唯一标识符：
- 内置插件：`{name}@builtin`
- Marketplace 插件：`{name}@{marketplace-name}`

## 插件目录结构

```
my-plugin/
├── plugin.json          # 清单文件（可选）
├── commands/            # 自定义斜杠命令
│   ├── build.md
│   └── deploy.md
├── agents/              # 自定义 AI 代理
│   └── test-runner.md
├── skills/              # 技能目录
│   └── my-skill/
│       └── SKILL.md
├── hooks/               # 钩子配置
│   └── hooks.json
└── .mcp.json            # MCP 服务器配置（可选）
```

## 插件组件类型

插件可提供以下组件：
- `commands`：自定义斜杠命令
- `agents`：AI 代理
- `skills`：技能目录
- `hooks`：钩子配置
- `output-styles`：输出样式

## LoadedPlugin 结构

```typescript
type LoadedPlugin = {
  name: string
  manifest: PluginManifest
  path: string
  source: string          // pluginId
  repository: string
  enabled: boolean
  isBuiltin: boolean
  hooksConfig?: HooksSettings
  mcpServers?: Record<string, McpServerConfig>
}
```

## MCP 服务器作用域

为避免不同插件间的服务器名称冲突，系统为插件 MCP 服务器添加作用域前缀：
```typescript
const scopedName = `plugin:${pluginName}:${name}`
```

## Connections

- [插件系统](../concepts/plugin-system.md) - 插件的核心概念
- [MCP 集成](../concepts/mcp-integration.md) - MCP 服务器配置
- [Hook 系统](../concepts/hook-system.md) - 生命周期钩子

## Open Questions

- Marketplace 系统何时实现？
- 插件依赖解析的具体逻辑？

## Sources

- `src/plugins/builtinPlugins.ts`
- `src/utils/plugins/schemas.ts`