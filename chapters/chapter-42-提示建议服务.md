# 第四十二章：提示建议服务

> 本章基于 Claude Code 源代码分析，请以最新版本为准。

## 42.1 引言

Claude Code 的提示建议服务（Prompt Suggestion Service）是一个智能预测系统，能够在每次对话轮次结束后，预测用户可能输入的下一步内容。这种 AI 驱动的建议机制旨在提升交互效率，减少用户思考下一步操作的时间。

提示建议服务的核心设计理念包括：

- **预测性**：基于对话上下文，预测用户自然想要输入的内容
- **轻量化**：通过 Forked Agent 机制复用主线程的 Prompt Cache，避免重复计算
- **可过滤**：严格的过滤机制确保建议符合用户习惯，而非 AI 视角
- **推测执行**：可选的 Speculation 功能可提前执行预测的命令，进一步节省时间

本章将深入分析提示建议服务的架构设计、生成机制、上下文感知策略、过滤规则以及与 UI 的集成方式。

```mermaid
flowchart TB
    subgraph Trigger["触发条件"]
        T1[助手消息完成]
        T2{检查启用状态}
        T3{检查抑制条件}
    end

    subgraph Generation["建议生成"]
        G1[保存 CacheSafeParams]
        G2[创建 Forked Agent]
        G3[发送建议提示]
        G4[提取文本建议]
    end

    subgraph Filtering["过滤检查"]
        F1{空建议?}
        F2{字数检查}
        F3{格式检查}
        F4{语气检查}
        F5{Claude视角检查}
    end

    subgraph Speculation["推测执行"]
        S1{启用 Speculation?}
        S2[启动推测流程]
        S3[创建 Overlay 文件层]
        S4[执行预测操作]
        S5[记录边界状态]
    end

    subgraph UI["UI集成"]
        U1[更新 AppState]
        U2[显示建议文本]
        U3[用户按 Tab 接受]
        U4[用户按 Enter 接受]
        U5[记录分析事件]
    end

    T1 --> T2
    T2 -->|已启用| T3
    T2 -->|未启用| END[结束]
    T3 -->|无抑制| G1
    T3 -->|有抑制| END

    G1 --> G2
    G2 --> G3
    G3 --> G4
    G4 --> F1

    F1 -->|非空| F2
    F1 -->|空| END
    F2 -->|合格| F3
    F2 -->|不合格| END
    F3 -->|合格| F4
    F3 -->|不合格| END
    F4 -->|合格| F5
    F4 -->|不合格| END
    F5 -->|合格| S1
    F5 -->|不合格| END

    S1 -->|启用| S2
    S1 -->|禁用| U1
    S2 --> S3
    S3 --> S4
    S4 --> S5
    S5 --> U1

    U1 --> U2
    U2 --> U3
    U2 --> U4
    U3 --> U5
    U4 --> U5

    figure-42-1: Claude Code 提示建议服务完整流程图
```

## 42.2 服务启用判断

提示建议服务并非无条件启用，系统通过多层检查确定是否允许生成建议。

### 42.2.1 启用条件检查

启用判断逻辑位于 `promptSuggestion.ts`（shouldEnablePromptSuggestion 函数区域）：

```typescript
export function shouldEnablePromptSuggestion(): boolean {
  // 1. 环境变量覆盖（测试优先）
  const envOverride = process.env.CLAUDE_CODE_ENABLE_PROMPT_SUGGESTION
  if (isEnvDefinedFalsy(envOverride)) {
    logEvent('tengu_prompt_suggestion_init', { enabled: false, source: 'env' })
    return false
  }
  if (isEnvTruthy(envOverride)) {
    logEvent('tengu_prompt_suggestion_init', { enabled: true, source: 'env' })
    return true
  }

  // 2. GrowthBook Feature Flag 检查
  if (!getFeatureValue_CACHED_MAY_BE_STALE('tengu_chomp_inflection', false)) {
    logEvent('tengu_prompt_suggestion_init', { enabled: false, source: 'growthbook' })
    return false
  }

  // 3. 非交互模式禁用（print mode, piped input, SDK）
  if (getIsNonInteractiveSession()) {
    logEvent('tengu_prompt_suggestion_init', { enabled: false, source: 'non_interactive' })
    return false
  }

  // 4. Swarm teammate 禁用（只有 leader 显示建议）
  if (isAgentSwarmsEnabled() && isTeammate()) {
    logEvent('tengu_prompt_suggestion_init', { enabled: false, source: 'swarm_teammate' })
    return false
  }

  // 5. 用户设置最终决定
  const enabled = getInitialSettings()?.promptSuggestionEnabled !== false
  return enabled
}
```

