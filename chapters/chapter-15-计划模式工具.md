# 第15章 计划模式工具

计划模式（Plan Mode）是 Claude Code 提供的一种工作流机制，用于在执行复杂的实现任务之前进行充分的探索和规划。通过进入计划模式，Claude 可以在不修改任何代码的情况下，先深入理解代码库结构、分析现有模式，然后设计一个经过用户审批的实现方案。这种"先规划后执行"的模式能有效避免方向性错误，提高开发效率。

## 15.1 计划模式概述

计划模式的核心价值在于：

1. **降低风险**：对于涉及多个文件、多种实现方案的复杂任务，先获得用户确认再动手
2. **提高效率**：避免因方向错误导致的返工
3. **增强透明度**：用户可以提前了解实现思路，参与到决策过程中
4. **文档化**：生成的计划文件可以作为实现过程的参考和记录

计划模式的典型应用场景包括：新功能实现、架构决策、重构任务、多文件修改等。对于简单的单行修复或明确的任务，则不建议使用计划模式。

```mermaid
sequenceDiagram
    participant User as 用户
    participant Claude as Claude
    participant Permission as 权限系统
    participant PlanFile as 计划文件
    participant Tool as 工具系统

    User->>Claude: 提出复杂任务请求
    Claude->>Tool: 调用 EnterPlanModeTool
    Tool->>Permission: 发起权限请求
    Permission->>User: "是否进入计划模式?"
    User->>Permission: 确认进入
    Permission->>Tool: 返回 allow
    Tool->>Claude: 进入计划模式成功

    Note over Claude: 只读探索阶段
    Claude->>Tool: 使用 Glob/Grep/Read 探索代码库
    Claude->>PlanFile: 编写/编辑计划文件

    Claude->>Tool: 调用 ExitPlanModeTool
    Tool->>Permission: 发起审批请求
    Permission->>User: 展示计划内容
    User->>Permission: 批准/拒绝/编辑
    Permission->>Tool: 返回审批结果

    alt 用户批准
        Tool->>Claude: 退出计划模式，开始实现
        Claude->>Tool: 执行代码修改等操作
    else 用户拒绝
        Tool->>Claude: 计划被拒绝，继续规划
    end

    figure-15-1: 计划模式完整流程时序图
```

## 15.2 EnterPlanModeTool - 进入计划模式

`EnterPlanModeTool` 是触发计划模式的入口工具。该工具本身不接受任何参数，其核心作用是请求用户批准进入计划模式，并完成权限上下文的切换。

### 15.2.1 工具定义

工具定义位于 `/Users/hw/workspaces/projects/claude-wiki/src/tools/EnterPlanModeTool/EnterPlanModeTool.ts`：

```typescript
// 第21-25行：输入输出 Schema 定义
const inputSchema = lazySchema(() =>
  z.strictObject({
    // No parameters needed - 该工具无需任何参数
  }),
)

const outputSchema = lazySchema(() =>
  z.object({
    message: z.string().describe('Confirmation that plan mode was entered'),
  }),
)
```

工具的核心配置（第36-76行）：

```typescript
export const EnterPlanModeTool: Tool<InputSchema, Output> = buildTool({
  name: ENTER_PLAN_MODE_TOOL_NAME,
  searchHint: 'switch to plan mode to design an approach before coding',
  maxResultSizeChars: 100_000,
  async description() {
    return 'Requests permission to enter plan mode for complex tasks requiring exploration and design'
  },
  shouldDefer: true,  // 延迟执行，等待权限审批
  isConcurrencySafe() {
    return true  // 与其他工具并发安全
  },
  isReadOnly() {
    return true  // 只读操作，不修改文件
  },
  // ... 其他配置
})
```

关键特性说明：

- `shouldDefer: true`：工具需要用户交互确认，在权限流程中延迟执行
- `isReadOnly: true`：进入计划模式本身不产生文件修改
- `isEnabled()` 方法检查 channels 功能是否活跃，避免用户无法退出计划模式的陷阱

