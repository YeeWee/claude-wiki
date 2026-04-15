---
title: "Teammate协作"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-14-任务管理工具集.md, chapter-15-计划模式工具.md]
related: []
---

Teammate 是 Claude Code 的多 Agent 协作机制，支持任务分配、计划审批和团队协调。

## 核心组件

### 任务系统集成

- 自动所有者分配（Agent 声明任务时）
- 邮箱通知（所有者变更）
- Agent 状态检查（是否忙碌）

### 计划审批流程

设置 `planModeRequired` 时：
1. 计划发送到 team leader mailbox
2. Leader 审批或拒绝
3. 响应发送到 teammate inbox

## Agent 状态检查

`claimTaskWithBusyCheck` 检查 Agent 是否忙碌于其他任务，防止过载。

## 任务声明机制

```typescript
type ClaimTaskResult = {
  success: boolean
  reason?: 'task_not_found' | 'already_claimed' | 'already_resolved' | 'blocked' | 'agent_busy'
  task?: Task
  busyWithTasks?: string[]
  blockedByTasks?: string[]
}
```

## 邮箱通信

任务分配和计划审批通过 mailbox 实现：
```typescript
{
  type: 'task_assignment' | 'plan_approval_request',
  taskId: string,
  subject: string,
  description: string,
  assignedBy: string,
  timestamp: string
}
```

## Agent 工具限制

**所有 Agent 禁用的工具**：
- ASK_USER_QUESTION_TOOL_NAME
- ENTER_PLAN_MODE_TOOL_NAME
- EXIT_PLAN_MODE_V2_TOOL_NAME
- AGENT_TOOL_NAME（防递归）

## Connections

- [任务系统](task-system.md)
- [计划模式](plan-mode.md)

## Open Questions

- 如何优化多 Agent 负载均衡？

## Sources

- `chapters/chapter-14-任务管理工具集.md`
- `chapters/chapter-15-计划模式工具.md`