启用检查的优先级顺序：

| 优先级 | 来源 | 说明 |
|--------|------|------|
| 1 | 环境变量 | 测试或强制启用/禁用 |
| 2 | Feature Flag | 渐进式发布控制 |
| 3 | 非交互检测 | SDK/Print 模式不显示 |
| 4 | Swarm teammate | 多 Agent 场景仅 leader |
| 5 | 用户设置 | 默认启用，可手动关闭 |

### 42.2.2 抑制条件检查

即使服务已启用，特定状态会抑制建议生成。抑制判断定义在 getSuggestionSuppressReason 函数区域：

```typescript
export function getSuggestionSuppressReason(appState: AppState): string | null {
  if (!appState.promptSuggestionEnabled) return 'disabled'
  if (appState.pendingWorkerRequest || appState.pendingSandboxRequest)
    return 'pending_permission'
  if (appState.elicitation.queue.length > 0) return 'elicitation_active'
  if (appState.toolPermissionContext.mode === 'plan') return 'plan_mode'
  if (
    process.env.USER_TYPE === 'external' &&
    currentLimits.status !== 'allowed'
  )
    return 'rate_limit'
  return null
}
```

抑制原因分类：

- **disabled**：用户禁用建议功能
- **pending_permission**：有待处理的权限请求
- **elicitation_active**：有活跃的交互式询问队列
- **plan_mode**：计划模式（需要手动操作）
- **rate_limit**：外部用户达到使用限制

## 42.3 建议生成机制

### 42.3.1 Forked Agent 架构

建议生成使用 Forked Agent 机制，这是 Claude Code 的子 Agent 执行框架。核心优势是复用主线程的 Prompt Cache，避免重复处理系统提示和工具定义。

CacheSafeParams 类型定义（`forkedAgent.ts` 类型定义区域）：

```typescript
export type CacheSafeParams = {
  /** System prompt - must match parent for cache hits */
  systemPrompt: SystemPrompt
  /** User context - prepended to messages, affects cache */
  userContext: { [k: string]: string }
  /** System context - appended to system prompt, affects cache */
  systemContext: { [k: string]: string }
  /** Tool use context containing tools, model, and other options */
  toolUseContext: ToolUseContext
  /** Parent context messages for prompt cache sharing */
  forkContextMessages: Message[]
}
```

CacheSafeParams 在每次主线程采样完成后保存（`stopHooks.ts` 保存区域）：

```typescript
if (querySource === 'repl_main_thread' || querySource === 'sdk') {
  saveCacheSafeParams(createCacheSafeParams(stopHookContext))
}
```

### 42.3.2 建议提示模板

建议提示使用专用的提示模板，引导模型预测用户意图。模板定义在 SUGGESTION_PROMPT 常量区域：

```typescript
const SUGGESTION_PROMPT = `[SUGGESTION MODE: Suggest what the user might naturally type next into Claude Code.]

FIRST: Look at the user's recent messages and original request.

Your job is to predict what THEY would type - not what you think they should do.

THE TEST: Would they think "I was just about to type that"?

EXAMPLES:
User asked "fix the bug and run tests", bug is fixed -> "run the tests"
After code written -> "try it out"
Claude offers options -> suggest the one the user would likely pick, based on conversation
Claude asks to continue -> "yes" or "go ahead"
Task complete, obvious follow-up -> "commit this" or "push it"
After error or misunderstanding -> silence (let them assess/correct)

Be specific: "run the tests" beats "continue".

NEVER SUGGEST:
- Evaluative ("looks good", "thanks")
- Questions ("what about...?")
- Claude-voice ("Let me...", "I'll...", "Here's...")
- New ideas they didn't ask about
- Multiple sentences

Stay silent if the next step isn't obvious from what the user said.

Format: 2-12 words, match the user's style. Or nothing.

Reply with ONLY the suggestion, no quotes or explanation.`
```

