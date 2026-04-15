---
title: "对话流程"
type: source
created: 2026-04-15
updated: 2026-04-15
sources: ["chapter-40-对话流程.md"]
related: []
---

## 概述

对话流程是 Claude Code 与用户交互的核心机制，从用户输入到 Claude 响应完成涉及多个复杂阶段：消息接收、上下文管理、API 请求、工具执行、错误恢复、状态更新。

## 核心要点

### 核心文件

| 文件 | 作用 |
|------|------|
| QueryEngine.ts | 对话引擎类，管理会话状态和生命周期 |
| query.ts | 核心查询循环，处理消息轮次和流式响应 |
| StreamingToolExecutor.ts | 流式工具执行器 |
| processUserInput.ts | 用户输入处理 |

### State 对象设计

跨轮次的可变状态，关键字段：
- `messages` - 当前轮次的消息数组
- `autoCompactTracking` - 自动压缩追踪状态
- `maxOutputTokensRecoveryCount` - 输出 token 限制恢复计数
- `turnCount` - 当前轮次计数
- `transition` - 上一次循环的继续原因

### Continue 转换机制

转换类型：
- `collapse_drain_retry` - collapse drain 重试
- `reactive_compact_retry` - 响应式压缩重试
- `max_output_tokens_recovery` - 输出限制恢复
- `max_output_tokens_escalate` - 输出限制升级
- `stop_hook_blocking` - Stop Hook 阻塞

### StreamingToolExecutor 设计

工具状态：`queued` | `executing` | `completed` | `yielded`

并发控制规则：
- 无正在执行的工具：可以立即执行
- 并发安全工具：可与其他并发安全工具同时执行
- 非并发安全工具：必须独占执行

### 错误恢复机制

| 错误类型 | 处理策略 |
|---------|---------|
| prompt-too-long (413) | collapse drain -> reactive compact |
| max_output_tokens | escalate -> multi-turn resume |
| rate_limit | fallback to alternate model |
| image_size_error | strip oversized media |
| API 连接错误 | withRetry 自动重试 |

### Prompt-Too-Long 恢复

两级策略：
1. 第一级：collapse drain（低成本，保留粒度上下文）
2. 第二级：reactive compact（完整摘要）

### Max_Output_Tokens 恢复

两级策略：
1. 第一级：escalate（从 8k 提升到 64k）
2. 第二级：multi-turn recovery（最多 3 次）

### 模型降级处理

`FallbackTriggeredError` 时切换备用模型，清理孤立消息，重建工具执行器。

### 中断处理

消耗剩余结果（生成合成 tool_result），Chicago MCP 清理，跳过 submit-interrupt 的中断消息。

## 关键洞察

对话流程架构确保了 Claude Code 在复杂任务场景下的稳定性和可靠性，关键设计原则：状态分离、流式处理、并发控制、优雅降级。

## 开放问题

- 多级错误恢复的阈值参数如何动态调优？
- 并发安全工具的分类标准是否需要更细粒度的控制？

## Sources

- `chapters/chapter-40-对话流程.md`