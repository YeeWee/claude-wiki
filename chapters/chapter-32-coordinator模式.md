# 第三十二章：Coordinator 模式

## 32.1 引言

Coordinator 模式是 Claude Code 实现多 Agent 编排的核心机制。在该模式下，主 Agent 扮演协调者角色，负责任务分解、Worker 启动、结果综合和用户沟通；Worker Agent 执行具体的研究、实现和验证任务。这种分工协作模式使 Claude Code 能够高效处理复杂的多步骤任务。

Coordinator 模式的核心价值：

1. **并行执行**：多个 Worker 可以同时运行，充分利用异步特性
2. **任务分解**：复杂任务分解为独立子任务，由专业 Worker 处理
3. **知识综合**：Coordinator 综合多个 Worker 的研究结果，做出决策
4. **上下文隔离**：每个 Worker 独立运行，避免上下文污染

Coordinator 模式定义在 `/Users/hw/workspaces/projects/claude-wiki/src/coordinator/coordinatorMode.ts`，本章深入分析其设计哲学和实现机制。

---

## 32.2 多 Agent 架构设计

### 32.2.1 启用方式

Coordinator 模式通过环境变量启用（第36-41行）：

```typescript
export function isCoordinatorMode(): boolean {
  if (feature('COORDINATOR_MODE')) {
    return isEnvTruthy(process.env.CLAUDE_CODE_COORDINATOR_MODE)
  }
  return false
}
```

启用条件：
- Feature gate `COORDINATOR_MODE` 需开启
- 环境变量 `CLAUDE_CODE_COORDINATOR_MODE` 设置为 truthy 值

会话恢复时，系统会检查并匹配存储的模式（第49-78行）：

```typescript
export function matchSessionMode(
  sessionMode: 'coordinator' | 'normal' | undefined,
): string | undefined {
  // 无存储模式（旧会话）— 不处理
  if (!sessionMode) {
    return undefined
  }

  const currentIsCoordinator = isCoordinatorMode()
  const sessionIsCoordinator = sessionMode === 'coordinator'

  if (currentIsCoordinator === sessionIsCoordinator) {
    return undefined
  }

  // 翻转环境变量
  if (sessionIsCoordinator) {
    process.env.CLAUDE_CODE_COORDINATOR_MODE = '1'
  } else {
    delete process.env.CLAUDE_CODE_COORDINATOR_MODE
  }

  logEvent('tengu_coordinator_mode_switched', { to: sessionMode })

  return sessionIsCoordinator
    ? 'Entered coordinator mode to match resumed session.'
    : 'Exited coordinator mode to match resumed session.'
}
```

### 32.2.2 整体架构

```mermaid
flowchart TD
    subgraph UserLayer["用户层"]
        A[用户请求] --> B[Coordinator Agent]
    end
    
    subgraph CoordinatorLayer["Coordinator 层"]
        B --> C{任务分析}
        C -->|研究任务| D1[并行启动研究 Workers]
        C -->|实现任务| D2[启动实现 Worker]
        C -->|验证任务| D3[启动验证 Worker]
        
        D1 --> E[接收 task-notification]
        D2 --> E
        D3 --> E
        
        E --> F{结果综合}
        F -->|继续任务| G[SendMessage 继续 Worker]
        F -->|需要修正| H[TaskStop 停止 Worker]
        F -->|完成| I[向用户报告]
        
        G --> D2
        H --> J[停止信号]
        J --> K[重新启动正确方向 Worker]
    end
    
    subgraph WorkerPool["Worker Pool"]
        D1 --> W1[Worker 1: 研究 A]
        D1 --> W2[Worker 2: 研究 B]
        D2 --> W3[Worker 3: 实现]
        D3 --> W4[Worker 4: 验证]
        
        W1 --> N1[task-notification]
        W2 --> N2[task-notification]
        W3 --> N3[task-notification]
        W4 --> N4[task-notification]
        
        N1 --> E
        N2 --> E
        N3 --> E
        N4 --> E
    end
    
    subgraph KnowledgeLayer["知识共享层"]
        S[Scratchpad 目录]
        W1 --> S
        W2 --> S
        W3 --> S
        W4 --> S
        S --> B
    end
    
    figure-32-1: Coordinator 多 Agent 编排架构图
```

### 32.2.3 Coordinator 工具集