提示模板的核心设计原则：

1. **用户视角**：预测用户想输入的内容，而非 AI 认为应该做的事情
2. **确定性测试**：只有当用户会想"我正要输入这个"时才生成建议
3. **简洁格式**：2-12 个词，匹配用户风格
4. **明确否定**：禁止评估性、疑问性、Claude 视角的建议

### 42.3.3 生成流程实现

核心生成函数 `generateSuggestion()` 位于生成流程区域：

```typescript
export async function generateSuggestion(
  abortController: AbortController,
  promptId: PromptVariant,
  cacheSafeParams: CacheSafeParams,
): Promise<{ suggestion: string | null; generationRequestId: string | null }> {
  const prompt = SUGGESTION_PROMPTS[promptId]

  // 通过 canUseTool 回调拒绝所有工具，而非传递 tools:[]（会破坏缓存）
  const canUseTool = async () => ({
    behavior: 'deny' as const,
    message: 'No tools needed for suggestion',
    decisionReason: { type: 'other' as const, reason: 'suggestion only' },
  })

  // 关键：不覆盖任何与主线程不同的 API 参数
  // Fork 复用主线程的 Prompt Cache，需要发送相同的 cache-key 参数
  const result = await runForkedAgent({
    promptMessages: [createUserMessage({ content: prompt })],
    cacheSafeParams,  // 不覆盖 tools/thinking 设置 - 会破坏缓存
    canUseTool,
    querySource: 'prompt_suggestion',
    forkLabel: 'prompt_suggestion',
    overrides: { abortController },
    skipTranscript: true,
    skipCacheWrite: true,
  })

  // 从所有助手消息中提取文本建议
  const firstAssistantMsg = result.messages.find(m => m.type === 'assistant')
  const generationRequestId = firstAssistantMsg?.requestId ?? null

  for (const msg of result.messages) {
    if (msg.type !== 'assistant') continue
    const textBlock = msg.message.content.find(b => b.type === 'text')
    if (textBlock?.type === 'text') {
      const suggestion = textBlock.text.trim()
      if (suggestion) {
        return { suggestion, generationRequestId }
      }
    }
  }

  return { suggestion: null, generationRequestId }
}
```

缓存复用的关键约束：

| 参数 | 处理方式 | 原因 |
|------|----------|------|
| tools | 通过 canUseTool 拒绝 | tools:[] 会破坏缓存 |
| thinking config | 不覆盖 | 是缓存键的一部分 |
| maxOutputTokens | 不设置 | 会间接改变 budget_tokens |
| abortController | 可覆盖 | 不发送到 API |
| skipTranscript | 可设置 | 仅客户端行为 |

### 42.3.4 统一生成入口

`tryGenerateSuggestion()` 是共享的生成入口（统一生成入口区域）：

```typescript
export async function tryGenerateSuggestion(
  abortController: AbortController,
  messages: Message[],
  getAppState: () => AppState,
  cacheSafeParams: CacheSafeParams,
  source?: 'cli' | 'sdk',
): Promise<{
  suggestion: string
  promptId: PromptVariant
  generationRequestId: string | null
} | null> {
  // 前置检查
  if (abortController.signal.aborted) return null

  // 要求至少 2 轮助手消息（避免早期对话生成无意义建议）
  const assistantTurnCount = count(messages, m => m.type === 'assistant')
  if (assistantTurnCount < 2) return null

  // 最后一条助手消息是错误时不生成
  const lastAssistantMessage = getLastAssistantMessage(messages)
  if (lastAssistantMessage?.isApiErrorMessage) return null

  // 父请求缓存冷却时不生成（超过 10,000 未缓存 tokens）
  const cacheReason = getParentCacheSuppressReason(lastAssistantMessage)
  if (cacheReason) return null

  // 状态抑制检查
  const appState = getAppState()
  const suppressReason = getSuggestionSuppressReason(appState)
  if (suppressReason) return null

  // 生成并过滤
  const promptId = getPromptVariant()
  const { suggestion, generationRequestId } = await generateSuggestion(...)
  if (shouldFilterSuggestion(suggestion, promptId, source)) return null

  return { suggestion, promptId, generationRequestId }
}
```

## 42.4 上下文感知策略

### 42.4.1 消息历史要求

