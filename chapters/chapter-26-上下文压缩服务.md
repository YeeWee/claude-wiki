# 第二十六章：上下文压缩服务

## 26.1 引言

Claude Code 作为 AI 编程助手，需要维护长对话历史以保持上下文连贯性。然而，Claude API 的上下文窗口有限制（例如 Claude 3.5 Sonnet 约 200K tokens），当对话历史超出限制时，必须进行压缩以释放空间。上下文压缩服务（Compaction Service）正是为解决这一问题而设计的核心模块。

压缩服务的核心目标包括：
- **释放上下文空间**：通过摘要和裁剪减少历史消息占用的 token 数
- **保留关键信息**：确保压缩后仍保留必要的技术细节、决策和进度
- **保证缓存效率**：尽可能利用 Prompt Cache 减少重复计算开销
- **用户友好**：提供自动和手动两种压缩方式，减少用户干预

本章将深入分析 Claude Code 的压缩算法实现，包括触发机制、Token 预算管理、消息摘要策略、工具结果截断和历史裁剪机制。

```mermaid
flowchart TB
    subgraph Input["输入：对话消息"]
        M[消息数组]
    end

    subgraph Trigger["触发判断"]
        T1{Token 达到阈值?}
        T2{时间间隔超限?}
        T3{手动 /compact?}
    end

    subgraph MicroCompact["微压缩层"]
        MC1[收集可压缩工具 ID]
        MC2[时间触发判断]
        MC3[清理旧工具结果]
        MC4[缓存编辑路径]
    end

    subgraph MainCompact["主压缩层"]
        C1{Session Memory 可用?}
        C2[Session Memory 压缩]
        C3[传统压缩]
        C4[生成摘要]
    end

    subgraph Cleanup["清理层"]
        CL1[清理缓存状态]
        CL2[重置 Microcompact]
        CL3[Session Start Hooks]
    end

    subgraph Output["输出：压缩后消息"]
        BM[边界标记]
        SM[摘要消息]
        KP[保留消息]
        AT[附件消息]
    end

    M --> T1
    T1 -->|达到阈值| MC1
    T2 -->|超限| MC2
    T3 -->|手动触发| MC1

    MC1 --> MC2
    MC2 -->|触发| MC3
    MC2 -->|不触发| MC4
    MC3 --> C1
    MC4 --> C1

    C1 -->|可用| C2
    C1 -->|不可用| C3
    C3 --> C4

    C2 --> CL1
    C4 --> CL1

    CL1 --> CL2
    CL2 --> CL3
    CL3 --> BM

    BM --> SM
    SM --> KP
    KP --> AT

    figure-26-1: Claude Code 上下文压缩流程图
```

## 26.2 压缩算法概述

Claude Code 的压缩服务采用多层渐进式策略，从轻量级到重量级依次尝试：

### 26.2.1 压缩层级结构

压缩服务定义在 `src/services/compact/` 目录下，包含以下核心模块：

| 模块 | 文件 | 功能 |
|------|------|------|
| 微压缩 | `microCompact.ts` | 清理旧工具结果，轻量级预处理 |
| 自动压缩 | `autoCompact.ts` | 监控阈值，触发压缩流程 |
| Session Memory 压缩 | `sessionMemoryCompact.ts` | 使用预提取的会话记忆替代 API 调用 |
| 主压缩 | `compact.ts` | 调用 Claude API 生成结构化摘要 |
| 消息分组 | `grouping.ts` | 按 API 调用轮次划分消息边界 |
| 压缩提示 | `prompt.ts` | 定义摘要生成的提示模板 |

### 26.2.2 压缩触发条件

压缩流程由三种触发条件驱动：

**1. 自动触发（Auto Compact）**

当 token 使用量达到阈值时自动触发。阈值计算逻辑位于 `autoCompact.ts` (第 72-91 行)：

