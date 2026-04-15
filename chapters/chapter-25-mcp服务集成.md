# 第25章：MCP服务集成

> **版本说明**：本文基于 Claude Code 源代码分析，请以最新版本为准。

## 25.1 引言

Model Context Protocol (MCP) 是 Anthropic 开发的开放协议，用于建立 AI 模型与外部工具、资源之间的标准化连接。Claude Code 通过 MCP 服务集成，实现了与各种外部系统（如文件系统、数据库、API服务、IDE等）的无缝交互。

MCP 在 Claude Code 中扮演着"能力扩展器"的角色。它允许 Claude Code：
- 调用外部工具执行特定任务
- 访问远程资源和数据源
- 通过 Elicitation 机制与用户进行交互式确认
- 动态加载插件提供的服务

本章将深入分析 Claude Code 的 MCP 服务实现，包括协议基础、传输机制、连接生命周期和能力暴露。

```mermaid
flowchart TB
    subgraph CC["Claude Code CLI"]
        MC["MCP Client<br/>client.ts"]
        CM["Connection Manager<br/>useManageMCPConnections.ts"]
        EH["Elicitation Handler<br/>elicitationHandler.ts"]
        CFG["Config Manager<br/>config.ts"]
    end

    subgraph SDK["@modelcontextprotocol/sdk"]
        CL["Client Class"]
        ST["StdioTransport"]
        SST["SSETransport"]
        HTT["HTTPTransport"]
        WST["WebSocketTransport"]
    end

    subgraph Servers["MCP Servers"]
        STD["stdio Server<br/>本地进程"]
        SSE["SSE Server<br/>Server-Sent Events"]
        HTTP["HTTP Server<br/>Streamable HTTP"]
        WS["WebSocket Server"]
        INP["In-Process Server<br/>Chrome/ComputerUse"]
    end

    MC --> CL
    CL --> ST --> STD
    CL --> SST --> SSE
    CL --> HTT --> HTTP
    CL --> WST --> WS

    MC --> EH
    CM --> MC
    CFG --> CM

    MC -->|"createLinkedTransportPair"| INP

    figure-25-1: MCP集成架构图
```

## 25.2 MCP协议概述

### 25.2.1 协议基础

MCP 基于 JSON-RPC 2.0 协议，定义了客户端与服务器之间的通信规范。核心概念包括：

1. **工具 (Tools)**: 可被调用的函数，执行特定操作
2. **资源 (Resources)**: 可被读取的数据源
3. **提示 (Prompts)**: 预定义的提示模板
4. **能力 (Capabilities)**: 服务器声明支持的功能

Claude Code 使用 `@modelcontextprotocol/sdk` 作为 MCP 客户端实现，核心导入来自 `client.ts`:

```typescript
// src/services/mcp/client.ts:1-38
import { Client } from '@modelcontextprotocol/sdk/client/index.js'
import {
  SSEClientTransport,
  type SSEClientTransportOptions,
} from '@modelcontextprotocol/sdk/client/sse.js'
import { StdioClientTransport } from '@modelcontextprotocol/sdk/client/stdio.js'
import {
  StreamableHTTPClientTransport,
  type StreamableHTTPClientTransportOptions,
} from '@modelcontextprotocol/sdk/client/streamableHttp.js'
import {
  CallToolResultSchema,
  ElicitRequestSchema,
  type ElicitRequestURLParams,
  type ElicitResult,
  ErrorCode,
  type JSONRPCMessage,
  type ListPromptsResult,
  ListPromptsResultSchema,
  ListResourcesResultSchema,
  ListToolsResultSchema,
  McpError,
} from '@modelcontextprotocol/sdk/types.js'
```

### 25.2.2 配置类型系统

MCP 服务器的配置通过 Zod Schema 进行严格类型定义，确保配置数据的合法性。`types.ts` 文件定义了完整的配置类型体系：

```typescript
// src/services/mcp/types.ts:23-26
export const TransportSchema = lazySchema(() =>
  z.enum(['stdio', 'sse', 'sse-ide', 'http', 'ws', 'sdk']),
)
export type Transport = z.infer<ReturnType<typeof TransportSchema>>
```

配置范围 (ConfigScope) 定义了 MCP 配置的来源层级：

```typescript
// src/services/mcp/types.ts:10-21
export const ConfigScopeSchema = lazySchema(() =>
  z.enum([
    'local',      // 本地私有配置
    'user',       // 用户全局配置
    'project',    // 项目共享配置 (.mcp.json)
    'dynamic',    // 动态配置 (命令行)
    'enterprise', // 企业托管配置
    'claudeai',   // claude.ai 连接器
    'managed',    // 受管配置
  ]),
)
```

服务器连接状态类型定义了连接的生命周期状态：

