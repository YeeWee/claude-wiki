---
title: "Tool Type"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-05-类型系统基础.md]
related: [../concepts/command-type.md, ../concepts/taskstate-type.md, ../concepts/message-type.md]
---

Tool Type 是 Claude Code 工具系统的核心类型定义，采用三参数泛型设计 `Tool<Input, Output, P>`，定义工具的完整生命周期方法：执行、权限、渲染、并发安全等。通过 buildTool 工厂函数提供安全默认值。

## 泛型接口

### 三参数泛型

```typescript
type Tool<
  Input extends AnyObject = AnyObject,   // Zod schema 验证对象
  Output = unknown,                       // 输出数据类型
  P extends ToolProgressData = ToolProgressData,  // 进度数据类型
>
```

### 泛型参数说明

| 参数 | 约束 | 用途 |
|------|------|------|
| Input | extends AnyObject | Zod schema，输入验证 |
| Output | 默认 unknown | 工具返回数据类型 |
| P | extends ToolProgressData | 进度回调数据类型 |

## 核心字段

### 标识字段

- `name`：工具唯一标识名
- `aliases`：向后兼容的别名列表
- `searchHint`：ToolSearch 关键字提示

### 执行字段

- `inputSchema`：输入验证 Zod schema
- `call()`：核心执行方法
- `description()`：描述生成方法

### 能力声明字段

- `isEnabled()`：是否启用
- `isConcurrencySafe()`：并发安全检查
- `isReadOnly()`：只读操作检查
- `isDestructive()`：破坏性操作检查

### 配置字段

- `maxResultSizeChars`：结果最大字符数
- `shouldDefer`：延迟加载标记
- `alwaysLoad`：始终加载标记
- `mcpInfo`：MCP 服务器信息

## ToolUseContext

工具执行时的上下文类型：

```typescript
type ToolUseContext = {
  options: {
    commands, debug, mainLoopModel, tools, verbose,
    mcpClients, mcpResources, agentDefinitions
  },
  abortController: AbortController,
  messages: Message[],
  getAppState, setAppState,
  setToolJSX, addNotification,
}
```

分类：

- **配置选项**：运行时参数
- **资源引用**：可用外部资源
- **状态访问**：当前状态
- **状态更新**：状态修改入口

## ToolResult

```typescript
type ToolResult<T> = {
  data: T,
  newMessages?: Message[],
  contextModifier?: (context) => context,
  mcpMeta?: { _meta, structuredContent },
}
```

## buildTool 工厂

### 失败关闭策略

```typescript
const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: () => false,  // 默认不安全
  isReadOnly: () => false,         // 默认写操作
  isDestructive: () => false,
}
```

确保不安全的工具不会被错误地并发执行。

## Connections

- [Command Type](../concepts/command-type.md) - 命令类型
- [TaskState Type](../concepts/taskstate-type.md) - 任务状态
- [Message Type](../concepts/message-type.md) - 消息类型

## Open Questions

- ToolUseContext 的完整字段定义
- contextModifier 的具体用途

## Sources

- `chapters/chapter-05-类型系统基础.md`