```typescript
export function getAutoCompactThreshold(model: string): number {
  const effectiveContextWindow = getEffectiveContextWindowSize(model)

  const autocompactThreshold =
    effectiveContextWindow - AUTOCOMPACT_BUFFER_TOKENS  // 13,000 tokens 缓冲

  // 支持环境变量覆盖测试
  const envPercent = process.env.CLAUDE_AUTOCOMPACT_PCT_OVERRIDE
  if (envPercent) {
    const parsed = parseFloat(envPercent)
    if (!isNaN(parsed) && parsed > 0 && parsed <= 100) {
      const percentageThreshold = Math.floor(
        effectiveContextWindow * (parsed / 100),
      )
      return Math.min(percentageThreshold, autocompactThreshold)
    }
  }

  return autocompactThreshold
}
```

**2. 时间触发（Time-based Microcompact）**

当距离上次助手消息超过 API 缓存 TTL（默认 60 分钟）时，服务器端缓存已失效，此时清理旧工具结果不会增加额外开销。配置定义在 `timeBasedMCConfig.ts`：

```typescript
export type TimeBasedMCConfig = {
  enabled: boolean
  gapThresholdMinutes: number  // 默认 60 分钟
  keepRecent: number           // 保留最近 5 个工具结果
}
```

**3. 手动触发（Manual Compact）**

用户执行 `/compact` 命令时触发，位于 `src/commands/compact/compact.ts`。

### 26.2.3 微压缩流程

微压缩（Microcompact）是最轻量的压缩方式，仅清理工具结果而不生成摘要。核心逻辑在 `microCompact.ts` (第 253-293 行)：

```typescript
export async function microcompactMessages(
  messages: Message[],
  toolUseContext?: ToolUseContext,
  querySource?: QuerySource,
): Promise<MicrocompactResult> {
  // 清除抑制标志
  clearCompactWarningSuppression()

  // 时间触发优先，服务器缓存已失效则直接清理
  const timeBasedResult = maybeTimeBasedMicrocompact(messages, querySource)
  if (timeBasedResult) {
    return timeBasedResult
  }

  // 缓存编辑路径：使用 API cache_edits 删除而不破坏缓存
  if (feature('CACHED_MICROCOMPACT')) {
    const mod = await getCachedMCModule()
    if (
      mod.isCachedMicrocompactEnabled() &&
      mod.isModelSupportedForCacheEditing(model) &&
      isMainThreadSource(querySource)
    ) {
      return await cachedMicrocompactPath(messages, querySource)
    }
  }

  // 无需压缩，返回原消息
  return { messages }
}
```

可压缩的工具类型定义在第 41-50 行：

```typescript
const COMPACTABLE_TOOLS = new Set<string>([
  FILE_READ_TOOL_NAME,
  ...SHELL_TOOL_NAMES,
  GREP_TOOL_NAME,
  GLOB_TOOL_NAME,
  WEB_SEARCH_TOOL_NAME,
  WEB_FETCH_TOOL_NAME,
  FILE_EDIT_TOOL_NAME,
  FILE_WRITE_TOOL_NAME,
])
```

## 26.3 Token 预算管理

Token 预算管理是压缩服务的核心决策依据，定义了各级阈值和缓冲值。

### 26.3.1 预算常量定义

位于 `autoCompact.ts` (第 62-70 行)：

```typescript
export const AUTOCOMPACT_BUFFER_TOKENS = 13_000        // 自动压缩缓冲
export const WARNING_THRESHOLD_BUFFER_TOKENS = 20_000 // 警告阈值缓冲
export const ERROR_THRESHOLD_BUFFER_TOKENS = 20_000   // 错误阈值缓冲
export const MANUAL_COMPACT_BUFFER_TOKENS = 3_000     // 手动压缩缓冲
const MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3        // 连续失败熔断
```

### 26.3.2 预算计算函数

`getEffectiveContextWindowSize()` 计算有效上下文窗口，扣除输出 token 预留：

