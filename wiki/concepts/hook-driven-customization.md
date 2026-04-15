---
title: "Hook-driven Customization"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-46-架构设计原则总结.md]
related: []
---

Hook-driven Customization 是 Claude Code 的生命周期定制化机制，允许用户在关键节点注入自定义逻辑。

## 事件类型（25+ 种）

| 类别 | 事件类型 |
|------|---------|
| 工具执行 | PreToolUse, PostToolUse, PostToolUseFailure |
| 会话生命周期 | SessionStart, SessionEnd, Stop, StopFailure |
| 子代理 | SubagentStart, SubagentStop |
| 压缩 | PreCompact, PostCompact |
| 权限 | PermissionRequest, PermissionDenied |
| 用户交互 | UserPromptSubmit, Notification |
| 配置 | ConfigChange, InstructionsLoaded, CwdChanged |
| 文件 | FileChanged |
| Worktree | WorktreeCreate, WorktreeRemove |

## Hook 类型

| 类型 | 说明 |
|------|------|
| Command Hook | 执行 shell 命令 |
| Prompt Hook | AI 模型执行 |
| Agent Hook | 启动子代理 |
| HTTP Hook | 发送 HTTP 请求 |
| Callback Hook | TypeScript 回调 |

## Hook 能力

- permissionDecision：自动权限决策
- updatedInput：修改工具输入
- blockExecution：阻断工具执行
- CLAUDE_ENV_FILE：环境变量注入
- watchPaths：文件监控路径

## Connections

- 无

## Open Questions

- Hook 执行顺序如何确定？

## Sources

- `chapters/chapter-46-架构设计原则总结.md`