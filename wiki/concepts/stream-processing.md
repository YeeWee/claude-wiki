---
title: "Stream Processing"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources:
  - ../sources/2026-04-15-chapter-24-api客户端服务.md
related:
  - ../concepts/api-client.md
---

# Stream Processing

## 概要

流式响应处理是 API 客户端的核心功能，通过 SSE（Server-Sent Events）流接收 Claude API 的响应，实时累积状态并 yield 中间结果。

## 流事件处理

| 事件类型 | 处理逻辑 |
|----------|----------|
| `message_start` | 记录 TTFB，初始化 usage |
| `content_block_start` | 初始化 contentBlock |
| `content_block_delta` | 累积文本/工具输入 |
| `content_block_stop` | 创建 AssistantMessage 并 yield |
| `message_delta` | 更新 usage, stop_reason，计算成本 |

## 流空闲超时检测

**Stream Watchdog**：超过 90 秒无数据则中断流，触发非流式降级。

```typescript
const STREAM_IDLE_TIMEOUT_MS = 90_000

function resetStreamIdleTimer(): void {
  streamIdleTimer = setTimeout(() => {
    streamIdleAborted = true
    releaseStreamResources()
  }, STREAM_IDLE_TIMEOUT_MS)
}
```

## Stall 检测

检测并记录流中断（30 秒阈值）：
```typescript
const STALL_THRESHOLD_MS = 30_000
if (timeSinceLastEvent > STALL_THRESHOLD_MS) {
  stallCount++
  totalStallTime += timeSinceLastEvent
}
```

## 非流式降级

流失败时自动降级到非流式请求：
```typescript
if (streamingError) {
  const result = yield* executeNonStreamingRequest(...)
  const m: AssistantMessage = { message: result, type: 'assistant' }
  yield m
}
```

## Usage 累积

流式 API 提供累积值而非增量，需要保护已设置的值不被显式 0 覆盖：
```typescript
input_tokens: partUsage.input_tokens > 0
  ? partUsage.input_tokens
  : usage.input_tokens  // 防止用 0 覆盖
```

## Connections

- [API Client](../concepts/api-client.md) - API 通信层
- [Token Tracking](../concepts/token-tracking.md) - 使用量追踪

## Sources

- `src/services/api/claude.ts`