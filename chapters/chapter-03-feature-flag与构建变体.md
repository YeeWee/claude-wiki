# 第三章：Feature Flag 与构建变体

## 3.1 引言：为什么需要 Feature Flag

在现代软件开发中，Feature Flag（特性开关）是一种强大的技术，允许团队在不修改代码的情况下控制功能的启用或禁用。Claude Code 作为 Anthropic 的官方 CLI 工具，同时服务于内部用户（称为 "ant"）和外部用户（称为 "external"），因此需要一套灵活的构建变体系统来：

1. **隔离内部功能**：某些功能仅对 Anthropic 内部员工可用，不应出现在公开发布版本中
2. **渐进式发布**：通过 GrowthBook 等平台控制功能的灰度发布
3. **减小构建体积**：通过 Dead Code Elimination（DCE）移除不需要的代码分支
4. **环境差异化**：根据运行环境动态加载不同的工具集

```mermaid
flowchart TB
    subgraph BuildTime["构建时"]
        A[源代码] --> B[Bun Bundler]
        B --> C{构建变体}
        C -->|ant| D[内部构建<br/>完整功能集]
        C -->|external| E[外部构建<br/>精简功能集]
    end

    subgraph CompileTime["编译时处理"]
        F["feature('FLAG')"] --> G{静态评估}
        G -->|true| H[保留代码分支]
        G -->|false| I[DCE 移除分支]
    end

    subgraph Runtime["运行时"]
        J[GrowthBook API] --> K[动态配置]
        K --> L[getFeatureValue_CACHED_MAY_BE_STALE]
        L --> M[功能开关决策]
    end

    D --> F
    E --> F
    H --> M
    I --> M

    figure-03-1: Feature Flag 流程图
```

## 3.2 `bun:bundle` feature 函数机制

### 3.2.1 基本原理

Claude Code 使用 Bun 作为构建工具，`bun:bundle` 是 Bun 提供的特殊模块，其中的 `feature` 函数在构建时被静态评估。

```typescript
// src/constants/system.ts
import { feature } from 'bun:bundle'

// 构建时评估：如果 NATIVE_CLIENT_ATTESTATION 未启用，整个分支被移除
const cch = feature('NATIVE_CLIENT_ATTESTATION') ? ' cch=00000;' : ''
```

`feature()` 函数的工作原理：

1. **构建时注入**：Bun bundler 在编译阶段读取构建配置
2. **静态替换**：将 `feature('FLAG_NAME')` 替换为布尔常量
3. **DCE 优化**：基于替换后的常量进行死代码消除

### 3.2.2 使用模式

**正面模式（推荐）**：
```typescript
// src/bridge/bridgeEnabled.ts
export function isBridgeEnabled(): boolean {
  // 正面模式：条件为真时执行，否则返回默认值
  return feature('BRIDGE_MODE')
    ? isClaudeAISubscriber() &&
        getFeatureValue_CACHED_MAY_BE_STALE('tengu_ccr_bridge', false)
    : false
}
```

**反面模式（不推荐）**：
```typescript
// 这种写法不会消除字符串字面量
if (!feature('BRIDGE_MODE')) {
  return  // 字符串 'BRIDGE_MODE' 仍会保留在构建产物中
}
```

正面模式的优势在于：当功能未启用时，整个三元表达式被替换为 `false`，相关代码和字符串字面量都会被消除。

## 3.3 Dead Code Elimination (DCE) 原理

### 3.3.1 构建变体字符串替换

Claude Code 使用一种巧妙的技术来区分构建变体：

```typescript
// 构建时，"external" 被替换为实际的 USER_TYPE 值
if ("external" === 'ant') {
  // ant 构建时："ant" === 'ant' → true，代码保留
  // external 构建时："external" === 'ant' → false，代码移除
}
```

这种模式在代码库中广泛使用：

```typescript
// src/tools/AgentTool/AgentTool.tsx
isolation: ("external" === 'ant'
  ? z.enum(['worktree', 'remote'])
  : z.enum(['worktree'])
).optional()

// src/commands/ultraplan.tsx
isEnabled: () => "external" === 'ant'
```

### 3.3.2 条件导入模式

对于包含敏感功能或需要从外部构建中完全移除的模块，使用条件 `require`：

```typescript
// src/tools.ts - 死代码消除的条件导入
/* eslint-disable @typescript-eslint/no-require-imports */
const REPLTool =
  process.env.USER_TYPE === 'ant'
    ? require('./tools/REPLTool/REPLTool.js').REPLTool
    : null

const SleepTool =
  feature('PROACTIVE') || feature('KAIROS')
    ? require('./tools/SleepTool/SleepTool.js').SleepTool
    : null

const cronTools = feature('AGENT_TRIGGERS')
  ? [
      require('./tools/ScheduleCronTool/CronCreateTool.js').CronCreateTool,
      require('./tools/ScheduleCronTool/CronDeleteTool.js').CronDeleteTool,
      require('./tools/ScheduleCronTool/CronListTool.js').CronListTool,
    ]
  : []
/* eslint-enable @typescript-eslint/no-require-imports */
```

