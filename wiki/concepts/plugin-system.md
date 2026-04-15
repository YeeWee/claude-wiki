---
title: "Plugin System"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources:
  - ../sources/2026-04-15-chapter-22-插件系统.md
related:
  - ../entities/plugin.md
  - ../concepts/builtin-plugin.md
  - ../concepts/mcp-integration.md
---

# Plugin System

## 概要

插件系统是 Claude Code 的扩展机制，允许用户通过安装插件来增强 CLI 的功能。插件可以提供自定义命令、AI 代理、技能、钩子、输出样式以及 MCP 服务器集成。

## 插件与内置技能的区分

| 类型 | 特点 | 控制方式 |
|------|------|----------|
| 内置插件 | 随 CLI 发布 | `/plugin` UI 启用/禁用 |
| 内置技能 | 编译进二进制 | 对所有用户自动可用 |

## 插件标识符格式

使用 `{name}@{marketplace}` 格式：
- 内置插件：`{name}@builtin`
- Marketplace 插件：`{name}@{marketplace-name}`

## 插件组件

插件可提供以下组件类型：
- `commands`：自定义斜杠命令
- `agents`：AI 代理
- `skills`：技能目录
- `hooks`：钩子配置
- `output-styles`：输出样式

## 清单 Schema

通过 Zod schema 进行严格验证：
```typescript
const PluginManifestMetadataSchema = z.object({
  name: z.string().min(1).refine(name => !name.includes(' ')),
  version: z.string().optional(),
  description: z.string().optional(),
  dependencies: z.array(DependencyRefSchema).optional(),
  // ...
})
```

## 状态管理

启用状态优先级：用户偏好 > 插件默认 > true

## MCP 服务器作用域

为避免冲突，添加插件前缀：
```typescript
const scopedName = `plugin:${pluginName}:${name}`
```

## 错误处理

类型化错误便于诊断：
- `path-not-found`、`network-error`
- `manifest-parse-error`、`manifest-validation-error`
- `mcp-config-invalid`、`dependency-unsatisfied` 等

## Connections

- [Plugin](../entities/plugin.md) - 插件实体
- [Builtin Plugin](../concepts/builtin-plugin.md) - 内置插件机制
- [MCP Integration](../concepts/mcp-integration.md) - MCP 服务器配置
- [Hook System](../concepts/hook-system.md) - 生命周期钩子

## Sources

- `src/plugins/builtinPlugins.ts`
- `src/utils/plugins/schemas.ts`