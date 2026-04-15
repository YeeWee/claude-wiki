---
title: "第二十三章：Hook 系统"
type: source
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-23-hook系统.md]
related:
  - ../concepts/hook-system.md
  - ../concepts/hook-event.md
  - ../concepts/async-hook.md
---

# 第二十三章：Hook 系统

## 概要

Hook（钩子）系统是 Claude Code 的生命周期定制化机制，允许用户在特定事件触发时执行自定义的 shell 命令或 TypeScript 回调。通过 Hook 系统，用户可以拦截工具调用、管理会话生命周期、处理权限请求、监控配置变更。

## 核心要点

### Hook 事件类型

系统定义了 25+ 种事件类型：

| 类别 | 事件类型 |
|------|---------|
| 工具执行 | `PreToolUse`, `PostToolUse`, `PostToolUseFailure` |
| 会话生命周期 | `SessionStart`, `SessionEnd`, `Stop`, `StopFailure` |
| 子代理 | `SubagentStart`, `SubagentStop` |
| 压缩 | `PreCompact`, `PostCompact` |
| 权限 | `PermissionRequest`, `PermissionDenied` |
| 用户交互 | `UserPromptSubmit`, `Notification` |
| 配置/状态 | `ConfigChange`, `InstructionsLoaded`, `CwdChanged`, `FileChanged` |
| Worktree | `WorktreeCreate`, `WorktreeRemove` |
| 任务 | `TaskCreated`, `TaskCompleted` |
| MCP | `Elicitation`, `ElicitationResult` |

### Hook 类型

| 类型 | 说明 |
|------|------|
| `command` | 执行 Shell 命令 |
| `prompt` | 使用 AI 模型执行提示 |
| `agent` | 启动子代理执行任务 |
| `http` | 发送 HTTP 请求 |
| `callback` | 内部 TypeScript 回调（SDK 注册） |
| `function` | 会话级 TypeScript 函数 Hook |

### 会话 Hook 特殊能力

**SessionStart Hook**：
- 环境变量注入：通过 `CLAUDE_ENV_FILE` 写入 `.sh` 文件定义环境变量
- 文件监控路径：返回 `watchPaths` 设置 `FileChanged` Hook 监控路径
- 初始用户消息：返回 `initialUserMessage` 作为会话初始输入

**SessionEnd Hook**：
- 在 REPL 外执行，不产生面向模型的消息
- 失败信息直接写入 stderr
- 执行后清理所有会话级 Hook

### 异步 Hook 执行

Hook 通过输出首行的 JSON 响应声明异步模式：
```json
{"async": true, "asyncTimeout": 15000}
```

异步 Hook 通过 `AsyncHookRegistry` 管理，系统定期检查已完成的异步 Hook。

### 权限 Hook

`PermissionRequest` Hook 可以直接做出决策，返回 `decision` 字段：
- `behavior: 'allow'`：自动批准，可选返回 `updatedInput` 和 `updatedPermissions`
- `behavior: 'deny'`：自动拒绝，可选返回 `message` 和 `interrupt`

## 源文件位置

- `src/utils/hooks.ts`：Hook 执行核心逻辑
- `src/types/hooks.ts`：Hook 类型定义
- `hooks/AsyncHookRegistry.ts`：异步 Hook 注册表
- `src/utils/hooks/hookEvents.ts`：Hook 事件广播机制
- `src/entrypoints/sdk/coreTypes.ts`：HOOK_EVENTS 常量

## Connections

- [Hook 系统](../concepts/hook-system.md) - Hook 的核心概念
- [Hook Event](../concepts/hook-event.md) - 事件类型详解
- [异步 Hook](../concepts/async-hook.md) - 异步执行机制

## Open Questions

- Hook 执行超时（TOOL_HOOK_EXECUTION_TIMEOUT_MS）的具体值？
- 多个 Hook 的执行顺序如何确定？
- Hook 安全验证的工作区信任机制细节？

## Sources

- `chapters/chapter-23-hook系统.md`