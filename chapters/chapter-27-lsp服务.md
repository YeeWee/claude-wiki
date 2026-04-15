# 第27章：LSP服务

## 27.1 引言

Language Server Protocol (LSP) 是微软开发的开放协议，用于建立编辑器与语言服务器之间的标准化通信。Claude Code 通过 LSP 服务集成，实现了代码智能功能，包括代码诊断、定义跳转、引用查找等。

LSP 在 Claude Code 中扮演着"代码理解器"的角色。它允许 Claude Code：
- 获取实时的代码诊断信息（错误、警告）
- 理解代码结构和语义
- 提供智能的代码上下文
- 支持多种编程语言

与 MCP 不同，LSP 专注于代码智能而非通用工具扩展。本章将深入分析 Claude Code 的 LSP 服务实现，包括服务器管理、协议集成、诊断处理和多语言支持。

```mermaid
sequenceDiagram
    participant CC as Claude Code CLI
    participant SM as LSPServerManager
    participant SI as LSPServerInstance
    participant LC as LSPClient
    participant LS as Language Server进程

    CC->>SM: initializeLspServerManager()
    SM->>SM: loadAllPluginsCacheOnly()
    SM->>SM: getAllLspServers()
    SM->>SI: createLSPServerInstance(name, config)
    SM->>SM: build extensionMap
    SM-->>CC: 初始化完成

    CC->>SM: openFile(filePath, content)
    SM->>SM: getServerForFile(filePath)
    SM->>SI: ensureServerStarted(filePath)
    SI->>LC: start(command, args)
    LC->>LS: spawn 子进程
    LS-->>LC: spawn 成功
    LC->>LS: initialize(params)
    LS-->>LC: InitializeResult + capabilities
    LC->>LS: initialized notification
    SI-->>SM: 服务器运行中
    SM->>SI: sendNotification("textDocument/didOpen")
    SI->>LC: sendNotification()
    LC->>LS: textDocument/didOpen
    LS-->>LC: publishDiagnostics notification
    LC->>SI: onNotification handler
    SI->>SM: registerPendingLSPDiagnostic()
    SM-->>CC: 诊断已注册

    figure-27-1: LSP交互时序图
```

## 27.2 语言服务器管理

### 27.2.1 全局单例管理

Claude Code 使用单例模式管理 LSP 服务，通过 `manager.ts` 提供全局访问入口：

```typescript
// src/services/lsp/manager.ts:14-41
type InitializationState = 'not-started' | 'pending' | 'success' | 'failed'

let lspManagerInstance: LSPServerManager | undefined
let initializationState: InitializationState = 'not-started'
let initializationError: Error | undefined
let initializationGeneration = 0
let initializationPromise: Promise<void> | undefined
```

初始化状态机包含四种状态：
- `not-started`: 未启动，初始状态
- `pending`: 正在初始化中
- `success`: 成功初始化
- `failed`: 初始化失败

初始化函数采用异步非阻塞模式，避免阻塞 Claude Code 启动流程：

```typescript
// src/services/lsp/manager.ts:145-208
export function initializeLspServerManager(): void {
  // --bare 模式下不启动 LSP
  if (isBareMode()) {
    return
  }

  // 幂等性检查：已初始化或正在初始化时跳过
  if (lspManagerInstance !== undefined && initializationState !== 'failed') {
    return
  }

  // 创建管理器实例并标记为 pending
  lspManagerInstance = createLSPServerManager()
  initializationState = 'pending'

  // 增加代数计数器，防止过期的初始化 Promise 更新状态
  const currentGeneration = ++initializationGeneration

  // 异步初始化，不阻塞主流程
  initializationPromise = lspManagerInstance
    .initialize()
    .then(() => {
      if (currentGeneration === initializationGeneration) {
        initializationState = 'success'
        // 注册诊断通知处理器
        if (lspManagerInstance) {
          registerLSPNotificationHandlers(lspManagerInstance)
        }
      }
    })
    .catch((error: unknown) => {
      if (currentGeneration === initializationGeneration) {
        initializationState = 'failed'
        initializationError = error as Error
        lspManagerInstance = undefined
      }
    })
}
```

代数计数器 (`initializationGeneration`) 是关键设计，用于处理并发初始化请求。当快速调用多次初始化时，只有最后一次初始化的结果会被采纳。

