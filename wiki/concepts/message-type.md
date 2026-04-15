---
title: "Message Type"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-05-类型系统基础.md]
related: [../concepts/tool-type.md, ../concepts/taskstate-type.md]
---

Message Type 是 Claude Code 消息的类型体系，定义对话中各种消息的结构。包括 UserMessage（用户输入）、AssistantMessage（AI 响应）、ProgressMessage（工具进度）、SystemMessage（系统信息）等类型，构成完整的消息流转机制。

## 消息类型概览

### 类型层次

```text
Message (基类)
├── UserMessage (用户输入)
├── AssistantMessage (AI 响应)
├── AttachmentMessage (文件附件)
├── ProgressMessage (工具进度)
└── SystemMessage (系统信息)
```

### 共享字段

所有消息类型共享：

- `type`：消息类型标识
- `uuid`：唯一标识符
- `timestamp`：时间戳

## UserMessage

用户输入消息：

```typescript
type UserMessage = {
  type: 'user',
  message: { role: 'user', content: string | ContentBlockParam[] },
  isMeta?: boolean,           // 系统生成的元消息
  isVirtual?: boolean,        // 虚拟消息
  toolUseResult?: unknown,    // 工具调用结果
  imagePasteIds?: string[],   // 图片粘贴 ID
  origin?: MessageOrigin,     // 消息来源
}
```

### 特殊标记

- `isMeta`：非用户直接输入，如权限请求
- `isVirtual`：不发送到 API，仅用于 UI 显示
- `toolUseResult`：工具执行结果回填

## AssistantMessage

AI 响应消息：

```typescript
type AssistantMessage = {
  type: 'assistant',
  message: BetaMessage,       // SDK 消息对象
  requestId?: string,         // API 请求 ID
  apiError?: APIError,        // API 错误
  usage?: Usage,              // Token 统计
}
```

### BetaMessage 结构

```typescript
type BetaMessage = {
  id: UUID,
  model: string,
  role: 'assistant',
  stop_reason: string,        // 停止原因
  content: BetaContentBlock[], // 内容块数组
  usage: { input_tokens, output_tokens, cache_read, cache_write },
}
```

## ProgressMessage

工具执行进度：

```typescript
type ProgressMessage<P = ToolProgressData> = {
  type: 'progress',
  data: P,                    // 进度数据
  toolUseID: string,          // 工具使用 ID
  parentToolUseID?: string,   // 父工具 ID（嵌套）
}
```

用于实时显示工具执行状态。

## SystemMessage

系统级信息：

```typescript
type SystemMessage = {
  type: 'system',
  subtype: string,            // 子类型
  content: string,            // 内容
  level: 'info' | 'warning' | 'error',
}
```

### 子类型

| 子类型 | 说明 |
|--------|------|
| `permission_retry` | 权限重试通知 |
| `api_error` | API 错误通知 |
| `api_metrics` | API 指标 |
| `turn_duration` | 轮次时长 |
| `compact_boundary` | 压缩边界 |
| `bridge_status` | 桥接状态 |
| `informational` | 一般信息 |

## Connections

- [Tool Type](../concepts/tool-type.md) - 工具类型
- [TaskState Type](../concepts/taskstate-type.md) - 任务状态

## Open Questions

- BetaContentBlock 的完整类型定义
- 消息压缩（compaction）的处理逻辑

## Sources

- `chapters/chapter-05-类型系统基础.md`