### 15.2.2 Prompt 指令设计

工具的 Prompt 设计根据用户类型分为两种版本，位于 `/Users/hw/workspaces/projects/claude-wiki/src/tools/EnterPlanModeTool/prompt.ts`。

**外部用户版本（第16-99行）** 倾向于主动使用：

```typescript
function getEnterPlanModeToolPromptExternal(): string {
  return `Use this tool proactively when you're about to start a non-trivial implementation task.

## When to Use This Tool

**Prefer using EnterPlanMode** for implementation tasks unless they're simple. Use it when ANY of these conditions apply:

1. **New Feature Implementation**: Adding meaningful new functionality
2. **Multiple Valid Approaches**: The task can be solved in several different ways
3. **Code Modifications**: Changes that affect existing behavior or structure
4. **Architectural Decisions**: The task requires choosing between patterns or technologies
5. **Multi-File Changes**: The task will likely touch more than 2-3 files
6. **Unclear Requirements**: You need to explore before understanding the full scope
7. **User Preferences Matter**: The implementation could reasonably go multiple ways

## When NOT to Use This Tool

Only skip EnterPlanMode for simple tasks:
- Single-line or few-line fixes (typos, obvious bugs, small tweaks)
- Adding a single function with clear requirements
- Tasks where the user has given very specific, detailed instructions
- Pure research/exploration tasks (use the Agent tool with explore agent instead)
`
}
```

**Ant 用户版本（第101-164行）** 更加谨慎：

```typescript
function getEnterPlanModeToolPromptAnt(): string {
  return `Use this tool when a task has genuine ambiguity about the right approach.

## When to Use This Tool

Plan mode is valuable when the implementation approach is genuinely unclear. Use it when:

1. **Significant Architectural Ambiguity**: Multiple reasonable approaches exist
2. **Unclear Requirements**: You need to explore and clarify before you can make progress
3. **High-Impact Restructuring**: The task will significantly restructure existing code

## When NOT to Use This Tool

Skip plan mode when you can reasonably infer the right approach:
- The task is straightforward even if it touches multiple files
- The user's request is specific enough that the implementation path is clear
- Bug fixes where the fix is clear once you understand the bug
- The user says something like "can we work on X" or "let's do X" — just get started
`
}
```

两种版本的核心差异在于：外部用户版本鼓励主动使用，而 Ant 版本强调只在真正模糊的情况下使用，否则直接开始工作。

### 15.2.3 执行逻辑

工具的核心执行逻辑（第77-102行）：

```typescript
async call(_input, context) {
  // 禁止在 Agent 上下文中使用（第78-80行）
  if (context.agentId) {
    throw new Error('EnterPlanMode tool cannot be used in agent contexts')
  }

  const appState = context.getAppState()
  // 处理计划模式转换（第83-84行）
  handlePlanModeTransition(appState.toolPermissionContext.mode, 'plan')

  // 更新权限上下文（第88-95行）
  context.setAppState(prev => ({
    ...prev,
    toolPermissionContext: applyPermissionUpdate(
      prepareContextForPlanMode(prev.toolPermissionContext),
      { type: 'setMode', mode: 'plan', destination: 'session' },
    ),
  }))

  return {
    data: {
      message:
        'Entered plan mode. You should now focus on exploring the codebase and designing an implementation approach.',
    },
  }
}
```

执行流程的关键步骤：

1. **检查 Agent 上下文**：禁止在子 Agent 中使用计划模式
2. **处理模式转换**：调用 `handlePlanModeTransition` 记录转换事件
3. **准备权限上下文**：调用 `prepareContextForPlanMode` 保存当前模式并设置 `prePlanMode`
4. **更新 AppState**：将权限模式切换为 `plan`

### 15.2.4 权限上下文准备

`prepareContextForPlanMode` 函数位于 `/Users/hw/workspaces/projects/claude-wiki/src/utils/permissions/permissionSetup.ts`（第1462-1493行）：