```typescript
export function getEffectiveContextWindowSize(model: string): number {
  const reservedTokensForSummary = Math.min(
    getMaxOutputTokensForModel(model),
    MAX_OUTPUT_TOKENS_FOR_SUMMARY,  // 20,000 tokens
  )
  let contextWindow = getContextWindowForModel(model, getSdkBetas())

  // 支持环境变量限制窗口大小
  const autoCompactWindow = process.env.CLAUDE_CODE_AUTO_COMPACT_WINDOW
  if (autoCompactWindow) {
    const parsed = parseInt(autoCompactWindow, 10)
    if (!isNaN(parsed) && parsed > 0) {
      contextWindow = Math.min(contextWindow, parsed)
    }
  }

  return contextWindow - reservedTokensForSummary
}
```

### 26.3.3 Token 警告状态计算

`calculateTokenWarningState()` 函数返回完整的阈值判断结果：

```typescript
export function calculateTokenWarningState(
  tokenUsage: number,
  model: string,
): {
  percentLeft: number
  isAboveWarningThreshold: boolean
  isAboveErrorThreshold: boolean
  isAboveAutoCompactThreshold: boolean
  isAtBlockingLimit: boolean
} {
  const autoCompactThreshold = getAutoCompactThreshold(model)
  const threshold = isAutoCompactEnabled()
    ? autoCompactThreshold
    : getEffectiveContextWindowSize(model)

  // 计算剩余百分比
  const percentLeft = Math.max(
    0,
    Math.round(((threshold - tokenUsage) / threshold) * 100),
  )

  // 判断各级阈值
  const warningThreshold = threshold - WARNING_THRESHOLD_BUFFER_TOKENS
  const errorThreshold = threshold - ERROR_THRESHOLD_BUFFER_TOKENS
  const isAboveWarningThreshold = tokenUsage >= warningThreshold
  const isAboveErrorThreshold = tokenUsage >= errorThreshold
  const isAboveAutoCompactThreshold =
    isAutoCompactEnabled() && tokenUsage >= autoCompactThreshold

  // 阻塞限制（强制停止点）
  const blockingLimit = actualContextWindow - MANUAL_COMPACT_BUFFER_TOKENS
  const isAtBlockingLimit = tokenUsage >= blockingLimit

  return { percentLeft, isAboveWarningThreshold, isAboveErrorThreshold,
           isAboveAutoCompactThreshold, isAtBlockingLimit }
}
```

### 26.3.4 Token 估算方法

消息 token 估算使用 `estimateMessageTokens()`，考虑各种消息块类型：

```typescript
export function estimateMessageTokens(messages: Message[]): number {
  let totalTokens = 0

  for (const message of messages) {
    for (const block of message.message.content) {
      if (block.type === 'text') {
        totalTokens += roughTokenCountEstimation(block.text)
      } else if (block.type === 'tool_result') {
        totalTokens += calculateToolResultTokens(block)
      } else if (block.type === 'image' || block.type === 'document') {
        totalTokens += IMAGE_MAX_TOKEN_SIZE  // 固定 2000 tokens
      } else if (block.type === 'thinking') {
        totalTokens += roughTokenCountEstimation(block.thinking)
      } else if (block.type === 'tool_use') {
        totalTokens += roughTokenCountEstimation(
          block.name + jsonStringify(block.input ?? {}),
        )
      }
    }
  }

  // 保守估计，增加 4/3 系数
  return Math.ceil(totalTokens * (4 / 3))
}
```

## 26.4 消息摘要策略

当微压缩不足以释放空间时，系统调用 Claude API 生成结构化摘要。

### 26.4.1 摘要提示模板

摘要提示模板定义在 `prompt.ts`，包含分析块和摘要块两部分：

```typescript
const BASE_COMPACT_PROMPT = `Your task is to create a detailed summary of the
conversation so far, paying close attention to the user's explicit requests and
your previous actions.