```typescript
// src/services/mcp/types.ts:180-227
export type ConnectedMCPServer = {
  client: Client
  name: string
  type: 'connected'
  capabilities: ServerCapabilities
  serverInfo?: { name: string; version: string }
  instructions?: string
  config: ScopedMcpServerConfig
  cleanup: () => Promise<void>
}

export type FailedMCPServer = {
  name: string
  type: 'failed'
  config: ScopedMcpServerConfig
  error?: string
}

export type NeedsAuthMCPServer = {
  name: string
  type: 'needs-auth'
  config: ScopedMcpServerConfig
}

export type PendingMCPServer = {
  name: string
  type: 'pending'
  config: ScopedMcpServerConfig
  reconnectAttempt?: number
  maxReconnectAttempts?: number
}

export type DisabledMCPServer = {
  name: string
  type: 'disabled'
  config: ScopedMcpServerConfig
}

export type MCPServerConnection =
  | ConnectedMCPServer
  | FailedMCPServer
  | NeedsAuthMCPServer
  | PendingMCPServer
  | DisabledMCPServer
```

## 25.3 传输类型

### 25.3.1 Stdio传输

Stdio 传输是最常用的 MCP 通信方式，通过标准输入/输出流与本地进程通信。适用于本地安装的 MCP 服务器。

```typescript
// src/services/mcp/client.ts:944-958
} else if (serverRef.type === 'stdio' || !serverRef.type) {
  const finalCommand =
    process.env.CLAUDE_CODE_SHELL_PREFIX || serverRef.command
  const finalArgs = process.env.CLAUDE_CODE_SHELL_PREFIX
    ? [[serverRef.command, ...serverRef.args].join(' ')]
    : serverRef.args
  transport = new StdioClientTransport({
    command: finalCommand,
    args: finalArgs,
    env: {
      ...subprocessEnv(),
      ...serverRef.env,
    } as Record<string, string>,
    stderr: 'pipe', // 防止 MCP 服务器的错误输出打印到 UI
  })
}
```

Stdio 传输的清理过程包含优雅终止逻辑，确保子进程正确关闭：

```typescript
// src/services/mcp/client.ts:1429-1562
// Stdio 传输清理 - 发送 SIGINT/SIGTERM/SIGKILL 信号序列
if (serverRef.type === 'stdio') {
  try {
    const stdioTransport = transport as StdioClientTransport
    const childPid = stdioTransport.pid

    if (childPid) {
      logMCPDebug(name, 'Sending SIGINT to MCP server process')
      try {
        process.kill(childPid, 'SIGINT')
      } catch (error) {
        logMCPDebug(name, `Error sending SIGINT: ${error}`)
        return
      }
      // 等待优雅关闭，100ms 后尝试 SIGTERM，400ms 后尝试 SIGKILL
      // 总清理时间不超过 500ms
    }
  }
}
```

### 25.3.2 SSE传输

Server-Sent Events (SSE) 传输用于与 HTTP 服务器建立持久连接，服务器通过事件流推送消息。

```typescript
// src/services/mcp/client.ts:619-677
if (serverRef.type === 'sse') {
  // 创建认证提供者
  const authProvider = new ClaudeAuthProvider(name, serverRef)

  // 获取组合头部（静态 + 动态）
  const combinedHeaders = await getMcpServerHeaders(name, serverRef)

  const transportOptions: SSEClientTransportOptions = {
    authProvider,
    // 使用每个请求的新超时包装器
    fetch: wrapFetchWithTimeout(
      wrapFetchWithStepUpDetection(createFetchWithInit(), authProvider),
    ),
    requestInit: {
      headers: {
        'User-Agent': getMCPUserAgent(),
        ...combinedHeaders,
      },
    },
  }

  // EventSource 连接是长期存活的，不应应用超时
  transportOptions.eventSourceInit = {
    fetch: async (url: string | URL, init?: RequestInit) => {
      const authHeaders: Record<string, string> = {}
      const tokens = await authProvider.tokens()
      if (tokens) {
        authHeaders.Authorization = `Bearer ${tokens.access_token}`
      }
      return fetch(url, {
        ...init,
        headers: {
          'User-Agent': getMCPUserAgent(),
          ...authHeaders,
          ...init?.headers,
          ...combinedHeaders,
          Accept: 'text/event-stream',
        },
      })
    },
  }

  transport = new SSEClientTransport(
    new URL(serverRef.url),
    transportOptions,
  )
}
```

### 25.3.3 HTTP传输

Streamable HTTP 是 MCP 规范定义的现代化传输方式，支持双向 JSON-RPC 通信。