建议生成需要足够的对话历史作为上下文。系统要求至少 2 轮助手消息：

```typescript
const assistantTurnCount = count(messages, m => m.type === 'assistant')
if (assistantTurnCount < 2) {
  logSuggestionSuppressed('early_conversation', undefined, undefined, source)
  return null
}
```

早期对话抑制的原因：

- 单轮对话缺乏足够上下文判断用户意图
- 首次响应后用户可能有多种选择方向
- 避免生成与用户期望不符的建议

### 42.4.2 缓存温度检查

父请求的缓存状态影响建议生成的经济性。当未缓存 tokens 过多时，Fork 请求成本过高：

```typescript
const MAX_PARENT_UNCACHED_TOKENS = 10_000

export function getParentCacheSuppressReason(
  lastAssistantMessage: ReturnType<typeof getLastAssistantMessage>,
): string | null {
  if (!lastAssistantMessage) return null

  const usage = lastAssistantMessage.message.usage
  const inputTokens = usage.input_tokens ?? 0
  const cacheWriteTokens = usage.cache_creation_input_tokens ?? 0
  const outputTokens = usage.output_tokens ?? 0

  return inputTokens + cacheWriteTokens + outputTokens >
    MAX_PARENT_UNCACHED_TOKENS
    ? 'cache_cold'
    : null
}
```

缓存冷却抑制（cache_cold）的场景：

- 长对话导致大量未缓存 tokens
- 新会话开始（无缓存基础）
- 模型切换导致缓存失效

### 42.4.3 上下文消息传递

Forked Agent 通过 `forkContextMessages` 继承主线程的对话历史：

```typescript
export type CacheSafeParams = {
  ...
  forkContextMessages: Message[]  // 父上下文消息
}
```

这些消息与建议提示拼接，形成完整的输入上下文：

```typescript
const augmentedContext: REPLHookContext = {
  ...context,
  messages: [
    ...context.messages,
    createUserMessage({ content: suggestionText }),
    ...speculatedMessages,
  ],
}
```

## 42.5 用户偏好学习

提示建议服务通过多种机制学习和适应用户偏好，持续改进建议质量。

### 42.5.1 风格匹配策略

提示模板明确要求匹配用户风格（Format 配置区域）：

```typescript
Format: 2-12 words, match the user's style. Or nothing.
```

用户风格匹配的实现维度：

| 维度 | 实现方式 | 示例 |
|------|----------|------|
| **语言风格** | 提示模板引导 | 简短命令 vs 详细描述 |
| **操作习惯** | 过滤器例外 | 允许常用单字命令 |
| **偏好命令** | 斜杠命令豁免 | `/commit`、`/review` 等 |

### 42.5.2 常用命令学习

过滤器包含用户常用命令的例外列表（ALLOWED_SINGLE_WORDS 常量区域）：

```typescript
const ALLOWED_SINGLE_WORDS = new Set([
  // Affirmatives - 用户确认习惯
  'yes', 'yeah', 'yep', 'yea', 'yup', 'sure', 'ok', 'okay',
  // Actions - 用户常用操作命令
  'push', 'commit', 'deploy', 'stop', 'continue', 'check', 'exit', 'quit',
  // Negation - 用户否定习惯
  'no'
])
```

这些例外反映了实际用户输入模式：

1. **确认词多样性**：不同用户偏好不同的确认表达
2. **Git 操作高频**：`commit`、`push` 是开发者常用操作
3. **会话控制**：`exit`、`quit`、`stop` 用于中断流程

### 42.5.3 分析数据收集

每次建议交互都记录详细的指标数据（logSuggestionOutcome 函数区域）：

```typescript
export function logSuggestionOutcome(
  suggestion: string,
  userInput: string,
  emittedAt: number,
  promptId: PromptVariant,
  generationRequestId: string | null,
): void {
  const similarity =
    Math.round((userInput.length / (suggestion.length || 1)) * 100) / 100
  const wasAccepted = userInput === suggestion
  const timeMs = Math.max(0, Date.now() - emittedAt)

  logEvent('tengu_prompt_suggestion', {
    source: 'sdk',
    outcome: wasAccepted ? 'accepted' : 'ignored',
    prompt_id: promptId,
    generationRequestId,
    ...(wasAccepted && { timeToAcceptMs: timeMs }),
    ...(!wasAccepted && { timeToIgnoreMs: timeMs }),
    similarity,
    // 内部用户记录完整建议内容用于分析
    ...(process.env.USER_TYPE === 'ant' && {
      suggestion,
      userInput,
    }),
  })
}
```