Coordinator 模式下的可用工具定义在 `/Users/hw/workspaces/projects/claude-wiki/src/constants/tools.ts`（第107-112行）：

```typescript
export const COORDINATOR_MODE_ALLOWED_TOOLS = new Set([
  AGENT_TOOL_NAME,      // 启动新 Worker
  TASK_STOP_TOOL_NAME,  // 停止运行中的 Worker
  SEND_MESSAGE_TOOL_NAME, // 继续 Worker 执行
  SYNTHETIC_OUTPUT_TOOL_NAME, // 输出工具
])
```

Coordinator 只能使用这四个工具：
- **AgentTool**：启动 Worker 执行任务
- **TaskStop**：停止错误方向的 Worker
- **SendMessage**：向已停止或正在运行的 Worker 发送消息
- **SyntheticOutput**：向用户输出结果

---

## 32.3 Worker Spawning 机制

### 32.3.1 Worker 工具配置

Coordinator 启动的 Worker 有独立的工具配置（第80-108行）：

```typescript
export function getCoordinatorUserContext(
  mcpClients: ReadonlyArray<{ name: string }>,
  scratchpadDir?: string,
): { [k: string]: string } {
  if (!isCoordinatorMode()) {
    return {}
  }

  // 简化模式：仅 Bash、Read、Edit
  const workerTools = isEnvTruthy(process.env.CLAUDE_CODE_SIMPLE)
    ? [BASH_TOOL_NAME, FILE_READ_TOOL_NAME, FILE_EDIT_TOOL_NAME]
        .sort()
        .join(', ')
    // 完整模式：使用异步 Agent 工具白名单
    : Array.from(ASYNC_AGENT_ALLOWED_TOOLS)
        .filter(name => !INTERNAL_WORKER_TOOLS.has(name))
        .sort()
        .join(', ')

  let content = `Workers spawned via the ${AGENT_TOOL_NAME} tool have access to these tools: ${workerTools}`

  // MCP 工具继承
  if (mcpClients.length > 0) {
    const serverNames = mcpClients.map(c => c.name).join(', ')
    content += `\n\nWorkers also have access to MCP tools from connected MCP servers: ${serverNames}`
  }

  // Scratchpad 共享目录
  if (scratchpadDir && isScratchpadGateEnabled()) {
    content += `\n\nScratchpad directory: ${scratchpadDir}\nWorkers can read and write here without permission prompts. Use this for durable cross-worker knowledge — structure files however fits the work.`
  }

  return { workerToolsContext: content }
}
```

**INTERNAL_WORKER_TOOLS** 定义了 Worker 禁用的内部工具（第29-34行）：

```typescript
const INTERNAL_WORKER_TOOLS = new Set([
  TEAM_CREATE_TOOL_NAME,
  TEAM_DELETE_TOOL_NAME,
  SEND_MESSAGE_TOOL_NAME,
  SYNTHETIC_OUTPUT_TOOL_NAME,
])
```

这些工具是 Coordinator 专用的，Worker 不能使用。

### 32.3.2 ASYNC_AGENT_ALLOWED_TOOLS

异步 Worker 可用的工具白名单（`src/constants/tools.ts` 第55-71行）：

```typescript
export const ASYNC_AGENT_ALLOWED_TOOLS = new Set([
  FILE_READ_TOOL_NAME,      // 文件读取
  WEB_SEARCH_TOOL_NAME,     // 网络搜索
  TODO_WRITE_TOOL_NAME,     // Todo 管理
  GREP_TOOL_NAME,           // 内容搜索
  WEB_FETCH_TOOL_NAME,      // URL 内容获取
  GLOB_TOOL_NAME,           // 文件模式匹配
  ...SHELL_TOOL_NAMES,      // Shell 命令
  FILE_EDIT_TOOL_NAME,      // 文件编辑
  FILE_WRITE_TOOL_NAME,     // 文件写入
  NOTEBOOK_EDIT_TOOL_NAME,  // Jupyter 编辑
  SKILL_TOOL_NAME,          // 技能调用
  SYNTHETIC_OUTPUT_TOOL_NAME, // 输出
  TOOL_SEARCH_TOOL_NAME,    // 工具搜索
  ENTER_WORKTREE_TOOL_NAME, // 进入 Worktree
  EXIT_WORKTREE_TOOL_NAME,  // 退出 Worktree
])
```

**Worker 禁用的工具及原因**（第91-102行注释）：