${DETAILED_ANALYSIS_INSTRUCTION_BASE}

Your summary should include the following sections:

1. Primary Request and Intent: Capture all of the user's explicit requests...
2. Key Technical Concepts: List all important technical concepts...
3. Files and Code Sections: Enumerate specific files and code sections...
4. Errors and fixes: List all errors that you ran into...
5. Problem Solving: Document problems solved...
6. All user messages: List ALL user messages...
7. Pending Tasks: Outline any pending tasks...
8. Current Work: Describe in detail what was being worked on...
9. Optional Next Step: List the next step...
`
```

分析指令要求模型先进行深度分析：

```typescript
const DETAILED_ANALYSIS_INSTRUCTION_BASE = `Before providing your final summary,
wrap your analysis in <analysis> tags to organize your thoughts...

1. Chronologically analyze each message and section...
2. Double-check for technical accuracy and completeness...`
```

### 26.4.2 部分压缩提示

部分压缩支持两种方向：

- **'from' 方向**：保留早期消息，摘要后续消息
- **'up_to' 方向**：摘要早期消息，保留后续消息

```typescript
export function getPartialCompactPrompt(
  customInstructions?: string,
  direction: PartialCompactDirection = 'from',
): string {
  const template =
    direction === 'up_to'
      ? PARTIAL_COMPACT_UP_TO_PROMPT
      : PARTIAL_COMPACT_PROMPT

  let prompt = NO_TOOLS_PREAMBLE + template
  if (customInstructions && customInstructions.trim() !== '') {
    prompt += `\n\nAdditional Instructions:\n${customInstructions}`
  }
  prompt += NO_TOOLS_TRAILER
  return prompt
}
```

### 26.4.3 摘要格式化

`formatCompactSummary()` 清理原始摘要输出：

```typescript
export function formatCompactSummary(summary: string): string {
  let formattedSummary = summary

  // 移除分析块（仅用于起草，不保留在最终摘要）
  formattedSummary = formattedSummary.replace(
    /<analysis>[\s\S]*?<\/analysis>/,
    '',
  )

  // 提取并格式化摘要块
  const summaryMatch = formattedSummary.match(/<summary>([\s\S]*?)<\/summary>/)
  if (summaryMatch) {
    const content = summaryMatch[1] || ''
    formattedSummary = formattedSummary.replace(
      /<summary>[\s\S]*?<\/summary>/,
      `Summary:\n${content.trim()}`,
    )
  }

  return formattedSummary.trim()
}
```

### 26.4.4 Session Memory 压缩

Session Memory 压缩是一种优化路径，使用预提取的会话记忆替代 API 调用生成摘要：

```typescript
export async function trySessionMemoryCompaction(
  messages: Message[],
  agentId?: AgentId,
  autoCompactThreshold?: number,
): Promise<CompactionResult | null> {
  if (!shouldUseSessionMemoryCompaction()) {
    return null
  }

  await initSessionMemoryCompactConfig()
  await waitForSessionMemoryExtraction()

  const sessionMemory = await getSessionMemoryContent()

  // Session Memory 文件不存在或为空模板时回退到传统压缩
  if (!sessionMemory || await isSessionMemoryEmpty(sessionMemory)) {
    return null
  }

  // 计算保留消息的起始索引
  const startIndex = calculateMessagesToKeepIndex(
    messages,
    lastSummarizedIndex,
  )
  const messagesToKeep = messages
    .slice(startIndex)
    .filter(m => !isCompactBoundaryMessage(m))

  // 生成压缩结果
  const compactionResult = createCompactionResultFromSessionMemory(
    messages,
    sessionMemory,
    messagesToKeep,
    hookResults,
    transcriptPath,
    agentId,
  )

  return compactionResult
}
```

配置参数控制保留范围：

