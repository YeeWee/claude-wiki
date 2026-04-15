---
title: "Error Recovery"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: ["chapter-40-对话流程.md"]
related: ["query-engine", "tool-concurrency-control"]
---

Claude Code 的错误恢复机制提供多层保障，确保对话在各种异常情况下能优雅恢复。

## 错误类型分类

| 错误类型 | 处理策略 |
|---------|---------|
| prompt-too-long (413) | collapse drain -> reactive compact |
| max_output_tokens | escalate -> multi-turn resume |
| rate_limit | fallback to alternate model |
| image_size_error | strip oversized media |
| API 连接错误 | withRetry 自动重试 |

## Prompt-Too-Long 恢复（413）

两级策略：
1. **collapse drain**：低成本，保留粒度上下文（通过 contextCollapse）
2. **reactive compact**：完整摘要（通过 reactiveCompact.tryReactiveCompact）

## Max_Output_Tokens 恢复

两级策略：
1. **escalate**：从 8k 提升到 64k（ESCALATED_MAX_TOKENS）
2. **multi-turn recovery**：最多 3 次（MAX_OUTPUT_TOKENS_RECOVERY_LIMIT），注入恢复消息继续对话

## 模型降级处理

`FallbackTriggeredError` 时：
1. 切换备用模型
2. 清理孤立消息（yieldMissingToolResultBlocks）
3. 重建工具执行器
4. 剥离签名块（thinking 签名是模型绑定）
5. 输出警告消息

## 中断处理

- 消耗剩余结果（生成合成 tool_result）
- Chicago MCP 清理
- 跳过 submit-interrupt 的中断消息
- 返回 `{ reason: 'aborted_streaming' }`

## Continue 转换机制

转换类型定义：
- `collapse_drain_retry`
- `reactive_compact_retry`
- `max_output_tokens_recovery`
- `max_output_tokens_escalate`
- `stop_hook_blocking`

## Connections

- [QueryEngine](../entities/query-engine.md) - 状态管理
- [Tool Concurrency Control](../concepts/tool-concurrency-control.md) - 工具失败中断

## Open Questions

- 多级错误恢复的阈值参数如何动态调优？
- 模型降级后的用户体验如何优化？
- 恢复策略的优先级排序依据是什么？

## Sources

- `chapters/chapter-40-对话流程.md`