### 27.2.2 服务器管理器接口

`LSPServerManager` 管理多个 LSP 服务器实例，根据文件扩展名路由请求：

```typescript
// src/services/lsp/LSPServerManager.ts:16-43
export type LSPServerManager = {
  initialize(): Promise<void>
  shutdown(): Promise<void>
  getServerForFile(filePath: string): LSPServerInstance | undefined
  ensureServerStarted(filePath: string): Promise<LSPServerInstance | undefined>
  sendRequest<T>(filePath: string, method: string, params: unknown): Promise<T | undefined>
  getAllServers(): Map<string, LSPServerInstance>
  openFile(filePath: string, content: string): Promise<void>
  changeFile(filePath: string, content: string): Promise<void>
  saveFile(filePath: string): Promise<void>
  closeFile(filePath: string): Promise<void>
  isFileOpen(filePath: string): boolean
}
```

管理器使用闭包封装私有状态，避免使用类：

```typescript
// src/services/lsp/LSPServerManager.ts:59-64
export function createLSPServerManager(): LSPServerManager {
  const servers: Map<string, LSPServerInstance> = new Map()
  const extensionMap: Map<string, string[]> = new Map()
  const openedFiles: Map<string, string> = new Map()
  // ...
}
```

三个核心状态：
- `servers`: 服务器名称到实例的映射
- `extensionMap`: 文件扩展名到服务器名称的映射
- `openedFiles`: 已打开文件的 URI 到服务器名称的映射

### 27.2.3 配置加载

LSP 服务器配置仅通过插件提供，不支持用户/项目直接配置：

```typescript
// src/services/lsp/config.ts:14-79
export async function getAllLspServers(): Promise<{
  servers: Record<string, ScopedLspServerConfig>
}> {
  const allServers: Record<string, ScopedLspServerConfig> = {}

  // 获取所有启用的插件
  const { enabled: plugins } = await loadAllPluginsCacheOnly()

  // 并行加载各插件的 LSP 配置
  const results = await Promise.all(
    plugins.map(async plugin => {
      const errors: PluginError[] = []
      try {
        const scopedServers = await getPluginLspServers(plugin, errors)
        return { plugin, scopedServers, errors }
      } catch (e) {
        // 单个插件失败不影响其他插件
        return { plugin, scopedServers: undefined, errors }
      }
    }),
  )

  // 合并所有服务器配置
  for (const { plugin, scopedServers, errors } of results) {
    if (scopedServers) {
      Object.assign(allServers, scopedServers)
    }
  }

  return { servers: allServers }
}
```

插件配置 Schema 定义了完整的 LSP 服务器配置结构：

```typescript
// src/utils/plugins/schemas.ts:708-788
export const LspServerConfigSchema = lazySchema(() =>
  z.strictObject({
    command: z.string().min(1)  // LSP 服务器命令
      .describe('Command to execute the LSP server'),
    args: z.array(nonEmptyString()).optional()  // 命令行参数
      .describe('Command-line arguments to pass to the server'),
    extensionToLanguage: z.record(fileExtension(), nonEmptyString())  // 文件扩展名映射
      .describe('Mapping from file extension to LSP language ID'),
    transport: z.enum(['stdio', 'socket']).default('stdio')  // 传输方式
      .describe('Communication transport mechanism'),
    env: z.record(z.string(), z.string()).optional()  // 环境变量
      .describe('Environment variables to set when starting the server'),
    initializationOptions: z.unknown().optional()  // 初始化选项
      .describe('Initialization options passed to the server'),
    workspaceFolder: z.string().optional()  // 工作区路径
      .describe('Workspace folder path to use for the server'),
    startupTimeout: z.number().int().positive().optional()  // 启动超时
      .describe('Maximum time to wait for server startup (milliseconds)'),
    maxRestarts: z.number().int().nonnegative().optional()  // 最大重启次数
      .describe('Maximum number of restart attempts before giving up'),
  }),
)
```

## 27.3 LSP 协议集成

### 27.3.1 客户端实现

`LSPClient` 使用 `vscode-jsonrpc` 库实现 JSON-RPC 通信：