```typescript
export function prepareContextForPlanMode(
  context: ToolPermissionContext,
): ToolPermissionContext {
  const currentMode = context.mode
  if (currentMode === 'plan') return context  // 已在计划模式，无需处理

  if (feature('TRANSCRIPT_CLASSIFIER')) {
    const planAutoMode = shouldPlanUseAutoMode()
    
    // 从 auto 模式进入计划模式
    if (currentMode === 'auto') {
      if (planAutoMode) {
        return { ...context, prePlanMode: 'auto' }  // 保持 auto 激活
      }
      autoModeStateModule?.setAutoModeActive(false)
      setNeedsAutoModeExitAttachment(true)
      return {
        ...restoreDangerousPermissions(context),
        prePlanMode: 'auto',
      }
    }
    
    // 从其他模式进入，启用 auto 模式（如果配置允许）
    if (planAutoMode && currentMode !== 'bypassPermissions') {
      autoModeStateModule?.setAutoModeActive(true)
      return {
        ...stripDangerousPermissionsForAutoMode(context),
        prePlanMode: currentMode,
      }
    }
  }

  return { ...context, prePlanMode: currentMode }
}
```

关键逻辑说明：

- `prePlanMode` 字段保存进入计划模式前的原始模式，供退出时恢复
- 如果用户启用了 `useAutoModeDuringPlan`，在计划模式中保持自动权限批准
- 从 `bypassPermissions` 模式进入时，不启用 auto 模式（安全考虑）

### 15.2.5 工具结果映射

进入计划模式后，系统会向 Claude 发送指令（第103-125行）：

```typescript
mapToolResultToToolResultBlockParam({ message }, toolUseID) {
  const instructions = isPlanModeInterviewPhaseEnabled()
    ? `${message}

DO NOT write or edit any files except the plan file. Detailed workflow instructions will follow.`
    : `${message}

In plan mode, you should:
1. Thoroughly explore the codebase to understand existing patterns
2. Identify similar features and architectural approaches
3. Consider multiple approaches and their trade-offs
4. Use AskUserQuestion if you need to clarify the approach
5. Design a concrete implementation strategy
6. When ready, use ExitPlanMode to present your plan for approval

Remember: DO NOT write or edit any files yet. This is a read-only exploration and planning phase.`

  return {
    type: 'tool_result',
    content: instructions,
    tool_use_id: toolUseID,
  }
}
```

如果启用了 Interview Phase（访谈阶段），系统会通过 `plan_mode` attachment 发送更详细的分阶段工作流指令。

## 15.3 ExitPlanModeV2Tool - 退出计划模式

`ExitPlanModeV2Tool` 是计划模式的出口工具，用于将编写的计划提交给用户审批，并在获得批准后退出计划模式开始实现。

### 15.3.1 工具定义

工具定义位于 `/Users/hw/workspaces/projects/claude-wiki/src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts`：

```typescript
// 第64-89行：输入 Schema 定义
const allowedPromptSchema = lazySchema(() =>
  z.object({
    tool: z.enum(['Bash']).describe('The tool this prompt applies to'),
    prompt: z
      .string()
      .describe(
        'Semantic description of the action, e.g. "run tests", "install dependencies"',
      ),
  }),
)

const inputSchema = lazySchema(() =>
  z
    .strictObject({
      // Prompt-based permissions requested by the plan
      allowedPrompts: z
        .array(allowedPromptSchema())
        .optional()
        .describe(
          'Prompt-based permissions needed to implement the plan.',
        ),
    })
    .passthrough(),
)
```

`allowedPrompts` 参数允许 Claude 请求语义化的权限批准，例如请求"运行测试"类别的 Bash 命令权限，而非具体命令。

输出 Schema（第110-143行）：

```typescript
export const outputSchema = lazySchema(() =>
  z.object({
    plan: z.string().nullable().describe('The plan that was presented to the user'),
    isAgent: z.boolean(),
    filePath: z.string().optional().describe('The plan file path'),
    hasTaskTool: z.boolean().optional().describe('Whether the Agent tool is available'),
    planWasEdited: z.boolean().optional().describe('True when the user edited the plan'),
    awaitingLeaderApproval: z.boolean().optional()
      .describe('When true, the teammate has sent a plan approval request to the team leader'),
    requestId: z.string().optional().describe('Unique identifier for the plan approval request'),
  }),
)
```