| 禁用工具 | 原因 |
|---------|------|
| AgentTool | 防止递归 Worker 创建 |
| TaskOutputTool | 防止递归 |
| ExitPlanModeTool | Plan Mode 是主线程抽象 |
| TaskStopTool | 需要访问主线程任务状态 |
| AskUserQuestionTool | Worker 不能主动询问用户 |

### 32.3.3 Worker 启动流程

```mermaid
sequenceDiagram
    participant Coordinator
    participant AgentTool
    participant AppState
    participant Worker
    
    Coordinator->>AgentTool: call({ subagent_type: "worker", prompt: "..." })
    AgentTool->>AgentTool: 检查 Coordinator 模式
    AgentTool->>AgentTool: shouldRunAsync = true (强制异步)
    AgentTool->>AgentTool: 组装 Worker 工具池
    AgentTool->>AppState: registerAsyncAgent()
    AppState-->>AgentTool: 返回 taskId 和 abortController
    AgentTool->>Worker: runAsyncAgentLifecycle()
    
    Note over Worker: Worker 执行任务
    
    Worker->>AgentTool: yield Message (每个迭代)
    AgentTool->>AppState: 更新进度
    Worker->>AgentTool: 完成
    AgentTool->>AgentTool: finalizeAgentTool()
    AgentTool->>AppState: completeAsyncAgent()
    AgentTool->>Coordinator: enqueueAgentNotification()
    
    Coordinator->>Coordinator: 接收 <task-notification>
    
    figure-32-2: Worker 启动流程时序图
```

Coordinator 模式下，Worker **强制异步执行**（`shouldRunAsync` 中包含 `isCoordinator` 检查），这是设计核心：

- Coordinator 不等待 Worker 完成
- Worker 完成后发送 `<task-notification>` 通知
- Coordinator 可同时启动多个 Worker 实现并行

---

## 32.4 Scratchpad 知识共享

### 32.4.1 Scratchpad 目录设计

Scratchpad 是跨 Worker 的知识共享目录，定义在 `/Users/hw/workspaces/projects/claude-wiki/src/utils/permissions/filesystem.ts`。

**路径格式**（第384-386行）：

```typescript
export function getScratchpadDir(): string {
  return join(getProjectTempDir(), getSessionId(), 'scratchpad')
}
// 格式: /tmp/claude-{uid}/{sanitized-cwd}/{sessionId}/scratchpad/
```

**启用检查**（第298-300行）：

```typescript
export function isScratchpadEnabled(): boolean {
  return checkStatsigFeatureGate_CACHED_MAY_BE_STATE('tengu_scratch')
}
```

**目录创建**（第394-407行）：

```typescript
export async function ensureScratchpadDir(): Promise<string> {
  if (!isScratchpadEnabled()) {
    throw new Error('Scratchpad directory feature is not enabled')
  }

  const fs = getFsImplementation()
  const scratchpadDir = getScratchpadDir()

  // 递归创建，安全权限（仅 owner 可访问）
  await fs.mkdir(scratchpadDir, { mode: 0o700 })

  return scratchpadDir
}
```

### 32.4.2 安全验证

路径验证防止路径穿越攻击（第410-424行）：

```typescript
function isScratchpadPath(absolutePath: string): boolean {
  if (!isScratchpadEnabled()) {
    return false
  }
  const scratchpadDir = getScratchpadDir()
  // 安全：规范化路径，解析 .. 段
  // 防止: echo "malicious" > scratchpad/../../../etc/passwd
  const normalizedPath = normalize(absolutePath)
  return (
    normalizedPath === scratchpadDir ||
    normalizedPath.startsWith(scratchpadDir + sep)
  )
}
```

### 32.4.3 Worker 知识共享机制

Scratchpad 在 Coordinator 系统提示中的作用（`coordinatorMode.ts` 第104-106行）：

```typescript
if (scratchpadDir && isScratchpadGateEnabled()) {
  content += `\n\nScratchpad directory: ${scratchpadDir}\nWorkers can read and write here without permission prompts. Use this for durable cross-worker knowledge — structure files however fits the work.`
}
```

**使用场景**：

1. **研究结果存储**：研究 Worker 将发现写入 Scratchpad，实现 Worker 读取
2. **中间状态持久化**：长时间任务保存进度，避免丢失
3. **跨 Worker 协调**：共享文件结构、发现、配置等