```typescript
// src/services/lsp/LSPClient.ts:21-41
export type LSPClient = {
  readonly capabilities: ServerCapabilities | undefined
  readonly isInitialized: boolean
  start: (command: string, args: string[], options?: { env?: Record<string, string>; cwd?: string }) => Promise<void>
  initialize: (params: InitializeParams) => Promise<InitializeResult>
  sendRequest: <TResult>(method: string, params: unknown) => Promise<TResult>
  sendNotification: (method: string, params: unknown) => Promise<void>
  onNotification: (method: string, handler: (params: unknown) => void) => void
  onRequest: <TParams, TResult>(method: string, handler: (params: TParams) => TResult | Promise<TResult>) => void
  stop: () => Promise<void>
}
```

客户端启动流程包含关键等待机制，确保进程成功启动：

```typescript
// src/services/lsp/LSPClient.ts:96-131
async start(command: string, args: string[], options?: { env?: Record<string, string>; cwd?: string }): Promise<void> {
  // 1. 启动 LSP 服务器进程
  process = spawn(command, args, {
    stdio: ['pipe', 'pipe', 'pipe'],
    env: { ...subprocessEnv(), ...options?.env },
    cwd: options?.cwd,
    windowsHide: true,  // Windows 上隐藏控制台窗口
  })

  // 1.5. 等待进程成功启动 - 关键步骤
  // spawn() 立即返回，但 error 事件异步触发
  // 如果在确认启动前使用流，会导致未处理的 Promise 拒绝
  const spawnedProcess = process
  await new Promise<void>((resolve, reject) => {
    spawnedProcess.once('spawn', resolve)
    spawnedProcess.once('error', reject)
  })

  // 捕获 stderr 用于诊断
  if (process.stderr) {
    process.stderr.on('data', (data: Buffer) => {
      logForDebugging(`[LSP SERVER ${serverName}] ${data.toString().trim()}`)
    })
  }

  // 2. 创建 JSON-RPC 连接
  const reader = new StreamMessageReader(process.stdout)
  const writer = new StreamMessageWriter(process.stdin)
  connection = createMessageConnection(reader, writer)

  // 2.5. 注册错误/关闭处理器 BEFORE listen()
  connection.onError(([error]) => {
    if (!isStopping) {
      startFailed = true
      startError = error
    }
  })

  connection.onClose(() => {
    if (!isStopping) {
      isInitialized = false
    }
  })

  // 3. 开始监听消息
  connection.listen()
}
```

### 27.3.2 初始化握手

LSP 协议要求客户端与服务器进行初始化握手：

```typescript
// src/services/lsp/LSPClient.ts:256-287
async initialize(params: InitializeParams): Promise<InitializeResult> {
  if (!connection) {
    throw new Error('LSP client not started')
  }

  // 发送 initialize 请求
  const result: InitializeResult = await connection.sendRequest('initialize', params)

  capabilities = result.capabilities

  // 发送 initialized 通知
  await connection.sendNotification('initialized', {})

  isInitialized = true
  return result
}
```

初始化参数包含客户端能力和工作区信息：

```typescript
// src/services/lsp/LSPServerInstance.ts:167-237
const initParams: InitializeParams = {
  processId: process.pid,

  // 初始化选项（某些服务器必需）
  initializationOptions: config.initializationOptions ?? {},


  // LSP 3.16+ 现代方式
  workspaceFolders: [
    {
      uri: workspaceUri,
      name: path.basename(workspaceFolder),
    },
  ],

  // 已弃用字段（某些服务器仍需要）
  rootPath: workspaceFolder,
  rootUri: workspaceUri,

  // 客户端能力声明
  capabilities: {
    workspace: {
      configuration: false,       // 不支持 workspace/configuration
      workspaceFolders: false,    // 不支持工作区文件夹变更
    },
    textDocument: {
      synchronization: {
        dynamicRegistration: false,
        willSave: false,
        willSaveWaitUntil: false,
        didSave: true,
      },
      publishDiagnostics: {
        relatedInformation: true,
        tagSupport: { valueSet: [1, 2] },  // Unnecessary, Deprecated
        versionSupport: false,
        codeDescriptionSupport: true,
        dataSupport: false,
      },
      hover: { contentFormat: ['markdown', 'plaintext'] },
      definition: { linkSupport: true },
      references: { dynamicRegistration: false },
      documentSymbol: { hierarchicalDocumentSymbolSupport: true },
      callHierarchy: { dynamicRegistration: false },
    },
    general: { positionEncodings: ['utf-16'] },
  },
}
```

