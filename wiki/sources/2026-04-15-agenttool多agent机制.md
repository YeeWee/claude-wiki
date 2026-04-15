---
title: "AgentTool 多 Agent 机制"
type: source
created: 2026-04-15
updated: 2026-04-15
sources: [chapters/chapter-12-agenttool多agent机制.md]
related: []
---

# AgentTool 多 Agent 机制

AgentTool 是 Claude Code 实现多 Agent 协作的核心机制，支持任务委托、并行执行、上下文隔离和后台运行。

## 核心内容

### Agent 类型

三类定义：
- BuiltInAgentDefinition：内置 Agent（general-purpose、Explore、Plan、Verification、fork 等）
- CustomAgentDefinition：用户/项目定义的 Agent
- PluginAgentDefinition：插件提供的 Agent

内置 Agent 特点：
- Explore：只读模式、omitClaudeMd、使用 haiku/inherit 模型
- Plan：只读模式、生成实施计划
- Fork：隐式触发、继承父上下文、权限冒泡

### 子 Agent 启动流程

runAgent 核心函数：
- Agent ID 创建
- 上下文优化（Explore/Plan 不加载 CLAUDE.md）
- 权限模式设置
- MCP 服务器初始化
- 工具上下文创建
- Query 循环执行

工具池组装：
- filterToolsForAgent 过滤禁用工具
- ALL_AGENT_DISALLOWED_TOOLS：所有子 Agent 禁用
- ASYNC_AGENT_ALLOWED_TOOLS：异步 Agent 白名单

### Worktree 隔离模式

独立 Git 工作树执行：
- 路径规范：`.claude/worktrees/<slug>`
- 智能清理：有变更保留，无变更自动清理
- Fork + Worktree 注入路径转换通知

### 后台执行

异步 Agent 生命周期：
- registerAsyncAgent 注册任务
- runAsyncAgentLifecycle 驱动执行
- task-notification XML 格式发送结果通知

### Coordinator 模式

协调者-Worker 模式：
- Coordinator 系统提示定义角色和工作流程
- Worker 工具配置通过白名单限制
- 并发原则：并行启动 Worker 提高效率

### Teammate 机制

多 Agent 协作：
- In-Process Teammate：同一进程运行，AsyncLocalStorage 隔离
- 通信机制：Mailbox 和 SendMessage 工具

## 关键文件

| 文件路径 | 主要内容 |
|---------|---------|
| `src/tools/AgentTool/AgentTool.tsx` | 主入口 |
| `src/tools/AgentTool/runAgent.ts` | 执行核心 |
| `src/tools/AgentTool/builtInAgents.ts` | 内置 Agent 定义 |
| `src/utils/worktree.ts` | Worktree 管理 |
| `src/coordinator/coordinatorMode.ts` | Coordinator 模式 |

## 设计启示

- 上下文优化节省 token
- Prompt 缓存最大化命中率
- 权限冒泡便于用户响应
- 智能清理减少资源占用

## Open Questions

- 多 Agent 协作是否支持更复杂的拓扑结构？
- Coordinator 模式是否可扩展更多 Worker 类型？

## Sources

- [第十二章：AgentTool 多 Agent 机制](../chapters/chapter-12-agenttool多agent机制.md)