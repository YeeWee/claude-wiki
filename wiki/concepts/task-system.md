---
title: "任务系统"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-14-任务管理工具集.md]
related: []
---

任务系统是 Claude Code 的结构化工作计划管理基础设施，支持任务生命周期、依赖关系和并发控制。

## 核心组件

### 五种任务工具

| 工具 | 功能 |
|------|------|
| TaskCreateTool | 创建任务 |
| TaskGetTool | 查询任务详情 |
| TaskListTool | 列出所有任务 |
| TaskUpdateTool | 更新任务状态 |
| TaskStopTool | 停止后台任务 |

### 状态模型

```
pending → in_progress → completed
pending → deleted
in_progress → pending（遇到阻塞）
```

后台任务系统使用不同状态模型：pending/running/completed/failed/killed。

## 依赖管理

### 双向阻塞关系

- **blocks**: 此任务阻塞的任务列表
- **blockedBy**: 阻塞此任务的任务列表

阻塞关系是双向一致的：A blocks B 意味着 A.blocks 包含 B 且 B.blockedBy 包含 A。

### 声明机制

`claimTask` 函数提供完整的声明检查：
1. 是否已被其他 Agent 声明
2. 任务是否已完成
3. 是否存在未解决的阻塞者

## 并发控制

### 高水位标记

防止 ID 重用，`.highwatermark` 文件存储最大已分配 ID。

### 文件锁

使用 lockfile 库保护并发操作：
- 30 次重试
- 5-100ms 超时

## 存储模型

每个任务为一个 JSON 文件，存储在 `.claude/tasks/` 目录。

## Teammate 集成

- 自动所有者分配
- 邮箱通知（所有者变更）
- Hook 验证机制

## Connections

- [TaskCreateTool](../entities/task-create-tool.md)
- [TaskUpdateTool](../entities/task-update-tool.md)
- [Teammate协作](teammate-collaboration.md)

## Open Questions

- 如何引入任务优先级机制？
- 如何优化大量任务的性能？

## Sources

- `chapters/chapter-14-任务管理工具集.md`