### 15.3.2 Prompt 指令

工具 Prompt 位于 `/Users/hw/workspaces/projects/claude-wiki/src/tools/ExitPlanModeTool/prompt.ts`：

```typescript
export const EXIT_PLAN_MODE_V2_TOOL_PROMPT = `Use this tool when you are in plan mode and have finished writing your plan to the plan file and are ready for user approval.

## How This Tool Works
- You should have already written your plan to the plan file specified in the plan mode system message
- This tool does NOT take the plan content as a parameter - it will read the plan from the file you wrote
- This tool simply signals that you're done planning and ready for the user to review and approve
- The user will see the contents of your plan file when they review it

## When to Use This Tool
IMPORTANT: Only use this tool when the task requires planning the implementation steps of a task that requires writing code. For research tasks where you're gathering information, searching files, reading files or in general trying to understand the codebase - do NOT use this tool.

## Before Using This Tool
Ensure your plan is complete and unambiguous:
- If you have unresolved questions about requirements or approach, use AskUserQuestion first
- Once your plan is finalized, use THIS tool to request approval

**Important:** Do NOT use AskUserQuestion to ask "Is this plan okay?" - that's exactly what THIS tool does.
`
```

关键要点：

- 工具不接受计划内容参数，而是从计划文件读取
- 仅用于需要编写代码的实现任务，不用于纯研究任务
- 如果有未解决的问题，应先使用 `AskUserQuestion` 工具

### 15.3.3 权限检查

权限检查逻辑（第221-239行）：

```typescript
async checkPermissions(input, context) {
  // 对于 teammate，跳过权限 UI（第226-231行）
  if (isTeammate()) {
    return {
      behavior: 'allow' as const,
      updatedInput: input,
    }
  }

  // 对于普通用户，需要确认（第234-238行）
  return {
    behavior: 'ask' as const,
    message: 'Exit plan mode?',
    updatedInput: input,
  }
}
```

对于 teammate（协作成员），如果设置了 `planModeRequired`，计划会发送给 team leader 审批；否则直接退出（自愿计划模式）。

### 15.3.4 执行逻辑

核心执行逻辑（第243-418行）处理多种场景：

```typescript
async call(input, context) {
  const isAgent = !!context.agentId
  const filePath = getPlanFilePath(context.agentId)

  // 从输入或磁盘获取计划内容（第251-253行）
  const inputPlan =
    'plan' in input && typeof input.plan === 'string' ? input.plan : undefined
  const plan = inputPlan ?? getPlan(context.agentId)

  // 同步磁盘写入（第258-261行）
  if (inputPlan !== undefined && filePath) {
    await writeFile(filePath, inputPlan, 'utf-8').catch(e => logError(e))
    void persistFileSnapshotIfRemote()
  }

  // teammate 需要 leader 审批的场景（第264-313行）
  if (isTeammate() && isPlanModeRequired()) {
    if (!plan) {
      throw new Error(
        `No plan file found at ${filePath}. Please write your plan before calling ExitPlanMode.`,
      )
    }
    const requestId = generateRequestId('plan_approval', formatAgentId(agentName, teamName))
    
    const approvalRequest = {
      type: 'plan_approval_request',
      from: agentName,
      timestamp: new Date().toISOString(),
      planFilePath: filePath,
      planContent: plan,
      requestId,
    }

    await writeToMailbox('team-lead', approvalRequest, teamName)
    
    return {
      data: {
        plan,
        isAgent: true,
        filePath,
        awaitingLeaderApproval: true,
        requestId,
      },
    }
  }

  // 普通用户：更新 AppState 退出计划模式（第357-403行）
  context.setAppState(prev => {
    if (prev.toolPermissionContext.mode !== 'plan') return prev
    setHasExitedPlanMode(true)
    setNeedsPlanModeExitAttachment(true)

    // 恢复到之前的模式
    let restoreMode = prev.toolPermissionContext.prePlanMode ?? 'default'

    // auto-mode gate 检查（断路器保护）
    if (restoreMode === 'auto' && !isAutoModeGateEnabled()) {
      restoreMode = 'default'
    }

    autoModeStateModule?.setAutoModeActive(restoreMode === 'auto')

    // 恢复权限规则
    let baseContext = prev.toolPermissionContext
    if (restoreMode !== 'auto' && prev.toolPermissionContext.strippedDangerousRules) {
      baseContext = restoreDangerousPermissions(baseContext)
    }

    return {
      ...prev,
      toolPermissionContext: {
        ...baseContext,
        mode: restoreMode,
        prePlanMode: undefined,
      },
    }
  })

  return {
    data: {
      plan,
      isAgent,
      filePath,
      hasTaskTool: hasTaskTool || undefined,
      planWasEdited: inputPlan !== undefined || undefined,
    },
  }
}
```