```typescript
export const DEFAULT_SM_COMPACT_CONFIG: SessionMemoryCompactConfig = {
  minTokens: 10_000,          // 最少保留 token 数
  minTextBlockMessages: 5,    // 最少保留文本消息数
  maxTokens: 40_000,          // 最大保留 token 数
}
```

## 26.5 工具结果截断

工具结果往往占用大量 token，是压缩的主要目标。

### 26.5.1 工具结果清理标记

时间微压缩使用固定标记替换工具结果内容：

```typescript
export const TIME_BASED_MC_CLEARED_MESSAGE = '[Old tool result content cleared]'
```

清理逻辑在 `maybeTimeBasedMicrocompact()` 中：

```typescript
function maybeTimeBasedMicrocompact(
  messages: Message[],
  querySource: QuerySource | undefined,
): MicrocompactResult | null {
  const trigger = evaluateTimeBasedTrigger(messages, querySource)
  if (!trigger) return null

  const compactableIds = collectCompactableToolIds(messages)
  const keepRecent = Math.max(1, config.keepRecent)  // 至少保留 1 个
  const keepSet = new Set(compactableIds.slice(-keepRecent))
  const clearSet = new Set(compactableIds.filter(id => !keepSet.has(id)))

  if (clearSet.size === 0) return null

  let tokensSaved = 0
  const result: Message[] = messages.map(message => {
    if (message.type !== 'user' || !Array.isArray(message.message.content)) {
      return message
    }
    const newContent = message.message.content.map(block => {
      if (
        block.type === 'tool_result' &&
        clearSet.has(block.tool_use_id) &&
        block.content !== TIME_BASED_MC_CLEARED_MESSAGE
      ) {
        tokensSaved += calculateToolResultTokens(block)
        return { ...block, content: TIME_BASED_MC_CLEARED_MESSAGE }
      }
      return block
    })
    return { ...message, message: { ...message.message, content: newContent } }
  })

  return { messages: result }
}
```

### 26.5.2 缓存编辑路径

Cached Microcompact 使用 API 的 `cache_edits` 功能删除工具结果而不破坏缓存：

```typescript
async function cachedMicrocompactPath(
  messages: Message[],
  querySource: QuerySource | undefined,
): Promise<MicrocompactResult> {
  const compactableToolIds = new Set(collectCompactableToolIds(messages))

  // 注册工具结果到全局状态
  for (const message of messages) {
    if (message.type === 'user' && Array.isArray(message.message.content)) {
      for (const block of message.message.content) {
        if (
          block.type === 'tool_result' &&
          compactableToolIds.has(block.tool_use_id)
        ) {
          mod.registerToolResult(state, block.tool_use_id)
        }
      }
    }
  }

  const toolsToDelete = mod.getToolResultsToDelete(state)

  if (toolsToDelete.length > 0) {
    // 创建 cache_edits 块
    const cacheEdits = mod.createCacheEditsBlock(state, toolsToDelete)
    pendingCacheEdits = cacheEdits

    return {
      messages,
      compactionInfo: {
        pendingCacheEdits: {
          trigger: 'auto',
          deletedToolIds: toolsToDelete,
          baselineCacheDeletedTokens: baseline,
        },
      },
    }
  }

  return { messages }
}
```

### 26.5.3 API 级工具清理策略

`apiMicrocompact.ts` 定义了 API 端的上下文管理策略：

```typescript
export type ContextEditStrategy =
  | {
      type: 'clear_tool_uses_20250919'
      trigger?: { type: 'input_tokens'; value: number }
      keep?: { type: 'tool_uses'; value: number }
      clear_tool_inputs?: boolean | string[]
      exclude_tools?: string[]
      clear_at_least?: { type: 'input_tokens'; value: number }
    }
  | {
      type: 'clear_thinking_20251015'
      keep: { type: 'thinking_turns'; value: number } | 'all'
    }
```

可清理工具结果列表：

