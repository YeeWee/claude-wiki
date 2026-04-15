---
title: "Worktree工具"
type: source
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-16-worktree工具.md]
related: []
---

Worktree 工具是 Claude Code 提供的隔离工作环境机制，基于 Git Worktree 实现并行任务处理和安全的实验环境。

## 核心要点

1. **隔离机制**：同一仓库的不同分支检出到不同目录
2. **并行任务处理**：不同任务在独立分支上并行进行
3. **安全的实验环境**：工作可保留或丢弃，不影响主分支
4. **会话状态隔离**：缓存、系统提示和内存文件独立管理

## Git Worktree 基础

```
主仓库 (main branch)     Worktree (feature branch)
├── .git/                ├── .git (文件，指向主仓库)
├── src/                 ├── src/
├── package.json         ├── package.json
                         └── (独立的工作目录)
```

优势：共享 .git 目录、独立分支、无需 clone、快速切换。

## Claude Code Worktree 架构

```
repo/
├── .claude/
│   ├── worktrees/
│   │   ├── worktree-feature-a/
│   │   ├── worktree-fix-bug/
│   │   └── agent-a7f3d2e/  (Agent worktree)
```

命名规则：`worktree-{slug}`，限制 64 字符，禁止 `.` 和 `..` 防止路径遍历。

## 工具概览

| 工具 | 功能 | 特性 |
|------|------|------|
| EnterWorktreeTool | 创建/进入 worktree | Hook 扩展支持、Fast Resume |
| ExitWorktreeTool | 退出 worktree | keep/remove 模式、安全验证 |

## 创建流程

1. 验证不在已存在的 worktree session
2. 切换到主仓库根目录
3. 创建 worktree（支持 sparse-checkout）
4. 复制 settings、配置 hooks、符号链接目录
5. 切换目录并清理缓存

### Fast Resume 优化

直接读取 `.git` 指针文件，跳过 spawn git subprocess，节省约 15ms。

### Sparse Checkout

配置 `worktree.sparsePaths` 只检出指定目录，优化大型仓库性能。

## 后创建设置

1. 复制 `settings.local.json`
2. 配置 `core.hooksPath`（解决 husky 问题）
3. 符号链接目录（如 node_modules）
4. 复制 `.worktreeinclude` 文件（gitignored 文件）

## 安全验证机制

`countWorktreeChanges` 函数检查未提交内容：
- 未提交文件数
- 相对原始 HEAD 的新提交数

**安全设计原则**：
- Fail-closed：无法确定状态时拒绝操作
- 显式确认：有未提交内容时必须设置 `discard_changes: true`

## Hook 扩展机制

支持非 Git 环境通过 WorktreeCreate/WorktreeRemove hooks 实现 VCS 隔离。

## Ephemeral Worktree Patterns

临时 worktree 用于 Agent、Workflow、Bridge、Template job：
```
agent-a[0-9a-f]{7}
wf_[0-9a-f]{8}-[0-9a-f]{3}-\d+
wf-\d+
bridge-[A-Za-z0-9_]+
job-[a-zA-Z0-9._-]{1,55}-[0-9a-f]{8}
```

## Connections

- [EnterWorktreeTool](../entities/enter-worktree-tool.md)
- [ExitWorktreeTool](../entities/exit-worktree-tool.md)
- [Git Worktree](../concepts/git-worktree.md)
- [隔离机制](../concepts/isolation-mechanism.md)

## Open Questions

- 如何优化 worktree 清理机制以处理进程异常退出导致的泄漏？
- 符号链接目录的最佳配置是什么？

## Sources

- `chapters/chapter-16-worktree工具.md`