**权限免除**：Workers 在 Scratchpad 目录读写无需权限提示，这是关键设计：

```typescript
// filesystem.ts 第1241-1245行
// 1.5. Allow writes to internal editable paths (plan files, scratchpad)
// This MUST come before isDangerousFilePathToAutoEdit check
const internalEditResult = checkEditableInternalPath(
  absolutePathForEdit,
  input,
  // ...
)
```

---

## 32.5 SendMessage 工具 - 消息路由

### 32.5.1 工具概述

SendMessage 工具用于继续运行中的或已停止的 Worker，定义在 `/Users/hw/workspaces/projects/claude-wiki/src/tools/SendMessageTool/SendMessageTool.ts`。

**Prompt 定义**（`prompt.ts` 第22-48行）：

```typescript
export function getPrompt(): string {
  return `
# SendMessage

Send a message to another agent.

\`\`\`json
{"to": "researcher", "summary": "assign task 1", "message": "start on task #1"}
\`\`\`

| \`to\` | |
|---|---|
| \`"researcher"\` | Teammate by name |
| \`"*"\` | Broadcast to all teammates |
| \`"agent-xyz"\` | Worker by agent ID |

Your plain text output is NOT visible to other agents — to communicate, you MUST call this tool.
`
}
```

### 32.5.2 消息路由流程

```mermaid
flowchart TD
    A[SendMessage.call] --> B{检查 input.to}
    B -->|"*"| C[handleBroadcast]
    B -->|"agent-xyz"| D{查找 Agent 任务}
    B -->|"name"| E[handleMessage]
    
    D --> F{任务状态}
    F -->|"running"| G[queuePendingMessage]
    F -->|"stopped"| H[resumeAgentBackground]
    F -->|"无任务"| I{尝试从 transcript 恢复}
    
    G --> J[消息入队等待下轮工具调用]
    H --> K[后台恢复 Agent]
    I -->|成功| K
    I -->|失败| L[错误：无 transcript]
    
    C --> M[广播到所有 Teammate]
    E --> N[写入 Mailbox]
    
    figure-32-3: SendMessage 消息路由流程