执行流程的核心步骤：

1. **获取计划内容**：优先从用户编辑的输入获取，否则从磁盘读取
2. **Teammate 特殊处理**：如果设置了 `planModeRequired`，发送审批请求到 leader mailbox
3. **模式恢复**：恢复到 `prePlanMode` 保存的模式，处理 auto-mode 断路器
4. **权限恢复**：如果从 auto 模式退出且之前剥离了危险权限，则恢复

### 15.3.5 工具结果映射

工具结果的映射（第419-492行）：

```typescript
mapToolResultToToolResultBlockParam(
  { isAgent, plan, filePath, hasTaskTool, planWasEdited, awaitingLeaderApproval, requestId },
  toolUseID,
) {
  // teammate 等待 leader 审批
  if (awaitingLeaderApproval) {
    return {
      type: 'tool_result',
      content: `Your plan has been submitted to the team lead for approval.

Plan file: ${filePath}

**What happens next:**
1. Wait for the team lead to review your plan
2. You will receive a message in your inbox with approval/rejection
3. If approved, you can proceed with implementation
4. If rejected, refine your plan based on the feedback

**Important:** Do NOT proceed until you receive approval.

Request ID: ${requestId}`,
      tool_use_id: toolUseID,
    }
  }

  // agent 简化响应
  if (isAgent) {
    return {
      type: 'tool_result',
      content: 'User has approved the plan. Please respond with "ok"',
      tool_use_id: toolUseID,
    }
  }

  // 空计划处理
  if (!plan || plan.trim() === '') {
    return {
      type: 'tool_result',
      content: 'User has approved exiting plan mode. You can now proceed.',
      tool_use_id: toolUseID,
    }
  }

  // 正常批准响应
  const planLabel = planWasEdited ? 'Approved Plan (edited by user)' : 'Approved Plan'
  return {
    type: 'tool_result',
    content: `User has approved your plan. You can now start coding.

Your plan has been saved to: ${filePath}

## ${planLabel}:
${plan}`,
    tool_use_id: toolUseID,
  }
}
```

## 15.4 计划文件管理

计划文件的管理逻辑位于 `/Users/hw/workspaces/projects/claude-wiki/src/utils/plans.ts`，负责计划文件的路径生成、读写操作和会话恢复。

### 15.4.1 计划文件路径

计划文件存储在 `.claude/plans/` 目录下，每个会话使用唯一的 slug 标识：

```typescript
// 第79-111行：获取计划目录
export const getPlansDirectory = memoize(function getPlansDirectory(): string {
  const settings = getInitialSettings()
  const settingsDir = settings.plansDirectory
  let plansPath: string

  if (settingsDir) {
    // 用户自定义目录（相对于项目根）
    const cwd = getCwd()
    const resolved = resolve(cwd, settingsDir)

    // 安全验证：防止路径穿越
    if (!resolved.startsWith(cwd + sep) && resolved !== cwd) {
      logError(new Error(`plansDirectory must be within project root: ${settingsDir}`))
      plansPath = join(getClaudeConfigHomeDir(), 'plans')
    } else {
      plansPath = resolved
    }
  } else {
    // 默认目录
    plansPath = join(getClaudeConfigHomeDir(), 'plans')
  }

  // 确保目录存在
  getFsImplementation().mkdirSync(plansPath)
  return plansPath
})
```