```typescript
// src/services/mcp/client.ts:784-865
} else if (serverRef.type === 'http') {
  logMCPDebug(name, `Initializing HTTP transport to ${serverRef.url}`)

  // 创建认证提供者
  const authProvider = new ClaudeAuthProvider(name, serverRef)
  const combinedHeaders = await getMcpServerHeaders(name, serverRef)

  const hasOAuthTokens = !!(await authProvider.tokens())

  const transportOptions: StreamableHTTPClientTransportOptions = {
    authProvider,
    fetch: wrapFetchWithTimeout(
      wrapFetchWithStepUpDetection(createFetchWithInit(), authProvider),
    ),
    requestInit: {
      headers: {
        'User-Agent': getMCPUserAgent(),
        ...(sessionIngressToken &&
          !hasOAuthTokens && {
            Authorization: `Bearer ${sessionIngressToken}`,
          }),
        ...combinedHeaders,
      },
    },
  }

  transport = new StreamableHTTPClientTransport(
    new URL(serverRef.url),
    transportOptions,
  )
}
```

HTTP 传输需要设置特定的 Accept 头部，以符合 MCP Streamable HTTP 规范：

```typescript
// src/services/mcp/client.ts:466-471
/**
 * MCP Streamable HTTP 规范要求客户端在每个 POST 中声明接受 JSON 和 SSE
 */
const MCP_STREAMABLE_HTTP_ACCEPT = 'application/json, text/event-stream'
```

### 25.3.4 WebSocket传输

WebSocket 传输提供双向实时通信能力：

```typescript
// src/services/mcp/client.ts:735-783
} else if (serverRef.type === 'ws') {
  logMCPDebug(
    name,
    `Initializing WebSocket transport to ${serverRef.url}`,
  )

  const combinedHeaders = await getMcpServerHeaders(name, serverRef)
  const tlsOptions = getWebSocketTLSOptions()
  const wsHeaders = {
    'User-Agent': getMCPUserAgent(),
    ...(sessionIngressToken && {
      Authorization: `Bearer ${sessionIngressToken}`,
    }),
    ...combinedHeaders,
  }

  let wsClient: WsClientLike
  if (typeof Bun !== 'undefined') {
    // Bun 的 WebSocket 支持头部/代理/TLS 选项
    wsClient = new globalThis.WebSocket(serverRef.url, {
      protocols: ['mcp'],
      headers: wsHeaders,
      proxy: getWebSocketProxyUrl(serverRef.url),
      tls: tlsOptions || undefined,
    })
  } else {
    wsClient = await createNodeWsClient(serverRef.url, {
      headers: wsHeaders,
      agent: getWebSocketProxyAgent(serverRef.url),
      ...(tlsOptions || {}),
    })
  }
  transport = new WebSocketTransport(wsClient)
}
```

### 25.3.5 进程内传输

对于特定场景（如 Chrome MCP、Computer Use），Claude Code 使用进程内传输避免启动重量级子进程：

```typescript
// src/services/mcp/InProcessTransport.ts:11-63
/**
 * 进程内链接传输对，用于在同一进程中运行 MCP 服务器和客户端
 * `send()` 在一侧发送消息，传递到另一侧的 `onmessage`
 */
class InProcessTransport implements Transport {
  private peer: InProcessTransport | undefined
  private closed = false

  onclose?: () => void
  onerror?: (error: Error) => void
  onmessage?: (message: JSONRPCMessage) => void

  async send(message: JSONRPCMessage): Promise<void> {
    if (this.closed) {
      throw new Error('Transport is closed')
    }
    // 异步传递到另一侧，避免同步请求/响应周期的栈深度问题
    queueMicrotask(() => {
      this.peer?.onmessage?.(message)
    })
  }

  async close(): Promise<void> {
    if (this.closed) return
    this.closed = true
    this.onclose?.()
    if (this.peer && !this.peer.closed) {
      this.peer.closed = true
      this.peer.onclose?.()
    }
  }
}

export function createLinkedTransportPair(): [Transport, Transport] {
  const a = new InProcessTransport()
  const b = new InProcessTransport()
  a._setPeer(b)
  b._setPeer(a)
  return [a, b]  // [clientTransport, serverTransport]
}
```

进程内传输的连接过程：

```typescript
// src/services/mcp/client.ts:906-924
} else if (
  (serverRef.type === 'stdio' || !serverRef.type) &&
  isClaudeInChromeMCPServer(name)
) {
  // 在进程内运行 Chrome MCP 服务器，避免启动 ~325 MB 的子进程
  const { createChromeContext } = await import(
    '../../utils/claudeInChrome/mcpServer.js'
  )
  const { createClaudeForChromeMcpServer } = await import(
    '@ant/claude-for-chrome-mcp'
  )
  const { createLinkedTransportPair } = await import(
    './InProcessTransport.js'
  )
  const context = createChromeContext(serverRef.env)
  inProcessServer = createClaudeForChromeMcpServer(context)
  const [clientTransport, serverTransport] = createLinkedTransportPair()
  await inProcessServer.connect(serverTransport)
  transport = clientTransport
}
```

### 25.3.6 SDK控制传输

SDK MCP 服务器运行在 SDK 进程内，需要特殊的桥接传输：

