---
title: "隔离机制"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-16-worktree工具.md]
related: []
---

隔离机制是 Claude Code 多任务并行和隔离实验的核心支撑。

## 核心功能

1. **隔离工作目录**：同一仓库中创建独立工作目录
2. **并行任务处理**：不同任务在独立分支上并行进行
3. **安全的实验环境**：工作可保留或丢弃
4. **会话状态隔离**：缓存、系统提示、内存文件独立

## Git Worktree 基础

Git Worktree 允许不同分支检出到不同目录，共享 .git 目录。

## Hook 扩展机制

非 Git 环境通过 WorktreeCreate/WorktreeRemove hooks 实现 VCS 隔离。

## 安全性设计

- 限制长度防止过长路径
- 禁止 `.` 和 `..` 防止路径遍历
- 使用白名单正则防止注入

## Scope Guard

ExitWorktreeTool 仅操作当前 session 创建的 worktree，不影响：
- 手动创建的 worktree
- 历史 session 的 worktree
- 非 EnterWorktree 创建的目录

## Session 状态隔离

切换 worktree 时清理：
- clearSystemPromptSections()
- clearMemoryFileCaches()
- getPlansDirectory.cache.clear()

## Ephemeral Worktree

临时 worktree 用于 Agent、Workflow、Bridge、Template job，需定期清理处理泄漏。

## Connections

- [Git Worktree](git-worktree.md)
- [EnterWorktreeTool](../entities/enter-worktree-tool.md)
- [ExitWorktreeTool](../entities/exit-worktree-tool.md)

## Open Questions

- 如何优化清理机制？

## Sources

- `chapters/chapter-16-worktree工具.md`