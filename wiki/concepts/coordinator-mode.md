---
title: "Coordinator 模式"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: [chapters/chapter-12-agenttool多agent机制.md]
related: [../entities/agent-tool.md]
---

# Coordinator 模式

Coordinator 模式是一种多 Agent 协作模式，主 Agent 作为协调者，Worker 执行具体任务。

## 核心角色

Coordinator 的职责：
- 帮助用户达成目标
- 指导 Worker 研究和实现
- 综合结果并与用户沟通
- 能直接回答问题时直接回答

## 任务阶段表

| Phase | Who | Purpose |
|-------|-----|---------|
| Research | Workers (parallel) | 调查代码库，查找文件 |
| Synthesis | Coordinator | 读取发现，制定规格 |
| Implementation | Workers | 执行针对性修改 |
| Verification | Workers | 测试变更有效性 |

## 并发原则

Worker 是异步的，独立任务应并行启动而非串行执行。

## Worker 工具配置

通过白名单限制可用工具：
- Simple 模式：Bash、Read、Edit
- 完整模式：ASYNC_AGENT_ALLOWED_TOOLS

MCP 工具自动继承。

## Scratchpad 共享目录

Worker 可读写共享目录，用于跨 Worker 持久化知识。

## Connections

- [AgentTool](../entities/agent-tool.md) - 启动 Worker

## Sources

- [第十二章：AgentTool 多 Agent 机制](../sources/2026-04-15-agenttool多agent机制.md)