```

### 32.5.3 Worker 继续机制

核心路由逻辑（`SendMessageTool.ts` 第802-874行）：

```typescript
// 路由到 in-process 子 Agent
if (typeof input.message === 'string' && input.to !== '*') {
  const appState = context.getAppState()
  const registered = appState.agentNameRegistry.get(input.to)
  const agentId = registered ?? toAgentId(input.to)
  
  if (agentId) {
    const task = appState.tasks[agentId]
    if (isLocalAgentTask(task) && !isMainSessionTask(task)) {
      if (task.status === 'running') {
        // 运行中：入队消息
        queuePendingMessage(
          agentId,
          input.message,
          context.setAppStateForTasks ?? context.setAppState,
        )
        return { data: { success: true, message: `Message queued for delivery...` } }
      }
      
      // 已停止：自动恢复
      try {
        const result = await resumeAgentBackground({
          agentId,
          prompt: input.message,
          toolUseContext: context,
          canUseTool,
          invokingRequestId: assistantMessage?.requestId,
        })
        return { data: { success: true, message: `Agent "${input.to}" was stopped; resumed it...` } }
      } catch (e) {
        return { data: { success: false, message: `Agent "${input.to}" could not be resumed...` } }
      }
    }
    
    // 任务已从状态清除：尝试从 transcript 恢复
    try {
      const result = await resumeAgentBackground({ ... })
      return { data: { success: true, message: `Agent "${input.to}" resumed from transcript...` } }
    } catch (e) {
      return { data: { success: false, message: `Agent "${input.to}" has no transcript...` } }
    }
  }
}
```

### 32.5.4 resumeAgentBackground

Agent 恢复的核心实现（`resumeAgent.ts` 第42-265行）：

```typescript
export async function resumeAgentBackground({
  agentId,
  prompt,
  toolUseContext,
  canUseTool,
  invokingRequestId,
}: {...}): Promise<ResumeAgentResult> {
  // 加载 transcript 和 metadata
  const [transcript, meta] = await Promise.all([
    getAgentTranscript(asAgentId(agentId)),
    readAgentMetadata(asAgentId(agentId)),
  ])
  
  if (!transcript) {
    throw new Error(`No transcript found for agent ID: ${agentId}`)
  }
  
  // 过滤消息，保留有效内容
  const resumedMessages = filterWhitespaceOnlyAssistantMessages(
    filterOrphanedThinkingOnlyMessages(
      filterUnresolvedToolUses(transcript.messages),
    ),
  )
  
  // 重建内容替换状态
  const resumedReplacementState = reconstructForSubagentResume(
    toolUseContext.contentReplacementState,
    resumedMessages,
    transcript.contentReplacements,
  )
  
  // 确定 Agent 定义
  let selectedAgent: AgentDefinition
  if (meta?.agentType === FORK_AGENT.agentType) {
    selectedAgent = FORK_AGENT
    isResumedFork = true
  } else if (meta?.agentType) {
    selectedAgent = toolUseContext.options.agentDefinitions.activeAgents.find(
      a => a.agentType === meta.agentType,
    ) ?? GENERAL_PURPOSE_AGENT
  } else {
    selectedAgent = GENERAL_PURPOSE_AGENT
  }
  
  // 注册后台任务
  const agentBackgroundTask = registerAsyncAgent({ ... })
  
  // 启动生命周期
  void runWithAgentContext(asyncAgentContext, () =>
    wrapWithCwd(() =>
      runAsyncAgentLifecycle({
        taskId: agentBackgroundTask.agentId,
        abortController: agentBackgroundTask.abortController!,
        makeStream: onCacheSafeParams =>
          runAgent({
            ...runAgentParams,
            promptMessages: [
              ...resumedMessages,
              createUserMessage({ content: prompt }),
            ],
            override: { agentId, abortController },
            onCacheSafeParams,
          }),
        // ...
      }),
    ),
  )
  
  return { agentId, description: uiDescription, outputFile: getTaskOutputPath(agentId) }
}
```

恢复机制的关键点：

1. **Transcript 加载**：从磁盘加载 Agent 的完整对话历史
2. **消息过滤**：移除无效的工具调用和空白消息
3. **内容重建**：恢复之前的工具结果替换状态
4. **追加消息**：将新消息追加到历史，继续执行

---

## 32.6 Coordinator 系统提示解析

### 32.6.1 角色定义

系统提示定义 Coordinator 的核心角色（第111-127行）：

```typescript
export function getCoordinatorSystemPrompt(): string {
  return `You are Claude Code, an AI assistant that orchestrates software engineering tasks across multiple workers.

## 1. Your Role

You are a **coordinator**. Your job is to:
- Help the user achieve their goal
- Direct workers to research, implement and verify code changes
- Synthesize results and communicate with the user
- Answer questions directly when possible — don't delegate work that you can handle without tools

Every message you send is to the user. Worker results and system notifications are internal signals, not conversation partners — never thank or acknowledge them. Summarize new information for the user as it arrives.
`
}
```

关键原则：
- Coordinator 直接与用户沟通，Worker 结果是内部信号
- 可以直接回答问题，不必都委托给 Worker
- 不会感谢或确认 Worker 通知，而是综合后向用户报告

### 32.6.2 任务阶段表

Coordinator 定义了清晰的任务阶段（第199-209行）：

```typescript
## 4. Task Workflow

| Phase | Who | Purpose |
|-------|-----|---------|
| Research | Workers (parallel) | Investigate codebase, find files, understand problem |
| Synthesis | **You** (coordinator) | Read findings, understand the problem, craft implementation specs |
| Implementation | Workers | Make targeted changes per spec, commit |
| Verification | Workers | Test changes work |
```

阶段职责：
- **Research**：Worker 并行执行，调查代码库
- **Synthesis**：Coordinator 必须阅读发现，理解问题，编写实现规范
- **Implementation**：Worker 按规范执行修改
- **Verification**：Worker 验证修改有效性

### 32.6.3 并发原则

并发是 Coordinator 的核心优势（第213-218行）：

```typescript
### Concurrency

**Parallelism is your superpower. Workers are async. Launch independent workers concurrently whenever possible — don't serialize work that can run simultaneously and look for opportunities to fan out. When doing research, cover multiple angles. To launch workers in parallel, make multiple tool calls in a single message.**

