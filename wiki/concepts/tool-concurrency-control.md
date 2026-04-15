---
title: "Tool Concurrency Control"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: ["chapter-40-对话流程.md"]
related: ["streaming-tool-executor", "error-recovery"]
---

Claude Code 的工具并发控制机制根据工具特性决定并发策略，平衡性能和安全性。

## 核心规则

并发控制通过 `canExecuteTool` 方法判断：

1. **无正在执行的工具**：可以立即执行
2. **并发安全工具**：可以与其他并发安全工具同时执行
3. **非并发安全工具**：必须独占执行，阻塞后续工具

## 工具状态流转

`queued` -> `executing` -> `completed` -> `yielded`

## TrackedTool 结构

```typescript
type TrackedTool = {
  id: string
  block: ToolUseBlock
  status: ToolStatus
  isConcurrencySafe: boolean  // 决定是否可并发
  promise?: Promise<void>
  results?: Message[]
  pendingProgress: Message[]
}
```

## 并发安全判断

工具通过 `isConcurrencySafe` 标记决定是否可以并发执行。典型的并发安全工具包括只读操作（Read、Glob、Grep），非并发安全工具包括写操作（Edit、Write、Bash）。

## 兄弟中断控制

`siblingAbortController` 用于在某个工具失败时中断同级工具执行。

## 进度消息处理

进度消息在工具执行过程中实时 yield，通过 `progressAvailableResolve` 通知等待者。

## Connections

- [StreamingToolExecutor](../entities/streaming-tool-executor.md) - 执行器实现
- [Error Recovery](../concepts/error-recovery.md) - 失败时的中断处理

## Open Questions

- 并发安全工具的分类标准是否需要更细粒度的控制？
- 工具执行超时的处理策略如何设计？
- 并发上限是否需要动态调整？

## Sources

- `chapters/chapter-40-对话流程.md`