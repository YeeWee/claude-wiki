---
title: "TaskUpdateTool"
type: entity
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-14-任务管理工具集.md]
related: []
---

TaskUpdateTool 是任务管理中最复杂的工具，支持状态转换、所有者分配、依赖关系建立。

## 核心功能

- 状态转换（pending/in_progress/completed/deleted）
- 所有者分配（Agent 名称）
- 依赖关系建立（blocks/blockedBy）
- 元数据合并

## 输入 Schema

```typescript
{
  taskId: string,           // 任务 ID
  subject?: string,         // 新标题
  description?: string,     // 新描述
  activeForm?: string,      // Spinner 文本
  status?: 'pending' | 'in_progress' | 'completed' | 'deleted',
  owner?: string,           // 新所有者
  addBlocks?: string[],     // 此任务阻塞的任务 ID
  addBlockedBy?: string[],  // 阻塞此任务的任务 ID
  metadata?: Record<string, unknown>  // 元数据（设为 null 删除键）
}
```

## 输出

```typescript
{
  success: boolean,
  taskId: string,
  updatedFields: string[],
  error?: string,
  statusChange?: { from: string, to: string }
}
```

## 关键逻辑

### 状态转换

1. 自动展开任务列表视图
2. 验证任务存在性
3. `deleted` 状态触发特殊删除逻辑
4. `completed` 状态执行 TaskCompleted Hooks

### 所有者自动分配

Teammate 模式下，标记为 in_progress 时自动分配当前 Agent 为 owner。

### 邮箱通知

所有者变更时通过邮箱通知新所有者。

### 依赖操作

- `addBlocks`: 调用 `blockTask(taskId, blockId)`
- `addBlockedBy`: 调用 `blockTask(blockerId, taskId)`（反向）

## Hook 验证

TaskCompleted Hooks 可阻止任务完成，返回阻塞错误时任务保持原状态。

## Connections

- [TaskCreateTool](task-create-tool.md)
- [TaskGetTool](task-get-tool.md)
- [任务依赖](../concepts/task-dependencies.md)

## Open Questions

- 如何处理循环依赖？

## Sources

- `chapters/chapter-14-任务管理工具集.md`