关键学习指标：

| 指标 | 学习用途 |
|------|----------|
| **outcome** | 整体建议质量评估 |
| **timeToAcceptMs** | 建议吸引力（快速接受 = 高质量） |
| **timeToIgnoreMs** | 建议干扰程度 |
| **similarity** | 预测准确度（用户输入与建议的相似度） |
| **suggestion/userInput** | 深度分析预测偏差 |

### 42.5.4 过滤器数据分析

所有被过滤的建议都记录原因（logSuggestionSuppressed 函数区域）：

```typescript
export function logSuggestionSuppressed(
  reason: string,
  suggestion?: string,
  promptId?: PromptVariant,
  source?: 'cli' | 'sdk',
): void {
  logEvent('tengu_prompt_suggestion', {
    source,
    outcome: 'suppressed',
    reason,
    prompt_id: promptId ?? getPromptVariant(),
    ...(process.env.USER_TYPE === 'ant' && suggestion && {
      suggestion,
    }),
  })
}
```

抑制原因统计用于改进：

- **early_conversation**：调整对话轮次阈值
- **cache_cold**：优化缓存策略
- **claude_voice**：改进提示模板强调用户视角
- **evaluative**：增强评估性文本检测

### 42.5.5 RL 数据关联

`generationRequestId` 用于将建议生成与后续用户行为关联：

```typescript
generationRequestId: string | null  // 用于 RL 数据集 joins
```

这种关联支持强化学习训练：

1. **生成阶段**：记录请求 ID
2. **交互阶段**：记录接受/忽略结果
3. **关联分析**：将生成参数与用户反馈匹配
4. **模型调优**：基于反馈改进生成策略

## 42.6 建议过滤规则

### 42.6.1 过滤器设计理念

过滤系统确保建议符合用户习惯而非 AI 视角。核心过滤器定义在 shouldFilterSuggestion 函数区域：

```typescript
export function shouldFilterSuggestion(
  suggestion: string | null,
  promptId: PromptVariant,
  source?: 'cli' | 'sdk',
): boolean {
  if (!suggestion) return true

  const lower = suggestion.toLowerCase()
  const wordCount = suggestion.trim().split(/\s+/).length

  const filters: Array<[string, () => boolean]> = [
    ['done', () => lower === 'done'],
    ['meta_text', () => 
      lower === 'nothing found' ||
      lower.startsWith('nothing to suggest') ||
      /\bsilence is\b|\bstay(s|ing)? silent\b/.test(lower) ||
      /^\W*silence\W*$/.test(lower)
    ],
    ['meta_wrapped', () => /^\(.*\)$|^\[.*\]$/.test(suggestion)],
    ['error_message', () =>
      lower.startsWith('api error:') ||
      lower.startsWith('prompt is too long')
    ],
    ['prefixed_label', () => /^\w+:\s/.test(suggestion)],
    ['too_few_words', () => {
      if (wordCount >= 2) return false
      if (suggestion.startsWith('/')) return false  // 允许斜杠命令
      const ALLOWED_SINGLE_WORDS = new Set([
        'yes', 'yeah', 'yep', 'yea', 'yup', 'sure', 'ok', 'okay',
        'push', 'commit', 'deploy', 'stop', 'continue', 'check', 'exit', 'quit',
        'no'
      ])
      return !ALLOWED_SINGLE_WORDS.has(lower)
    }],
    ['too_many_words', () => wordCount > 12],
    ['too_long', () => suggestion.length >= 100],
    ['multiple_sentences', () => /[.!?]\s+[A-Z]/.test(suggestion)],
    ['has_formatting', () => /[\n*]|\*\*/.test(suggestion)],
    ['evaluative', () =>
      /thanks|thank you|looks good|sounds good|that works|nice|great/.test(lower)
    ],
    ['claude_voice', () =>
      /^(let me|i'll|i've|i'm|i can|i think|here's|you can|you should)/i.test(suggestion)
    ],
  ]

  for (const [reason, check] of filters) {
    if (check()) {
      logSuggestionSuppressed(reason, suggestion, promptId, source)
      return true
    }
  }

  return false
}
```