```typescript
const TOOLS_CLEARABLE_RESULTS = [
  ...SHELL_TOOL_NAMES,
  GLOB_TOOL_NAME,
  GREP_TOOL_NAME,
  FILE_READ_TOOL_NAME,
  WEB_FETCH_TOOL_NAME,
  WEB_SEARCH_TOOL_NAME,
]
```

## 26.6 历史裁剪机制

当压缩请求本身超出 prompt-too-long 限制时，需要裁剪历史消息。

### 26.6.1 API 轮次分组

消息按 API 调用轮次分组，确保工具调用-结果配对完整性：

```typescript
export function groupMessagesByApiRound(messages: Message[]): Message[][] {
  const groups: Message[][] = []
  let current: Message[] = []
  let lastAssistantId: string | undefined

  for (const msg of messages) {
    // 当助手消息 ID 变化时，说明新 API 调用开始
    if (
      msg.type === 'assistant' &&
      msg.message.id !== lastAssistantId &&
      current.length > 0
    ) {
      groups.push(current)
      current = [msg]
    } else {
      current.push(msg)
    }
    if (msg.type === 'assistant') {
      lastAssistantId = msg.message.id
    }
  }

  if (current.length > 0) {
    groups.push(current)
  }
  return groups
}
```

### 26.6.2 Prompt-Too-Long 重试裁剪

当压缩 API 调用返回 prompt-too-long 时，裁剪最老的消息组：

```typescript
export function truncateHeadForPTLRetry(
  messages: Message[],
  ptlResponse: AssistantMessage,
): Message[] | null {
  // 移除之前重试的合成标记
  const input =
    messages[0]?.type === 'user' &&
    messages[0].isMeta &&
    messages[0].message.content === PTL_RETRY_MARKER
      ? messages.slice(1)
      : messages

  const groups = groupMessagesByApiRound(input)
  if (groups.length < 2) return null

  // 计算 token 缺口
  const tokenGap = getPromptTooLongTokenGap(ptlResponse)
  let dropCount: number
  if (tokenGap !== undefined) {
    let acc = 0
    dropCount = 0
    for (const g of groups) {
      acc += roughTokenCountEstimationForMessages(g)
      dropCount++
      if (acc >= tokenGap) break
    }
  } else {
    // 无法解析缺口时，丢弃 20%
    dropCount = Math.max(1, Math.floor(groups.length * 0.2))
  }

  // 至少保留一个组用于摘要
  dropCount = Math.min(dropCount, groups.length - 1)
  if (dropCount < 1) return null

  const sliced = groups.slice(dropCount).flat()

  // 如果裁剪后第一条是助手消息，添加合成用户标记满足 API 要求
  if (sliced[0]?.type === 'assistant') {
    return [
      createUserMessage({ content: PTL_RETRY_MARKER, isMeta: true }),
      ...sliced,
    ]
  }
  return sliced
}
```

### 26.6.3 工具配对完整性保护

裁剪时必须保持 tool_use 和 tool_result 配对完整：

```typescript
export function adjustIndexToPreserveAPIInvariants(
  messages: Message[],
  startIndex: number,
): number {
  if (startIndex <= 0 || startIndex >= messages.length) {
    return startIndex
  }

  let adjustedIndex = startIndex

  // 收集保留范围内的所有 tool_result ID
  const allToolResultIds: string[] = []
  for (let i = startIndex; i < messages.length; i++) {
    allToolResultIds.push(...getToolResultIds(messages[i]!))
  }

  if (allToolResultIds.length > 0) {
    // 找到匹配的 tool_use 消息并扩展保留范围
    const neededToolUseIds = new Set(
      allToolResultIds.filter(id => !toolUseIdsInKeptRange.has(id)),
    )
    for (let i = adjustedIndex - 1; i >= 0 && neededToolUseIds.size > 0; i--) {
      if (hasToolUseWithIds(messages[i]!, neededToolUseIds)) {
        adjustedIndex = i
        // 移除已找到的 ID
        for (const block of messages[i]!.message.content) {
          if (block.type === 'tool_use' && neededToolUseIds.has(block.id)) {
            neededToolUseIds.delete(block.id)
          }
        }
      }
    }
  }

  // 处理共享 message.id 的 thinking 块
  // 流式输出可能产生多个助手消息块共享同一 message.id
  const messageIdsInKeptRange = new Set<string>()
  for (let i = adjustedIndex; i < messages.length; i++) {
    if (messages[i]!.type === 'assistant' && messages[i]!.message.id) {
      messageIdsInKeptRange.add(messages[i]!.message.id)
    }
  }
  for (let i = adjustedIndex - 1; i >= 0; i--) {
    if (
      messages[i]!.type === 'assistant' &&
      messages[i]!.message.id &&
      messageIdsInKeptRange.has(messages[i]!.message.id)
    ) {
      adjustedIndex = i
    }
  }

  return adjustedIndex
}
```