### 3.3.3 DCE 的限制

Bun bundler 的 `feature()` 评估器有每个函数的复杂度预算限制：

```typescript
// src/tools/BashTool/bashPermissions.ts 注释
// DCE cliff: Bun's feature() evaluator has a per-function complexity budget.
// bashToolHasPermission is right at the limit. `import { X as Y }` aliases
// inside the import block count toward this budget; when they push it over
// the threshold Bun can no longer prove feature('BASH_CLASSIFIER') is a
// constant and silently evaluates the ternaries to `false`
```

解决方案：将导入别名移到顶层 `const` 语句：

```typescript
// 避免在导入块中使用别名，改用顶层 const
const bashCommandIsSafeAsync = bashCommandIsSafeAsync_DEPRECATED
const splitCommand = splitCommand_DEPRECATED
```

## 3.4 构建变体：ant vs external

### 3.4.1 USER_TYPE 环境变量

构建变体由 `process.env.USER_TYPE` 控制：

| 变体 | USER_TYPE | 说明 |
|------|-----------|------|
| ant | 'ant' | Anthropic 内部员工版本 |
| external | 'external' | 公开发布版本 |

```typescript
// src/cli/update.ts
const packageName =
  MACRO.PACKAGE_URL ||
  (process.env.USER_TYPE === 'ant'
    ? '@anthropic-ai/claude-cli'
    : '@anthropic-ai/claude-code')
```

### 3.4.2 功能差异

**ant-only 功能**：

```typescript
// src/tools.ts - ant 专属工具
...(process.env.USER_TYPE === 'ant' ? [ConfigTool] : []),
...(process.env.USER_TYPE === 'ant' ? [TungstenTool] : []),
...(process.env.USER_TYPE === 'ant' && REPLTool ? [REPLTool] : []),
```

**环境变量权限差异**：

```typescript
// src/tools/BashTool/bashPermissions.ts
const ANT_ONLY_SAFE_ENV_VARS = new Set([
  'ANTHROPIC_API_KEY',
  'CLAUDE_CODE_MODEL',
  'CLAUDE_CODE_MCP_SETTINGS',
  // ... 更多内部环境变量
])

// ant 用户可以访问更多安全环境变量
if (process.env.USER_TYPE === 'ant' && ANT_ONLY_SAFE_ENV_VARS.has(varName)) {
  // 允许访问
}
```

### 3.4.3 分析元数据差异

ant 用户享有额外的调试和分析功能：

```typescript
// src/services/analytics/firstPartyEventLogger.ts
if (process.env.USER_TYPE === 'ant') {
  logForDebugging(
    `[ANT-ONLY] 1P event: ${eventName} ${jsonStringify(metadata, null, 0)}`,
  )
}
```

```typescript
// src/utils/processUserInput/processSlashCommand.tsx
if ("external" === 'ant') {
  resultText = `[ANT-ONLY] API calls: ${getDisplayPath(getDumpPromptsPath(agentId))}\n${resultText}`;
}
```

## 3.5 条件加载模式详解

### 3.5.1 工具集条件加载

Claude Code 的工具加载是最典型的条件加载场景：

```typescript
// src/tools.ts - getAllBaseTools 函数
export function getAllBaseTools(): Tools {
  return [
    AgentTool,
    TaskOutputTool,
    BashTool,
    // 基础工具...

    // 条件加载的工具
    ...(hasEmbeddedSearchTools() ? [] : [GlobTool, GrepTool]),
    ...(process.env.USER_TYPE === 'ant' ? [ConfigTool] : []),
    ...(SuggestBackgroundPRTool ? [SuggestBackgroundPRTool] : []),
    ...(WebBrowserTool ? [WebBrowserTool] : []),
    ...(isTodoV2Enabled()
      ? [TaskCreateTool, TaskGetTool, TaskUpdateTool, TaskListTool]
      : []),
    ...(OverflowTestTool ? [OverflowTestTool] : []),
    ...(isWorktreeModeEnabled() ? [EnterWorktreeTool, ExitWorktreeTool] : []),
    // ... 更多条件加载
  ]
}
```

### 3.5.2 CLI 快速路径

CLI 入口点使用 `feature()` 进行快速路径判断：