### 27.3.3 通知处理机制

LSP 客户端支持注册通知处理器，用于接收服务器推送的消息：

```typescript
// src/services/lsp/LSPClient.ts:337-350
onNotification(method: string, handler: (params: unknown) => void): void {
  if (!connection) {
    // 惰性初始化支持：连接未就绪时排队处理器
    pendingHandlers.push({ method, handler })
    return
  }

  checkStartFailed()
  connection.onNotification(method, handler)
}
```

惰性初始化机制允许在服务器启动前注册处理器，启动后自动应用：

```typescript
// src/services/lsp/LSPClient.ts:229-244
// 启动成功后应用排队的处理器
for (const { method, handler } of pendingHandlers) {
  connection.onNotification(method, handler)
}
pendingHandlers.length = 0
```

## 27.4 服务器实例生命周期

### 27.4.1 状态机设计

`LSPServerInstance` 实现完整的状态机管理：

```typescript
// src/services/lsp/LSPServerInstance.ts:33-65
export type LSPServerInstance = {
  readonly name: string
  readonly config: ScopedLspServerConfig
  readonly state: LspServerState
  readonly startTime: Date | undefined
  readonly lastError: Error | undefined
  readonly restartCount: number
  start(): Promise<void>
  stop(): Promise<void>
  restart(): Promise<void>
  isHealthy(): boolean
  sendRequest<T>(method: string, params: unknown): Promise<T>
  sendNotification(method: string, params: unknown): Promise<void>
  onNotification(method: string, handler: (params: unknown) => void): void
  onRequest<TParams, TResult>(method: string, handler: (params: TParams) => TResult | Promise<TResult>) => void
}
```

状态转换规则：
- `stopped` → `starting` → `running`
- `running` → `stopping` → `stopped`
- 任意状态 → `error` (失败时)
- `error` → `starting` (重试时)

### 27.4.2 启动流程

启动函数包含崩溃恢复限制机制：

```typescript
// src/services/lsp/LSPServerInstance.ts:135-264
async function start(): Promise<void> {
  if (state === 'running' || state === 'starting') {
    return
  }

  // 崩溃恢复限制：防止持久崩溃的服务器无限重启
  const maxRestarts = config.maxRestarts ?? 3
  if (state === 'error' && crashRecoveryCount > maxRestarts) {
    const error = new Error(
      `LSP server '${name}' exceeded max crash recovery attempts (${maxRestarts})`
    )
    lastError = error
    throw error
  }

  try {
    state = 'starting'

    // 启动客户端
    await client.start(config.command, config.args || [], {
      env: config.env,
      cwd: config.workspaceFolder,
    })

    // 发送初始化请求
    const initPromise = client.initialize(initParams)
    if (config.startupTimeout !== undefined) {
      await withTimeout(initPromise, config.startupTimeout, `LSP server '${name}' timed out`)
    } else {
      await initPromise
    }

    state = 'running'
    startTime = new Date()
    crashRecoveryCount = 0
  } catch (error) {
    // 清理启动失败的进程
    client.stop().catch(() => {})
    initPromise?.catch(() => {})
    state = 'error'
    lastError = error as Error
    throw error
  }
}
```

超时包装器使用 Promise.race 实现：

```typescript
// src/services/lsp/LSPServerInstance.ts:499-511
function withTimeout<T>(promise: Promise<T>, ms: number, message: string): Promise<T> {
  let timer: ReturnType<typeof setTimeout>
  const timeoutPromise = new Promise<never>((_, reject) => {
    timer = setTimeout((rej, msg) => rej(new Error(msg)), ms, reject, message)
  })
  return Promise.race([promise, timeoutPromise]).finally(() => clearTimeout(timer!))
}
```

### 27.4.3 崩溃恢复

崩溃回调通过 `onCrash` 参数传递，更新实例状态：

```typescript
// src/services/lsp/LSPServerInstance.ts:121-125
const client = createLSPClient(name, error => {
  state = 'error'
  lastError = error
  crashRecoveryCount++
})
```

这确保崩溃的服务器在下次请求时能够被重启：

