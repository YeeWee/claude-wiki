---
title: "Builtin Plugin"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources:
  - ../sources/2026-04-15-chapter-22-插件系统.md
related:
  - ../concepts/plugin-system.md
---

# Builtin Plugin

## 概要

内置插件系统为 CLI 提供随发布附带的可切换功能。与硬编码的斜杠命令不同，内置插件在 `/plugin` UI 的 "Built-in" 区域显示，用户可启用/禁用，设置持久化到用户配置。

## 与 Bundled Skills 的区分

| 特性 | Built-in Plugin | Bundled Skill |
|------|-----------------|---------------|
| 控制方式 | `/plugin` UI | 编译进二进制 |
| 设置持久化 | 用户配置 | 无 |
| 可见性 | Built-in 区域 | 隐式可用 |

## 定义结构

```typescript
type BuiltinPluginDefinition = {
  name: string
  description: string
  version?: string
  skills?: BundledSkillDefinition[]
  hooks?: HooksSettings
  mcpServers?: Record<string, McpServerConfig>
  isAvailable?: () => boolean
  defaultEnabled?: boolean
}
```

## 注册机制

```typescript
const BUILTIN_PLUGINS: Map<string, BuiltinPluginDefinition> = new Map()

export function registerBuiltinPlugin(definition: BuiltinPluginDefinition): void {
  BUILTIN_PLUGINS.set(definition.name, definition)
}
```

## 状态管理

```typescript
// 启用状态：用户偏好 > 插件默认 > true
const isEnabled =
  userSetting !== undefined
    ? userSetting === true
    : (definition.defaultEnabled ?? true)
```

## 当前状态

内置插件系统已搭建框架，但尚未注册具体插件。此架构为未来将内置技能迁移为用户可切换的插件提供了基础设施。

## Connections

- [Plugin System](../concepts/plugin-system.md) - 插件的核心概念
- [Plugin](../entities/plugin.md) - 插件实体

## Sources

- `src/plugins/builtinPlugins.ts`
- `src/plugins/bundled/index.ts`