---
title: "Async Hook"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources:
  - ../sources/2026-04-15-chapter-23-hook系统.md
related:
  - ../concepts/hook-system.md
  - ../concepts/hook-event.md
---

# Async Hook

## 概要

异步 Hook 执行机制允许长时间运行的 Hook 在后台执行而不阻塞主流程，通过首行 JSON 响应声明异步模式。

## 异步响应协议

Hook 通过输出首行的 JSON 响应声明异步模式：
```json
{"async": true, "asyncTimeout": 15000}
```

当 Hook 输出首行包含 `{"async": true}` 时，进程会被移入后台继续执行。

## AsyncHookRegistry

异步 Hook 通过 `AsyncHookRegistry` 管理：

```typescript
type PendingAsyncHook = {
  processId: string
  hookId: string
  hookName: string
  hookEvent: HookEvent
  startTime: number
  timeout: number
  command: string
  responseAttachmentSent: boolean
  shellCommand?: ShellCommand
  stopProgressInterval: () => void
}
```

## 注册流程

```typescript
export function registerPendingAsyncHook({
  processId,
  hookId,
  asyncResponse,
  hookName,
  hookEvent,
}): void {
  const timeout = asyncResponse.asyncTimeout || 15000
  const stopProgressInterval = startHookProgressInterval({...})
  pendingHooks.set(processId, {...})
}
```

## 结果检查

`checkForAsyncHookResponses()` 函数定期检查已完成的异步 Hook：
- 检查每个待处理 Hook 的状态
- 解析 stdout 中的同步响应
- 移除已完成或被终止的 Hook

## 进度追踪

异步 Hook 执行过程中会定期发送进度事件：
- `started`：Hook 开始执行
- `progress`：定期输出更新
- `response`：最终结果

## Connections

- [Hook System](../concepts/hook-system.md) - Hook 的核心概念
- [Hook Event](../concepts/hook-event.md) - 事件类型

## Sources

- `hooks/AsyncHookRegistry.ts`
- `src/utils/hooks.ts`