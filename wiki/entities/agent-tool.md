---
title: "AgentTool"
type: entity
created: 2026-04-15
updated: 2026-04-15
sources: [chapters/chapter-12-agenttool多agent机制.md]
related: [../entities/agent-definition.md, ../concepts/worktree-isolation.md]
---

# AgentTool

AgentTool 是 Claude Code 实现多 Agent 协作的核心机制，支持任务委托、并行执行、上下文隔离和后台运行。

## 核心能力

1. **任务委托**：将复杂任务分解给专业子 Agent 处理
2. **并行执行**：同时启动多个子 Agent，提高效率
3. **上下文隔离**：子 Agent 拥有独立的工作目录和工具权限
4. **后台运行**：支持异步执行，主 Agent 可继续其他工作

## Agent 选择流程

1. 检查多 Agent spawn 条件
2. 检查 Fork 实验开启
3. 查找指定 Agent 定义
4. 检查权限规则
5. 执行子 Agent

## 内置 Agent

| Agent 类型 | 说明 | 特点 |
|------------|------|------|
| `general-purpose` | 通用研究 | 全工具访问 |
| `Explore` | 快速搜索 | 只读、omitClaudeMd |
| `Plan` | 规划 | 只读、生成计划 |
| `Verification` | 验证 | 验证代码变更 |
| `fork` | Fork 子 Agent | 继承父上下文 |

## Fork Agent 特点

- 隐式触发：不指定 subagent_type 时自动使用
- 上下文继承：继承父 Agent 完整对话历史
- 权限冒泡：权限请求显示在父终端
- Prompt 缓存共享：使用父的精确工具定义

## Connections

- [Agent Definition](../entities/agent-definition.md) - 类型定义
- [Worktree 隔离](../concepts/worktree-isolation.md) - 隔离模式
- [Coordinator 模式](../concepts/coordinator-mode.md) - 协作模式

## Sources

- [第十二章：AgentTool 多 Agent 机制](../sources/2026-04-15-agenttool多agent机制.md)