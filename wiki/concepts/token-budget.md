---
title: "Token预算管理"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: ["chapter-26-上下文压缩服务.md"]
related: []
---

Token 预算管理是 Claude Code 上下文压缩服务的核心决策依据。

## 预算常量

| 常量 | 值 | 用途 |
|------|-----|------|
| AUTOCOMPACT_BUFFER_TOKENS | 13,000 | 自动压缩缓冲 |
| WARNING_THRESHOLD_BUFFER_TOKENS | 20,000 | 警告阈值缓冲 |
| ERROR_THRESHOLD_BUFFER_TOKENS | 20,000 | 错误阈值缓冲 |
| MANUAL_COMPACT_BUFFER_TOKENS | 3,000 | 手动压缩缓冲 |
| MAX_OUTPUT_TOKENS_FOR_SUMMARY | 20,000 | 摘要输出上限 |

## 有效窗口计算

`getEffectiveContextWindowSize()`：
- 扣除输出 token 预留
- 支持环境变量限制窗口大小
- 考虑模型特性

## 阈值判断

`calculateTokenWarningState()` 返回：
- percentLeft: 剩余百分比
- isAboveWarningThreshold: 是否超过警告
- isAboveErrorThreshold: 是否超过错误
- isAboveAutoCompactThreshold: 是否触发自动压缩
- isAtBlockingLimit: 是否到达阻塞限制

## Token 估算

`estimateMessageTokens()` 考虑：
- text: roughTokenCountEstimation
- tool_result: calculateToolResultTokens
- image/document: 固定 2000 tokens
- thinking: roughTokenCountEstimation
- tool_use: 名称 + 参数

保守系数：4/3

## 环境变量覆盖

- `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE`: 百分比阈值覆盖
- `CLAUDE_CODE_AUTO_COMPACT_WINDOW`: 窗口大小限制

## Connections

- [上下文压缩](../concepts/context-compaction.md)
- [上下文压缩服务](../sources/2026-04-15-上下文压缩服务.md)

## Open Questions

- Token 估算的准确性如何提升？
- 动态预算调整策略？

## Sources

- `chapters/chapter-26-上下文压缩服务.md`