```typescript
// src/services/lsp/LSPServerManager.ts:215-236
async function ensureServerStarted(filePath: string): Promise<LSPServerInstance | undefined> {
  const server = getServerForFile(filePath)
  if (!server) return undefined

  if (server.state === 'stopped' || server.state === 'error') {
    try {
      await server.start()
    } catch (error) {
      throw error
    }
  }

  return server
}
```

### 27.4.4 请求重试机制

对于瞬态错误（如 rust-analyzer 累引期间的 "content modified"），客户端自动重试：

```typescript
// src/services/lsp/LSPServerInstance.ts:17-28
const LSP_ERROR_CONTENT_MODIFIED = -32801
const MAX_RETRIES_FOR_TRANSIENT_ERRORS = 3
const RETRY_BASE_DELAY_MS = 500

async function sendRequest<T>(method: string, params: unknown): Promise<T> {
  if (!isHealthy()) {
    throw new Error(`Cannot send request to LSP server '${name}': server is ${state}`)
  }

  let lastAttemptError: Error | undefined

  for (let attempt = 0; attempt <= MAX_RETRIES_FOR_TRANSIENT_ERRORS; attempt++) {
    try {
      return await client.sendRequest(method, params)
    } catch (error) {
      lastAttemptError = error as Error

      const errorCode = (error as { code?: number }).code
      const isContentModifiedError = errorCode === LSP_ERROR_CONTENT_MODIFIED

      if (isContentModifiedError && attempt < MAX_RETRIES_FOR_TRANSIENT_ERRORS) {
        const delay = RETRY_BASE_DELAY_MS * Math.pow(2, attempt)  // 指数退避
        await sleep(delay)
        continue
      }
      break
    }
  }

  throw new Error(`LSP request '${method}' failed: ${lastAttemptError?.message}`)
}
```

指数退避延迟：500ms → 1000ms → 2000ms

## 27.5 代码诊断系统

### 27.5.1 诊断注册机制

`LSPDiagnosticRegistry` 存储异步接收的诊断，遵循与 `AsyncHookRegistry` 相同的模式：

```typescript
// src/services/lsp/LSPDiagnosticRegistry.ts:21-56
export type PendingLSPDiagnostic = {
  serverName: string
  files: DiagnosticFile[]
  timestamp: number
  attachmentSent: boolean
}

const MAX_DIAGNOSTICS_PER_FILE = 10
const MAX_TOTAL_DIAGNOSTICS = 30
const MAX_DELIVERED_FILES = 500

const pendingDiagnostics = new Map<string, PendingLSPDiagnostic>()
const deliveredDiagnostics = new LRUCache<string, Set<string>>({ max: MAX_DELIVERED_FILES })
```

诊断流程：
1. LSP 服务器发送 `textDocument/publishDiagnostics` 通知
2. `registerPendingLSPDiagnostic()` 存储诊断
3. `checkForLSPDiagnostics()` 检索待处理诊断
4. 诊断作为 Attachment 交付到对话

### 27.5.2 诊断去重

跨轮次去重防止重复交付相同诊断：

```typescript
// src/services/lsp/LSPDiagnosticRegistry.ts:109-124
function createDiagnosticKey(diag: {
  message: string
  severity?: string
  range?: unknown
  source?: string
  code?: unknown
}): string {
  return jsonStringify({
    message: diag.message,
    severity: diag.severity,
    range: diag.range,
    source: diag.source || null,
    code: diag.code || null,
  })
}

function deduplicateDiagnosticFiles(allFiles: DiagnosticFile[]): DiagnosticFile[] {
  // ... 去重逻辑
  const previouslyDelivered = deliveredDiagnostics.get(file.uri) || new Set()

  for (const diag of file.diagnostics) {
    const key = createDiagnosticKey(diag)
    if (seenDiagnostics.has(key) || previouslyDelivered.has(key)) {
      continue  // 跳过重复诊断
    }
    seenDiagnostics.add(key)
    dedupedFile.diagnostics.push(diag)
  }
}
```

### 27.5.3 通知处理器注册

`passiveFeedback.ts` 注册诊断通知处理器：