路径生成逻辑（第119-129行）：

```typescript
export function getPlanFilePath(agentId?: AgentId): string {
  const planSlug = getPlanSlug(getSessionId())

  // 主会话：简单文件名
  if (!agentId) {
    return join(getPlansDirectory(), `${planSlug}.md`)
  }

  // 子 Agent：包含 Agent ID
  return join(getPlansDirectory(), `${planSlug}-agent-${agentId}.md`)
}
```

Slug 生成使用单词组合，确保可读性和唯一性（第32-49行）：

```typescript
export function getPlanSlug(sessionId?: SessionId): string {
  const id = sessionId ?? getSessionId()
  const cache = getPlanSlugCache()
  let slug = cache.get(id)
  
  if (!slug) {
    const plansDir = getPlansDirectory()
    // 最多尝试 10 次生成唯一 slug
    for (let i = 0; i < MAX_SLUG_RETRIES; i++) {
      slug = generateWordSlug()
      const filePath = join(plansDir, `${slug}.md`)
      if (!getFsImplementation().existsSync(filePath)) {
        break
      }
    }
    cache.set(id, slug!)
  }
  return slug!
}
```

### 15.4.2 计划文件读写

读取计划内容（第135-144行）：

```typescript
export function getPlan(agentId?: AgentId): string | null {
  const filePath = getPlanFilePath(agentId)
  try {
    return getFsImplementation().readFileSync(filePath, { encoding: 'utf-8' })
  } catch (error) {
    if (isENOENT(error)) return null  // 文件不存在返回 null
    logError(error)
    return null
  }
}
```

写入操作发生在 `ExitPlanModeV2Tool.call()` 中，通过 `writeFile` 写入用户编辑的计划内容。

### 15.4.3 会话恢复

恢复会话时，系统尝试从日志中恢复计划文件（第164-231行）：

```typescript
export async function copyPlanForResume(
  log: LogOption,
  targetSessionId?: SessionId,
): Promise<boolean> {
  const slug = getSlugFromLog(log)
  if (!slug) return false

  setPlanSlug(sessionId, slug)

  const planPath = join(getPlansDirectory(), `${slug}.md`)
  
  try {
    await getFsImplementation().readFile(planPath, { encoding: 'utf-8' })
    return true  // 文件存在
  } catch (e: unknown) {
    if (!isENOENT(e)) {
      logError(e)
      return false
    }

    // 远程会话（CCR）尝试从消息历史恢复
    if (getEnvironmentKind() === null) return false

    // 从文件快照恢复
    const snapshotPlan = findFileSnapshotEntry(log.messages, 'plan')
    let recovered: string | null = null
    if (snapshotPlan && snapshotPlan.content.length > 0) {
      recovered = snapshotPlan.content
    } else {
      // 从消息历史恢复
      recovered = recoverPlanFromMessages(log)
    }

    if (recovered) {
      await writeFile(planPath, recovered, { encoding: 'utf-8' })
      return true
    }
    return false
  }
}
```

恢复机制的三种来源（第279-326行）：

1. **ExitPlanMode tool_use input**：`normalizeToolInput` 将计划内容注入到 tool_use 输入中
2. **planContent 字段**：用户消息上的字段，在"清空上下文并实现"流程中设置
3. **plan_file_reference attachment**：auto-compact 创建的附件，用于跨压缩边界保留计划

### 15.4.4 Fork 会话处理

Fork 会话时生成新的 slug，避免覆盖原计划文件（第239-264行）：

