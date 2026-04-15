---
title: "Token Tracking"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources:
  - ../sources/2026-04-15-chapter-24-api客户端服务.md
related:
  - ../concepts/api-client.md
  - ../concepts/stream-processing.md
---

# Token Tracking

## 概要

Token 追踪是 API 客户端的核心功能，记录输入输出 token，计算成本，追踪缓存命中情况。

## NonNullableUsage 类型

```typescript
type NonNullableUsage = {
  input_tokens: number
  output_tokens: number
  cache_creation_input_tokens: number
  cache_read_input_tokens: number
  cache_creation: {
    ephemeral_1h_input_tokens: number
    ephemeral_5m_input_tokens: number
  }
  server_tool_use: {
    web_search_requests: number
    web_fetch_requests: number
  }
  service_tier: string
  inference_geo: string
  iterations: number
  speed: string
}
```

## 缓存 Token 细分

| 字段 | 说明 |
|------|------|
| `cache_read_input_tokens` | 从缓存读取（缓存命中） |
| `cache_creation_input_tokens` | 创建缓存（首次写入） |
| `ephemeral_1h_input_tokens` | 1 小时 TTL 缓存 |
| `ephemeral_5m_input_tokens` | 5 分钟 TTL 缓存 |

## 成本计算

`calculateUSDCost()` 函数计算请求成本，在 `message_delta` 事件处理时执行：
```typescript
const costUSD = calculateUSDCost(resolvedModel, usage)
costUSD += addToTotalSessionCost(costUSD, usage, options.model)
```

## 日志记录

`logAPISuccess()` 记录完整的 token 信息：
```typescript
logEvent('tengu_api_success', {
  model,
  inputTokens: usage.input_tokens,
  outputTokens: usage.output_tokens,
  cachedInputTokens: usage.cache_read_input_tokens ?? 0,
  uncachedInputTokens: usage.cache_creation_input_tokens ?? 0,
  messageTokens,
  durationMs,
  ttftMs,
  costUSD,
  requestId,
})
```

## 关键设计

流式 API 的 `message_delta` 可能发送显式 0 值，需要保护已设置的值：
```typescript
// 防止 message_delta 用 0 覆盖已设置的值
input_tokens: partUsage.input_tokens > 0
  ? partUsage.input_tokens
  : usage.input_tokens
```

## Connections

- [API Client](../concepts/api-client.md) - API 通信层
- [Stream Processing](../concepts/stream-processing.md) - 流处理机制

## Sources

- `src/services/api/logging.ts`
- `src/utils/modelCost.ts`