---
title: "ExitWorktreeTool"
type: entity
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-16-worktree工具.md]
related: []
---

ExitWorktreeTool 退出 Worktree 会话，支持保留或删除 worktree。

## 核心功能

- 退出 worktree 会话
- 保留或删除 worktree 和分支
- 恢复会话到原始目录

## 配置

| 配置项 | 值 |
|--------|-----|
| name | 'ExitWorktree' |
| shouldDefer | true |
| isDestructive | action === 'remove' |

## 输入 Schema

```typescript
{
  action: 'keep' | 'remove',     // 保留或删除
  discard_changes?: boolean      // 删除时确认丢弃内容
}
```

## 权限检查（安全验证）

```typescript
async validateInput(input) {
  // 检查是否有 worktree session
  if (!getCurrentWorktreeSession()) {
    return { result: false, message: 'No active session' }
  }
  
  // remove 时验证未提交内容
  if (input.action === 'remove' && !input.discard_changes) {
    const summary = await countWorktreeChanges(...)
    if (summary.changedFiles > 0 || summary.commits > 0) {
      return { result: false, message: 'Has uncommitted content' }
    }
  }
  return { result: true }
}
```

## Prompt Scope Guard

仅操作当前 session 创建的 worktree，不影响：
- 手动创建的 worktree
- 历史 session 的 worktree
- 非 EnterWorktree 创建的目录

## 执行逻辑

**keep**：
- 保持 worktree 和分支
- 恢复会话到原始目录

**remove**：
- 如果有 tmux session，终止它
- 执行 `git worktree remove --force`
- 执行 `git branch -D`
- 恢复会话到原始目录

## Session 状态恢复

```typescript
function restoreSessionToOriginalCwd(originalCwd, projectRootIsWorktree) {
  setCwd(originalCwd)
  setOriginalCwd(originalCwd)
  if (projectRootIsWorktree) {
    setProjectRoot(originalCwd)
    updateHooksConfigSnapshot()
  }
  saveWorktreeState(null)
  clearSystemPromptSections()
  clearMemoryFileCaches()
}
```

## 安全设计原则

- **Fail-closed**：无法确定状态时拒绝操作
- **显式确认**：有未提交内容时必须设置 `discard_changes: true`
- **变更列出**：错误消息详细列出文件数和提交数

## UI 渲染

```
{actionLabel} (branch {branch})
Returned to {originalCwd}
```

## Connections

- [EnterWorktreeTool](enter-worktree-tool.md)
- [Git Worktree](../concepts/git-worktree.md)

## Open Questions

- 如何处理强制删除时的数据恢复？

## Sources

- `chapters/chapter-16-worktree工具.md`