```typescript
// src/services/lsp/passiveFeedback.ts:125-279
export function registerLSPNotificationHandlers(
  manager: LSPServerManager,
): HandlerRegistrationResult {
  const servers = manager.getAllServers()

  const registrationErrors: Array<{ serverName: string; error: string }> = []
  let successCount = 0
  const diagnosticFailures: Map<string, { count: number; lastError: string }> = new Map()

  for (const [serverName, serverInstance] of servers.entries()) {
    try {
      serverInstance.onNotification(
        'textDocument/publishDiagnostics',
        (params: unknown) => {
          try {
            // 验证参数结构
            if (!params || typeof params !== 'object' || !('uri' in params) || !('diagnostics' in params)) {
              return
            }

            const diagnosticParams = params as PublishDiagnosticsParams

            // 转换 LSP 诊断格式
            const diagnosticFiles = formatDiagnosticsForAttachment(diagnosticParams)

            if (diagnosticFiles[0]?.diagnostics.length === 0) {
              return
            }

            // 注册诊断以异步交付
            registerPendingLSPDiagnostic({
              serverName,
              files: diagnosticFiles,
            })

            diagnosticFailures.delete(serverName)  // 成功时重置计数器
          } catch (error) {
            // 跟踪连续失败
            const failures = diagnosticFailures.get(serverName) || { count: 0, lastError: '' }
            failures.count++
            failures.lastError = err.message
            diagnosticFailures.set(serverName, failures)

            if (failures.count >= 3) {
              logForDebugging(`WARNING: LSP diagnostic handler for ${serverName} has failed ${failures.count} times`)
            }
          }
        },
      )
      successCount++
    } catch (error) {
      registrationErrors.push({ serverName, error: err.message })
    }
  }

  return { totalServers: servers.size, successCount, registrationErrors, diagnosticFailures }
}
```

### 27.5.4 严重程度映射

LSP 严重程度数字映射为 Claude 诊断格式：

```typescript
// src/services/lsp/passiveFeedback.ts:18-35
function mapLSPSeverity(lspSeverity: number | undefined): 'Error' | 'Warning' | 'Info' | 'Hint' {
  // LSP DiagnosticSeverity enum:
  // 1 = Error, 2 = Warning, 3 = Information, 4 = Hint
  switch (lspSeverity) {
    case 1: return 'Error'
    case 2: return 'Warning'
    case 3: return 'Info'
    case 4: return 'Hint'
    default: return 'Error'
  }
}
```

## 27.6 文件同步

### 27.6.1 文件打开

打开文件时发送 `textDocument/didOpen` 通知：

```typescript
// src/services/lsp/LSPServerManager.ts:270-310
async function openFile(filePath: string, content: string): Promise<void> {
  const server = await ensureServerStarted(filePath)
  if (!server) return

  const fileUri = pathToFileURL(path.resolve(filePath)).href

  // 跳过已打开的文件
  if (openedFiles.get(fileUri) === server.name) {
    return
  }

  // 获取语言 ID
  const ext = path.extname(filePath).toLowerCase()
  const languageId = server.config.extensionToLanguage[ext] || 'plaintext'

  try {
    await server.sendNotification('textDocument/didOpen', {
      textDocument: {
        uri: fileUri,
        languageId,
        version: 1,
        text: content,
      },
    })
    openedFiles.set(fileUri, server.name)
  } catch (error) {
    throw err
  }
}
```

### 27.6.2 文件变更

变更文件时发送 `textDocument/didChange` 通知：

```typescript
// src/services/lsp/LSPServerManager.ts:312-343
async function changeFile(filePath: string, content: string): Promise<void> {
  const server = getServerForFile(filePath)
  if (!server || server.state !== 'running') {
    return openFile(filePath, content)
  }

  const fileUri = pathToFileURL(path.resolve(filePath)).href

  // 文件未打开时先打开
  if (openedFiles.get(fileUri) !== server.name) {
    return openFile(filePath, content)
  }

  try {
    await server.sendNotification('textDocument/didChange', {
      textDocument: { uri: fileUri, version: 1 },
      contentChanges: [{ text: content }],
    })
  } catch (error) {
    throw err
  }
}
```

### 27.6.3 文件关闭

关闭文件时发送 `textDocument/didClose` 通知：