```typescript
// src/services/mcp/SdkControlTransport.ts:1-136
/**
 * SDK MCP 传输桥接
 *
 * 允许运行在 SDK 进程中的 MCP 服务器与 CLI 进程通信
 *
 * ## 消息流
 *
 * CLI → SDK (通过 SdkControlClientTransport):
 * 1. CLI 的 MCP Client 调用工具 → 发送 JSONRPC 请求
 * 2. 传输将消息包装为控制请求，包含 server_name 和 request_id
 * 3. 控制请求通过 stdout 发送到 SDK 进程
 * 4. SDK 的 StructuredIO 接收响应并路由回传输
 *
 * SDK → CLI (通过 SdkControlServerTransport):
 * 1. Query 接收控制请求，调用 transport.onmessage
 * 2. MCP 服务器处理消息，调用 transport.send()
 * 3. 传输通过回调发送响应
 */
export class SdkControlClientTransport implements Transport {
  constructor(
    private serverName: string,
    private sendMcpMessage: SendMcpMessageCallback,
  ) {}

  async send(message: JSONRPCMessage): Promise<void> {
    if (this.isClosed) {
      throw new Error('Transport is closed')
    }
    const response = await this.sendMcpMessage(this.serverName, message)
    if (this.onmessage) {
      this.onmessage(response)
    }
  }
}
```

## 25.4 服务器连接生命周期

### 25.4.1 连接初始化

`connectToServer` 函数是 MCP 连接的核心入口，使用 memoize 缓存避免重复连接：

```typescript
// src/services/mcp/client.ts:595-607
export const connectToServer = memoize(
  async (
    name: string,
    serverRef: ScopedMcpServerConfig,
    serverStats?: {
      totalServers: number
      stdioCount: number
      sseCount: number
      httpCount: number
    },
  ): Promise<MCPServerConnection> => {
    const connectStartTime = Date.now()
    let inProcessServer:
      | { connect(t: Transport): Promise<void>; close(): Promise<void> }
      | undefined
    try {
      let transport
      // ... 根据服务器类型创建传输
```

客户端创建时声明支持的 capabilities：

```typescript
// src/services/mcp/client.ts:985-1002
const client = new Client(
  {
    name: 'claude-code',
    title: 'Claude Code',
    version: MACRO.VERSION ?? 'unknown',
    description: "Anthropic's agentic coding tool",
    websiteUrl: PRODUCT_URL,
  },
  {
    capabilities: {
      roots: {},         // 支持根目录列表
      elicitation: {},   // 支持 Elicitation 交互
    },
  },
)
```

连接完成后获取服务器信息：

```typescript
// src/services/mcp/client.ts:1157-1187
const capabilities = client.getServerCapabilities()
const serverVersion = client.getServerVersion()
const rawInstructions = client.getInstructions()

// 截断过长的服务器指令（防止 token 消耗）
let instructions = rawInstructions
if (
  rawInstructions &&
  rawInstructions.length > MAX_MCP_DESCRIPTION_LENGTH
) {
  instructions =
    rawInstructions.slice(0, MAX_MCP_DESCRIPTION_LENGTH) + '… [truncated]'
}

// 记录连接成功日志
logMCPDebug(
  name,
  `Connection established with capabilities: ${jsonStringify({
    hasTools: !!capabilities?.tools,
    hasPrompts: !!capabilities?.prompts,
    hasResources: !!capabilities?.resources,
    hasResourceSubscribe: !!capabilities?.resources?.subscribe,
    serverVersion: serverVersion || 'unknown',
  })}`,
)
```

### 25.4.2 连接超时处理

MCP 连接设置超时机制，防止无限等待：

```typescript
// src/services/mcp/client.ts:1020-1077
const connectPromise = client.connect(transport)
const timeoutPromise = new Promise<never>((_, reject) => {
  const timeoutId = setTimeout(() => {
    const elapsed = Date.now() - connectStartTime
    logMCPDebug(
      name,
      `Connection timeout triggered after ${elapsed}ms`,
    )
    if (inProcessServer) {
      inProcessServer.close().catch(() => {})
    }
    transport.close().catch(() => {})
    reject(
      new TelemetrySafeError_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS(
        `MCP server "${name}" connection timed out`,
        'MCP connection timeout',
      ),
    )
  }, getConnectionTimeoutMs())  // 默认 30000ms
})
await Promise.race([connectPromise, timeoutPromise])
```

### 25.4.3 错误与关闭处理

连接建立后，注册增强的错误和关闭处理器：

