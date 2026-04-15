# Claude Code 调度平台需求文档

> 版本：v0.1 | 日期：2026-04-15 | 状态：草稿
> 基于 book/（50 章 Claude Code 源代码分析）和 wiki/（193 页知识库）文档编写

---

## 目录

1. [产品概述](#1-产品概述)
2. [功能需求](#2-功能需求)
3. [系统架构](#3-系统架构)
4. [技术实现方案](#4-技术实现方案)
5. [Agent 调度模型](#5-agent-调度模型)
6. [看板界面设计](#6-看板界面设计)
7. [会话管理](#7-会话管理)
8. [安全与权限](#8-安全与权限)
9. [监控与可观测性](#9-监控与可观测性)
10. [部署与运维](#10-部署与运维)
11. [开发计划](#11-开发计划)

---

## 1. 产品概述

### 1.1 产品愿景

**将多个 Claude Code 会话封装为可调度、可观察、可干预的"Agent 工人"，提供一个看板式管理平台，让用户像管理一支开发团队一样管理 AI Agent。**

核心隐喻：
- 每个 Claude Code 会话 = 一个正在工作的"人"
- 平台 = 项目经理看板 + 任务调度中心
- 用户 = 技术负责人，负责任务拆解、分配、审核

平台不替代 Claude Code 的能力，而是将其**会话生命周期**和**工具调用能力**封装为可编程、可调度的基础设施。

### 1.2 目标用户画像

| 角色 | 特征 | 核心需求 |
|------|------|----------|
| 技术管理者 | 负责多项目并行交付 | 全局视角看所有 Agent 状态，快速识别阻塞和异常 |
| 高级工程师 | 熟悉 Claude Code，需要规模化使用 | 同时调度 3-10 个 Agent 处理不同模块，自动分配 |
| 团队负责人 | 需要分配任务给 Agent 而非人类 | 创建任务、分配 Agent、设置优先级、审核结果 |

### 1.3 核心用例

**用例 1：多 Agent 并行开发**
```
用户场景：用户需要在 2 天内完成一个新功能的开发，包含前端、后端、测试三个模块。
操作路径：
1. 创建 3 个 Agent（前端专家、后端专家、测试工程师）
2. 创建 3 个任务，分别描述各模块需求
3. 将任务拖拽分配到对应 Agent
4. 在看板上观察进度，对卡住的 Agent 发送干预指令
5. Agent 完成后审核结果，合并到主分支
```

**用例 2：自动调度与故障恢复**
```
用户场景：用户提交 10 个独立的小任务，希望系统自动分配给空闲 Agent。
操作路径：
1. 批量创建 10 个任务，设置优先级
2. 选择"自动调度"策略
3. 系统根据 Agent 能力匹配和当前负载自动分配
4. 某 Agent 崩溃，系统自动重启并恢复未完成任务
5. 所有任务完成后，用户收到通知并批量审核
```

**用例 3：实时监控与干预**
```
用户场景：用户在看板上看到 Agent-2 正在修改一个关键文件，但方向似乎错误。
操作路径：
1. 点击 Agent-2 卡片，查看实时活动流
2. 发送干预消息："暂停，先阅读 wiki/concepts/auth-flow.md 再继续"
3. Agent 收到消息后调整行为
4. 或者：直接中断该 Agent 当前任务，重新分配
```

### 1.4 与现有系统的关系

本平台深度复用 Claude Code 已有的基础设施，而非从零构建：

| 平台能力 | 复用的现有代码 | 说明 |
|----------|---------------|------|
| Agent 会话管理 | `src/bridge/` 全目录 | Bridge 系统已实现完整的 session 创建、凭证获取、长轮询、心跳、归档流程 |
| Agent 定义 | `src/tools/AgentTool/builtInAgents.ts` | 已有 `BaseAgentDefinition` 类型，包含工具白名单、模型覆盖、Hook 配置等 |
| 任务状态机 | `src/Task.ts` | 已有 `TaskStateBase`、`TaskStatus`、`blocks`/`blockedBy` 依赖模型 |
| 多 Agent 编排 | `src/coordinator/coordinatorMode.ts` | 已有 orchestrator-worker 4 阶段工作流模式 |
| 实时事件流 | `src/bridge/replBridgeTransport.ts` | 已有 SSE/WebSocket 双版本传输、序列号追踪、回显去重 |
| 权限控制 | `src/permissions/` | 已有 6 种权限模式、5 层检查流程 |

**边界**：平台不修改 Claude Code 的核心执行逻辑，只在其外部提供编排、调度、观察层。

### 1.5 关键设计原则

1. **Agent 即会话**：每个 Agent 实例对应一个 Claude Code 会话进程，拥有独立的 `cse_*` 会话 ID
2. **状态外置**：Agent 内部状态通过 SDK 事件流暴露到平台，平台不侵入 Agent 进程
3. **故障隔离**：单个 Agent 崩溃不影响其他 Agent 和平台整体运行
4. **可干预优先**：用户可以随时干预、暂停、中断、重新分配，而非黑盒运行
5. **复用优先**：优先复用 Bridge API、Task 类型、Agent 定义等已有基础设施

---

## 2. 功能需求

### 2.1 Agent 池管理

#### 2.1.1 Agent 定义注册

平台维护 Agent 定义库，每个定义描述一个 Agent 的能力画像。复用 `src/tools/AgentTool/loadAgentsDir.ts` 中的 `BaseAgentDefinition` 类型体系，扩展平台特有字段：

```typescript
interface PlatformAgentDefinition extends BaseAgentDefinition {
  id: string                          // UUID
  name: string                        // 显示名称，如 "前端专家"
  description: string                 // 能力描述
  icon: string                        // 看板卡片图标
  color: AgentColorName               // 看板卡片颜色
  maxConcurrentTasks: number          // 最大并发任务数（默认 1）
  spawnTimeoutMs: number              // 启动超时（默认 30s）
  idleTimeoutMs: number               // 空闲自动回收时间（默认 1h）
  autoSpawn: boolean                  // 是否在平台启动时预创建
  // 以下继承自 BaseAgentDefinition:
  // allowedTools?: string[]
  // disallowedTools?: string[]
  // model?: string
  // maxTurns?: number
  // permissionMode?: PermissionMode
  // isolation?: 'worktree' | 'directory' | 'none'
  // mcpServers?: AgentMcpServerSpec[]
  // hooks?: HooksSettings
}
```

Agent 定义来源：
- **内置定义**：平台预置 5 种通用 Agent（通用开发、前端专家、后端专家、测试工程师、文档编写）
- **自定义定义**：用户通过界面创建，存储到数据库
- **导入定义**：从 `.claude/agents/` 目录导入已有的自定义 Agent

#### 2.1.2 Agent 实例化

Agent 实例是定义的运行时表现。实例化流程复用 `src/bridge/sessionRunner.ts` 的 `SessionSpawner` 接口：

```
Agent 实例化流程：
1. 根据定义创建 Claude Code 会话
   POST /v1/code/sessions → { session: { id: "cse_*" } }
2. 获取 Bridge 凭证
   POST /v1/code/sessions/{id}/bridge → { worker_jwt, worker_epoch, api_base_url }
3. Spawn 子进程
   设置环境变量：
     CLAUDE_CODE_SESSION_ACCESS_TOKEN={jwt}
     CLAUDE_CODE_ENVIRONMENT_KIND=bridge
     CLAUDE_CODE_USE_CCR_V2=1
   启动参数：
     --print --sdk-url {url} --session-id {id}
     --input-format stream-json --output-format stream-json
     --replay-user-messages
4. 建立 SSE 连接读取事件流
5. 注册到平台 Agent 实例注册表
```

实例生命周期：`creating → idle → running → stopping → stopped` 或 `error`

#### 2.1.3 Agent 健康监控

复用 `src/bridge/types.ts` 中的 `SessionActivity` 和 `SessionHandle` 接口：

```typescript
interface AgentHealthStatus {
  instanceId: string
  sessionId: string                // cse_*
  definitionId: string
  status: 'idle' | 'running' | 'error' | 'stopping'
  currentActivity: SessionActivity | null  // 环形缓冲区中的最新活动
  recentActivities: SessionActivity[]       // 最近 ~10 条活动
  lastHeartbeat: Date
  tokenUsage: {
    input: number
    output: number
    cacheRead: number
    cacheCreation: number
  }
  turnCount: number
  errorCount: number
  lastError: string | null
  uptime: number                    // 运行时长（ms）
}
```

健康检测机制：
- **心跳检测**：复用 `heartbeatWork` 机制，默认 120s 间隔
- **活动超时**：Agent 超过 `idleTimeoutMs` 无活动 → 标记为 idle
- **错误计数**：连续 spawn 失败 2 次 → 触发告警

#### 2.1.4 Agent 回收

Agent 空闲超时或用户手动停止时：
1. 调用 `SessionHandle.kill()` 发送 SIGTERM
2. 等待子进程退出（最多 5s）
3. 调用 `archiveSession` 归档会话记录
4. 从平台注册表移除实例
5. 更新数据库状态为 `stopped`

### 2.2 任务管理

#### 2.2.1 任务创建

复用 `src/Task.ts` 中的 `TaskStateBase` 和 `generateTaskId()` 模式，扩展平台字段：

```typescript
interface PlatformTask {
  id: string                        // 平台生成的 UUID
  taskId: string                    // 兼容 Task.ts 的 generateTaskId() 格式（如 'a1a2b3c4...'）
  title: string
  description: string               // Markdown 格式，可包含文件路径引用
  type: 'local_agent' | 'remote_agent' | 'local_bash'
  status: TaskStatus                // pending | running | completed | failed | killed
  priority: 'critical' | 'high' | 'normal' | 'low'
  assignedAgent: string | null      // Agent 实例 ID
  assignedBy: string | null         // 用户 ID
  createdAt: Date
  startedAt: Date | null
  completedAt: Date | null
  // 依赖管理（复用 Task.ts 的 blocks/blockedBy 模型）
  blockedBy: string[]               // 阻塞此任务的任务 ID 列表
  blocks: string[]                  // 被此任务阻塞的任务 ID 列表
  // 进度追踪
  progress: {
    tokenCount: number
    toolUseCount: number
    recentActivities: SessionActivity[]
    percentage: number              // 相对进度（估算）
  }
  // 输出
  outputFile: string | null         // 输出文件路径
  output: string | null             // 最终输出文本
  // 元数据
  metadata: Record<string, unknown>
  // 调度
  maxRetries: number                // 最大重试次数
  retryCount: number                // 当前重试次数
  timeoutMs: number | null          // 任务超时时间
}
```

#### 2.2.2 任务状态机

```
                    创建
                     │
                     ▼
                  ┌──────┐
                  │pending│ ← 任务已创建，等待分配
                  └──┬───┘
                     │ 分配 Agent
                     ▼
                  ┌──────┐
                  │running│ ← Agent 正在执行
                  └──┬───┘
                     │
              ┌──────┼──────┐
              ▼      ▼      ▼
           ┌────┐ ┌────┐ ┌──────┐
           │done│ │fail│ │killed│
           └────┘ └────┘ └──────┘
```

状态流转规则：
- `pending` → `running`：Agent 接受任务，开始执行
- `running` → `done`（completed）：Agent 正常完成
- `running` → `fail`（failed）：Agent 异常退出或超时
- `running` → `killed`：用户主动中断
- `fail` → `pending`：重试（retryCount < maxRetries）

#### 2.2.3 任务依赖

复用 `src/Task.ts` 的双向阻塞模型：
- `blockedBy`：这些任务完成后，本任务才能开始
- `blocks`：本任务完成后，这些任务才能开始

依赖检查（复用 `claimTask` 逻辑）：
```typescript
function canStartTask(task: PlatformTask, allTasks: Map<string, PlatformTask>): boolean {
  if (task.status !== 'pending') return false
  for (const blockerId of task.blockedBy) {
    const blocker = allTasks.get(blockerId)
    if (!blocker || blocker.status !== 'completed') return false
  }
  return true
}
```

#### 2.2.4 任务优先级

| 优先级 | 颜色 | 调度行为 |
|--------|------|----------|
| critical | 红色 | 抢占式调度，立即分配给最合适的 Agent |
| high | 橙色 | 优先调度，跳过等待队列 |
| normal | 蓝色 | 按创建顺序排队 |
| low | 灰色 | 仅在空闲时调度 |

### 2.3 调度引擎

#### 2.3.1 手动调度

用户通过看板界面手动分配：
- 拖拽任务卡片到 Agent 卡片
- 在任务详情页选择 Agent
- 支持批量分配（多选任务 → 分配到同一 Agent）

#### 2.3.2 自动调度

调度引擎根据以下因子自动选择最优 Agent：

```typescript
interface ScheduleDecision {
  taskId: string
  selectedAgent: string | null    // null = 无合适 Agent
  reason: string                  // 调度原因
  score: number                   // 匹配得分
  factors: {
    toolMatch: number             // 工具覆盖度
    loadScore: number             // 负载分数
    affinityScore: number         // 亲和性得分
    isolationScore: number        // 隔离性得分
  }
}
```

调度因子详解：

**工具覆盖度**：任务的工具需求 vs Agent 的 `allowedTools`
```
score = (任务需要的工具 ∩ Agent 允许的工具) / 任务需要的工具总数
```

**负载分数**：当前分配的任务数 / 最大并发任务数
```
score = 1 - (currentTasks / maxConcurrentTasks)
```

**亲和性得分**：相同 Git 仓库/目录的任务优先分配到同一 Agent（缓存复用）

**隔离性得分**：写操作任务分配到独立 worktree 的 Agent（复用 `spawnMode: 'worktree'`）

#### 2.3.3 调度策略

| 策略 | 适用场景 | 核心逻辑 |
|------|----------|----------|
| 能力匹配 | 任务需要特定工具 | 优先选择工具覆盖度最高的 Agent |
| 最少负载 | 通用开发任务 | 选择当前任务数最少的 Agent |
| 亲和性优先 | 同一仓库的多任务 | 优先选择已有该仓库缓存的 Agent |
| 隔离优先 | 并行写操作任务 | 分配不同 worktree 的 Agent |

用户可在设置中配置默认策略和策略权重。

### 2.4 看板界面

详见第 6 章详细设计。

### 2.5 干预与控制

#### 2.5.1 发送消息

向运行中的 Agent 发送文本指令：
- 复用 `src/bridge/types.ts` 的 `SessionHandle.writeStdin(data)` 方法
- 或通过 `sendEventToRemoteSession` API 发送 user 类型消息
- 消息显示在 Agent 的活动流中，Agent 在下一个 turn 响应

#### 2.5.2 中断执行

```
中断流程：
1. 用户点击"中断"按钮
2. 平台调用 SessionHandle.kill()
   - 向子进程发送 SIGTERM
   - 等待最多 5s
3. 如果未退出，调用 forceKill()
   - 发送 SIGKILL
4. 更新任务状态为 killed
5. 归档会话记录
```

#### 2.5.3 暂停/恢复

> **注意**：Claude Code 现有架构不支持真正的暂停/恢复。此功能通过以下模拟实现：
> - 暂停：发送"停止当前工作，保存进度到 scratchpad"指令
> - 恢复：从 scratchpad 读取进度，发送"继续之前的工作"指令

复用 `src/coordinator/coordinatorMode.ts` 中的 Scratchpad 模式（`/tmp/claude-{uid}/{cwd}/{sessionId}/scratchpad/`）。

#### 2.5.4 权限审批

当 Agent 需要用户确认权限时（如写入文件、执行命令）：
1. Agent 发出 `PermissionRequest` 事件
2. 平台通过 SSE 推送到前端
3. 前端弹出审批对话框
4. 用户选择 Allow/Deny
5. 平台通过 `sendPermissionResponseEvent` API 回复
   - 复用 `src/bridge/types.ts` 的 `BridgeApiClient.sendPermissionResponseEvent`

### 2.6 会话管理

详见第 7 章详细设计。

---

## 3. 系统架构

### 3.1 架构总览

```
┌─────────────────────────────────────────────────────────────────┐
│                        前端 Web 应用                              │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌───────────┐  │
│  │  看板面板   │  │ Agent 管理 │  │ 任务管理    │  │ 实时监控   │  │
│  │ KanbanBoard│  │ AgentPool  │  │ TaskManager│  │ LiveFeed   │  │
│  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘  └─────┬─────┘  │
│        └────────────────┴───────────────┴───────────────┘        │
│                              ↕ WebSocket                         │
├─────────────────────────────────────────────────────────────────┤
│                      调度平台后端                                 │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌───────────┐  │
│  │  API 网关   │  │ 调度引擎    │  │ 会话管理器  │  │ 事件总线   │  │
│  │  (Hono)    │  │ Scheduler  │  │ SessionMgr │  │ EventBus   │  │
│  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘  └─────┬─────┘  │
│        └────────────────┴───────────────┴───────────────┘        │
│              ↕ HTTP               ↕ SSE/HTTP                      │
├─────────────────────────────────────────────────────────────────┤
│                  Claude Code Agent 进程层                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │  Agent-1     │  │  Agent-2     │  │  Agent-N     │           │
│  │  (ChildProc) │  │  (ChildProc) │  │  (ChildProc) │           │
│  │  cse_xxx1    │  │  cse_xxx2    │  │  cse_xxxN    │           │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘           │
│         └─────────────────┴─────────────────┘                    │
│                    ↕ stdout/stdin (stream-json)                  │
├─────────────────────────────────────────────────────────────────┤
│                    Claude.ai 云端 API                             │
│  POST /v1/code/sessions          │  POST /v1/code/sessions/bridge │
│  GET  /v1/environments/poll      │  Worker events stream (SSE)    │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 核心组件

#### 3.2.1 调度服务（Scheduler）

职责：任务分配、Agent 选择、负载均衡。

```typescript
interface SchedulerService {
  // 手动调度
  assignTask(taskId: string, agentId: string): Promise<void>
  // 自动调度
  autoSchedule(): Promise<ScheduleDecision[]>
  // 检查依赖就绪的任务
  checkReadyTasks(): Promise<void>
  // 获取调度策略
  getPolicy(): SchedulePolicy
  setPolicy(policy: SchedulePolicy): Promise<void>
}
```

#### 3.2.2 会话服务（SessionManager）

职责：封装 Bridge API，管理会话生命周期。

```typescript
interface SessionManagerService {
  // 创建会话
  createSession(definition: PlatformAgentDefinition): Promise<AgentInstance>
  // 终止会话
  stopSession(instanceId: string): Promise<void>
  // 发送消息
  sendMessage(instanceId: string, message: string): Promise<void>
  // 获取实时状态
  getInstanceStatus(instanceId: string): AgentHealthStatus
  // 刷新凭证
  refreshCredentials(instanceId: string): Promise<void>
}
```

复用 `src/bridge/bridgeApi.ts` 的 `BridgeApiClient` 实现以下操作：
- `registerBridgeEnvironment(config)`
- `pollForWork(environmentId, secret, signal)`
- `acknowledgeWork(environmentId, workId, sessionToken)`
- `stopWork(environmentId, workId, force)`
- `heartbeatWork(environmentId, workId, sessionToken)`
- `archiveSession(sessionId)`
- `sendPermissionResponseEvent(sessionId, event, sessionToken)`

#### 3.2.3 事件服务（EventBus）

职责：接收 Agent 事件，转发到前端。

```typescript
interface EventBusService {
  // 发布事件
  publish(event: AgentEvent): void
  // 订阅事件（前端 WebSocket）
  subscribe(clientId: string, filter?: EventFilter): void
  // 订阅单 Agent 事件流
  subscribeAgent(agentId: string, clientId: string): void
}
```

事件类型（复用 `src/bridge/types.ts` 的 `SessionActivityType`）：
- `tool_start`：Agent 开始使用工具
- `text`：Agent 输出文本
- `result`：Agent 完成任务
- `error`：Agent 错误
- `status_change`：Agent 状态变更
- `permission_request`：需要用户审批

#### 3.2.4 存储服务（Store）

职责：任务、Agent 定义、事件记录的持久化。

技术选型：PostgreSQL（主存储）+ Redis（实时状态缓存）

### 3.3 数据流设计

```
用户创建任务 → API 网关 → 存储服务写入 tasks 表 → 看板更新
                                              │
                                    调度引擎检测新任务
                                              │
                                    选择最优 Agent
                                              │
                                    会话管理器 → Spawn 子进程
                                              │
                                    Agent 开始执行
                                              │
                                    Agent 事件 → SSE → 事件总线
                                              │
                                    WebSocket → 前端看板更新
                                              │
                                    任务完成 → 更新状态 → 归档会话
```

### 3.4 关键设计决策

#### 决策 1：Bridge 复用 vs 新建 Agent 宿主

**方案 A：复用 Bridge 进程**
- 优势：已有完整的 session 创建、凭证管理、心跳、恢复机制
- 劣势：Bridge 设计为单 worker 模式，需要改造为多 session 并发

**方案 B：新建独立 Agent 宿主服务**
- 优势：完全控制，专为多 session 并发优化
- 劣势：重复实现已有的 Bridge 功能

**决策：方案 A — 复用 Bridge 的核心组件（BridgeApiClient、SessionSpawner），但重写多 session 并发管理层。**

理由：`src/bridge/` 已经实现了 90% 的 session 生命周期管理，重写成本高于改造成本。

#### 决策 2：WebSocket vs SSE

**方案 A：WebSocket（双向）**
- 优势：单连接双向通信
- 劣势：实现复杂度高，需要处理连接保活、重连、消息排序

**方案 B：SSE + HTTP POST（分离读写）**
- 优势：与 Claude Code v2 传输一致，实现简单
- 劣势：需要两个连接

**决策：方案 B — SSE（Agent → 平台）+ HTTP POST（平台 → Agent），与现有 Claude Code v2 传输架构对齐。**

前端到平台使用 WebSocket（双向实时），因为前端需要发送操作指令。

#### 决策 3：存储选型

| 组件 | 选型 | 理由 |
|------|------|------|
| 主存储 | PostgreSQL | 事务一致性、复杂查询（任务依赖、调度因子计算） |
| 实时缓存 | Redis | Agent 状态、任务进度、事件流缓冲区 |
| 事件归档 | PostgreSQL agent_events 表 | 结构化查询，支持按 Agent/时间/类型过滤 |
| 文件输出 | 本地文件系统 | Agent 产出文件（代码、文档）存储在本地 |

---

## 4. 技术实现方案

### 4.1 技术栈选型

#### 4.1.1 前端技术栈

| 技术 | 版本 | 用途 | 选型理由 |
|------|------|------|----------|
| Next.js | 15 (App Router) | 应用框架 | SSR/SSG 支持、API Route 内置 |
| React | 19 | UI 框架 | 与 Claude Code 现有 Ink UI 一致的 React 生态 |
| TypeScript | 5.x | 类型系统 | 与 Claude Code 源码一致的类型安全要求 |
| TailwindCSS | 4 | 样式系统 | 原子化 CSS，快速迭代 |
| dnd-kit | 最新 | 拖拽交互 | 轻量、React 原生、无障碍支持 |
| Zustand | 5.x | 状态管理 | 与 Claude Code `useAppState` 模式一致 |
| React Server Components | 内置 | 服务端渲染 | 看板初始数据 SSR 加载 |

**不选 Next.js 以前的 Pages Router**：App Router 的 Server Component 模式更适合看板的初始数据加载（首屏无需 JS 即可渲染看板骨架）。

#### 4.1.2 后端技术栈

| 技术 | 版本 | 用途 | 选型理由 |
|------|------|------|----------|
| Bun | 1.x | 运行时 | 与 Claude Code 一致，启动速度快 |
| Hono | 4.x | Web 框架 | 轻量、Edge-compatible、类型安全 |
| Zod | 3.x | 数据校验 | 与 Claude Code 配置系统一致的 Zod schema |
| Drizzle ORM | 最新 | 数据库 ORM | 类型安全、与 Bun 兼容 |
| PostgreSQL | 16 | 主数据库 | 事务一致性、JSONB 支持 |
| Redis | 7 | 缓存/消息 | Agent 状态缓存、SSE 事件分发 |
| node-cron | 内置 | 定时任务 | Token 刷新、心跳检测、自动回收 |

#### 4.1.3 实时通信

| 通道 | 协议 | 方向 | 内容 |
|------|------|------|------|
| Agent → 平台 | SSE | 单向 | Agent 事件流（tool_use、text、result、error） |
| 平台 → 前端 | WebSocket | 双向 | 前端操作指令 + 实时状态推送 |
| 平台 → Agent | HTTP POST | 单向 | 权限审批回复、干预消息 |

### 4.2 API 设计

#### 4.2.1 Agent 管理 API

```
GET    /api/v1/agents                        # 列出所有 Agent 定义
POST   /api/v1/agents                        # 创建 Agent 定义
GET    /api/v1/agents/:id                    # 获取 Agent 定义详情
PATCH  /api/v1/agents/:id                   # 更新 Agent 定义
DELETE /api/v1/agents/:id                   # 删除 Agent 定义
POST   /api/v1/agents/:id/spawn              # 启动 Agent 实例
POST   /api/v1/agents/:id/kill               # 终止 Agent 实例
GET    /api/v1/agents/:id/status             # 获取 Agent 实时状态
POST   /api/v1/agents/:id/message            # 向 Agent 发送消息
GET    /api/v1/agents/:id/stream             # SSE 端点（单 Agent 事件流）
GET    /api/v1/agents/:id/activities          # 获取最近活动记录
GET    /api/v1/agents/:id/metrics            # 获取 Agent 指标（Token 使用、Turn 数）
```

#### 4.2.2 任务管理 API

```
GET    /api/v1/tasks                         # 列出所有任务（看板数据源）
POST   /api/v1/tasks                         # 创建任务
GET    /api/v1/tasks/:id                     # 获取任务详情
PATCH  /api/v1/tasks/:id                    # 更新任务（状态、优先级、描述）
DELETE /api/v1/tasks/:id                    # 删除任务
POST   /api/v1/tasks/:id/assign              # 分配任务到 Agent
POST   /api/v1/tasks/:id/stop                # 停止任务执行
POST   /api/v1/tasks/:id/retry               # 重试失败任务
GET    /api/v1/tasks/:id/output              # 获取任务输出
GET    /api/v1/tasks/:id/events              # 获取任务事件流
POST   /api/v1/tasks/batch                   # 批量创建任务
GET    /api/v1/tasks/dependencies            # 获取任务依赖图（DAG）
```

#### 4.2.3 调度 API

```
POST   /api/v1/schedule/assign               # 手动调度：分配任务到 Agent
POST   /api/v1/schedule/auto                 # 触发自动调度
GET    /api/v1/schedule/policy               # 获取调度策略
PATCH  /api/v1/schedule/policy               # 更新调度策略
GET    /api/v1/schedule/decisions            # 获取最近调度决策记录
POST   /api/v1/schedule/rebalance            # 重新平衡负载（跨 Agent 迁移任务）
```

#### 4.2.4 系统 API

```
GET    /api/v1/health                        # 健康检查
GET    /api/v1/metrics                       # 系统指标
GET    /api/v1/config                        # 获取平台配置
PATCH  /api/v1/config                        # 更新平台配置
GET    /api/v1/logs                          # 查询平台日志
DELETE /api/v1/sessions/cleanup               # 清理残留会话
POST   /api/v1/emergency/stop-all             # 紧急停止所有 Agent
```

#### 4.2.5 实时推送

```
GET    /api/v1/stream                        # WebSocket 端点（前端主连接）
GET    /api/v1/stream/events                 # SSE 端点（前端事件订阅）
```

### 4.3 数据模型

#### 4.3.1 Agent 定义表

```sql
CREATE TABLE agent_definitions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name                VARCHAR(255) NOT NULL,
    description         TEXT NOT NULL DEFAULT '',
    agent_type          VARCHAR(50) NOT NULL DEFAULT 'general-purpose',
    -- 能力定义（继承 BaseAgentDefinition）
    allowed_tools       JSONB NOT NULL DEFAULT '[]',     -- 工具白名单
    disallowed_tools    JSONB NOT NULL DEFAULT '[]',     -- 工具黑名单
    skills              JSONB NOT NULL DEFAULT '[]',     -- 预加载技能
    model               VARCHAR(50),                     -- 模型覆盖（如 'claude-sonnet-4-6'）
    max_turns           INTEGER,                         -- 最大 agentic turns
    permission_mode     VARCHAR(20) DEFAULT 'default',   -- default/plan/auto/bypassPermissions
    effort              VARCHAR(10) DEFAULT 'medium',    -- low/medium/high
    -- 平台扩展字段
    icon                VARCHAR(50) DEFAULT 'robot',
    color               VARCHAR(20) DEFAULT 'blue',
    max_concurrent_tasks INTEGER DEFAULT 1,
    spawn_timeout_ms    INTEGER DEFAULT 30000,
    idle_timeout_ms     INTEGER DEFAULT 3600000,         -- 1h 空闲回收
    auto_spawn          BOOLEAN DEFAULT FALSE,
    isolation_mode      VARCHAR(20) DEFAULT 'none',      -- none/worktree/directory
    mcp_servers         JSONB NOT NULL DEFAULT '[]',     -- Agent 专属 MCP 配置
    hooks_config        JSONB NOT NULL DEFAULT '{}',     -- Hook 配置
    initial_prompt      TEXT,                            -- 首次对话 prepend 的内容
    -- 元数据
    source              VARCHAR(20) DEFAULT 'user',      -- built_in/user/imported
    created_by          UUID,                            -- 创建者 ID
    created_at          TIMESTAMP DEFAULT NOW(),
    updated_at          TIMESTAMP DEFAULT NOW(),
    deleted_at          TIMESTAMP                        -- 软删除
);

CREATE INDEX idx_agent_definitions_source ON agent_definitions(source);
CREATE INDEX idx_agent_definitions_deleted ON agent_definitions(deleted_at) WHERE deleted_at IS NOT NULL;
```

#### 4.3.2 Agent 实例表

```sql
CREATE TABLE agent_instances (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    definition_id       UUID NOT NULL REFERENCES agent_definitions(id),
    session_id          VARCHAR(255) NOT NULL,     -- cse_* 格式
    environment_id      UUID,
    worker_epoch        INTEGER DEFAULT 0,
    status              VARCHAR(20) NOT NULL DEFAULT 'creating',
    -- 运行状态
    current_task_id     UUID,                      -- 当前执行的任务 ID
    activity_type       VARCHAR(50),               -- tool_start/text/result/error
    activity_summary    TEXT,
    -- 资源使用
    token_input         INTEGER DEFAULT 0,
    token_output        INTEGER DEFAULT 0,
    token_cache_read    INTEGER DEFAULT 0,
    token_cache_create  INTEGER DEFAULT 0,
    turn_count          INTEGER DEFAULT 0,
    -- 时间
    started_at          TIMESTAMP DEFAULT NOW(),
    completed_at        TIMESTAMP,
    last_heartbeat      TIMESTAMP DEFAULT NOW(),
    last_activity_at     TIMESTAMP DEFAULT NOW(),
    -- 错误
    error_count         INTEGER DEFAULT 0,
    last_error          TEXT,
    -- 元数据
    working_dir         VARCHAR(500),              -- 工作目录
    git_branch          VARCHAR(255),              -- 当前分支
    metadata            JSONB NOT NULL DEFAULT '{}'
);

CREATE INDEX idx_agent_instances_definition ON agent_instances(definition_id);
CREATE INDEX idx_agent_instances_status ON agent_instances(status);
CREATE INDEX idx_agent_instances_session ON agent_instances(session_id);
CREATE INDEX idx_agent_instances_heartbeat ON agent_instances(last_heartbeat);
```

#### 4.3.3 任务表

```sql
CREATE TABLE tasks (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    task_id             VARCHAR(50) UNIQUE,        -- 兼容 generateTaskId() 格式
    title               VARCHAR(500) NOT NULL,
    description         TEXT NOT NULL DEFAULT '',
    type                VARCHAR(20) DEFAULT 'local_agent',
    status              VARCHAR(20) NOT NULL DEFAULT 'pending',
    priority            VARCHAR(10) DEFAULT 'normal',
    -- 分配
    assigned_agent      UUID REFERENCES agent_instances(id),
    assigned_by         UUID,                      -- 用户 ID（手动/自动）
    assigned_at         TIMESTAMP,
    -- 时间
    created_at          TIMESTAMP DEFAULT NOW(),
    started_at          TIMESTAMP,
    completed_at        TIMESTAMP,
    -- 依赖（复用 Task.ts 的 blocks/blockedBy 模型）
    blocked_by          JSONB NOT NULL DEFAULT '[]',
    blocks              JSONB NOT NULL DEFAULT '[]',
    -- 进度
    progress_data       JSONB NOT NULL DEFAULT '{
        "tokenCount": 0,
        "toolUseCount": 0,
        "recentActivities": [],
        "percentage": 0
    }',
    -- 输出
    output_file         VARCHAR(500),
    output_text         TEXT,
    -- 重试
    max_retries         INTEGER DEFAULT 0,
    retry_count         INTEGER DEFAULT 0,
    timeout_ms          INTEGER,
    -- 通知
    notified            BOOLEAN DEFAULT FALSE,
    -- 元数据
    metadata            JSONB NOT NULL DEFAULT '{}'
);

CREATE INDEX idx_tasks_status ON tasks(status);
CREATE INDEX idx_tasks_priority ON tasks(priority);
CREATE INDEX idx_tasks_assigned_agent ON tasks(assigned_agent);
CREATE INDEX idx_tasks_assigned_agent_status ON tasks(assigned_agent, status);
CREATE INDEX idx_tasks_created_at ON tasks(created_at DESC);
```

#### 4.3.4 事件流表

```sql
CREATE TABLE agent_events (
    id                  BIGSERIAL PRIMARY KEY,
    agent_instance_id   UUID NOT NULL REFERENCES agent_instances(id),
    task_id             UUID REFERENCES tasks(id),
    event_type          VARCHAR(50) NOT NULL,       -- tool_start/text/result/error/status_change/permission_request
    event_data          JSONB NOT NULL,
    sequence_num        BIGINT NOT NULL,            -- 序列号（断点续传）
    created_at          TIMESTAMP DEFAULT NOW()
);

-- 按时间分区（假设月级分区）
-- CREATE TABLE agent_events_2026_04 PARTITION OF agent_events
--     FOR VALUES FROM ('2026-04-01') TO ('2026-05-01');

CREATE INDEX idx_agent_events_agent ON agent_events(agent_instance_id, sequence_num);
CREATE INDEX idx_agent_events_type ON agent_events(event_type);
CREATE INDEX idx_agent_events_created ON agent_events(created_at DESC);
```

#### 4.3.5 调度策略配置表

```sql
CREATE TABLE schedule_policies (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name                VARCHAR(255) NOT NULL,
    is_default          BOOLEAN DEFAULT FALSE,
    -- 策略权重（总和 100）
    tool_match_weight   INTEGER DEFAULT 40,        -- 工具覆盖度权重
    load_balance_weight INTEGER DEFAULT 30,        -- 负载均衡权重
    affinity_weight     INTEGER DEFAULT 20,        -- 亲和性权重
    isolation_weight    INTEGER DEFAULT 10,        -- 隔离性权重
    -- 约束
    max_tasks_per_agent INTEGER DEFAULT 3,         -- 单 Agent 最大并发任务
    enable_auto_schedule BOOLEAN DEFAULT FALSE,    -- 是否启用自动调度
    auto_schedule_interval_ms INTEGER DEFAULT 5000, -- 自动调度间隔
    -- 元数据
    created_by          UUID,
    created_at          TIMESTAMP DEFAULT NOW(),
    updated_at          TIMESTAMP DEFAULT NOW()
);
```

### 4.4 前端架构

#### 4.4.1 页面路由

```
/                           → 重定向到 /dashboard
/dashboard                  → 看板主页面（SSR 初始数据）
/agents                     → Agent 定义管理
/agents/:id                 → Agent 详情（实例列表、活动历史、指标）
/tasks                      → 任务列表（表格视图，支持筛选）
/tasks/:id                  → 任务详情（描述、输出、事件流）
/settings                   → 平台配置（调度策略、权限、回收策略）
/logs                       → 平台日志查询
```

#### 4.4.2 状态管理（Zustand）

复用 Claude Code 的 `useAppState` selector 模式：

```typescript
interface PlatformStore {
  // Agent
  agents: Map<string, PlatformAgentDefinition>
  agentInstances: Map<string, AgentInstance>
  agentHealth: Map<string, AgentHealthStatus>
  // 任务
  tasks: Map<string, PlatformTask>
  // 调度
  schedulePolicy: SchedulePolicy
  scheduleDecisions: ScheduleDecision[]
  // 实时
  wsConnected: boolean
  lastEventTime: Date
  // 操作
  fetchAgents: () => Promise<void>
  fetchTasks: () => Promise<void>
  assignTask: (taskId: string, agentId: string) => Promise<void>
  spawnAgent: (definitionId: string) => Promise<void>
  killAgent: (instanceId: string) => Promise<void>
  sendMessage: (instanceId: string, message: string) => Promise<void>
}
```

#### 4.4.3 组件树

```
App
├── Header                    # 项目名、仓库信息、Agent 池状态
├── KanbanBoard              # 看板主组件
│   ├── KanbanColumn          # 列（待分配/执行中/审核中/已完成/失败）
│   │   └── TaskCard          # 任务卡片
│   │       ├── TaskTitle     # 标题 + 优先级标签
│   │       ├── TaskAgent     # 分配的 Agent 信息
│   │       ├── TaskProgress  # 进度条
│   │       ├── TaskActivity  # 最近活动摘要
│   │       └── TaskActions   # 操作按钮（详情/中断/重分配）
│   └── DragOverlay           # 拖拽浮层
├── AgentPoolBar             # 底部 Agent 池状态栏
│   └── AgentPill             # 单个 Agent 状态胶囊
├── ActivityFeed             # 右侧活动流面板
│   └── ActivityItem          # 单条活动记录
├── PermissionModal           # 权限审批弹窗
├── TaskDetailDrawer         # 任务详情侧栏
└── SettingsDrawer           # 设置侧栏
```

#### 4.4.4 实时更新机制

```
SSE（Agent → 平台）
  ↓
EventBus 服务
  ↓
Redis Pub/Sub（跨进程分发）
  ↓
WebSocket Server
  ↓
前端 EventSource → Zustand store 更新 → React 重渲染
```

高频事件优化：
- 复用 `FlushGate` 模式：每 500ms 聚合一次事件
- 活动流使用虚拟滚动（最多 300 条可见）
- Token 使用指标使用 `useSyncExternalStore` selector 模式，避免全量重渲染

### 4.5 后端架构

#### 4.5.1 服务层划分

```
src/
├── api/                    # Hono Router + 路由处理
│   ├── agents.ts           # Agent 管理 API
│   ├── tasks.ts            # 任务管理 API
│   ├── schedule.ts         # 调度 API
│   ├── stream.ts           # WebSocket/SSE 端点
│   └── system.ts           # 系统 API
├── services/
│   ├── scheduler.ts        # 调度引擎
│   ├── sessionManager.ts   # 会话管理（封装 Bridge API）
│   ├── eventBus.ts         # 事件总线
│   ├── agentHealth.ts      # Agent 健康检测
│   └── tokenRefresher.ts   # JWT Token 刷新调度器
├── models/                 # Drizzle ORM 数据模型
│   ├── agentDefinitions.ts
│   ├── agentInstances.ts
│   ├── tasks.ts
│   ├── agentEvents.ts
│   └── schedulePolicies.ts
├── lib/
│   ├── bridgeClient.ts     # BridgeApiClient 封装
│   ├── sessionSpawner.ts   # SessionSpawner 封装
│   └── types.ts            # 平台核心类型定义
├── middleware/
│   ├── auth.ts             # 认证中间件
│   ├── rateLimit.ts        # 限流中间件
│   └── errorHandler.ts     # 全局错误处理
└── index.ts                # 应用入口
```

#### 4.5.2 中间件

**认证中间件**：
```typescript
// Bearer Token 认证
const authMiddleware = async (c: Context, next: () => Promise<void>) => {
  const token = c.req.header('Authorization')?.replace('Bearer ', '')
  if (!token) return c.json({ error: 'Unauthorized' }, 401)
  const user = await verifyToken(token)
  if (!user) return c.json({ error: 'Invalid token' }, 401)
  c.set('user', user)
  await next()
}
```

**错误处理中间件**：
```typescript
// 全局错误处理 + 结构化错误响应
const errorHandler = async (err: Error, c: Context) => {
  c.var.logger.error({ err, path: c.req.path })
  if (err instanceof ValidationError) {
    return c.json({ error: 'validation_error', details: err.details }, 400)
  }
  if (err instanceof SessionError) {
    return c.json({ error: 'session_error', message: err.message }, 502)
  }
  return c.json({ error: 'internal_error' }, 500)
}
```

#### 4.5.3 重试机制

所有与 Claude.ai API 的交互都实现指数退避重试：

```typescript
async function withRetry<T>(
  fn: () => Promise<T>,
  options: { maxRetries?: number; baseDelayMs?: number } = {}
): Promise<T> {
  const { maxRetries = 3, baseDelayMs = 1000 } = options
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await fn()
    } catch (err) {
      if (i === maxRetries - 1) throw err
      const delay = baseDelayMs * Math.pow(2, i) * (0.5 + Math.random())
      await sleep(delay)
    }
  }
  throw new Error('unreachable')
}
```

### 4.6 核心类型定义

```typescript
// 平台 Agent（定义 + 实例 + 健康状态的聚合视图）
interface PlatformAgent {
  definition: PlatformAgentDefinition
  instances: AgentInstance[]
  health: AgentHealthStatus | null
}

// 调度策略
interface SchedulePolicy {
  id: string
  name: string
  toolMatchWeight: number
  loadBalanceWeight: number
  affinityWeight: number
  isolationWeight: number
  maxTasksPerAgent: number
  enableAutoSchedule: boolean
  autoScheduleIntervalMs: number
}

// Agent 活动（复用 src/bridge/types.ts SessionActivity）
interface SessionActivity {
  type: 'tool_start' | 'text' | 'result' | 'error'
  summary: string
  timestamp: number
}

// 事件过滤（前端订阅用）
interface EventFilter {
  agentIds?: string[]
  eventTypes?: string[]
  taskIds?: string[]
  since?: Date
}
```

---

## 5. Agent 调度模型

### 5.1 调度器架构

调度引擎采用**中央决策 + 分布式执行**模式，复用 Claude Code Bridge 的 `pollForWork` + `acknowledgeWork` + `heartbeatWork` 协议：

```
┌─────────────────────────────────────────────────┐
│                 调度引擎 (Scheduler)               │
│                                                   │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────┐  │
│  │ 任务队列     │  │ 策略评估器   │  │ DAG 解析器│  │
│  │ (按优先级)   │  │ (打分模型)   │  │ (依赖检查) │  │
│  └──────┬──────┘  └──────┬──────┘  └─────┬────┘  │
│         └────────────────┴───────────────┘        │
│                          │ 调度决策                 │
├──────────────────────────┼────────────────────────┤
│                          ▼                        │
│  ┌──────────────┐  ┌──────────────┐               │
│  │ Agent-1      │  │ Agent-2      │               │
│  │ idle: true   │  │ idle: false  │               │
│  │ tools: [...] │  │ tools: [...] │               │
│  │ load: 0/1    │  │ load: 1/1    │               │
│  └──────────────┘  └──────────────┘               │
└─────────────────────────────────────────────────┘
```

调度循环：
```
while (true) {
  1. 扫描 pending 任务（按优先级排序）
  2. 检查依赖是否就绪（DAG 解析）
  3. 评估可用 Agent（状态=idle 或 load < max）
  4. 计算调度得分（策略加权）
  5. 选择最高分 Agent → 分配任务
  6. sleep(调度间隔)
}
```

### 5.2 调度策略详解

#### 5.2.1 手动调度

用户通过看板拖拽触发，跳过自动评估流程，直接建立 Agent-task 绑定。

前置检查：
- Agent 状态必须为 `idle` 或 `running` 且 `load < maxConcurrentTasks`
- 任务依赖必须已就绪（所有 `blockedBy` 任务状态为 `completed`）
- Agent 的工具白名单覆盖任务所需的工具

#### 5.2.2 能力匹配调度

```
得分 = (覆盖的工具数 / 需要的工具数) × toolMatchWeight
```

任务所需工具从任务描述中推导（通过关键词匹配或 LLM 分类），例如：
- "编写 React 组件" → 需要 `Edit`、`Bash`（npm/dev server）
- "数据库迁移" → 需要 `Bash`、`FileRead`、`FileEdit`
- "写测试" → 需要 `Bash`、`FileRead`、`FileWrite`

#### 5.2.3 最少负载调度

```
得分 = (1 - currentTasks / maxConcurrentTasks) × loadBalanceWeight
```

当两个 Agent 能力相同时，选择负载较低的。

#### 5.2.4 亲和性调度

```
得分 = (共享同仓库的任务数) × affinityWeight
```

同一 Git 仓库的任务优先分配到同一个 Agent，因为：
- 复用已有的文件系统缓存
- 避免多个 Agent 对同一仓库的并发写入冲突

#### 5.2.5 隔离性调度

```
得分 = (Agent 是否运行在独立 worktree) × isolationWeight
```

当任务涉及文件写入时，优先分配给 `isolation_mode = 'worktree'` 的 Agent。

### 5.3 负载均衡

#### 5.3.1 容量控制

每个 Agent 的容量由 `maxConcurrentTasks` 控制（默认 1）：
- `load < maxConcurrentTasks`：可接受新任务
- `load >= maxConcurrentTasks`：任务进入等待队列

#### 5.3.2 负载指标

```typescript
interface AgentLoadMetrics {
  currentTasks: number           // 当前任务数
  maxTasks: number               // 最大并发任务数
  tokenUsageRate: number         // Token 使用速率（tokens/min）
  turnRate: number               // Turn 速率（turns/min）
  errorRate: number              // 错误率（errors/hour）
  idleDuration: number           // 空闲时长（ms）
  utilization: number            // 利用率 = currentTasks / maxTasks
}
```

#### 5.3.3 超载处理

当 Agent 池满载时：
1. 新任务进入等待队列（按优先级排序）
2. 触发 `agent_pool_full` 告警
3. 如果等待超过阈值（5min），自动 spawn 新 Agent 实例（如果 `auto_spawn` 开启）

### 5.4 故障恢复

#### 5.4.1 崩溃检测

```
心跳超时检测：
  last_heartbeat > 120s → 标记为疑似崩溃
  last_heartbeat > 300s → 确认崩溃

活动超时检测：
  last_activity_at > idleTimeoutMs → 标记为 idle
```

#### 5.4.2 自动重启

```
崩溃恢复流程：
1. 确认 Agent 崩溃（连续 2 次心跳超时）
2. 更新实例状态为 error
3. 查找未完成任务（status = running, assignedAgent = 崩溃实例）
4. 将任务状态回退到 pending
5. Spawn 新 Agent 实例（复用同一个定义）
6. 调度引擎重新分配任务
```

复用 `src/bridge/` 的 `bridgePointer` 崩溃恢复机制：
```
写入 {sessionId, environmentId, source} 到本地文件
重启时读取指针 → reconnectSession → 恢复
```

#### 5.4.3 任务状态持久化

任务状态存储在 PostgreSQL 中，独立于 Agent 生命周期：
- Agent 崩溃不影响任务记录
- 任务可以在新 Agent 实例上恢复执行
- 输出文件（`output_file`）保持不变

### 5.5 并发控制

#### 5.5.1 文件操作冲突检测

当多个 Agent 操作同一文件时：
```
冲突场景：
  Agent-A 正在 Edit src/auth.ts
  Agent-B 也想 Edit src/auth.ts

检测机制：
  平台维护 file_locks 表：
    file_path | agent_id | locked_at

  Agent 在 Edit 前检查文件锁：
    - 无锁 → 获得锁，执行编辑
    - 有锁 → 等待或报错
```

#### 5.5.2 Worktree 隔离

复用 `spawnMode: 'worktree'` 模式，为每个 Agent 创建独立的 Git worktree：
```
.git/worktrees/
├── agent-1-abc123/    → Agent-1 的工作目录
├── agent-2-def456/    → Agent-2 的工作目录
└── agent-3-ghi789/    → Agent-3 的工作目录
```

每个 worktree 对应一个独立分支，Agent 完成后通过 PR 合并到主分支。

#### 5.5.3 DAG 调度

任务依赖图使用拓扑排序：
```
A ──→ B ──→ D
      │
C ────┘

执行顺序：
1. A 和 C 可并行（无依赖）
2. A 和 C 都完成后，B 开始
3. B 完成后，D 开始
```

复用 `src/Task.ts` 的 `claimTask` 检查逻辑：
```typescript
function claimTask(taskId: string, agentId: string): boolean {
  const task = getTask(taskId)
  const agent = getAgent(agentId)
  if (!task || !agent) return false
  // 检查阻塞
  for (const blockerId of task.blockedBy) {
    if (getTask(blockerId)?.status !== 'completed') return false
  }
  // 检查 Agent 是否繁忙
  if (agent.currentTasks >= agent.maxConcurrentTasks) return false
  // 分配
  assignTask(taskId, agentId)
  return true
}
```

---

## 6. 看板界面设计

### 6.1 整体布局

```
┌──────────────────────────────────────────────────────────────────────┐
│ Header                                                               │
│ 📋 Claude Code Dispatch Platform  |  📂 my-project (main) | 3/5 忙碌 │
│                                                [⚙️ 设置] [👤 Admin] │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─ 待分配 (2) ─┐ ┌─ 执行中 (3) ─┐ ┌─ 审核中 (1) ─┐ ┌─ 已完成 ─┐   │
│  │              │ │              │ │              │ │          │   │
│  │ ┌──────────┐ │ │ ┌──────────┐ │ │ ┌──────────┐ │ │ ┌──────┐ │   │
│  │ │ 编写登录页│ │ │ │ 编写API   │ │ │ │ 写单元测试│ │ │ │ 需求  │ │   │
│  │ │ Agent-1  │ │ │ │ Agent-2  │ │ │ │ Agent-3  │ │ │ │ 分析  │ │   │
│  │ │ 🔴 高    │ │ │ │ 🟢 普通  │ │ │ │ 🔴 高    │ │ │ │      │ │   │
│  │ │ ████░ 60%│ │ │ │ ██░░░ 30%│ │ │ │ ⏳ 等待  │ │ │ │ 完成 │ │   │
│  │ └──────────┘ │ │ └──────────┘ │ │ └──────────┘ │ │ └──────┘ │   │
│  │ ┌──────────┐ │ │ ┌──────────┘ │ │              │ │ ┌──────┐ │   │
│  │ │ 数据库迁移│ │ │ │ ┌────────┐ │ │              │ │ │ 文档  │ │   │
│  │ │ 未分配    │ │ │ │ 修复Bug │ │ │              │ │ │ 编写  │ │   │
│  │ │ ⚪ 低     │ │ │ │ Agent-2 │ │ │              │ │ │      │ │   │
│  │ │           │ │ │ │ █████ 95%│ │ │              │ │ │ 完成 │ │   │
│  │ └──────────┘ │ │ └────────┘ │ │              │ │ └──────┘ │   │
│  └──────────────┘ └──────────────┘ └──────────────┘ └──────────┘   │
│                                                                      │
├──────────────────────────────────────────────────────────────────────┤
│ Agent 池                                                             │
│ [🟢 Agent-1: 编写登录页] [🟢 Agent-2: 2 tasks] [◌ Agent-3: idle]    │
│ [🟢 Agent-4: 文档编写] [◌ Agent-5: idle]                    [+ 新增] │
├──────────────────────────────────────────────────────────────────────┤
│ Activity Feed (最近 50 条)                                           │
│ [14:32:01] 🟢 Agent-2 正在编辑 src/api/auth.ts...                    │
│ [14:31:45] 🔴 Agent-1 完成: 编写登录页 (60%, 3200 tokens)            │
│ [14:31:30] ⚠️ Agent-3 等待审核: 单元测试编写完成                      │
│ [14:30:15] ◌ Agent-5 空闲超过 1h                                    │
└──────────────────────────────────────────────────────────────────────┘
```

### 6.2 响应式适配

| 屏幕宽度 | 布局 |
|----------|------|
| > 1400px | 6 列看板 + 右侧活动流 |
| 1024-1400px | 6 列看板（活动流折叠为抽屉） |
| 768-1024px | 2×3 网格布局 |
| < 768px | 单列可切换视图（只看板/只看 Agent/只看活动） |

### 6.3 TaskCard 组件设计

```typescript
interface TaskCardProps {
  task: PlatformTask
  agent?: PlatformAgent
  onAssign: (taskId: string, agentId: string) => void
  onStop: (taskId: string) => void
  onRetry: (taskId: string) => void
  onViewDetail: (taskId: string) => void
}
```

视觉规范：
- **优先级标签**：红色（critical）、橙色（high）、蓝色（normal）、灰色（low）
- **进度条**：高度 4px，颜色随进度变化（红色 → 黄色 → 绿色）
- **Agent 信息**：头像 + 名称，点击跳转到 Agent 详情
- **活动摘要**：最新一条 `SessionActivity.summary`，超过 2 行截断
- **悬停操作**：查看详情、中断、重新分配（仅悬停时显示）

### 6.4 实时更新机制

```
Agent 事件 → SSE → 后端 EventBus → Redis Pub/Sub
  → WebSocket Server → 前端 EventSource
    → Zustand store dispatch
      → React re-render (selected columns only)
```

优化措施：
- **列级更新**：仅更新状态发生变化的列，非全量重渲染
- **卡片虚拟化**：每列超过 20 个卡片启用虚拟滚动
- **事件聚合**：复用 `FlushGate` 模式，500ms 窗口内的事件合并为一次更新
- **进度动画**：进度条使用 CSS transition，避免 React 状态变化导致的重渲染

### 6.5 Agent 池状态栏

每个 Agent 显示为一个"状态胶囊"：
```
[🟢 名称] [当前活动摘要] [进度条/空闲标记]
```

颜色编码：
- 🟢 绿色：running，正常工作
- 🟡 黄色：idle，等待任务
- 🔴 红色：error，异常
- ⚪ 灰色：stopped，已停止
- 💜 紫色：审核中（等待用户审批）

交互：
- 点击 Agent 胶囊 → 打开 Agent 详情抽屉
- 拖拽任务到 Agent 胶囊 → 分配任务
- 右键菜单 → 终止/重启/查看指标

### 6.6 权限审批弹窗

当 Agent 需要用户确认时：

```
┌─────────────────────────────────────┐
│  🔒 权限请求                         │
│                                     │
│  Agent-2 请求执行:                   │
│  Write → src/components/AuthForm.tsx│
│                                     │
│  原因: "需要创建登录表单组件"         │
│                                     │
│  [✅ 允许]    [❌ 拒绝]    [📝 备注]  │
└─────────────────────────────────────┘
```

复用 `src/permissions/` 的 5 层检查流程信息，在弹窗中展示：
- 请求的操作类型（文件写入/命令执行）
- 目标路径
- 匹配的规则（如果有）
- 安全风险评估

---

## 7. 会话管理

### 7.1 会话生命周期

```
创建 → 注册环境 → 获取凭证 → Spawn 子进程 → 运行 → (心跳维持) → 完成/失败 → 归档

状态流转：
creating → idle → running → stopping → stopped
    ↓         ↓        ↓         ↓
  error ← error ←  error ← error
```

### 7.2 会话创建流程（5 步）

```
Step 1: 创建 Claude Code 会话
  POST /v1/code/sessions
  Body: { title: "Agent-{id}", bridge: {}, tags: ["dispatch-platform"] }
  Response: { session: { id: "cse_xxxxx" } }

Step 2: 获取 Bridge 凭证
  POST /v1/code/sessions/cse_xxxxx/bridge
  Auth: OAuth Token (from Claude AI)
  Response: {
    worker_jwt: "eyJ...",
    worker_epoch: 1,
    api_base_url: "https://api.claude.ai/api",
    expires_in: 3600
  }

Step 3: Spawn 子进程
  环境变量:
    CLAUDE_CODE_SESSION_ACCESS_TOKEN={worker_jwt}
    CLAUDE_CODE_ENVIRONMENT_KIND=bridge
    CLAUDE_CODE_USE_CCR_V2=1
    CLAUDE_CODE_POST_FOR_SESSION_INGRESS_V2=1
    CLAUDE_CODE_WORKER_EPOCH={worker_epoch}
  启动命令:
    claude --print \
      --sdk-url {api_base_url} \
      --session-id cse_xxxxx \
      --input-format stream-json \
      --output-format stream-json \
      --replay-user-messages \
      --permission-mode {permissionMode}

Step 4: 建立 SSE 连接
  GET /worker/events/stream
  Headers: Authorization: Bearer {worker_jwt}
  → 读取 NDJSON 事件流

Step 5: 注册到平台
  写入 agent_instances 表
  更新 Zustand store
  在看板显示
```

### 7.3 会话恢复

利用 `bridgePointer` 机制：
```
正常情况：
  agent_instances 表中有完整记录

进程崩溃：
  1. 读取本地 bridgePointer 文件
     {sessionId, environmentId, source}
  2. 调用 reconnectSession
     POST /v1/environments/{id}/bridge/reconnect
     Body: { session_id: "cse_xxxxx" }
  3. 停止残留 worker
  4. 重新排队 session
  5. 重新 pollForWork → 获取新的 WorkResponse
  6. Spawn 新进程接管
```

### 7.4 Token 管理

JWT Token 生命周期：
```
获取凭证 → worker_jwt (expires_in: 3600s)
  ↓
Token 刷新调度器（提前 5 分钟刷新）
  ↓
调用 /bridge → 获取新的 worker_jwt + worker_epoch
  ↓
更新运行中会话的凭证 (updateAccessToken)
  ↓
重建 SSE 连接（因为 worker_epoch 变了）
```

刷新调度器：
```typescript
class TokenRefresher {
  private timer: NodeJS.Timeout

  scheduleRefresh(instance: AgentInstance) {
    const expiresIn = instance.credentialExpiresIn  // 从凭证获取
    const refreshAt = expiresIn - 300  // 提前 5 分钟
    this.timer = setTimeout(() => {
      this.refresh(instance)
    }, refreshAt * 1000)
  }

  async refresh(instance: AgentInstance) {
    const creds = await this.sessionApi.getBridgeCredentials(instance.sessionId)
    await this.db.updateAgentCredentials(instance.id, creds)
    // worker_epoch 可能变化，需要重建传输
    if (creds.worker_epoch !== instance.workerEpoch) {
      await this.rebuildTransport(instance, creds)
    }
  }
}
```

### 7.5 会话终止

| 终止原因 | 状态 | 流程 |
|----------|------|------|
| 正常完成 | completed | Agent 输出 result → 自动归档 |
| 用户中断 | killed | kill() → SIGTERM → forceKill() → 归档 |
| 异常退出 | failed | 子进程 exit code != 0 → 归档 |
| 超时 | failed | DEFAULT_SESSION_TIMEOUT_MS = 24h → kill → 归档 |
| 空闲回收 | stopped | idleTimeoutMs 到期 → kill → 归档 |

归档流程（复用 `archiveSession` API）：
```
1. 确保子进程已终止
2. POST /v1/sessions/{id}/archive
3. 更新 agent_instances.status = stopped
4. 释放 Agent 实例容量
5. 如果有关联任务，更新任务状态
```

---

## 8. 安全与权限

### 8.1 认证体系

三层认证模型：

| 层级 | 认证方式 | 用途 |
|------|----------|------|
| 用户 → 平台 | OAuth 2.0 / Bearer Token | 前端用户访问平台 API |
| 平台 → Claude.ai API | OAuth Token (from `getClaudeAIOAuthTokens`) | 创建会话、获取凭证 |
| 平台 → Agent 进程 | JWT + worker_epoch | 会话认证、SSE 连接 |

#### 8.1.1 用户认证

```
用户登录 → OAuth 授权 → 获取 access_token
  → 平台签发 JWT (有效期 24h)
    → 前端存储 JWT
      → 所有 API 请求携带 Authorization: Bearer {jwt}
```

#### 8.1.2 Agent 认证

Agent 进程通过环境变量获取 JWT：
```
CLAUDE_CODE_SESSION_ACCESS_TOKEN={worker_jwt}
```

平台通过 SSE 连接时：
```
GET /worker/events/stream
Authorization: Bearer {worker_jwt}
```

### 8.2 权限控制

#### 8.2.1 平台级 RBAC

| 角色 | 权限 |
|------|------|
| Admin | 全部操作（管理 Agent 定义、调度策略、系统配置） |
| Manager | 创建/编辑/分配任务、启动/终止 Agent、审批权限 |
| Viewer | 只读看板、查看活动流、查看 Agent 指标 |

#### 8.2.2 Agent 级权限模式

复用 Claude Code 的 6 种权限模式：

| 模式 | 行为 | 适用场景 |
|------|------|----------|
| default | 文件读取自动允许，写入需确认 | 通用开发 |
| plan | 进入计划模式，先输出方案再执行 | 复杂任务审核 |
| acceptEdits | 自动允许文件写入 | 可信 Agent |
| bypassPermissions | 跳过所有权限检查 | 完全信任 |
| dontAsk | 自动拒绝需要确认的操作 | 只读 Agent |
| auto | AI 分类器自动判断 | 平衡安全与效率 |

#### 8.2.3 文件操作权限

复用 5 层检查流程：
```
1. Deny rules → 明确禁止的路径匹配
2. Internal paths → 保护 .claude/ 等内部目录
3. Safety checks → 危险文件检测 (.gitconfig, .bashrc)
4. Working dir → 确保操作在允许的工作目录内
5. Allow rules → 明确允许的路径匹配
→ Default → 根据权限模式决定
```

### 8.3 沙箱隔离

| 隔离级别 | 实现 | 效果 |
|----------|------|------|
| 目录隔离 | 每个 Agent 独立目录或 worktree | 文件操作互不影响 |
| 进程隔离 | macOS seatbelt / Linux bubblewrap | 系统调用受限 |
| 网络隔离 | 可配置的 `allowedDomains` / `deniedDomains` | 控制出站请求 |
| 上下文隔离 | 每个 Agent 独立 ToolUseContext | 不共享内部状态 |

### 8.4 安全边界

```
┌─────────────────────────────────────┐
│         平台 (Trusted)               │
│  ┌───────────────────────────────┐  │
│  │     Agent 进程 (Semi-Trusted)  │  │
│  │                               │  │
│  │  Claude AI API (External)     │  │
│  │                               │  │
│  └───────────────────────────────┘  │
│                                     │
│  本地文件系统 (Trusted boundary)     │
└─────────────────────────────────────┘
```

安全原则：
1. Agent 不能互相访问彼此的上下文
2. 平台不能直接访问 Agent 的文件系统（仅通过 SDK 消息交互）
3. 敏感操作（文件写入、命令执行）在本地执行，不经过平台
4. 平台到 Agent 的通信通过 SSE（单向读取）+ HTTP POST（写入）
5. Agent 凭证（JWT）仅用于 SSE 连接，不暴露给前端

### 8.5 速率限制

| 端点 | 限制 | 窗口 |
|------|------|------|
| /api/v1/agents/spawn | 10 次 | 5 分钟 |
| /api/v1/tasks | 100 次 | 5 分钟 |
| /api/v1/schedule/auto | 20 次 | 5 分钟 |
| /api/v1/stream (WebSocket) | 5 连接 | 每用户 |
| SSE 连接 | 10 连接 | 每用户 |

实现：滑动窗口计数器，基于 Redis `INCR` + `EXPIRE`。

---

## 9. 监控与可观测性

### 9.1 日志系统

#### 9.1.1 Agent 日志

复用 `src/bridge/types.ts` 的 `SessionHandle.lastStderr` 环形缓冲区：

```typescript
interface AgentLogEntry {
  timestamp: Date
  level: 'info' | 'warn' | 'error'
  instanceId: string
  source: 'stderr' | 'platform' | 'bridge' | 'claude-api'
  message: string
  context?: Record<string, unknown>
}
```

Agent stderr 通过 `lastStderr` 环形缓冲区（默认容量 10 行）捕获：
```typescript
// 在 SessionSpawner 中解析 NDJSON 时捕获 stderr
if (line.type === 'status' && line.subtype === 'error') {
  handle.stderr.push(line.message)
  if (handle.stderr.length > 10) handle.stderr.shift()
}
```

#### 9.1.2 平台日志

结构化 JSON 日志（Bun 内置 `console.log` 或 pino）：
```json
{
  "level": "error",
  "time": "2026-04-15T14:32:01.123Z",
  "service": "session-manager",
  "instanceId": "uuid-xxx",
  "sessionId": "cse_xxxxx",
  "message": "Failed to refresh credentials",
  "error": "401 Unauthorized",
  "stack": "..."
}
```

日志级别：
- `error`：需要立即处理的问题（spawn 失败、凭证过期）
- `warn`：需要关注的问题（心跳延迟、Token 即将过期）
- `info`：正常操作的记录（Agent 启动、任务分配、会话归档）
- `debug`：详细的调试信息（HTTP 请求/响应、SSE 事件）

#### 9.1.3 会话日志

复用 transcript 文件机制（`recordSidechainTranscript`）：
- 每个会话的 SDK 消息记录到本地 JSON 文件
- 文件格式：`.claude/transcripts/{sessionId}.jsonl`
- 包含完整的对话历史、工具调用、结果
- 用于事后审计和问题排查

### 9.2 指标采集

#### 9.2.1 Agent 指标

复用 `src/` 的 `StatsStore` 模式进行指标聚合：

| 指标 | 类型 | 采集方式 |
|------|------|----------|
| agent.active | Gauge | 状态为 running 的 Agent 数 |
| agent.idle | Gauge | 状态为 idle 的 Agent 数 |
| agent.error | Counter | spawn 失败次数 |
| agent.token.input | Counter | 累计输入 Token |
| agent.token.output | Counter | 累计输出 Token |
| agent.token.cache_read | Counter | 缓存读取 Token |
| agent.token.cache_create | Counter | 缓存创建 Token |
| agent.turns | Counter | 累计 Turn 数 |
| agent.uptime | Gauge | 运行时长（s） |
| agent.load | Gauge | 当前任务数 |

#### 9.2.2 任务指标

| 指标 | 类型 | 采集方式 |
|------|------|----------|
| task.created | Counter | 任务创建数 |
| task.completed | Counter | 任务完成数 |
| task.failed | Counter | 任务失败数 |
| task.killed | Counter | 任务被中止数 |
| task.duration | Histogram | 任务完成时长 |
| task.queue_depth | Gauge | 等待分配的任务数 |
| task.retry | Counter | 任务重试次数 |

#### 9.2.3 系统指标

| 指标 | 类型 | 采集方式 |
|------|------|----------|
| api.request_duration | Histogram | API 请求延迟 |
| api.error_rate | Gauge | API 错误率 |
| sse.connections | Gauge | 活跃 SSE 连接数 |
| ws.connections | Gauge | 活跃 WebSocket 连接数 |
| db.pool.active | Gauge | 数据库活跃连接数 |
| db.pool.idle | Gauge | 数据库空闲连接数 |
| db.query_duration | Histogram | 数据库查询延迟 |
| redis.operations | Counter | Redis 操作次数 |

### 9.3 告警规则

复用 `src/bridge/` 的 `BridgeFatalError` 错误分类机制：

| 告警 | 条件 | 级别 | 动作 |
|------|------|------|------|
| Agent 崩溃 | 连续 2 次 spawn 失败 | Critical | 邮件/Slack 通知，暂停自动分配 |
| 任务超时 | 运行超过预估时间 2 倍 | Warning | 看板标记，通知 Manager |
| 资源满载 | Agent 池满载 10 分钟 | Warning | 建议扩容 |
| Token 预算超限 | 累计 Token 使用超过日预算 | Critical | 暂停所有新任务创建 |
| 凭证过期 | JWT 过期且无法刷新 | Critical | 检查 OAuth 配置 |
| 数据库连接耗尽 | 连接池满载 | Critical | 紧急排查 |

### 9.4 健康检查

#### 9.4.1 Agent 心跳

复用 `heartbeatWork` 机制：
```
每 120s 调用一次 heartbeatWork
Response: { lease_extended: boolean, state: string }

如果 lease_extended = false → Agent 可能已崩溃
如果 state = "stopped" → Agent 已停止
```

#### 9.4.2 平台健康端点

```
GET /api/v1/health
Response:
{
  "status": "healthy" | "degraded" | "unhealthy",
  "checks": {
    "database": { "status": "ok", "latency_ms": 5 },
    "redis": { "status": "ok", "latency_ms": 1 },
    "claude_api": { "status": "ok", "last_check": "..." },
    "agent_pool": {
      "total": 5,
      "active": 3,
      "idle": 2,
      "error": 0
    },
    "task_queue": { "pending": 2, "running": 3 },
    "uptime_seconds": 86400
  }
}
```

#### 9.4.3 数据库健康

```
连接池状态：
  active / max 连接数
  等待获取连接的请求数

慢查询检测：
  超过 500ms 的查询记录
  按 SQL 模式聚合延迟统计
```

### 9.5 遥测集成

#### 9.5.1 OpenTelemetry

复用 `chapter-28` 的 OpenTelemetry 集成：
```typescript
import { trace, metrics } from '@opentelemetry/api'

const tracer = trace.getTracer('dispatch-platform')
const meter = metrics.getMeter('dispatch-platform')

// 分布式追踪
const span = tracer.startSpan('task.assign')
span.setAttribute('task.id', taskId)
span.setAttribute('agent.id', agentId)
// ... 操作
span.end()

// 自定义指标
const taskCounter = meter.createCounter('task.completed')
taskCounter.add(1, { priority: task.priority })
```

#### 9.5.2 采样策略

复用 GrowthBook 采样策略：
- 生产环境：10% 采样（减少存储成本）
- 错误请求：100% 采样
- 关键操作（spawn/kill）：100% 采样

#### 9.5.3 导出目标

| 目标 | 协议 | 内容 |
|------|------|------|
| Datadog | OTLP | Traces + Metrics + Logs |
| Prometheus | Pull | Metrics |
| 本地文件 | JSONL | Logs（开发环境） |

---

## 10. 部署与运维

### 10.1 部署架构

```
                    用户浏览器
                         │
                         ▼
                      CDN (静态资源)
                         │
                         ▼
                  ┌─────────────┐
                  │  API Gateway │ (Nginx / Cloudflare)
                  │  - TLS 终止  │
                  │  - 限流      │
                  │  - 路由      │
                  └──────┬──────┘
                         │
                         ▼
              ┌──────────────────────┐
              │  调度平台 Pod (K8s)    │
              │                      │
              │  ┌────────────────┐  │
              │  │ Hono API Server │  │
              │  │ (Bun, 多实例)    │  │
              │  └────────────────┘  │
              │  ┌────────────────┐  │
              │  │ Agent Manager   │  │
              │  │ (运行 Claude CLI)│  │
              │  └────────────────┘  │
              │  ┌────────────────┐  │
              │  │ Scheduler       │  │
              │  │ (定时任务)       │  │
              │  └────────────────┘  │
              │  ┌────────────────┐  │
              │  │ EventBus        │  │
              │  │ (SSE + WS)      │  │
              │  └────────────────┘  │
              └──────────────────────┘
                         │
              ┌──────────┼──────────┐
              ▼          ▼          ▼
         PostgreSQL   Redis   对象存储
         (RDS)       (EC)    (S3, 可选)
```

### 10.2 基础设施要求

#### 10.2.1 计算资源

| 组件 | CPU | 内存 | 存储 | 数量 |
|------|-----|------|------|------|
| API Server | 1 核 | 512MB | - | 2+ |
| Agent Manager | 2 核/Agent | 4GB/Agent | 10GB/Agent | N |
| Scheduler | 0.5 核 | 256MB | - | 1 |
| EventBus | 1 核 | 1GB | - | 1 |

**关键约束**：每个 Agent 进程需要 1-2 CPU 核心 + 2-4GB 内存（Claude Code CLI 的最低要求）。

#### 10.2.2 数据库资源

| 组件 | 规格 | 说明 |
|------|------|------|
| PostgreSQL | 2 核 4GB, 100GB SSD | 主存储，RDS 或自建 |
| Redis | 1 核 2GB | 缓存 + Pub/Sub + 限流 |
| 对象存储 (可选) | S3 兼容 | 会话 transcript 归档 |

#### 10.2.3 网络要求

- 出站：访问 `api.anthropic.com`（Claude API）、`claude.ai`（OAuth/Bridge API）
- 入站：用户浏览器 → CDN → API Gateway（HTTPS 443）
- 内部：Pod 间通信（VPC 内网）

### 10.3 扩展策略

#### 10.3.1 水平扩展（API 层）

API Server 无状态，可水平扩展：
```
Nginx → [API-1, API-2, API-3]
```

会话亲和性：WebSocket 连接需要 sticky session（基于 agentInstanceId 的 cookie 或 header）。

#### 10.3.2 Agent 扩展

Agent Manager 按 capacity 自动扩缩容：
```
if (pending_tasks > idle_agents * 2 && total_agents < max_agents) {
  spawn_new_agent()
}
if (idle_agents > active_agents * 2 && total_agents > min_agents) {
  recycle_idle_agent()
}
```

#### 10.3.3 数据库扩展

- **读写分离**：主库写、只读副本读（看板查询）
- **连接池**：PgBouncer 或 Drizzle 内置连接池
- **分片**：event 表按时间分区（月度分区），历史数据归档

### 10.4 CI/CD 流程

```
开发者 push → GitHub Actions
  ↓
  1. 类型检查 (tsc --noEmit)
  2. 单元测试 (vitest)
  3. 集成测试 (本地 PostgreSQL + Redis)
  4. 构建 Docker 镜像
  5. 推送到 ECR
  ↓
  6. 部署到 staging 环境
  7. 冒烟测试（自动创建/销毁测试 Agent）
  8. 生产发布（滚动更新）
```

Agent 配置通过 GitOps 管理：
- `agent-definitions/` 目录下的 YAML 文件
- PR 变更 → 审查 → 合并 → 自动同步到数据库

### 10.5 备份与恢复

#### 10.5.1 数据库备份

| 类型 | 频率 | 保留 | 工具 |
|------|------|------|------|
| 全量备份 | 每日 | 30 天 | pg_dump |
| WAL 归档 | 连续 | 7 天 | WAL-G |
| 事件归档 | 月度分区 | 1 年 | 移动分区到冷存储 |

#### 10.5.2 Agent 配置版本化

- Agent 定义每次变更创建新版本
- 支持回滚到历史版本
- 版本信息存储在 `agent_definition_versions` 表

#### 10.5.3 会话输出归档

- Agent 产出的代码文件存储在本地目录
- 会话归档时打包为 tar.gz
- 可选上传到 S3 进行长期存储

### 10.6 运维手册

#### 10.6.1 Agent 池扩容

```
1. 登录平台管理界面
2. 进入设置 → Agent 池
3. 点击"新增 Agent"
4. 选择 Agent 定义、设置数量
5. 确认后批量启动
```

#### 10.6.2 会话异常排查

```
1. 查看 Agent 活动流（最近 10 条活动）
2. 查看 stderr 环形缓冲区
3. 查看任务事件流（agent_events 表）
4. 查看会话 transcript（.claude/transcripts/{sessionId}.jsonl）
5. 检查 Claude.ai API 状态（是否有 429/500）
```

#### 10.6.3 紧急停止所有 Agent

```
1. 调用 POST /api/v1/emergency/stop-all
   - 向所有 Agent 发送 SIGTERM
   - 等待 10s
   - SIGKILL 未退出的进程
2. 清理残留会话
   - 调用 DELETE /api/v1/sessions/cleanup
3. 停止调度引擎
   - 更新 schedule_policies.enable_auto_schedule = false
4. 调查根因
5. 修复后重启
```

---

## 11. 开发计划

### 11.1 Phase 1：基础架构（Week 1-3）

**目标**：搭建平台骨架，实现基本 CRUD 和看板静态展示。

| 任务 | 交付物 | 验证标准 |
|------|--------|----------|
| 搭建后端框架 | Hono + PostgreSQL + Drizzle 项目骨架 | `bun run dev` 启动成功，健康端点返回 200 |
| 数据库 Schema | 5 张核心表的 DDL + 迁移脚本 | `drizzle-kit push` 成功执行 |
| Agent 定义 CRUD | POST/GET/PATCH/DELETE /api/v1/agents | 创建/查询/更新/删除 Agent 定义 |
| 任务管理 CRUD | POST/GET/PATCH/DELETE /api/v1/tasks | 创建/查询/更新/删除任务 |
| 前端骨架 | Next.js App Router + 看板布局 | 看板页面可访问，6 列显示 |
| 看板静态数据 | 从 API 读取任务数据渲染看板 | 卡片正确显示任务信息 |

**里程碑 M1**（Week 3 结束）：基本 CRUD + 看板静态展示可用。

### 11.2 Phase 2：会话集成（Week 4-6）

**目标**：Agent 可运行，状态实时同步到看板。

| 任务 | 交付物 | 验证标准 |
|------|--------|----------|
| 封装 Bridge API 客户端 | bridgeClient.ts（复用 createBridgeApiClient） | 可创建会话、获取凭证、spawn 子进程 |
| Agent spawn 生命周期 | SessionManager.createSession/stopSession | spawn → 运行 → 停止全流程可用 |
| SSE 事件流接收 | EventBus 接收 Agent 事件 | agent_events 表正确记录事件 |
| SSE → WebSocket → 前端 | 实时推送链路打通 | 看板状态随 Agent 活动实时更新 |
| 凭证刷新调度器 | TokenRefresher 服务 | JWT 过期前自动刷新，连接不中断 |

**里程碑 M2**（Week 6 结束）：Agent 可运行、状态实时同步到看板。

### 11.3 Phase 3：调度引擎（Week 7-9）

**目标**：自动调度可用，任务依赖按 DAG 执行。

| 任务 | 交付物 | 验证标准 |
|------|--------|----------|
| 手动调度（拖拽） | dnd-kit 集成 + assign API | 拖拽任务到 Agent 卡片 → 任务分配成功 |
| 自动调度引擎 | Scheduler 服务（4 种策略） | 调用 /schedule/auto → 返回调度决策 |
| 任务依赖 DAG 执行 | blockedBy/blocks 解析 + 拓扑排序 | 创建有依赖的任务 → 依赖完成后自动触发 |
| Agent 故障检测与恢复 | 心跳超时检测 + 自动重启 | 手动 kill Agent 进程 → 平台检测并自动重启 |
| 调度策略配置界面 | 设置页面 | 可调整策略权重，保存后生效 |

**里程碑 M3**（Week 9 结束）：自动调度可用、任务依赖 DAG 执行、故障自动恢复。

### 11.4 Phase 4：干预与控制（Week 10-12）

**目标**：完整干预能力，权限审批可用。

| 任务 | 交付物 | 验证标准 |
|------|--------|----------|
| 向 Agent 发送消息 | sendMessage API + 前端输入框 | Agent 收到消息并在下一 turn 响应 |
| 任务中断 | stopTask API + kill 按钮 | 点击中断 → Agent 停止 → 任务状态 killed |
| 权限审批界面 | PermissionModal 组件 + 审批 API | Agent 请求权限 → 前端弹窗 → 用户审批 |
| 暂停/恢复（模拟） | Scratchpad 模式 | 暂停 → 保存进度 → 恢复 → 继续工作 |
| 错误处理与重试 | 全局错误中间件 + 任务重试 | 任务失败 → 重试按钮 → 重新分配执行 |

**里程碑 M4**（Week 12 结束）：完整干预能力、权限审批、模拟暂停恢复。

### 11.5 Phase 5：生产就绪（Week 13-15）

**目标**：生产环境部署，安全加固。

| 任务 | 交付物 | 验证标准 |
|------|--------|----------|
| 认证和授权 | OAuth + JWT + RBAC | 不同角色访问受限 |
| 速率限制 | Redis 滑动窗口限流 | 超过限制 → 429 Too Many Requests |
| 安全加固 | 沙箱配置、输入校验、CSP | 安全扫描通过 |
| 监控告警 | OpenTelemetry + 告警规则 | 模拟崩溃 → 告警触发 |
| 性能优化 | 数据库索引、缓存、虚拟滚动 | 看板 100 任务 < 1s 加载 |
| 文档和运维手册 | README、API 文档、Runbook | 新成员可按文档部署和运维 |

**里程碑 M5**（Week 15 结束）：生产环境部署。

### 11.6 风险与缓解

| 风险 | 影响 | 概率 | 缓解策略 |
|------|------|------|----------|
| Claude API 限流 | Agent 无法启动 | 中 | 实现请求队列 + 指数退避重试 |
| Agent 内存泄漏 | 长时间运行崩溃 | 低 | 设置 idleTimeoutMs 自动回收 |
| 数据库连接耗尽 | 平台不可用 | 低 | 连接池 + PgBouncer + 告警 |
| 并发写入冲突 | Agent 间文件冲突 | 中 | worktree 隔离 + 文件锁 |
| 前端性能 | 大量任务导致卡顿 | 低 | 虚拟化 + 事件聚合 + 分页 |
| 凭证过期 | SSE 连接断开 | 低 | Token 刷新调度器 + 自动重连 |

### 11.7 里程碑总结

```
Week 1 ── M1 ── M2 ── M3 ── M4 ── M5 ── Week 15
         CRUD   Agent  调度   干预   生产
         看板   可运行  引擎   控制   就绪
```

每个里程碑都是一次内部 Demo，展示给利益相关方审查。

---

## 附录

### A. 术语表

| 术语 | 说明 |
|------|------|
| Agent | Claude Code 会话的封装，代表一个"AI 工人" |
| Agent 定义 | Agent 的能力画像（工具白名单、模型、权限等） |
| Agent 实例 | Agent 定义的运行时表现，对应一个 Claude Code 会话 |
| 任务 | 分配给 Agent 执行的工作单元 |
| 调度 | 将任务分配到 Agent 的过程 |
| Bridge | Claude Code 的远程会话管理系统 |
| SSE | Server-Sent Events，服务端推送技术 |
| Token | Claude API 的计量单位（输入/输出文本片段） |
| Worktree | Git 的独立工作目录，用于隔离多 Agent 操作 |
| Scratchpad | Agent 间共享的临时目录 |

### B. 关键代码引用索引

| 现有代码路径 | 引用场景 |
|-------------|----------|
| `src/bridge/types.ts` | BridgeConfig、SessionHandle、WorkSecret、SessionActivity 类型定义 |
| `src/bridge/bridgeApi.ts` | BridgeApiClient 实现（registerEnvironment、pollForWork、acknowledgeWork） |
| `src/bridge/codeSessionApi.ts` | 会话创建/凭证获取的 HTTP API 封装 |
| `src/bridge/sessionRunner.ts` | SessionSpawner 接口，子进程 spawn |
| `src/Task.ts` | TaskType、TaskStatus、TaskStateBase、generateTaskId |
| `src/coordinator/coordinatorMode.ts` | Orchestrator-Worker 模式、Scratchpad |
| `src/tools/AgentTool/` | Agent 定义、工具过滤、异步 Agent 生命周期 |
| `src/QueryEngine.ts` | 会话编排核心（消息流、工具执行） |
| `wiki/concepts/coordinator-mode.md` | 编排器架构知识 |
| `wiki/entities/bridge-system.md` | 桥接系统知识 |
| `wiki/concepts/task-management.md` | 任务管理系统知识 |
| `wiki/concepts/agent-tool.md` | Agent 工具系统知识 |
| `wiki/concepts/hook-system.md` | Hook 系统知识 |

### C. 文档版本历史

| 版本 | 日期 | 变更 |
|------|------|------|
| v0.1 | 2026-04-15 | 初始版本，11 章完整需求文档 |