```typescript
// src/entrypoints/cli.tsx
async function main(): Promise<void> {
  const args = process.argv.slice(2);

  // --version 快速路径：零模块加载
  if (args.length === 1 && args[0] === '--version') {
    console.log(`${MACRO.VERSION} (Claude Code)`);
    return;
  }

  // ant-only: --dump-system-prompt
  if (feature('DUMP_SYSTEM_PROMPT') && args[0] === '--dump-system-prompt') {
    // 整个分支在 external 构建中被移除
    const { getSystemPrompt } = await import('../constants/prompts.js');
    console.log(await getSystemPrompt([], model));
    return;
  }

  // Bridge 模式快速路径
  if (feature('BRIDGE_MODE') && args[0] === 'remote-control') {
    // feature() 必须内联以保证 DCE
    await bridgeMain();
    return;
  }
}
```

### 3.5.3 模块级条件加载

某些模块包含敏感字符串，需要完全隔离：

```typescript
// src/setup.ts
if (feature('COMMIT_ATTRIBUTION')) {
  // Dynamic import to enable dead code elimination
  // (module contains excluded strings).
  setImmediate(() => {
    void import('../utils/gitAttribution.js').then(module => {
      module.setupGitAttribution()
    })
  })
}
```

### 3.5.4 React 组件中的条件渲染

```typescript
// src/tools/AgentTool/UI.tsx
if ("external" !== 'ant') {
  // external 构建中跳过调试检查
  if (isBeingDebugged()) {
    // 调试检测逻辑
  }
}

// 条件渲染 UI 元素
{"external" === 'ant' && <MessageResponse>
  {/* ant-only UI 组件 */}
</MessageResponse>}
```

## 3.6 GrowthBook 动态配置

### 3.6.1 缓存机制

GrowthBook 提供运行时动态配置，支持缓存以避免阻塞：

```typescript
// src/services/analytics/growthbook.ts
/**
 * 非阻塞获取功能值，可能使用缓存（可能过期）
 * 适用于启动关键路径和同步上下文
 */
export function getFeatureValue_CACHED_MAY_BE_STALE<T>(
  feature: string,
  defaultValue: T,
): T {
  // 优先级：env 覆盖 > 配置覆盖 > 内存缓存 > 磁盘缓存 > 默认值
  const overrides = getEnvOverrides()
  if (overrides && feature in overrides) {
    return overrides[feature] as T
  }
  // ... 缓存检查逻辑
}
```

### 3.6.2 ant-only 配置覆盖

ant 用户可以通过 `/config` 命令或环境变量覆盖 GrowthBook 配置：

```typescript
// src/services/analytics/growthbook.ts
function getEnvOverrides(): Record<string, unknown> | null {
  if (process.env.USER_TYPE === 'ant') {
    const raw = process.env.CLAUDE_INTERNAL_FC_OVERRIDES
    if (raw) {
      return JSON.parse(raw)
    }
  }
  return null
}

function getConfigOverrides(): Record<string, unknown> | undefined {
  if (process.env.USER_TYPE !== 'ant') return undefined
  return getGlobalConfig().growthBookOverrides
}
```

## 3.7 最佳实践总结

### 3.7.1 Feature Flag 使用规范

1. **使用正面模式**：`feature('FLAG') ? enabledCode : defaultValue`
2. **保持 feature() 内联**：不要将结果存储到变量中再判断
3. **动态导入敏感模块**：使用 `await import()` 或 `require()` 条件加载
4. **避免复杂导入别名**：将别名移到顶层 `const`

### 3.7.2 构建变体区分规范

1. **使用 `"external" === 'ant'`**：而非 `process.env.USER_TYPE === 'ant'`
2. **ANT-ONLY 标记**：在注释中明确标注 `[ANT-ONLY]`
3. **环境变量检查**：`process.env.USER_TYPE === 'ant'` 用于运行时判断

### 3.7.3 代码组织建议

```typescript
// 推荐的组织方式
/* eslint-disable @typescript-eslint/no-require-imports */
const OptionalTool = feature('FLAG')
  ? require('./OptionalTool.js').OptionalTool
  : null
/* eslint-enable @typescript-eslint/no-require-imports */

// 工具集组装
const tools = [
  BaseTool1,
  BaseTool2,
  ...(OptionalTool ? [OptionalTool] : []),
  ...(feature('OTHER_FLAG') ? [OtherTool] : []),
]
```

## 小结

本章介绍了 Claude Code 的 Feature Flag 和构建变体系统，包括：

- `bun:bundle` feature 函数的构建时静态评估机制
- Dead Code Elimination 原理和实践模式
- ant 与 external 两种构建变体的差异
- 条件加载模式在工具集、CLI 入口和 React 组件中的应用
- GrowthBook 动态配置与缓存机制

这套系统使得 Claude Code 能够灵活地管理内部和外部版本的功能差异，同时保持构建产物的精简和高效。