```typescript
// src/services/mcp/client.ts:1217-1402
// 存储原始处理器
const originalOnerror = client.onerror
const originalOnclose = client.onclose

// 连续错误计数，超过阈值触发重连
let consecutiveConnectionErrors = 0
const MAX_ERRORS_BEFORE_RECONNECT = 3

// 增强错误处理器
client.onerror = (error: Error) => {
  const uptime = Date.now() - connectionStartTime
  hasErrorOccurred = true
  const transportType = serverRef.type || 'stdio'

  logMCPDebug(
    name,
    `${transportType.toUpperCase()} connection dropped after ${Math.floor(uptime / 1000)}s uptime`,
  )

  // HTTP 传输的会话过期检测
  if (
    (transportType === 'http' || transportType === 'claudeai-proxy') &&
    isMcpSessionExpiredError(error)
  ) {
    logMCPDebug(
      name,
      `MCP session expired, triggering reconnection`,
    )
    closeTransportAndRejectPending('session expired')
    return
  }

  // SSE/HTTP 传输的终端错误检测
  if (isTerminalConnectionError(error.message)) {
    consecutiveConnectionErrors++
    if (consecutiveConnectionErrors >= MAX_ERRORS_BEFORE_RECONNECT) {
      consecutiveConnectionErrors = 0
      closeTransportAndRejectPending('max consecutive terminal errors')
    }
  }

  if (originalOnerror) {
    originalOnerror(error)
  }
}

// 增强关闭处理器
client.onclose = () => {
  const uptime = Date.now() - connectionStartTime

  logMCPDebug(
    name,
    `${transportType.toUpperCase()} connection closed after ${Math.floor(uptime / 1000)}s`,
  )

  // 清除 memoize 缓存以便下次操作重连
  const key = getServerCacheKey(name, serverRef)
  fetchToolsForClient.cache.delete(name)
  fetchResourcesForClient.cache.delete(name)
  fetchCommandsForClient.cache.delete(name)
  connectToServer.cache.delete(key)

  if (originalOnclose) {
    originalOnclose()
  }
}
```

### 25.4.4 连接管理 Hook

`useManageMCPConnections` Hook 管理 MCP 连接的生命周期和状态同步：

```typescript
// src/services/mcp/useManageMCPConnections.ts:143-200
export function useManageMCPConnections(
  dynamicMcpConfig: Record<string, ScopedMcpServerConfig> | undefined,
  isStrictMcpConfig = false,
) {
  const store = useAppStateStore()
  const setAppState = useSetAppState()

  // 重连计时器引用，允许取消
  const reconnectTimersRef = useRef<Map<string, NodeJS.Timeout>>(new Map())

  // 批量状态更新队列
  const pendingUpdatesRef = useRef<PendingUpdate[]>([])
  const flushTimerRef = useRef<ReturnType<typeof setTimeout> | null>(null)

  // 更新服务器状态、工具、命令和资源
  // 更新通过 setTimeout 批量处理，合并 MCP_BATCH_FLUSH_MS 内的更新
  const updateServer = useCallback((update: PendingUpdate) => {
    pendingUpdatesRef.current.push(update)
    if (flushTimerRef.current === null) {
      flushTimerRef.current = setTimeout(
        flushPendingUpdates,
        MCP_BATCH_FLUSH_MS,  // 16ms
      )
    }
  }, [setAppState])
```

重连机制使用指数退避策略：

```typescript
// src/services/mcp/useManageMCPConnections.ts:87-90
const MAX_RECONNECT_ATTEMPTS = 5
const INITIAL_BACKOFF_MS = 1000
const MAX_BACKOFF_MS = 30000
```

## 25.5 工具/资源/命令暴露

### 25.5.1 工具获取

连接成功后，通过 `fetchToolsForClient` 获取服务器暴露的工具：

```typescript
// src/services/mcp/client.ts:1743-1799
export const fetchToolsForClient = memoizeWithLRU(
  async (client: MCPServerConnection): Promise<Tool[]> => {
    if (client.type !== 'connected') return []

    try {
      if (!client.capabilities?.tools) {
        return []
      }

      const result = (await client.client.request(
        { method: 'tools/list' },
        ListToolsResultSchema,
      )) as ListToolsResult

      // 对 MCP 服务器返回的工具数据进行 Unicode 清理
      const toolsToProcess = recursivelySanitizeUnicode(result.tools)

      // 将 MCP 工具转换为 Claude Code 的 Tool 格式
      return toolsToProcess.map((tool): Tool => {
        const fullyQualifiedName = buildMcpToolName(client.name, tool.name)
        return {
          ...MCPTool,
          name: fullyQualifiedName,  // mcp__serverName__toolName
          mcpInfo: { serverName: client.name, toolName: tool.name },
          isMcp: true,
          searchHint: tool._meta?.['anthropic/searchHint'],
          alwaysLoad: tool._meta?.['anthropic/alwaysLoad'] === true,
          async description() {
            return tool.description ?? ''
          },
          isConcurrencySafe() {
            return tool.annotations?.readOnlyHint ?? false
          },
          isReadOnly() {
            return tool.annotations?.readOnlyHint ?? false
          },
        }
      })
    }
  },
  MCP_FETCH_CACHE_SIZE,  // 最大缓存 20 个服务器
)
```

