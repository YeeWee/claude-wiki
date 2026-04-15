---
title: "类型系统基础"
type: source
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-05-类型系统基础.md]
related: [../concepts/tool-type.md, ../concepts/command-type.md, ../concepts/taskstate-type.md, ../concepts/message-type.md]
---

Claude Code 的类型系统为整个架构提供坚实的契约基础，通过泛型约束、联合类型、不可变性标记和类型守卫等 TypeScript 特性，定义 Tool、Command、TaskState、Message 等核心类型接口，确保模块间的类型安全和编译时错误检测。

## 类型系统的角色

### 核心价值

1. **契约定义**：类型定义模块间的接口契约，编译时发现不兼容问题
2. **文档化作用**：类型本身就是最好的文档
3. **安全保障**：防止空值引用、类型混淆、API 签名错误

### 设计原则

| 原则 | 说明 | 示例 |
|------|------|------|
| 泛型约束 | 使用泛型提高复用性 | `Tool<Input, Output, P>` |
| 联合类型 | 表示多种可能性 | `TaskState = ShellTask \| AgentTask \| ...` |
| 不可变性 | readonly 标记不可变集合 | `Tools = readonly Tool[]` |
| 可选标记 | ? 明确可选字段 | `aliases?: string[]` |
| 类型守卫 | 函数实现运行时类型判断 | `isBackgroundTask()` |

## Tool 类型

### 泛型接口

```typescript
type Tool<
  Input extends AnyObject = AnyObject,
  Output = unknown,
  P extends ToolProgressData = ToolProgressData,
>
```

- **Input**：Zod schema 验证的对象
- **Output**：任意类型返回值
- **P**：进度数据类型

### 核心字段

- `name`：工具唯一标识
- `inputSchema`：输入验证 schema
- `call()`：核心执行方法
- `description()`：描述生成方法
- `isEnabled()`：是否启用
- `isConcurrencySafe()`：并发安全检查
- `isReadOnly()`：只读操作检查
- `isDestructive()`：破坏性操作检查

### ToolUseContext

工具执行时的上下文类型，包含：

- **配置选项**：debug、verbose、mainLoopModel
- **资源引用**：tools、commands、mcpClients
- **状态访问**：getAppState、messages
- **状态更新**：setAppState、setToolJSX
- **中断控制**：abortController
- **通知系统**：addNotification、sendOSNotification

### buildTool 工厂

采用"失败关闭"策略：

- `isConcurrencySafe` 默认 `false`
- `isReadOnly` 默认 `false`

## Command 类型

联合类型：`Command = CommandBase & (PromptCommand | LocalCommand | LocalJSXCommand)`

| 类型 | 说明 | 执行方式 |
|------|------|----------|
| PromptCommand | 提示命令 | 生成提示注入对话 |
| LocalCommand | 本地命令 | 懒加载模块执行 |
| LocalJSXCommand | JSX 命令 | 返回 React 组件 |

### CommandBase

共享字段：availability、description、name、aliases、isHidden、isEnabled、argumentHint、whenToUse

## TaskState 类型

联合类型涵盖七种任务状态：

| 任务类型 | 说明 |
|----------|------|
| LocalShellTaskState | Shell 命令执行 |
| LocalAgentTaskState | 子 Agent 执行 |
| RemoteAgentTaskState | 远程 Agent |
| InProcessTeammateTaskState | 团队协作 |
| LocalWorkflowTaskState | 工作流脚本 |
| MonitorMcpTaskState | MCP 监控 |
| DreamTaskState | 自动 Dream |

### isBackgroundTask 类型守卫

返回 `task is BackgroundTaskState`，实现安全的类型缩小。

## Message 类型体系

| 消息类型 | 说明 |
|----------|------|
| UserMessage | 用户输入，包含工具调用结果 |
| AssistantMessage | AI 响应，包含内容块和 token 统计 |
| AttachmentMessage | 文件附件 |
| ProgressMessage | 工具执行进度 |
| SystemMessage | 系统级信息 |

## Connections

- [Tool Type](../concepts/tool-type.md) - 工具类型详解
- [Command Type](../concepts/command-type.md) - 命令类型详解
- [TaskState Type](../concepts/taskstate-type.md) - 任务状态类型
- [Message Type](../concepts/message-type.md) - 消息类型体系

## Open Questions

- ToolUseContext 的完整字段列表和类型定义
- Message 类型与 Anthropic SDK 的 BetaMessage 结构映射
- 类型守卫在复杂嵌套场景下的性能影响

## Sources

- `chapters/chapter-05-类型系统基础.md`