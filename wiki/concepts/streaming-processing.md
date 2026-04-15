---
title: "Streaming Processing"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: ["chapter-38-query-engine.md", "chapter-40-对话流程.md", "chapter-41-语音模式.md"]
related: ["query-engine", "streaming-tool-executor"]
---

Claude Code 采用异步生成器（AsyncGenerator）模式实现流式处理，支持实时进度显示和增量消息输出。

## 核心模式

`submitMessage()` 返回 `AsyncGenerator<SDKMessage>`，允许 SDK 客户端实时接收流式事件，实现打字效果和实时反馈。

## 流式消息处理

`handleMessageFromStream` 函数处理流式响应事件：
- `content_block_start` - 设置流模式（thinking/responding/tool-input）
- `content_block_delta` - 增量内容（text_delta/input_json_delta/thinking_delta）
- `content_block_stop` - 完成内容块
- `message_start/message_delta/message_stop` - 使用量追踪

## StreamingToolUse 类型

```typescript
type StreamingToolUse = {
  index: number
  contentBlock: BetaToolUseBlock
  unparsedToolInput: string
}
```

## 语音流式处理

WebSocket 双向通信：
- 实时转录：TranscriptText（临时）、TranscriptEndpoint（最终）
- 保活机制：每 8 秒发送 KeepAlive
- 控制消息：CloseStream 结束流

## 使用量追踪机制

分层追踪：message_start 重置 -> message_delta 累加 -> message_stop 累积总量。

## 流式输出控制

- `includePartialMessages` 控制是否输出部分消息
- 进度消息实时 yield（通过 `progressAvailableResolve` 通知）

## Connections

- [QueryEngine](../entities/query-engine.md) - submitMessage 实现
- [StreamingToolExecutor](../entities/streaming-tool-executor.md) - 工具结果流式收集
- [VoiceProvider](../entities/voice-provider.md) - 语音流式转录

## Open Questions

- 流式处理在高延迟网络下的用户体验如何优化？
- 部分消息输出的缓存策略如何设计？

## Sources

- `chapters/chapter-38-query-engine.md`
- `chapters/chapter-40-对话流程.md`
- `chapters/chapter-41-语音模式.md`