工具名称遵循 `mcp__serverName__toolName` 格式：

```typescript
// src/services/mcp/mcpStringUtils.ts
export function buildMcpToolName(serverName: string, toolName: string): string {
  const normalizedServerName = normalizeNameForMCP(serverName)
  const normalizedToolName = normalizeNameForMCP(toolName)
  return `mcp__${normalizedServerName}__${normalizedToolName}`
}
```

### 25.5.2 资源获取

资源通过 `fetchResourcesForClient` 获取：

```typescript
// src/services/mcp/client.ts:2000-2032
export const fetchResourcesForClient = memoizeWithLRU(
  async (client: MCPServerConnection): Promise<ServerResource[]> => {
    if (client.type !== 'connected') return []

    try {
      if (!client.capabilities?.resources) {
        return []
      }

      const result = await client.client.request(
        { method: 'resources/list' },
        ListResourcesResultSchema,
      )

      if (!result.resources) return []

      return result.resources.map(resource => ({
        ...resource,
        server: client.name,  // 标记服务器来源
      }))
    }
  },
)
```

### 25.5.3 命令（提示）获取

MCP prompts 转换为 Claude Code 的 Command：

```typescript
// src/services/mcp/client.ts:2033-2099
export const fetchCommandsForClient = memoizeWithLRU(
  async (client: MCPServerConnection): Promise<Command[]> => {
    if (client.type !== 'connected') return []

    try {
      if (!client.capabilities?.prompts) {
        return []
      }

      const result = (await client.client.request(
        { method: 'prompts/list' },
        ListPromptsResultSchema,
      )) as ListPromptsResult

      if (!result.prompts) return []

      return result.prompts.map(prompt => {
        const promptName = buildMcpToolName(client.name, prompt.name)
        return {
          type: 'prompt',
          name: promptName,
          description: prompt.description ?? '',
          isMcp: true,
          arguments: prompt.arguments,
          serverName: client.name,
          // ... 其他属性
        }
      })
    }
  },
)
```

### 25.5.4 批量获取

`getMcpToolsCommandsAndResources` 函数批量获取所有能力：

```typescript
// src/services/mcp/client.ts:2172-2192
const [tools, mcpCommands, mcpSkills, resources] = await Promise.all([
  fetchToolsForClient(client),
  fetchCommandsForClient(client),
  feature('MCP_SKILLS') && supportsResources
    ? fetchMcpSkillsForClient!(client)
    : Promise.resolve([]),
  supportsResources ? fetchResourcesForClient(client) : Promise.resolve([]),
])
const commands = [...mcpCommands, ...mcpSkills]

// 检查是否需要添加资源工具
const resourceTools: Tool[] = []
if (supportsResources) {
  const hasResourceTools = [ListMcpResourcesTool, ReadMcpResourceTool].some(
    tool => tools.some(t => toolMatchesName(t, tool.name)),
  )
  if (!hasResourceTools) {
    resourceTools.push(ListMcpResourcesTool, ReadMcpResourceTool)
  }
}
```

## 25.6 Elicitation 处理

### 25.6.1 Elicitation 协议

Elicitation 是 MCP 服务器向用户请求信息或确认的机制。支持两种模式：

1. **Form 模式**: 服务器请求结构化数据输入
2. **URL 模式**: 服务器引导用户访问 URL 并等待确认

```typescript
// src/services/mcp/elicitationHandler.ts:29-47
export type ElicitationRequestEvent = {
  serverName: string
  requestId: string | number
  params: ElicitRequestParams
  signal: AbortSignal
  respond: (response: ElicitResult) => void
  waitingState?: ElicitationWaitingState
  onWaitingDismiss?: (action: 'dismiss' | 'retry' | 'cancel') => void
  completed?: boolean
}
```

### 25.6.2 处理器注册

在连接成功后注册 Elicitation 处理器：