```typescript
export async function copyPlanForFork(
  log: LogOption,
  targetSessionId: SessionId,
): Promise<boolean> {
  const originalSlug = getSlugFromLog(log)
  if (!originalSlug) return false

  const originalPlanPath = join(plansDir, `${originalSlug}.md`)
  
  // 生成新 slug（不复用原 slug）
  const newSlug = getPlanSlug(targetSessionId)
  const newPlanPath = join(plansDir, `${newSlug}.md`)
  
  try {
    await copyFile(originalPlanPath, newPlanPath)
    return true
  } catch (error) {
    if (isENOENT(error)) return false
    logError(error)
    return false
  }
}
```

## 15.5 计划审批流程

### 15.5.1 用户审批 UI

用户审批 UI 位于 `/Users/hw/workspaces/projects/claude-wiki/src/components/permissions/ExitPlanModePermissionRequest/ExitPlanModePermissionRequest.tsx`。

该组件展示计划内容，并提供批准、拒绝和编辑选项。用户可以直接编辑计划内容，编辑后的版本会通过 `permissionResult.updatedInput.plan` 传递给工具。

### 15.5.2 Teammate Leader 审批

对于设置了 `planModeRequired` 的 teammate，计划审批流程发送到 team leader 的 mailbox：

```typescript
// ExitPlanModeV2Tool.ts 第278-296行
const approvalRequest = {
  type: 'plan_approval_request',
  from: agentName,
  timestamp: new Date().toISOString(),
  planFilePath: filePath,
  planContent: plan,
  requestId,
}

await writeToMailbox('team-lead', {
  from: agentName,
  text: jsonStringify(approvalRequest),
  timestamp: new Date().toISOString(),
}, teamName)
```

Leader 通过 mailbox 接收审批请求，批准或拒绝后发送响应到 teammate 的 inbox。

### 15.5.3 计划模式指令

计划模式的详细指令通过 `plan_mode` attachment 发送，位于 `/Users/hw/workspaces/projects/claude-wiki/src/utils/messages.ts`。

**五阶段工作流指令**（第3207-3250行）：

```typescript
function getPlanModeV2Instructions(attachment): UserMessage[] {
  const content = `Plan mode is active. The user indicated that they do not want you to execute yet.

## Plan File Info:
${planFileInfo}

## Plan Workflow

### Phase 1: Initial Understanding
Goal: Gain a comprehensive understanding of the user's request by reading through code and asking them questions.
Critical: In this phase you should only use the ${EXPLORE_AGENT.agentType} subagent type.

### Phase 2: Problem Discovery
Goal: Build a deeper understanding of the problem space...

### Phase 3: Solution Design
Goal: Converge on a solution approach...

### Phase 4: Final Plan
Goal: Write your final plan to the plan file (the only file you can edit).
- Include a Context section explaining why this change is being made
- Include only your recommended approach, not all alternatives
- List the paths of critical files to be modified
- Reference existing functions and utilities to reuse
- Include a verification section for testing

### Phase 5: Present Plan
Goal: Present your plan to the user for approval...
`
}
```

**Interview Phase 工作流**（用于 Ant 用户或启用了该功能的用户）提供更详细的迭代式规划流程。

**计划文件结构实验**（第3156-3189行）：系统通过 `tengu_pewter_ledger` feature gate 控制计划文件的结构建议：

- **Control**：包含 Context、推荐方案、文件路径、验证方法
- **Trim**：一行 Context、文件修改列表、验证命令
- **Cut**：禁止 Context/Background、一行描述每文件修改
- **Cap**：禁止 Context/Background/Overview，硬限制 40 行

### 15.5.4 模式恢复与附件

退出计划模式时，系统发送 `plan_mode_exit` attachment 通知 Claude：

```typescript
// messages.ts 第3848-3859行
case 'plan_mode_exit': {
  const planReference = attachment.planExists
    ? ` The plan file is located at ${attachment.planFilePath} if you need to reference it.`
    : ''
  const content = `## Exited Plan Mode

