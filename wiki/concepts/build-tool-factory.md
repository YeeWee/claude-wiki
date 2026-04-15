---
title: "buildTool 工厂函数"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: [chapters/chapter-08-工具架构总览.md]
related: [../entities/tool-interface.md]
---

# buildTool 工厂函数

buildTool 函数简化工具定义，为常用方法提供安全默认值。

## 定义

```typescript
export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D> {
  return {
    ...TOOL_DEFAULTS,
    userFacingName: () => def.name,
    ...def,
  } as BuiltTool<D>
}
```

## 默认值策略（失败关闭）

| 方法 | 默认值 | 安全含义 |
|------|--------|----------|
| `isEnabled` | `true` | 默认启用 |
| `isConcurrencySafe` | `false` | 默认不安全 |
| `isReadOnly` | `false` | 默认假设写操作 |
| `checkPermissions` | 委托通用系统 | 标准权限流程 |

## ToolDef 类型

允许省略可默认化的方法：
```typescript
export type ToolDef = Omit<Tool, DefaultableToolKeys> &
  Partial<Pick<Tool, DefaultableToolKeys>>
```

可默认化的字段：
- isEnabled
- isConcurrencySafe
- isReadOnly
- isDestructive
- checkPermissions
- toAutoClassifierInput
- userFacingName

## Connections

- [Tool 接口](../entities/tool-interface.md) - 工具定义结构

## Sources

- [第八章：工具架构总览](../sources/2026-04-15-工具架构总览.md)