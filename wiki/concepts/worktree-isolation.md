---
title: "Worktree 隔离"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: [chapters/chapter-12-agenttool多agent机制.md]
related: [../entities/agent-tool.md]
---

# Worktree 隔离

Worktree 隔离模式让子 Agent 在独立的 Git 工作树中运行，避免与主工作目录冲突。

## 启用方式

```typescript
// Agent 定义中指定
isolation: 'worktree'

// 或调用时指定
AgentTool({ isolation: 'worktree', ... })
```

## 路径规范

标准路径：`.claude/worktrees/<slug>`

Slug 验证：
- 正则：`^[a-zA-Z0-9._-]+$`
- 最大长度：64 字符

## 创建流程

1. validateWorktreeSlug
2. findGitRoot
3. 检查 Worktree Create Hook
4. 执行 git worktree add 或 Hook

## 智能清理

Agent 完成后检查变更：
- 有变更：保留 worktree
- 无变更：自动清理（移除 worktree 和分支）

变更检测：比较 HEAD 与初始 commit。

## Fork + Worktree 通知

注入路径转换通知，告知 Agent 继承的父路径需转换为 worktree 根。

## Connections

- [AgentTool](../entities/agent-tool.md) - 使用 Worktree 隔离

## Sources

- [第十二章：AgentTool 多 Agent 机制](../sources/2026-04-15-agenttool多agent机制.md)