---
title: "TaskCreateTool"
type: entity
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-14-任务管理工具集.md]
related: []
---

TaskCreateTool 在任务列表中创建新任务，是复杂工作流的起点。

## 核心功能

- 创建任务并分配唯一 ID
- 执行 TaskCreated Hooks 验证
- 自动展开任务列表视图

## 输入 Schema

```typescript
{
  subject: string,          // 任务标题（祈使语气）
  description: string,      // 详细说明
  activeForm?: string,      // Spinner 显示文本
  metadata?: Record<string, unknown>  // 任意元数据
}
```

## 创建流程

1. 调用 `createTask` 创建任务（状态: pending）
2. 执行 TaskCreated Hooks
3. Hook 返回阻塞错误时删除任务并抛出异常
4. 自动展开任务列表视图

## 输出

```typescript
{
  task: {
    id: string,      // 分配的任务 ID
    subject: string  // 任务标题
  }
}
```

## 使用场景

- 复杂多步骤任务（3+ 步骤）
- 非平凡复杂任务
- 计划模式中的工作跟踪
- 用户明确请求 todo 列表
- 用户提供多个任务

## Teammate 模式提示

- 描述需足够详细供其他 agent 理解
- 新任务状态为 pending，无 owner
- 使用 TaskUpdateTool 分配 owner

## Connections

- [TaskUpdateTool](task-update-tool.md)
- [TaskGetTool](task-get-tool.md)
- [TaskListTool](task-list-tool.md)
- [任务系统](../concepts/task-system.md)

## Open Questions

- 如何确定最佳的任务粒度？

## Sources

- `chapters/chapter-14-任务管理工具集.md`