Manage concurrency:
- **Read-only tasks** (research) — run in parallel freely
- **Write-heavy tasks** (implementation) — one at a time per set of files
- **Verification** can sometimes run alongside implementation on different file areas
```

关键策略：
- 研究任务并行启动多个 Worker，覆盖多个角度
- 实现任务串行执行，避免文件冲突
- 验证任务可与实现并行（不同文件区域）

### 32.6.4 task-notification 格式

Worker 结果通过 `<task-notification>` XML 通知（第143-185行）：

```typescript
### AgentTool Results

Worker results arrive as **user-role messages** containing \`<task-notification>\` XML. They look like user messages but are not. Distinguish them by the \`<task-notification>\` opening tag.

Format:

\`\`\`xml
<task-notification>
<task-id>{agentId}</task-id>
<status>completed|failed|killed</status>
<summary>{human-readable status summary}</summary>
<result>{agent's final text response}</result>
<usage>
  <total_tokens>N</total_tokens>
  <tool_uses>N</tool_uses>
  <duration_ms>N</duration_ms>
</usage>
</task-notification>
\`\`\`

- \`<result>\` and \`<usage>\` are optional sections
- The \`<task-id>\` value is the agent ID — use SendMessage with that ID as \`to\` to continue that worker
```

### 32.6.5 Worker Prompt 编写指南

Coordinator 必须综合研究结果，编写精确的 Worker Prompt（第251-334行）：

**核心原则**：

```typescript
### Always synthesize — your most important job

When workers report research findings, **you must understand them before directing follow-up work**. Read the findings. Identify the approach. Then write a prompt that proves you understood by including specific file paths, line numbers, and exactly what to change.

Never write "based on your findings" or "based on the research." These phrases delegate understanding to the worker instead of doing it yourself. You never hand off understanding to another worker.

// 反模式 — 懒惰委托
${AGENT_TOOL_NAME}({ prompt: "Based on your findings, fix the auth bug", ... })

// 正确 — 综合后的规范
${AGENT_TOOL_NAME}({ prompt: "Fix the null pointer in src/auth/validate.ts:42. The user field on Session is undefined when sessions expire but the token remains cached. Add a null check before user.id access — if null, return 401 with 'Session expired'. Commit and report the hash.", ... })
```

**Continue vs Spawn 决策表**（第284-292行）：

| 情况 | 机制 | 原因 |
|------|------|------|
| 研究探索的正是需要编辑的文件 | Continue（SendMessage） | Worker 已有文件上下文，获得清晰计划 |
| 研究广泛但实现范围窄 | Spawn fresh | 避免拖入探索噪声，专注上下文更清晰 |
| 修正失败或延续近期工作 | Continue | Worker 有错误上下文，知道刚尝试了什么 |
| 验证不同 Worker 的代码 | Spawn fresh | 验证者应以新视角看代码，不携带实现假设 |
| 第一次实现尝试方向完全错误 | Spawn fresh | 错误方向上下文污染重试，干净起点避免锚定 |

---

## 32.7 TaskStop 工具

### 32.7.1 工具概述

TaskStop 用于停止运行中的 Worker，当 Coordinator 发现 Worker 方向错误时使用。

Prompt 定义（`prompt.ts`）：

```typescript
export const DESCRIPTION = `
- Stops a running background task by its ID
- Takes a task_id parameter identifying the task to stop
- Returns a success or failure status
- Use this tool when you need to terminate a long-running task
`
```

### 32.7.2 使用场景

Coordinator 系统提示中的使用指导（第237-249行）：

```typescript
### Stopping Workers

Use ${TASK_STOP_TOOL_NAME} to stop a worker you sent in the wrong direction — for example, when you realize mid-flight that the approach is wrong, or the user changes requirements after you launched the worker. Pass the \`task_id\` from the ${AGENT_TOOL_NAME} tool's launch result. Stopped workers can be continued with ${SEND_MESSAGE_TOOL_NAME}.

\`\`\`
// Launched a worker to refactor auth to use JWT
${AGENT_TOOL_NAME}({ description: "Refactor auth to JWT", subagent_type: "worker", prompt: "..." })
// ... returns task_id: "agent-x7q" ...

// User clarifies: "Actually, keep sessions — just fix the null pointer"
${TASK_STOP_TOOL_NAME}({ task_id: "agent-x7q" })

// Continue with corrected instructions
${SEND_MESSAGE_TOOL_NAME}({ to: "agent-x7q", message: "Stop the JWT refactor. Instead, fix the null pointer..." })
\`\`\`
```

---

