---
title: "Tool 接口"
type: entity
created: 2026-04-15
updated: 2026-04-15
sources: [chapters/chapter-08-工具架构总览.md]
related: [../concepts/build-tool-factory.md, ../concepts/tool-lifecycle.md]
---

# Tool 接口

Tool 类型是 Claude Code 工具系统的核心接口，定义在 `src/Tool.ts`，是一个泛型接口，支持输入验证、输出类型和进度类型。

## 定义结构

```typescript
export type Tool<
  Input extends AnyObject = AnyObject,
  Output = unknown,
  P extends ToolProgressData = ToolProgressData,
> = {
  // 核心字段和方法
}
```

## 核心字段

| 字段名 | 类型 | 说明 |
|--------|------|------|
| `name` | `string` | 工具唯一标识名 |
| `aliases` | `string[]?` | 工具别名（向后兼容） |
| `inputSchema` | `Input` | Zod 输入验证 schema |
| `outputSchema` | `z.ZodType?` | 输出验证 schema |
| `maxResultSizeChars` | `number` | 结果最大字符数 |
| `mcpInfo` | `object?` | MCP 服务器信息 |

## 核心方法

| 方法名 | 返回类型 | 说明 |
|--------|----------|------|
| `call()` | `Promise<ToolResult<Output>>` | 执行工具 |
| `description()` | `Promise<string>` | 生成工具描述 |
| `prompt()` | `Promise<string>` | 生成系统提示 |
| `isEnabled()` | `boolean` | 是否启用 |
| `isConcurrencySafe()` | `boolean` | 是否可并发执行 |
| `isReadOnly()` | `boolean` | 是否只读操作 |
| `checkPermissions()` | `Promise<PermissionResult>` | 权限检查 |

## UI 渲染方法

- `renderToolUseMessage()`：渲染工具使用消息
- `renderToolResultMessage()`：渲染工具结果
- `renderToolUseProgressMessage()`：渲染进度
- `renderGroupedToolUse()`：渲染并行工具组

## Connections

- [buildTool 工厂](../concepts/build-tool-factory.md) - 简化工具定义
- [工具生命周期](../concepts/tool-lifecycle.md) - 从定义到执行
- [BashTool](../entities/bash-tool.md) - 具体工具实现

## Sources

- [第八章：工具架构总览](../sources/2026-04-15-工具架构总览.md)