---
title: "Hook Event"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources:
  - ../sources/2026-04-15-chapter-23-hook系统.md
related:
  - ../concepts/hook-system.md
---

# Hook Event

## 概要

Hook Event 是 Hook 系统的触发点，定义了在 Claude Code 运行过程中可以拦截的关键节点。

## 事件类型列表

| 事件类型 | 触发时机 | 主要用途 |
|---------|---------|---------|
| `PreToolUse` | 工具执行前 | 输入验证、权限拦截 |
| `PostToolUse` | 工具执行成功后 | 结果处理、日志记录 |
| `PostToolUseFailure` | 工具执行失败后 | 错误处理、重试逻辑 |
| `SessionStart` | 会话启动时 | 初始化环境 |
| `SessionEnd` | 会话结束时 | 清理资源 |
| `Stop` | 主线程停止时 | 最终输出处理 |
| `StopFailure` | 主线程错误停止时 | 错误报告 |
| `SubagentStart` | 子代理启动时 | 子代理初始化 |
| `SubagentStop` | 子代理停止时 | 子代理结果处理 |
| `Setup` | 初始化触发时 | 环境设置 |
| `PreCompact` | 会话压缩前 | 自定义压缩指令 |
| `PostCompact` | 会话压缩后 | 压缩后处理 |
| `UserPromptSubmit` | 用户提交提示时 | 提示预处理 |
| `Notification` | 通知发送时 | 通知拦截 |
| `PermissionRequest` | 权限请求时 | 自动批准/拒绝 |
| `PermissionDenied` | 权限被拒绝时 | 记录拒绝原因 |
| `TeammateIdle` | Teammate 空闲时 | 防止空闲 |
| `TaskCreated` | 任务创建时 | 任务验证 |
| `TaskCompleted` | 任务完成时 | 完成后处理 |
| `Elicitation` | MCP Elicitation 请求时 | 自动响应 |
| `ElicitationResult` | Elicitation 结果时 | 结果处理 |
| `ConfigChange` | 配置变更时 | 变更审计 |
| `InstructionsLoaded` | 指令文件加载时 | 加载监控 |
| `CwdChanged` | 工作目录变更时 | 目录切换处理 |
| `FileChanged` | 文件变更时 | 文件监控 |
| `WorktreeCreate` | Worktree 创建时 | Worktree 定位 |
| `WorktreeRemove` | Worktree 移除时 | 清理处理 |

## 会话 Hook 特殊能力

### SessionStart

- **环境变量注入**：通过 `CLAUDE_ENV_FILE` 写入 `.sh` 文件
- **文件监控路径**：返回 `watchPaths` 设置监控
- **初始用户消息**：返回 `initialUserMessage` 作为初始输入

### SessionEnd

- 在 REPL 外执行
- 失败信息直接写入 stderr
- 执行后清理所有会话级 Hook

## Connections

- [Hook System](../concepts/hook-system.md) - Hook 的核心概念
- [Async Hook](../concepts/async-hook.md) - 异步执行机制

## Sources

- `src/entrypoints/sdk/coreTypes.ts`