### 42.6.2 过滤器分类详解

| 类别 | 过滤器 | 说明 |
|------|--------|------|
| **元文本** | done, meta_text, meta_wrapped | 模型输出的无意义标记 |
| **错误消息** | error_message | API 错误被当作建议 |
| **格式问题** | prefixed_label, has_formatting | 带标签或格式的文本 |
| **长度问题** | too_few_words, too_many_words, too_long | 字数超出 2-12 词范围 |
| **多句问题** | multiple_sentences | 包含多个完整句子 |
| **语气问题** | evaluative | 评估性而非行动性 |
| **视角问题** | claude_voice | AI 视角而非用户视角 |

### 42.6.3 特殊例外处理

单词建议的例外处理：

```typescript
const ALLOWED_SINGLE_WORDS = new Set([
  // 肯定词
  'yes', 'yeah', 'yep', 'yea', 'yup', 'sure', 'ok', 'okay',
  // 行动词
  'push', 'commit', 'deploy', 'stop', 'continue', 'check', 'exit', 'quit',
  // 否定词
  'no'
])
```

斜杠命令例外：

```typescript
if (suggestion.startsWith('/')) return false  // 允许所有斜杠命令
```

这些例外反映了真实用户输入习惯。

## 42.7 推测执行机制

### 42.7.1 Speculation 概述

Speculation（推测执行）是提示建议服务的高级功能。当用户接受建议时，系统可能已经提前执行了预测的操作，从而节省等待时间。

推测执行架构位于 `speculation.ts`：

| 模块 | 功能 |
|------|------|
| `startSpeculation()` | 启动推测执行流程 |
| `acceptSpeculation()` | 用户接受时合并结果 |
| `abortSpeculation()` | 用户拒绝时中止执行 |
| `handleSpeculationAccept()` | 处理接受的完整流程 |

### 42.7.2 Overlay 文件隔离

推测执行使用 Overlay（覆盖层）机制隔离文件操作，确保未接受时不会影响实际文件：

```typescript
function getOverlayPath(id: string): string {
  return join(getClaudeTempDir(), 'speculation', String(process.pid), id)
}
```

文件写入时的 Copy-on-Write 逻辑（写入重定向区域）：

```typescript
if (isWriteTool) {
  // Copy-on-write: 先复制原文件到 overlay
  if (!writtenPathsRef.current.has(rel)) {
    const overlayFile = join(overlayPath, rel)
    await mkdir(dirname(overlayFile), { recursive: true })
    try {
      await copyFile(join(cwd, rel), overlayFile)
    } catch {
      // 原文件可能不存在（新建文件）
    }
    writtenPathsRef.current.add(rel)
  }
  // 将写入重定向到 overlay
  input = { ...input, [pathKey]: join(overlayPath, rel) }
}
```

### 42.7.3 执行边界控制

推测执行有明确的边界，遇到需要用户确认的操作时停止：

```typescript
const WRITE_TOOLS = new Set(['Edit', 'Write', 'NotebookEdit'])
const SAFE_READ_ONLY_TOOLS = new Set([
  'Read', 'Glob', 'Grep', 'ToolSearch', 'LSP', 'TaskGet', 'TaskList'
])
```

边界类型定义：

```typescript
type CompletionBoundary =
  | { type: 'bash'; command: string; completedAt: number }
  | { type: 'edit'; toolName: string; filePath: string; completedAt: number }
  | { type: 'denied_tool'; toolName: string; detail: string; completedAt: number }
  | { type: 'complete'; completedAt: number; outputTokens: number }
```

边界触发逻辑（边界控制区域）：

```typescript
// 文件编辑边界：需要权限确认
if (isWriteTool) {
  const canAutoAcceptEdits =
    mode === 'acceptEdits' ||
    mode === 'bypassPermissions' ||
    (mode === 'plan' && isBypassPermissionsModeAvailable)

  if (!canAutoAcceptEdits) {
    updateActiveSpeculationState(setAppState, () => ({
      boundary: { type: 'edit', toolName: tool.name, filePath: editPath ?? '' }
    }))
    abortController.abort()
    return denySpeculation('Speculation paused: file edit requires permission')
  }
}

// Bash 命令边界：非只读命令停止
if (tool.name === 'Bash') {
  if (!command || checkReadOnlyConstraints({ command }, ...).behavior !== 'allow') {
    updateActiveSpeculationState(setAppState, () => ({
      boundary: { type: 'bash', command }
    }))
    abortController.abort()
    return denySpeculation('Speculation paused: bash boundary')
  }
}
```