```typescript
// src/services/mcp/elicitationHandler.ts:68-212
export function registerElicitationHandler(
  client: Client,
  serverName: string,
  setAppState: (f: (prevState: AppState) => AppState) => void,
): void {
  try {
    client.setRequestHandler(ElicitRequestSchema, async (request, extra) => {
      logMCPDebug(
        serverName,
        `Received elicitation request: ${jsonStringify(request)}`,
      )

      const mode = getElicitationMode(request.params)

      // 先运行 elicitation hooks - 可以通过编程方式提供响应
      const hookResponse = await runElicitationHooks(
        serverName,
        request.params,
        extra.signal,
      )
      if (hookResponse) {
        logMCPDebug(
          serverName,
          `Elicitation resolved by hook: ${jsonStringify(hookResponse)}`,
        )
        return hookResponse
      }

      // 将请求加入队列，等待用户交互
      const response = new Promise<ElicitResult>(resolve => {
        const onAbort = () => {
          resolve({ action: 'cancel' })
        }

        if (extra.signal.aborted) {
          onAbort()
          return
        }

        setAppState(prev => ({
          ...prev,
          elicitation: {
            queue: [
              ...prev.elicitation.queue,
              {
                serverName,
                requestId: extra.requestId,
                params: request.params,
                signal: extra.signal,
                waitingState: elicitationId ? { actionLabel: 'Skip confirmation' } : undefined,
                respond: (result: ElicitResult) => {
                  extra.signal.removeEventListener('abort', onAbort)
                  resolve(result)
                },
              },
            ],
          },
        }))

        extra.signal.addEventListener('abort', onAbort, { once: true })
      })

      const rawResult = await response
      return await runElicitationResultHooks(
        serverName,
        rawResult,
        extra.signal,
        mode,
        elicitationId,
      )
    })

    // 注册完成通知处理器（URL 模式）
    client.setNotificationHandler(
      ElicitationCompleteNotificationSchema,
      notification => {
        const { elicitationId } = notification.params
        logMCPDebug(
          serverName,
          `Received elicitation completion notification: ${elicitationId}`,
        )
        // 更新队列中对应请求的 completed 标志
        setAppState(prev => {
          const idx = findElicitationInQueue(
            prev.elicitation.queue,
            serverName,
            elicitationId,
          )
          if (idx === -1) return prev
          const queue = [...prev.elicitation.queue]
          queue[idx] = { ...queue[idx], completed: true }
          return { ...prev, elicitation: { queue } }
        })
      },
    )
  } catch {
    // 客户端未创建 elicitation capability - 无需注册
    return
  }
}
```

### 25.6.3 Hook 集成

Elicitation 支持通过 Hook 进行程序化处理：

```typescript
// src/services/mcp/elicitationHandler.ts:214-257
export async function runElicitationHooks(
  serverName: string,
  params: ElicitRequestParams,
  signal: AbortSignal,
): Promise<ElicitResult | undefined> {
  try {
    const mode = params.mode === 'url' ? 'url' : 'form'
    const url = 'url' in params ? (params.url as string) : undefined
    const elicitationId =
      'elicitationId' in params
        ? (params.elicitationId as string | undefined)
        : undefined

    const { elicitationResponse, blockingError } =
      await executeElicitationHooks({
        serverName,
        message: params.message,
        requestedSchema: 'requestedSchema' in params ? params.requestedSchema : undefined,
        signal,
        mode,
        url,
        elicitationId,
      })

    if (blockingError) {
      return { action: 'decline' }
    }

    if (elicitationResponse) {
      return {
        action: elicitationResponse.action,
        content: elicitationResponse.content,
      }
    }

    return undefined
  } catch (error) {
    logMCPError(serverName, `Elicitation hook error: ${error}`)
    return undefined
  }
}
```

### 25.6.4 默认响应

在连接初始化期间，设置默认的 Elicitation 响应：

```typescript
// src/services/mcp/client.ts:1188-1197
// 注册默认 elicitation 处理器，在 registerElicitationHandler 覆盖之前
// 返回 cancel 响应
client.setRequestHandler(ElicitRequestSchema, async request => {
  logMCPDebug(
    name,
    `Elicitation request received during initialization`,
  )
  return { action: 'cancel' as const }
})
```

## 25.7 认证处理

### 25.7.1 OAuth 认证

远程 MCP 服务器（SSE/HTTP）支持 OAuth 认证：

```typescript
// src/services/mcp/types.ts:43-56
const McpOAuthConfigSchema = lazySchema(() =>
  z.object({
    clientId: z.string().optional(),
    callbackPort: z.number().int().positive().optional(),
    authServerMetadataUrl: z
      .string()
      .url()
      .startsWith('https://', {
        message: 'authServerMetadataUrl must use https://',
      })
      .optional(),
    xaa: McpXaaConfigSchema().optional(),  // Cross-App Access
  }),
)
```

认证失败时，服务器状态变为 `needs-auth`：

```typescript
// src/services/mcp/client.ts:340-361
function handleRemoteAuthFailure(
  name: string,
  serverRef: ScopedMcpServerConfig,
  transportType: 'sse' | 'http' | 'claudeai-proxy',
): MCPServerConnection {
  logEvent('tengu_mcp_server_needs_auth', {
    transportType,
    ...mcpBaseUrlAnalytics(serverRef),
  })
  logMCPDebug(
    name,
    `Authentication required for ${transportType} server`,
  )
  setMcpAuthCacheEntry(name)  // 缓存 needs-auth 状态 15 分钟
  return { name, type: 'needs-auth', config: serverRef }
}
```

### 25.7.2 Claude.ai 代理认证

Claude.ai 代理使用特殊的认证包装：