## 32.8 会话示例分析

### 32.8.1 完整协作流程

Coordinator 系统提示中提供的示例（第336-369行）：

```typescript
## 6. Example Session

User: "There's a null pointer in the auth module. Can you fix it?"

You:
  Let me investigate first.

  ${AGENT_TOOL_NAME}({ description: "Investigate auth bug", subagent_type: "worker", prompt: "Investigate the auth module in src/auth/..." })
  ${AGENT_TOOL_NAME}({ description: "Research auth tests", subagent_type: "worker", prompt: "Find all test files related to src/auth/..." })

  Investigating from two angles — I'll report back with findings.

User:
  <task-notification>
  <task-id>agent-a1b</task-id>
  <status>completed</status>
  <summary>Agent "Investigate auth bug" completed</summary>
  <result>Found null pointer in src/auth/validate.ts:42...</result>
  </task-notification>

You:
  Found the bug — null pointer in validate.ts:42.

  ${SEND_MESSAGE_TOOL_NAME}({ to: "agent-a1b", message: "Fix the null pointer in src/auth/validate.ts:42..." })

  Fix is in progress.

User:
  How's it going?

You:
  Fix for the new test is in progress. Still waiting to hear back about the test suite.
```

### 32.8.2 流程分解

1. **并行研究**：启动两个 Worker 分别调查 Bug 和测试结构
2. **通知接收**：第一个 Worker 完成，发送 task-notification
3. **结果综合**：Coordinator 读取发现，确定问题位置
4. **Worker 继续**：SendMessage 发送修复指令到已完成的 Worker
5. **用户沟通**：向用户报告进度，等待其他 Worker

---

## 32.9 总结

Coordinator 模式是 Claude Code 多 Agent 编排的核心机制：

| 特性 | 说明 | 关键文件 |
|------|------|----------|
| 模式启用 | 环境变量 + Feature gate | `coordinatorMode.ts:36-41` |
| Coordinator 工具集 | Agent、Stop、SendMessage、Output | `constants/tools.ts:107-112` |
| Worker 工具集 | ASYNC_AGENT_ALLOWED_TOOLS | `constants/tools.ts:55-71` |
| Scratchpad | 跨 Worker 知识共享 | `filesystem.ts:384-407` |
| SendMessage | 继续 Worker 执行 | `SendMessageTool.ts` |
| TaskStop | 停止错误方向 Worker | `TaskStopTool/prompt.ts` |
| resumeAgent | 后台恢复 Agent | `resumeAgent.ts` |

设计亮点：

1. **强制异步**：Coordinator 模式下 Worker 必须异步运行
2. **上下文隔离**：Worker 工具集受限，不能访问 Coordinator 工具
3. **知识综合**：Coordinator 必须阅读并理解 Worker 结果，编写精确规范
4. **并发优势**：研究阶段并行启动 Worker，充分利用异步特性
5. **Scratchpad 共享**：跨 Worker 知识持久化，无需权限提示

---

## 附录：关键源文件索引

| 文件路径 | 主要职责 |
|----------|----------|
| `/Users/hw/workspaces/projects/claude-wiki/src/coordinator/coordinatorMode.ts` | Coordinator 模式定义、系统提示、Worker 工具配置 |
| `/Users/hw/workspaces/projects/claude-wiki/src/constants/tools.ts` | 工具白名单定义（ASYNC_AGENT_ALLOWED_TOOLS、COORDINATOR_MODE_ALLOWED_TOOLS） |
| `/Users/hw/workspaces/projects/claude-wiki/src/tools/SendMessageTool/SendMessageTool.ts` | 消息路由、Worker 继续机制 |
| `/Users/hw/workspaces/projects/claude-wiki/src/tools/SendMessageTool/prompt.ts` | SendMessage Prompt 定义 |
| `/Users/hw/workspaces/projects/claude-wiki/src/tools/AgentTool/resumeAgent.ts` | Agent 后台恢复实现 |
| `/Users/hw/workspaces/projects/claude-wiki/src/tools/TaskStopTool/prompt.ts` | TaskStop Prompt 定义 |
| `/Users/hw/workspaces/projects/claude-wiki/src/utils/permissions/filesystem.ts` | Scratchpad 目录管理 |
| `/Users/hw/workspaces/projects/claude-wiki/src/QueryEngine.ts` | Coordinator UserContext 注入 |