### 42.7.4 Pipeline 建议

当推测执行完成时，系统可以生成下一个 Pipeline 建议，形成连续预测链：

```typescript
// Pipeline: 在等待用户接受时生成下一个建议
void generatePipelinedSuggestion(
  contextRef.current,
  suggestionText,
  messagesRef.current,
  setAppState,
  abortController,
)
```

Pipeline 建议的上下文包含已推测的消息：

```typescript
const augmentedContext: REPLHookContext = {
  ...context,
  messages: [
    ...context.messages,
    createUserMessage({ content: suggestionText }),
    ...speculatedMessages,  // 已推测执行的消息
  ],
}
```

## 42.8 UI 集成

### 42.8.1 AppState 状态结构

提示建议状态存储在 AppState 中（`AppStateStore.ts` promptSuggestion 字段区域）：

```typescript
promptSuggestion: {
  text: string | null            // 建议文本内容
  promptId: 'user_intent' | 'stated_intent' | null  // 提示变体 ID
  shownAt: number                // 显示时间戳
  acceptedAt: number             // 接受时间戳
  generationRequestId: string | null  // 生成请求 ID（用于 RL 数据关联）
}
```

### 42.8.2 usePromptSuggestion Hook

React Hook `usePromptSuggestion` 管理建议的生命周期（`hooks/usePromptSuggestion.ts`）：

```typescript
export function usePromptSuggestion({
  inputValue,
  isAssistantResponding,
}: Props): {
  suggestion: string | null
  markAccepted: () => void
  markShown: () => void
  logOutcomeAtSubmission: (finalInput: string, opts?: { skipReset: boolean }) => void
}
```

核心功能：

1. **suggestion 计算**：响应中或输入非空时隐藏建议
   ```typescript
   const suggestion = isAssistantResponding || inputValue.length > 0
     ? null
     : suggestionText
   ```

2. **markShown**：首次显示时记录时间戳
   ```typescript
   const markShown = useCallback(() => {
     setAppState(prev => {
       if (prev.promptSuggestion.shownAt !== 0 || !prev.promptSuggestion.text) {
         return prev
       }
       return {
         ...prev,
         promptSuggestion: { ...prev.promptSuggestion, shownAt: Date.now() }
       }
     })
   }, [setAppState])
   ```

3. **markAccepted**：接受时记录时间戳（用于区分 Tab vs Enter）
   ```typescript
   const markAccepted = useCallback(() => {
     if (!isValidSuggestion) return
     setAppState(prev => ({
       ...prev,
       promptSuggestion: { ...prev.promptSuggestion, acceptedAt: Date.now() }
     }))
   }, [isValidSuggestion, setAppState])
   ```

### 42.8.3 PromptInput 组件集成

PromptInput 组件负责建议的显示和交互（`PromptInput.tsx` 建议接受判断区域）：

```typescript
// 建议接受判断
const suggestionText = promptSuggestionState.text
const inputMatchesSuggestion = inputParam.trim() === '' || inputParam === suggestionText

if (inputMatchesSuggestion && suggestionText && !hasImages && !state.viewingAgentTaskId) {
  // Speculation 激活时的特殊处理
  if (speculation.status === 'active') {
    markAccepted()
    logOutcomeAtSubmission(suggestionText, { skipReset: true })  // 不中止推测
    void onSubmitProp(suggestionText, ..., {
      state: speculation,
      speculationSessionTimeSavedMs,
      setAppState
    })
    return  // Skip normal query
  }

  // 常规建议接受（需要 shownAt > 0）
  if (promptSuggestionState.shownAt > 0) {
    markAccepted()
    inputParam = suggestionText
  }
}
```

建议显示作为 placeholder（placeholder 显示区域）：

```typescript
const placeholder = showPromptSuggestion && promptSuggestion
  ? promptSuggestion
  : defaultPlaceholder
```

### 42.8.4 分析事件记录

