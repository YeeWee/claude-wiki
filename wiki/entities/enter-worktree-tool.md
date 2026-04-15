---
title: "EnterWorktreeTool"
type: entity
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-16-worktree工具.md]
related: []
---

EnterWorktreeTool 创建隔离的 Git Worktree 工作环境并切换会话到其中。

## 核心功能

- 创建 Git Worktree（或通过 Hook）
- 切换会话到 worktree 目录
- 清理缓存和状态

## 配置

| 配置项 | 值 |
|--------|-----|
| name | 'EnterWorktree' |
| maxResultSizeChars | 100_000 |
| shouldDefer | true |

## 输入 Schema

```typescript
{
  name?: string  // Worktree 名称，限制 64 字符
}
```

命名规则：字母、数字、点、下划线、短横线，禁止 `.` 和 `..`。

## Prompt 关键要点

**When to Use**：用户明确说 "worktree"
**When NOT to Use**：创建分支、切换分支、修复 bug、开发功能（除非明确提到 worktree）

## 执行逻辑

1. 验证不在已存在的 worktree session
2. 切换到主仓库根目录
3. 创建 worktree（支持 sparse-checkout）
4. 执行后创建设置（复制 settings、配置 hooks、符号链接）
5. 切换目录并清理缓存

## Fast Resume 优化

直接读取 `.git` 指针文件，跳过 spawn git subprocess，节省约 15ms。

## Sparse Checkout

配置 `worktree.sparsePaths` 只检出指定目录。

## Hook 扩展

非 Git 环境可通过 `WorktreeCreate` Hook 实现 VCS 隔离。

## UI 渲染

```
Switched to worktree on branch {branch}
{worktreePath}
```

## Connections

- [ExitWorktreeTool](exit-worktree-tool.md)
- [Git Worktree](../concepts/git-worktree.md)

## Open Questions

- 如何处理创建失败后的清理？

## Sources

- `chapters/chapter-16-worktree工具.md`