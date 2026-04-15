---
title: "后台执行机制"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: [chapters/chapter-09-bashtool深度解析.md, chapters/chapter-12-agenttool多agent机制.md]
related: [../entities/bash-tool.md, ../entities/agent-tool.md]
---

# 后台执行机制

后台执行支持长时间运行任务的异步处理，主线程可继续其他工作。

## BashTool 后台触发

| 触发方式 | 条件 |
|---------|------|
| 显式请求 | `run_in_background: true` |
| 自动超时 | 命令超过 15 秒 |
| 用户手动 | Ctrl+B 快捷键 |

### Kairos 自动后台化

主线程阻塞命令超过 15 秒自动后台化。

### 后台任务结果

包含任务 ID、输出路径、触发方式说明。

## AgentTool 后台触发

| 触发方式 | 条件 |
|---------|------|
| 显式请求 | `run_in_background: true` |
| Agent 定义 | `background: true` |
| Coordinator | Coordinator 模式强制异步 |
| Fork 实验 | forceAsync 标志 |

### 异步 Agent 生命周期

1. registerAsyncAgent 注册任务
2. runAsyncAgentLifecycle 驱动执行
3. 完成时发送 task-notification

### 任务通知格式

```xml
<task-notification>
<task-id>{agentId}</task-id>
<status>completed|failed|killed</status>
<summary>{status summary}</summary>
<result>{final response}</result>
<usage>
  <total_tokens>N</total_tokens>
  <tool_uses>N</tool_uses>
  <duration_ms>N</duration_ms>
</usage>
</task-notification>
```

## Connections

- [BashTool](../entities/bash-tool.md) - 命令后台执行
- [AgentTool](../entities/agent-tool.md) - Agent 后台执行

## Sources

- [第九章：BashTool 深度解析](../sources/2026-04-15-bashtool深度解析.md)
- [第十二章：AgentTool 多 Agent 机制](../sources/2026-04-15-agenttool多agent机制.md)