完整的用户交互记录用于改进建议质量：

```typescript
logEvent('tengu_prompt_suggestion', {
  source: 'cli',
  outcome: wasAccepted ? 'accepted' : 'ignored',
  prompt_id: promptId,
  generationRequestId,
  ...(wasAccepted && {
    acceptMethod: tabWasPressed ? 'tab' : 'enter',
    timeToAcceptMs: timeMs - shownAt
  }),
  ...(!wasAccepted && {
    timeToIgnoreMs: timeMs - shownAt
  }),
  ...(firstKeystrokeAt.current > 0 && {
    timeToFirstKeystrokeMs: firstKeystrokeAt.current - shownAt
  }),
  wasFocusedWhenShown,
  similarity: Math.round((finalInput.length / (suggestionText?.length || 1)) * 100) / 100
})
```

记录的关键指标：

| 指标 | 用途 |
|------|------|
| outcome | accepted/ignored/suppressed |
| acceptMethod | Tab 或 Enter 接受 |
| timeToAcceptMs | 从显示到接受的时间 |
| timeToIgnoreMs | 从显示到忽略的时间 |
| timeToFirstKeystrokeMs | 用户首次输入时间 |
| wasFocusedWhenShown | 显示时终端焦点状态 |
| similarity | 用户输入与建议的相似度 |

## 42.9 执行触发流程

### 42.9.1 Stop Hooks 集成

提示建议在 Stop Hooks 中触发执行（`stopHooks.ts` 引入区域）：

```typescript
import { executePromptSuggestion } from '../services/PromptSuggestion/promptSuggestion.js'
```

在 `handleStopHooks()` 中保存上下文参数：

```typescript
const stopHookContext: REPLHookContext = {
  messages: [...messagesForQuery, ...assistantMessages],
  systemPrompt,
  userContext,
  systemContext,
  toolUseContext,
  querySource,
}

if (querySource === 'repl_main_thread' || querySource === 'sdk') {
  saveCacheSafeParams(createCacheSafeParams(stopHookContext))
}
```

### 42.9.2 执行函数实现

`executePromptSuggestion()` 是 CLI TUI 的执行入口（执行函数实现区域）：

```typescript
export async function executePromptSuggestion(
  context: REPLHookContext,
): Promise<void> {
  // 仅在主线程触发
  if (context.querySource !== 'repl_main_thread') return

  currentAbortController = new AbortController()
  const abortController = currentAbortController
  const cacheSafeParams = createCacheSafeParams(context)

  try {
    const result = await tryGenerateSuggestion(
      abortController,
      context.messages,
      context.toolUseContext.getAppState,
      cacheSafeParams,
      'cli'
    )
    if (!result) return

    // 更新 AppState 显示建议
    context.toolUseContext.setAppState(prev => ({
      ...prev,
      promptSuggestion: {
        text: result.suggestion,
        promptId: result.promptId,
        shownAt: 0,
        acceptedAt: 0,
        generationRequestId: result.generationRequestId,
      },
    }))

    // 启动推测执行（如果启用）
    if (isSpeculationEnabled() && result.suggestion) {
      void startSpeculation(
        result.suggestion,
        context,
        context.toolUseContext.setAppState,
        false,
        cacheSafeParams
      )
    }
  } catch (error) {
    if (error instanceof Error && error.name === 'AbortError') return
    logError(toError(error))
  }
}
```

## 42.10 小结

Claude Code 的提示建议服务是一个精细设计的智能预测系统：

1. **分层启用检查**：从环境变量到 Feature Flag 到用户设置，多层控制启用状态
2. **缓存复用架构**：Forked Agent 复用主线程 Prompt Cache，降低生成成本
3. **用户视角提示**：提示模板引导模型预测用户意图而非 AI 建议
4. **严格过滤规则**：12 类过滤器确保建议符合用户习惯和格式要求
5. **推测执行扩展**：Overlay 隔离和边界控制实现安全的提前执行
6. **Pipeline 连续预测**：推测完成时生成下一个建议，形成预测链
7. **详细分析记录**：完整的交互指标用于持续改进建议质量

这种设计在保证建议质量的同时，最大化利用缓存机制降低成本，并通过推测执行进一步节省用户等待时间，是 AI 辅助交互体验的重要创新。