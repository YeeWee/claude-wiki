---
title: "Git Worktree"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-16-worktree工具.md]
related: []
---

Git Worktree 是 Git 2.5+ 引入的功能，允许将同一仓库的不同分支检出到不同目录。

## 核心概念

```
主仓库 (main branch)     Worktree (feature branch)
├── .git/                ├── .git (文件，指向主仓库)
├── src/                 ├── src/
├── package.json         ├── package.json
                         └── (独立的工作目录)
```

## 优势

| 特性 | 说明 |
|------|------|
| 共享 .git 目录 | 多个工作目录共享同一 Git 历史 |
| 独立分支 | 每个工作目录可 checkout 不同分支 |
| 无需 clone | 避免完整 clone 的磁盘开销和时间 |
| 快速切换 | 直接切换目录即可切换工作环境 |

## Claude Worktree 架构

```
repo/
├── .claude/
│   ├── worktrees/
│   │   ├── worktree-feature-a/
│   │   ├── worktree-fix-bug/
│   │   └── agent-a7f3d2e/  (Agent worktree)
```

## 工具支持

| 工具 | 功能 |
|------|------|
| EnterWorktreeTool | 创建/进入 worktree |
| ExitWorktreeTool | 退出 worktree |

## Fast Resume

直接读取 `.git` 指针文件，跳过 spawn git subprocess，节省约 15ms。

## Sparse Checkout

配置 `worktree.sparsePaths` 只检出指定目录，优化大型仓库性能。

## 后创建设置

1. 复制 `settings.local.json`
2. 配置 `core.hooksPath`
3. 符号链接目录（如 node_modules）
4. 复制 `.worktreeinclude` 文件

## Ephemeral Worktree

临时 worktree 用于 Agent、Workflow、Bridge、Template job：
- `agent-a[0-9a-f]{7}`
- `wf_[0-9a-f]{8}-[0-9a-f]{3}-\d+`
- `bridge-[A-Za-z0-9_]+`

## 安全性设计

- 限制长度防止过长路径
- 禁止 `.` 和 `..` 防止路径遍历
- 使用白名单正则防止注入

## Connections

- [EnterWorktreeTool](../entities/enter-worktree-tool.md)
- [ExitWorktreeTool](../entities/exit-worktree-tool.md)
- [隔离机制](isolation-mechanism.md)

## Open Questions

- 如何处理进程异常退出导致的泄漏？

## Sources

- `chapters/chapter-16-worktree工具.md`