```typescript
// src/services/lsp/LSPServerManager.ts:377-400
async function closeFile(filePath: string): Promise<void> {
  const server = getServerForFile(filePath)
  if (!server || server.state !== 'running') return

  const fileUri = pathToFileURL(path.resolve(filePath)).href

  try {
    await server.sendNotification('textDocument/didClose', {
      textDocument: { uri: fileUri },
    })
    openedFiles.delete(fileUri)
  } catch (error) {
    throw err
  }
}
```

## 27.7 多语言支持

### 27.7.1 扩展名映射

服务器配置包含文件扩展名到语言 ID 的映射：

```typescript
// src/services/lsp/LSPServerManager.ts:89-117
async function initialize(): Promise<void> {
  // 构建扩展名 → 服务器映射
  for (const [serverName, config] of Object.entries(serverConfigs)) {
    // 从 extensionToLanguage 派生文件扩展名列表
    const fileExtensions = Object.keys(config.extensionToLanguage)
    for (const ext of fileExtensions) {
      const normalized = ext.toLowerCase()
      if (!extensionMap.has(normalized)) {
        extensionMap.set(normalized, [])
      }
      extensionMap.get(normalized)?.push(serverName)
    }

    // 创建服务器实例
    const instance = createLSPServerInstance(serverName, config)
    servers.set(serverName, instance)
  }
}
```

### 27.7.2 服务器路由

根据文件扩展名路由到对应服务器：

```typescript
// src/services/lsp/LSPServerManager.ts:192-207
function getServerForFile(filePath: string): LSPServerInstance | undefined {
  const ext = path.extname(filePath).toLowerCase()
  const serverNames = extensionMap.get(ext)

  if (!serverNames || serverNames.length === 0) {
    return undefined
  }

  // 使用第一个注册的服务器
  const serverName = serverNames[0]
  return servers.get(serverName)
}
```

### 27.7.3 插件 LSP 配置示例

插件可以通过 `.lsp.json` 或 manifest 提供 LSP 配置：

```json
{
  "typescript": {
    "command": "typescript-language-server",
    "args": ["--stdio"],
    "extensionToLanguage": {
      ".ts": "typescript",
      ".tsx": "typescriptreact",
      ".js": "javascript",
      ".jsx": "javascriptreact"
    }
  },
  "python": {
    "command": "pyright-langserver",
    "args": ["--stdio"],
    "extensionToLanguage": {
      ".py": "python"
    },
    "initializationOptions": {
      "analysis": {
        "autoSearchPaths": true
      }
    }
  }
}
```

插件配置会被添加作用域前缀：

```typescript
// src/utils/plugins/lspPluginIntegration.ts:298-315
export function addPluginScopeToLspServers(
  servers: Record<string, LspServerConfig>,
  pluginName: string,
): Record<string, ScopedLspServerConfig> {
  const scopedServers: Record<string, ScopedLspServerConfig> = {}

  for (const [name, config] of Object.entries(servers)) {
    // 添加插件前缀避免冲突
    const scopedName = `plugin:${pluginName}:${name}`
    scopedServers[scopedName] = {
      ...config,
      scope: 'dynamic',
      source: pluginName,
    }
  }

  return scopedServers
}
```

## 27.8 小结

本章深入分析了 Claude Code 的 LSP 服务架构：

1. **单例管理**: 使用状态机和代数计数器实现幂等初始化，异步非阻塞模式避免阻塞启动流程

2. **服务器管理**: 通过闭包封装状态，文件扩展名映射实现多服务器路由，惰性启动优化资源使用

3. **协议集成**: 使用 vscode-jsonrpc 实现 JSON-RPC 通信，完整的初始化握手和错误处理机制

4. **生命周期**: 状态机设计管理服务器状态，崩溃恢复限制防止无限重启，瞬态错误自动重试

5. **诊断系统**: 异步接收诊断通知，LRU 缓存实现跨轮次去重，容量限制防止诊断洪泛

6. **文件同步**: didOpen/didChange/didClose 通知同步文件状态，跟踪已打开文件避免重复通知

7. **多语言支持**: 扩展名映射路由请求，插件作用域避免冲突，灵活的配置 Schema 支持各种语言服务器

LSP 服务为 Claude Code 提供了实时的代码智能，使 Claude 能够理解代码结构、获取诊断信息，从而提供更精准的代码分析和修改建议。