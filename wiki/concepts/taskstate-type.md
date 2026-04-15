---
title: "TaskState Type"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-05-类型系统基础.md]
related: [../concepts/tool-type.md, ../concepts/message-type.md]
---

TaskState Type 是 Claude Code 任务状态的类型定义，采用联合类型涵盖七种任务形态，配合 `isBackgroundTask` 类型守卫实现安全的类型缩小，用于判断任务是否应显示在后台任务指示器。

## 联合类型定义

### 七种任务类型

```typescript
type TaskState =
  | LocalShellTaskState      // Shell 命令执行
  | LocalAgentTaskState      // 子 Agent 执行
  | RemoteAgentTaskState     // 远程 Agent
  | InProcessTeammateTaskState  // 团队协作
  | LocalWorkflowTaskState   // 工作流脚本
  | MonitorMcpTaskState      // MCP 监控
  | DreamTaskState           // 自动 Dream
```

### 任务类型说明

| 任务类型 | 来源 | 特点 |
|----------|------|------|
| LocalShellTask | Bash 工具 | 命令执行 |
| LocalAgentTask | Agent 工具 | 子进程 Agent |
| RemoteAgentTask | 远程服务 | 异地 Agent |
| InProcessTeammateTask | 团队功能 | 进程内协作 |
| LocalWorkflowTask | 工作流 | 脚本执行 |
| MonitorMcpTask | MCP 协议 | 服务器监控 |
| DreamTask | 自动化 | 自动任务 |

## BackgroundTaskState

```typescript
type BackgroundTaskState =
  | LocalShellTaskState
  | LocalAgentTaskState
  | RemoteAgentTaskState
  | InProcessTeammateTaskState
  | LocalWorkflowTaskState
  | MonitorMcpTaskState
  | DreamTaskState
```

后台任务与 TaskState 的区别在于 `status` 和 `isBackgrounded` 字段判断。

## isBackgroundTask 类型守卫

### 函数签名

```typescript
function isBackgroundTask(task: TaskState): task is BackgroundTaskState
```

返回 `task is BackgroundTaskState` 是类型守卫签名：

- 返回 `true`：TypeScript 将 `task` 类型缩小为 `BackgroundTaskState`
- 返回 `false`：类型保持为 `TaskState`

### 实现逻辑

```typescript
function isBackgroundTask(task: TaskState): task is BackgroundTaskState {
  // 状态检查：必须是 running 或 pending
  if (task.status !== 'running' && task.status !== 'pending') {
    return false
  }
  // Foreground 检查：isBackgrounded === false 不算后台
  if ('isBackgrounded' in task && task.isBackgrounded === false) {
    return false
  }
  return true
}
```

## 类型守卫机制

### 编译时类型缩小

```typescript
const task: TaskState = ...

if (isBackgroundTask(task)) {
  // task 类型为 BackgroundTaskState
  // 可安全访问 BackgroundTaskState 专属字段
} else {
  // task 类型仍为 TaskState
}
```

### 运行时检查 + 类型缩小

类型守卫结合运行时检查和 TypeScript 类型系统：

1. 运行时：检查 `status` 和 `isBackgrounded`
2. 编译时：TypeScript 根据返回值缩小类型

## Connections

- [Tool Type](../concepts/tool-type.md) - 工具类型
- [Message Type](../concepts/message-type.md) - 消息类型

## Open Questions

- 各任务类型的完整字段定义
- 任务状态转换的完整生命周期

## Sources

- `chapters/chapter-05-类型系统基础.md`