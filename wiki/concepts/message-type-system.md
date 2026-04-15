---
title: "Message Type System"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: ["chapter-39-消息处理.md"]
related: ["query-engine", "streaming-processing"]
---

Claude Code 的消息类型系统定义了一套完整的消息类型体系，用于表示对话中的各种信息形式。

## 主要消息类型

| 类型 | 说明 | 核心字段 |
|------|------|----------|
| UserMessage | 用户输入 | `message.content`, `isMeta`, `toolUseResult` |
| AssistantMessage | AI 响应 | `message.content`, `usage`, `stop_reason` |
| AttachmentMessage | 文件/Hook等附加信息 | `attachment.type` |
| ProgressMessage | 工具执行进度 | `data`, `toolUseID` |
| SystemMessage | 系统级别信息 | `subtype`, `level` |

## 内容块类型

- TextBlock - 文本内容
- ToolUseBlock - 工具调用请求
- ThinkingBlock - 思考内容（扩展思考）
- RedactedThinkingBlock - 被遮蔽的思考
- ToolResultBlock - 工具执行结果

## 附件类型

- FileAttachment - 用户 @-mention 的文件
- CompactFileReferenceAttachment - 压缩后的文件引用
- PDFReferenceAttachment - PDF 文件引用
- HookAttachment - Hook 执行结果

## 系统消息子类型

`informational`、`api_error`、`compact_boundary`、`api_metrics`、`turn_duration`、`stop_hook_summary`、`permission_retry`

## 规范化流程

`normalizeMessagesForAPI` 处理步骤：
1. 重排序附件消息
2. 合连续用户消息
3. 提升 tool_result 到内容开头
4. 过滤孤立 thinking 消息
5. 验证图片大小

## 查找系统

`buildMessageLookups` 构建 O(1) 复杂度查找表：
- `siblingToolUseIDs` - 同级工具调用
- `progressMessagesByToolUseID` - 进度消息
- `toolResultByToolUseID` - 工具结果

## Connections

- [QueryEngine](../entities/query-engine.md) - 消息存储管理
- [Streaming Processing](../concepts/streaming-processing.md) - 流式消息处理

## Open Questions

- UUID 派生机制在大规模消息拆分时的碰撞风险如何？
- 附件类型的扩展机制是否需要更灵活的设计？

## Sources

- `chapters/chapter-39-消息处理.md`