## 26.7 压缩后清理

压缩完成后需要清理各种缓存和状态，位于 `postCompactCleanup.ts`：

```typescript
export function runPostCompactCleanup(querySource?: QuerySource): void {
  const isMainThreadCompact =
    querySource === undefined ||
    querySource.startsWith('repl_main_thread') ||
    querySource === 'sdk'

  resetMicrocompactState()

  // 重置 Context Collapse 状态
  if (feature('CONTEXT_COLLAPSE')) {
    if (isMainThreadCompact) {
      require('../contextCollapse/index.js').resetContextCollapse()
    }
  }

  if (isMainThreadCompact) {
    getUserContext.cache.clear?.()
    resetGetMemoryFilesCache('compact')
  }

  // 清理系统提示区段
  clearSystemPromptSections()
  clearClassifierApprovals()
  clearSpeculativeChecks()
  clearBetaTracingState()
  clearSessionMessagesCache()
}
```

## 26.8 压缩边界标记

压缩完成后生成边界标记（Boundary Marker），用于标识压缩发生点：

```typescript
const boundaryMarker = createCompactBoundaryMessage(
  isAutoCompact ? 'auto' : 'manual',
  preCompactTokenCount ?? 0,
  messages.at(-1)?.uuid,
)

// 携带已发现工具状态
const preCompactDiscovered = extractDiscoveredToolNames(messages)
if (preCompactDiscovered.size > 0) {
  boundaryMarker.compactMetadata.preCompactDiscoveredTools = [
    ...preCompactDiscovered,
  ].sort()
}
```

边界标记还支持保留片段元数据，用于链接保留的消息：

```typescript
export function annotateBoundaryWithPreservedSegment(
  boundary: SystemCompactBoundaryMessage,
  anchorUuid: UUID,
  messagesToKeep: readonly Message[] | undefined,
): SystemCompactBoundaryMessage {
  const keep = messagesToKeep ?? []
  if (keep.length === 0) return boundary
  return {
    ...boundary,
    compactMetadata: {
      ...boundary.compactMetadata,
      preservedSegment: {
        headUuid: keep[0]!.uuid,
        anchorUuid,
        tailUuid: keep.at(-1)!.uuid,
      },
    },
  }
}
```

## 26.9 小结

Claude Code 的上下文压缩服务是一个多层次、渐进式的智能系统：

1. **分层策略**：从微压缩到 Session Memory 压缩再到传统 API 摘要，按代价递增尝试
2. **缓存优化**：时间触发与缓存 TTL 同步，缓存编辑路径保持缓存有效性
3. **Token 预算**：多级阈值（警告、错误、阻塞）确保可控性
4. **完整性保护**：工具配对和 thinking 块合并的完整性检查
5. **结构化摘要**：详细的 9 部分摘要模板确保关键信息不丢失

这种设计在保证上下文连续性的同时，最大化利用有限 token 空间，是长对话 AI 系统的核心基础设施。