```typescript
// src/services/mcp/client.ts:372-422
export function createClaudeAiProxyFetch(innerFetch: FetchLike): FetchLike {
  return async (url, init) => {
    const doRequest = async () => {
      await checkAndRefreshOAuthTokenIfNeeded()
      const currentTokens = getClaudeAIOAuthTokens()
      if (!currentTokens) {
        throw new Error('No claude.ai OAuth token available')
      }
      const headers = new Headers(init?.headers)
      headers.set('Authorization', `Bearer ${currentTokens.accessToken}`)
      const response = await innerFetch(url, { ...init, headers })
      return { response, sentToken: currentTokens.accessToken }
    }

    const { response, sentToken } = await doRequest()
    if (response.status !== 401) {
      return response
    }
    // 401 时尝试刷新 token 并重试
    const tokenChanged = await handleOAuth401Error(sentToken).catch(() => false)
    if (!tokenChanged) {
      return response
    }
    return (await doRequest()).response
  }
}
```

## 25.8 配置管理

### 25.8.1 配置优先级

MCP 配置按优先级合并，手动配置优先于插件配置：

```typescript
// src/services/mcp/config.ts:223-266
export function dedupPluginMcpServers(
  pluginServers: Record<string, ScopedMcpServerConfig>,
  manualServers: Record<string, ScopedMcpServerConfig>,
): {
  servers: Record<string, ScopedMcpServerConfig>
  suppressed: Array<{ name: string; duplicateOf: string }>
} {
  // 基于签名去重：相同命令或 URL 视为同一服务器
  const manualSigs = new Map<string, string>()
  for (const [name, config] of Object.entries(manualServers)) {
    const sig = getMcpServerSignature(config)
    if (sig && !manualSigs.has(sig)) manualSigs.set(sig, name)
  }

  const servers: Record<string, ScopedMcpServerConfig> = {}
  const suppressed: Array<{ name: string; duplicateOf: string }> = []

  for (const [name, config] of Object.entries(pluginServers)) {
    const sig = getMcpServerSignature(config)
    const manualDup = manualSigs.get(sig)
    if (manualDup !== undefined) {
      suppressed.push({ name, duplicateOf: manualDup })
      continue  // 手动配置优先
    }
    servers[name] = config
  }
  return { servers, suppressed }
}
```

### 25.8.2 配置签名计算

用于检测配置变更和去重：

```typescript
// src/services/mcp/config.ts:202-212
export function getMcpServerSignature(config: McpServerConfig): string | null {
  const cmd = getServerCommandArray(config)
  if (cmd) {
    return `stdio:${jsonStringify(cmd)}`
  }
  const url = getServerUrl(config)
  if (url) {
    return `url:${unwrapCcrProxyUrl(url)}`
  }
  return null  // SDK 类型无签名
}
```

### 25.8.3 配置范围描述

`utils.ts` 提供配置来源的友好描述：

```typescript
// src/services/mcp/utils.ts:263-298
export function describeMcpConfigFilePath(scope: ConfigScope): string {
  switch (scope) {
    case 'user':
      return getGlobalClaudeFile()
    case 'project':
      return join(getCwd(), '.mcp.json')
    case 'local':
      return `${getGlobalClaudeFile()} [project: ${getCwd()}]`
    case 'dynamic':
      return 'Dynamically configured'
    case 'enterprise':
      return getEnterpriseMcpFilePath()
    case 'claudeai':
      return 'claude.ai'
    default:
      return scope
  }
}

export function getScopeLabel(scope: ConfigScope): string {
  switch (scope) {
    case 'local':
      return 'Local config (private to you in this project)'
    case 'project':
      return 'Project config (shared via .mcp.json)'
    case 'user':
      return 'User config (available in all your projects)'
    case 'dynamic':
      return 'Dynamic config (from command line)'
    case 'enterprise':
      return 'Enterprise config (managed by your organization)'
    case 'claudeai':
      return 'claude.ai config'
    default:
      return scope
  }
}
```

## 25.9 小结

本章深入分析了 Claude Code 的 MCP 服务集成架构：

1. **协议基础**: MCP 基于 JSON-RPC 2.0，定义了工具、资源、提示和能力四大核心概念

2. **传输多样性**: 支持 Stdio、SSE、HTTP、WebSocket、进程内和 SDK 控制等多种传输方式，适配不同场景

3. **生命周期管理**: 从连接初始化、超时处理、错误恢复到优雅关闭，实现了完整的连接生命周期

4. **能力暴露**: 通过 memoize 缓存高效获取工具、资源、命令，并转换为 Claude Code 内部格式

5. **Elicitation 交互**: 支持 Form 和 URL 两种模式的用户交互，可通过 Hook 程序化处理

6. **认证支持**: OAuth 认证、Claude.ai 代理认证，以及 needs-auth 状态缓存

7. **配置管理**: 多层级配置优先级，基于签名的去重机制

MCP 服务集成使 Claude Code 能够无缝扩展能力，与外部系统建立标准化连接，是 Claude Code 作为"agentic coding tool"的核心基础设施。