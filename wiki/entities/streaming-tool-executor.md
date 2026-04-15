---
title: "StreamingToolExecutor"
type: entity
created: 2026-04-15
updated: 2026-04-15
sources: ["chapter-40-对话流程.md"]
related: ["query-engine", "tool-concurrency-control"]
---

StreamingToolExecutor 是 Claude Code 的流式工具执行器，实现工具的并发执行控制。

## 核心职责

- 管理工具执行状态（queued/executing/completed/yielded）
- 实现并发控制策略
- 处理进度消息实时 yield
- 协调工具结果收集

## 工具状态追踪

```typescript
type TrackedTool = {
  id: string
  block: ToolUseBlock
  assistantMessage: AssistantMessage
  status: ToolStatus
  isConcurrencySafe: boolean
  promise?: Promise<void>
  results?: Message[]
  pendingProgress: Message[]
}
```

## 并发控制规则

1. 无正在执行的工具：可以立即执行
2. 并发安全工具：可与其他并发安全工具同时执行
3. 非并发安全工具：必须独占执行，阻塞后续工具

## 关键成员

- `tools: TrackedTool[]` - 工具追踪数组
- `siblingAbortController` - 同级中断控制
- `progressAvailableResolve` - 进度可用通知

## Connections

- [QueryEngine](../entities/query-engine.md) - 对话编排器
- [Tool Concurrency Control](../concepts/tool-concurrency-control.md) - 并发控制概念

## Open Questions

- 并发安全工具的分类标准是否需要更细粒度的控制？
- 工具执行超时的处理策略如何设计？

## Sources

- `chapters/chapter-40-对话流程.md`