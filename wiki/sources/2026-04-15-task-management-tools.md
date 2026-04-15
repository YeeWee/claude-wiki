---
title: "任务管理工具集"
type: source
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-14-任务管理工具集.md]
related: []
---

任务管理工具集提供完整的任务生命周期管理能力，支持复杂的多步骤工作流和任务依赖关系管理。

## 核心要点

1. **五种核心工具**：TaskCreateTool、TaskGetTool、TaskListTool、TaskUpdateTool、TaskStopTool
2. **状态模型**：pending → in_progress → completed/deleted
3. **依赖管理**：双向阻塞关系（blocks/blockedBy）
4. **并发控制**：文件锁机制支持多 Agent 协作

## 任务状态模型

```
pending → in_progress → completed
pending → deleted
in_progress → pending (遇到阻塞)
in_progress → deleted
```

任务状态定义在 `/src/utils/tasks.ts`，后台任务系统使用不同状态模型（pending/running/completed/failed/killed）。

## 工具概览

| 工具 | 功能 | 关键特性 |
|------|------|----------|
| TaskCreateTool | 创建任务 | Hook 验证、自动展开视图 |
| TaskGetTool | 查询任务 | 完整详情、依赖关系 |
| TaskListTool | 列出任务 | 摘要格式、过滤已完成依赖 |
| TaskUpdateTool | 更新任务 | 状态转换、所有者分配、依赖建立 |
| TaskStopTool | 停止任务 | 后台任务终止、向后兼容 |

## 依赖关系模型

- **blocks**: 此任务阻塞的任务列表
- **blockedBy**: 阻塞此任务的任务列表

阻塞关系是双向的：A blocks B 意味着 A 的 `blocks` 列表包含 B，B 的 `blockedBy` 列表包含 A。

## 并发控制机制

### 高水位标记

防止 ID 重用，使用 `.highwatermark` 文件存储最大已分配 ID。

### 文件锁

使用 lockfile 库保护并发操作，配置 30 次重试、5-100ms 超时。

## 任务声明机制

`claimTask` 函数提供完整的声明检查：
1. 检查是否已被其他 Agent 声明
2. 检查任务是否已完成
3. 检查是否存在未解决的阻塞者
4. 执行声明操作

`claimTaskWithBusyCheck` 还会检查 Agent 是否忙碌于其他任务。

## Teammate 集成

- 自动所有者分配（Agent 声明任务时）
- 邮箱通知（所有者变更时）
- Hook 验证机制（创建/完成时）

## Connections

- [TaskCreateTool](../entities/task-create-tool.md)
- [TaskUpdateTool](../entities/task-update-tool.md)
- [任务系统](../concepts/task-system.md)
- [Teammate协作](../concepts/teammate-collaboration.md)

## Open Questions

- 如何优化大量任务场景下的性能？
- 任务优先级机制是否需要引入？

## Sources

- `chapters/chapter-14-任务管理工具集.md`