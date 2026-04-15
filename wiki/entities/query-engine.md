---
title: "QueryEngine"
type: entity
created: 2026-04-15
updated: 2026-04-15
sources: ["chapter-38-query-engine.md", "chapter-40-对话流程.md"]
related: ["streaming-tool-executor", "message-type-system", "error-recovery"]
---

QueryEngine 是 Claude Code 对话系统的核心编排器类，管理整个对话的生命周期。

## 核心职责

- **生命周期管理**：每个 QueryEngine 实例对应一个完整的对话会话
- **状态持久化**：跨多个 turn 保存消息、使用量和权限状态
- **协议适配**：将内部消息格式转换为 SDK 消息格式

## 关键成员变量

| 成员变量 | 用途 | 生命周期 |
|---------|------|----------|
| `mutableMessages` | 存储对话中所有消息 | 会话级，跨 turn 持久 |
| `permissionDenials` | 记录被拒绝的工具调用 | 会话级，用于 SDK 报告 |
| `totalUsage` | 累计 API 使用量 | 会话级，用于成本追踪 |
| `discoveredSkillNames` | 追踪已发现的技能名 | Turn 级，每轮清空 |
| `readFileState` | 文件内容缓存 | 会话级 |
| `abortController` | 支持中断执行 | 会话级 |

## 核心方法

`submitMessage()` 是核心入口，返回 AsyncGenerator，处理流程包括：权限包装、系统提示构建、用户输入处理、会话持久化、技能/插件加载、query 循环。

## 设计原则

使 SDK 模式和 REPL 模式共享相同的对话处理逻辑，同时保持状态隔离和可测试性。

## Connections

- [StreamingToolExecutor](../entities/streaming-tool-executor.md) - 工具执行协调
- [Message Type System](../concepts/message-type-system.md) - 消息类型定义
- [Error Recovery](../concepts/error-recovery.md) - 错误恢复机制

## Open Questions

- Snip Replay 机制在长 SDK 会话中的内存泄漏预防效果如何？
- 权限拒绝追踪的累积上限是否需要设计？

## Sources

- `chapters/chapter-38-query-engine.md`
- `chapters/chapter-40-对话流程.md`