You have exited plan mode. You can now make edits, run tools, and take actions.${planReference}`

  return wrapMessagesInSystemReminder([
    createUserMessage({ content, isMeta: true }),
  ])
}
```

如果从 auto 模式退出，还会发送 `auto_mode_exit` attachment，提醒 Claude 增强用户交互。

## 15.6 配置与功能开关

计划模式相关配置位于 `/Users/hw/workspaces/projects/claude-wiki/src/utils/planModeV2.ts`：

```typescript
// 第5-29行：Agent 数量配置
export function getPlanModeV2AgentCount(): number {
  if (process.env.CLAUDE_CODE_PLAN_V2_AGENT_COUNT) {
    const count = parseInt(process.env.CLAUDE_CODE_PLAN_V2_AGENT_COUNT, 10)
    if (!isNaN(count) && count > 0 && count <= 10) return count
  }

  const subscriptionType = getSubscriptionType()
  const rateLimitTier = getRateLimitTier()

  // Max 和 Enterprise/Team 用户可使用 3 个 agent
  if (subscriptionType === 'max' && rateLimitTier === 'default_claude_max_20x') {
    return 3
  }
  if (subscriptionType === 'enterprise' || subscriptionType === 'team') {
    return 3
  }
  return 1
}

// 第50-62行：Interview Phase 开关
export function isPlanModeInterviewPhaseEnabled(): boolean {
  // Ant 用户始终启用
  if (process.env.USER_TYPE === 'ant') return true

  const env = process.env.CLAUDE_CODE_PLAN_MODE_INTERVIEW_PHASE
  if (isEnvTruthy(env)) return true
  if (isEnvDefinedFalsy(env)) return false

  // 通过 feature gate 控制
  return getFeatureValue_CACHED_MAY_BE_STALE('tengu_plan_mode_interview_phase', false)
}
```

## 15.7 总结

计划模式工具（EnterPlanModeTool 和 ExitPlanModeV2Tool）构成了 Claude Code 的"规划-执行"工作流的核心：

1. **EnterPlanModeTool**：作为入口，请求用户批准进入只读探索阶段，完成权限上下文切换
2. **ExitPlanModeV2Tool**：作为出口，提交计划给用户审批，处理 teammate leader 审批流程，恢复原模式
3. **计划文件管理**：通过 slug 生成唯一文件路径，支持会话恢复和 fork 处理
4. **审批流程**：普通用户通过 UI 确认，teammate 通过 mailbox 发送 leader 审批请求

这套机制确保了复杂任务的实现方向正确，减少了因理解偏差导致的返工，同时通过计划文件留档实现了实现过程的可追溯性。

---

**相关文件路径**：

- `/Users/hw/workspaces/projects/claude-wiki/src/tools/EnterPlanModeTool/EnterPlanModeTool.ts`
- `/Users/hw/workspaces/projects/claude-wiki/src/tools/EnterPlanModeTool/prompt.ts`
- `/Users/hw/workspaces/projects/claude-wiki/src/tools/EnterPlanModeTool/UI.tsx`
- `/Users/hw/workspaces/projects/claude-wiki/src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts`
- `/Users/hw/workspaces/projects/claude-wiki/src/tools/ExitPlanModeTool/prompt.ts`
- `/Users/hw/workspaces/projects/claude-wiki/src/tools/ExitPlanModeTool/constants.ts`
- `/Users/hw/workspaces/projects/claude-wiki/src/tools/ExitPlanModeTool/UI.tsx`
- `/Users/hw/workspaces/projects/claude-wiki/src/utils/plans.ts`
- `/Users/hw/workspaces/projects/claude-wiki/src/utils/planModeV2.ts`
- `/Users/hw/workspaces/projects/claude-wiki/src/utils/permissions/permissionSetup.ts`
- `/Users/hw/workspaces/projects/claude-wiki/src/utils/messages.ts`
- `/Users/hw/workspaces/projects/claude-wiki/src/components/permissions/ExitPlanModePermissionRequest/ExitPlanModePermissionRequest.tsx`