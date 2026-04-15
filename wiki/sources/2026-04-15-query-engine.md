---
title: "Query Engine"
type: source
created: 2026-04-15
updated: 2026-04-15
sources: ["chapter-38-query-engine.md"]
related: []
---

## 概述

Query Engine 是 Claude Code 对话系统的核心编排器，将对话生命周期管理、消息状态跟踪、工具执行协调和 Compaction 触发等功能封装成一个独立的类，为 SDK 模式和 REPL 模式提供统一的对话处理接口。

## 核心要点

### 设计目标

- **生命周期管理**：每个 QueryEngine 实例对应一个完整的对话会话
- **状态持久化**：跨多个 turn 保存消息、使用量和权限状态
- **协议适配**：将内部消息格式转换为 SDK 消息格式

### 核心成员变量

| 成员变量 | 用途 | 生命周期 |
|---------|------|----------|
| `mutableMessages` | 存储对话中所有消息 | 会话级，跨 turn 持久 |
| `permissionDenials` | 记录被拒绝的工具调用 | 会话级，用于 SDK 报告 |
| `totalUsage` | 累计 API 使用量 | 会话级，用于成本追踪 |
| `discoveredSkillNames` | 追踪已发现的技能名 | Turn 级，每轮清空 |
| `readFileState` | 文件内容缓存 | 会话级，避免重复读取 |
| `abortController` | 支持中断执行 | 会话级 |

### submitMessage 方法

核心方法是异步生成器 `submitMessage()`，处理流程：
1. 权限包装 - 包装 `canUseTool` 函数追踪权限拒绝
2. 系统提示构建 - 调用 `fetchSystemPromptParts`
3. 用户输入处理 - 调用 `processUserInput`
4. 会话持久化 - 写入 transcript
5. 技能/插件加载
6. query 循环 - 进入核心查询循环

### 使用量追踪机制

分层追踪：message_start 时重置，message_delta 时累加，message_stop 时累积到总量。

### Compaction 协调

处理 compact_boundary 系统消息，持久化压缩边界前的消息，释放旧消息支持 GC。

### 公共 API

| 方法 | 返回类型 | 用途 |
|------|---------|------|
| `submitMessage()` | `AsyncGenerator<SDKMessage>` | 提交消息并流式返回结果 |
| `interrupt()` | `void` | 中断当前执行 |
| `getMessages()` | `readonly Message[]` | 获取当前消息列表 |
| `getReadFileState()` | `FileStateCache` | 获取文件读取缓存 |
| `getSessionId()` | `string` | 获取会话 ID |
| `setModel()` | `void` | 动态设置模型 |

### ask() 便捷函数

QueryEngine 的便捷封装，用于一次性查询。

## 关键洞察

QueryEngine 使 SDK 模式和 REPL 模式共享相同的对话处理逻辑，同时保持状态隔离和可测试性。它是 Claude Code 架构中承上启下的关键组件。

## 开放问题

- Snip Replay 机制在长 SDK 会话中的内存泄漏预防效果如何？
- 权限拒绝追踪的累积上限是否需要设计？

## Sources

- `chapters/chapter-38-query-engine.md`