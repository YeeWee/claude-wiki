# Wiki Index

_Last updated: 2026-04-15 | Pages: 193 | Sources: 50 | Entities: 47 | Concepts: 92_

## Sources
| Page | Summary | Date | Source file |
|---|---|---|---|
| [项目概述](sources/2026-04-15-project-overview.md) | Claude Code 项目整体架构与开发环境配置 | 2026-04-15 | chapter-01 |
| [入口点与启动](sources/2026-04-15-entry-points.md) | CLI、Bridge、Daemon、Fast Path 多入口启动流程 | 2026-04-15 | chapter-02 |
| [Feature Flags](sources/2026-04-15-feature-flags.md) | GrowthBook 特性开关与构建变体 DCE | 2026-04-15 | chapter-03 |
| [配置系统](sources/2026-04-15-configuration-system.md) | 四层配置合并与环境变量 Schema 验证 | 2026-04-15 | chapter-04 |
| [类型系统](sources/2026-04-15-type-system.md) | Tool/Command/TaskState/Message 类型体系 | 2026-04-15 | chapter-05 |
| [AppState](sources/2026-04-15-appstate-management.md) | Zustand Store + React 集成的状态管理 | 2026-04-15 | chapter-06 |
| [React Context](sources/2026-04-15-react-context体系.md) | 分层 Context、Notification 优先级队列、Stats | 2026-04-15 | chapter-07 |
| [工具架构总览](sources/2026-04-15-工具架构总览.md) | Tool 接口、buildTool 工厂、条件加载 | 2026-04-15 | chapter-08 |
| [BashTool](sources/2026-04-15-bashtool深度解析.md) | Sandbox 安全机制、权限检查、后台执行 | 2026-04-15 | chapter-09 |
| [文件工具集](sources/2026-04-15-文件操作工具集.md) | Read/Edit/Write、缓存机制、先读后写约束 | 2026-04-15 | chapter-10 |
| [搜索工具集](sources/2026-04-15-搜索工具集.md) | Glob/Grep、ripgrep 集成、输出模式 | 2026-04-15 | chapter-11 |
| [AgentTool](sources/2026-04-15-agenttool多agent机制.md) | Agent 类型、Worktree 隔离、Coordinator 模式 | 2026-04-15 | chapter-12 |
| [Web工具集](sources/2026-04-15-web-tools.md) | WebFetch/WebSearch、缓存与超时策略 | 2026-04-15 | chapter-13 |
| [任务管理](sources/2026-04-15-task-management-tools.md) | TaskCreate/Update、结构化任务追踪 | 2026-04-15 | chapter-14 |
| [计划模式](sources/2026-04-15-plan-mode-tools.md) | EnterPlanMode/ExitPlanModeV2、规划-执行工作流 | 2026-04-15 | chapter-15 |
| [Worktree](sources/2026-04-15-worktree-tools.md) | Git 工作树隔离、分支管理 | 2026-04-15 | chapter-16 |
| [其他工具](sources/2026-04-15-miscellaneous-tools.md) | Skill/MCP/Config/AskUserQuestion | 2026-04-15 | chapter-17 |
| [命令系统](sources/2026-04-15-command-system-architecture.md) | Prompt/Local/LocalJSX 命令类型架构 | 2026-04-15 | chapter-18 |
| [内置命令](sources/2026-04-15-chapter-19-内置命令详解.md) | 用户交互入口点详解 | 2026-04-15 | chapter-19 |
| [技能系统](sources/2026-04-15-chapter-20-技能系统架构.md) | Skill 定义、Frontmatter、Fork Context | 2026-04-15 | chapter-20 |
| [技能实战](sources/2026-04-15-chapter-21-技能实战解析.md) | 技能配置与执行案例 | 2026-04-15 | chapter-21 |
| [插件系统](sources/2026-04-15-chapter-22-插件系统.md) | Plugin 机制、内置插件、MCP 集成 | 2026-04-15 | chapter-22 |
| [Hook系统](sources/2026-04-15-chapter-23-hook系统.md) | Hook 事件类型、异步执行 | 2026-04-15 | chapter-23 |
| [API客户端](sources/2026-04-15-chapter-24-api客户端服务.md) | Stream 处理、Token 追踪 | 2026-04-15 | chapter-24 |
| [MCP集成](sources/2026-04-15-mcp服务集成.md) | MCP 协议、Transport 层、生命周期 | 2026-04-15 | chapter-25 |
| [上下文压缩](sources/2026-04-15-上下文压缩服务.md) | Token 预算、摘要模板、历史裁剪 | 2026-04-15 | chapter-26 |
| [LSP服务](sources/2026-04-15-lsp服务.md) | LSP 协议、代码诊断、文件同步 | 2026-04-15 | chapter-27 |
| [分析服务](sources/2026-04-15-分析与服务.md) | GrowthBook、OpenTelemetry、事件采样 | 2026-04-15 | chapter-28 |
| [Bridge架构](sources/2026-04-15-bridge系统架构.md) | WebSocket/SSE 传输、心跳重连 | 2026-04-15 | chapter-29 |
| [REPL Bridge](sources/2026-04-15-repl-bridge.md) | REPL 消息流转、FlushGate | 2026-04-15 | chapter-30 |
| [远程会话](sources/2026-04-15-remote-session-management.md) | JWT 刷新、SSE 传输、会话生命周期 | 2026-04-15 | chapter-31 |
| [Coordinator](sources/2026-04-15-coordinator-mode.md) | Multi-agent 编排、Worker、Scratchpad | 2026-04-15 | chapter-32 |
| [权限系统](sources/2026-04-15-permission-system.md) | Permission modes、规则匹配、危险模式 | 2026-04-15 | chapter-33 |
| [沙箱安全](sources/2026-04-15-sandbox-security.md) | FAIL-CLOSED、bare git repo 攻击防护 | 2026-04-15 | chapter-34 |
| [Ink UI](sources/2026-04-15-ink-ui-framework.md) | React reconciler、Yoga 布局、ANSI diff | 2026-04-15 | chapter-35 |
| [焦点交互](sources/2026-04-15-focus-and-interaction.md) | FocusManager、文本选择、键盘解析 | 2026-04-15 | chapter-36 |
| [组件架构](sources/2026-04-15-component-architecture.md) | React+Ink 分层组件架构 | 2026-04-15 | chapter-37 |
| [QueryEngine](sources/2026-04-15-query-engine.md) | 对话编排器核心类 | 2026-04-15 | chapter-38 |
| [消息处理](sources/2026-04-15-message-processing.md) | 消息类型与内容块体系 | 2026-04-15 | chapter-39 |
| [对话流程](sources/2026-04-15-dialogue-flow.md) | 流式处理、并发控制、错误恢复 | 2026-04-15 | chapter-40 |
| [语音模式](sources/2026-04-15-voice-mode.md) | VoiceProvider、语音状态管理 | 2026-04-15 | chapter-41 |
| [提示建议](sources/2026-04-15-prompt-suggestion.md) | AI 驱动提示建议系统 | 2026-04-15 | chapter-42 |
| [远程设置](sources/2026-04-15-remote-managed-settings-sync.md) | Checksum 校验、优雅降级 | 2026-04-15 | chapter-43 |
| [历史回放](sources/2026-04-15-session-history.md) | 会话历史机制 | 2026-04-15 | chapter-44 |
| [调试诊断](sources/2026-04-15-debugging-and-diagnosis.md) | /doctor 命令、调试日志 | 2026-04-15 | chapter-45 |
| [架构原则](sources/2026-04-15-architecture-design-principles.md) | Monolith+动态导入、Fail-close | 2026-04-15 | chapter-46 |
| [性能优化](sources/2026-04-15-performance-optimization.md) | Lazy Loading、虚拟滚动 | 2026-04-15 | chapter-47 |
| [可扩展性](sources/2026-04-15-extensibility-design.md) | Tool/Command/Skill/MCP 扩展方式 | 2026-04-15 | chapter-48 |
| [学习资源](sources/2026-04-15-learning-resources.md) | 学习路径与社区资源 | 2026-04-15 | chapter-49 |
| [附录](sources/2026-04-15-appendix.md) | 补充材料 | 2026-04-15 | chapter-50 |

## Entities
| Page | Summary |
|---|---|
| [Anthropic](entities/anthropic.md) | Claude Code 开发公司 |
| [Bun](entities/bun.md) | JavaScript 运行时 |
| [React/Ink](entities/react-ink.md) | React 终端 UI 框架 |
| [GrowthBook](entities/growthbook.md) | Feature Flag 配置平台 |
| [MCP](entities/mcp.md) | Model Context Protocol |
| [Datadog](entities/datadog.md) | 监控与分析平台 |
| [OpenTelemetry](entities/opentelemetry.md) | 遥测标准 |
| [GitHub](entities/github.md) | 代码托管平台 |
| [Language Server](entities/language-server.md) | LSP 协议实现 |
| [QueryEngine](entities/query-engine.md) | 对话编排核心类 |
| [StreamingToolExecutor](entities/streaming-tool-executor.md) | 流式工具执行器 |
| [VoiceProvider](entities/voice-provider.md) | 语音状态管理 |
| [ThemeProvider](entities/theme-provider.md) | 主题上下文 |
| [SandboxManager](entities/sandbox-manager.md) | 沙箱安全适配层 |
| [FocusManager](entities/focus-manager.md) | DOM 式焦点管理 |
| [BashTool](entities/bash-tool.md) | 命令执行引擎 |
| [ToolInterface](entities/tool-interface.md) | 工具系统核心接口 |
| [FileStateCache](entities/file-state-cache.md) | 文件状态缓存 |
| [FileReadTool](entities/file-read-tool.md) | 多格式文件读取 |
| [FileEditTool](entities/file-edit-tool.md) | 确字符串替换 |
| [FileWriteTool](entities/file-write-tool.md) | 文件创建/覆盖 |
| [GlobTool](entities/glob-tool.md) | 文件名匹配 |
| [GrepTool](entities/grep-tool.md) | 内容搜索 |
| [Ripgrep](entities/ripgrep.md) | 搜索底层引擎 |
| [AgentTool](entities/agent-tool.md) | Multi-agent 协作核心 |
| [AgentDefinition](entities/agent-definition.md) | Agent 类型定义 |
| [StatsStore](entities/stats-store.md) | 统计数据存储 |
| [WebFetchTool](entities/web-fetch-tool.md) | 网页抓取 |
| [WebSearchTool](entities/web-search-tool.md) | 网络搜索 |
| [TaskCreateTool](entities/task-create-tool.md) | 任务创建 |
| [TaskUpdateTool](entities/task-update-tool.md) | 任务更新 |
| [EnterPlanModeTool](entities/enter-plan-mode-tool.md) | 进入计划模式 |
| [ExitPlanModeV2Tool](entities/exit-plan-mode-v2-tool.md) | 退出计划模式 |
| [EnterWorktreeTool](entities/enter-worktree-tool.md) | 创建 worktree |
| [ExitWorktreeTool](entities/exit-worktree-tool.md) | 退出 worktree |
| [SkillTool](entities/skill-tool.md) | 技能执行 |
| [MCPTool](entities/mcp-tool.md) | MCP 调用 |
| [ConfigTool](entities/config-tool.md) | 配置管理 |
| [AskUserQuestionTool](entities/ask-user-question-tool.md) | 用户问答 |
| [PromptCommand](entities/prompt-command.md) | 提示生成命令 |
| [LocalCommand](entities/local-command.md) | 本地逻辑命令 |
| [LocalJSXCommand](entities/local-jsx-command.md) | JSX UI 命令 |
| [BundledSkill](entities/bundled-skill.md) | 内置技能 |
| [Plugin](entities/plugin.md) | 插件实体 |
| [CoordinatorMode](entities/coordinator-mode.md) | Multi-agent 编排模式 |
| [PermissionRule](entities/permission-rule.md) | 权限规则结构 |
| [Scratchpad](entities/scratchpad.md) | Worker 间知识共享 |

## Concepts
| Page | Summary |
|---|---|
| [CLI Architecture](concepts/cli-architecture.md) | 多入口点架构设计 |
| [Feature Flag](concepts/feature-flag.md) | 构建时+运行时特性开关 |
| [DCE](concepts/dead-code-elimination.md) | 死代码消除技术 |
| [Daemon Mode](concepts/daemon-mode.md) | 后台守护进程 |
| [Bridge Mode](concepts/bridge-mode.md) | 远程控制模式 |
| [Fast Path](concepts/fast-path.md) | 快速启动路径 |
| [Init Sequence](concepts/init-sequence.md) | 启动初始化序列 |
| [Build Variants](concepts/build-variants.md) | 构建变体配置 |
| [Settings Layers](concepts/settings-layers.md) | 四层配置合并 |
| [Environment Variables](concepts/environment-variables.md) | 环境变量管理 |
| [Schema Validation](concepts/schema-validation.md) | 配置验证机制 |
| [Remote Managed Settings](concepts/remote-managed-settings.md) | 远程设置同步 |
| [Checksum Validation](concepts/checksum-validation.md) | 设置校验机制 |
| [Graceful Degradation](concepts/graceful-degradation.md) | 容错策略 |
| [Store Pattern](concepts/store-pattern.md) | Zustand 状态管理 |
| [Selector Pattern](concepts/selector-pattern.md) | 状态选择器 |
| [React Integration](concepts/react-integration.md) | React 状态集成 |
| [AsyncGenerator Pattern](concepts/async-generator-pattern.md) | 流式进度模式 |
| [buildTool Factory](concepts/build-tool-factory.md) | 工具定义工厂 |
| [Tool Type](concepts/tool-type.md) | 工具类型定义 |
| [Command Type](concepts/command-type.md) | 命令类型定义 |
| [TaskState Type](concepts/taskstate-type.md) | 任务状态类型 |
| [Message Type](concepts/message-type.md) | 消息类型定义 |
| [Sandbox Isolation](concepts/sandbox-isolation.md) | 操作系统级安全隔离 |
| [Read Before Write](concepts/read-before-write.md) | 文件操作安全约束 |
| [LRU Cache](concepts/lru-cache.md) | 双重限制缓存 |
| [Reservoir Sampling](concepts/reservoir-sampling.md) | 无偏分布估算 |
| [Priority Queue](concepts/priority-queue.md) | 通知优先级管理 |
| [Folding Mechanism](concepts/folding-mechanism.md) | 通知折叠机制 |
| [Worktree Isolation](concepts/worktree-isolation.md) | Git 工作树隔离 |
| [Coordinator Pattern](concepts/coordinator-mode.md) | 协调者-Worker 协作 |
| [Async Worker Execution](concepts/async-worker-execution.md) | 异步 Worker 执行 |
| [Background Execution](concepts/background-execution.md) | 异步任务机制 |
| [Permission Check Flow](concepts/permission-check-flow.md) | 五层权限防护 |
| [Permission Modes](concepts/permission-modes.md) | default/plan/auto/bypass |
| [Permission Context](concepts/permission-context.md) | 权限检查上下文 |
| [Permission System](concepts/permission-system.md) | 权限系统架构 |
| [Command System](concepts/command-system.md) | 用户交互入口点 |
| [Command Types](concepts/command-types.md) | 命令类型分类 |
| [Skill System](concepts/skill-system.md) | Skill 定义与执行 |
| [Frontmatter](concepts/frontmatter.md) | Skill YAML 配置 |
| [Fork Context](concepts/fork-context.md) | Fork 执行上下文 |
| [Plugin System](concepts/plugin-system.md) | Plugin 机制 |
| [Builtin Plugin](concepts/builtin-plugin.md) | 内置插件 |
| [MCP Integration](concepts/mcp-integration.md) | MCP 集成架构 |
| [MCP Protocol](concepts/mcp-protocol.md) | MCP 协议规范 |
| [MCP Skill Builder](concepts/mcp-skill-builder.md) | MCP 技能构建 |
| [Hook System](concepts/hook-system.md) | 事件驱动扩展 |
| [Hook Event](concepts/hook-event.md) | Hook 事件类型 |
| [Async Hook](concepts/async-hook.md) | 异步 Hook 执行 |
| [Hook Driven Customization](concepts/hook-driven-customization.md) | Hook 定制模式 |
| [Transport Layer](concepts/transport-layer.md) | 传输层抽象 |
| [WebSocket Transport](concepts/websocket-transport.md) | WebSocket 传输 |
| [SSE Transport](concepts/sse-transport.md) | SSE 传输 |
| [Context Compaction](concepts/context-compaction.md) | Token 预算管理 |
| [Token Budget](concepts/token-budget.md) | Token 预算分配 |
| [LSP Protocol](concepts/lsp-protocol.md) | Language Server Protocol |
| [Code Diagnosis](concepts/code-diagnosis.md) | 代码诊断机制 |
| [Event Logging](concepts/event-logging.md) | 双通道导出 |
| [Bridge System](concepts/bridge-system.md) | 远程控制架构 |
| [REPL Bridge](concepts/repl-bridge.md) | REPL 消息流转 |
| [Session Management](concepts/session-management.md) | 生命周期管理 |
| [Session History](concepts/session-history.md) | 会话历史机制 |
| [Fail-Closed Principle](concepts/fail-closed-principle.md) | 安全哲学 |
| [Multi-agent Orchestration](concepts/multi-agent-orchestration.md) | Coordinator+Workers |
| [Focus Management](concepts/focus-management.md) | Tab 导航、栈恢复 |
| [Text Selection](concepts/text-selection.md) | anchor/focus 模型 |
| [Component Architecture](concepts/component-architecture.md) | React+Ink 分层 |
| [Message Type System](concepts/message-type-system.md) | 内容块体系 |
| [Streaming Processing](concepts/streaming-processing.md) | 流式处理模式 |
| [Stream Processing](concepts/stream-processing.md) | API 流处理 |
| [Token Tracking](concepts/token-tracking.md) | Token 追踪机制 |
| [API Client](concepts/api-client.md) | API 客户端设计 |
| [Tool Concurrency Control](concepts/tool-concurrency-control.md) | 工具执行策略 |
| [Error Recovery](concepts/error-recovery.md) | 多级恢复机制 |
| [Forked Agent](concepts/forked-agent.md) | 子 Agent 缓存复用 |
| [Speculation](concepts/speculation.md) | Overlay 隔离 |
| [Prompt Suggestion](concepts/prompt-suggestion.md) | AI 驱动系统 |
| [Task System](concepts/task-system.md) | 结构化工作计划 |
| [Plan Mode](concepts/plan-mode.md) | 规划-执行工作流 |
| [Git Worktree](concepts/git-worktree.md) | 隔离工作目录 |
| [Caching Mechanism](concepts/caching-mechanism.md) | LRU 优化 |
| [Teammate Collaboration](concepts/teammate-collaboration.md) | 多 Agent 协作 |
| [Isolation Mechanism](concepts/isolation-mechanism.md) | 多任务并行支撑 |
| [Doctor Command](concepts/doctor-command.md) | 诊断命令 |
| [Debug Logging](concepts/debug-logging.md) | 日志系统 |
| [Monolith Dynamic Import](concepts/monolith-dynamic-import.md) | 单 bundle+延迟加载 |
| [Tool Extensibility](concepts/tool-extensibility.md) | 工具扩展理念 |
| [Tool Extension](concepts/tool-extension.md) | 工具扩展实现 |
| [Command Extension](concepts/command-extension.md) | 命令扩展方式 |
| [Skill Extension](concepts/skill-extension.md) | 技能扩展方式 |
| [MCP Extension](concepts/mcp-extension.md) | MCP 扩展方式 |
| [Lazy Loading](concepts/lazy-loading.md) | 延迟加载 |
| [Virtual Scrolling](concepts/virtual-scrolling.md) | 大列表优化 |
| [Merge Strategy](concepts/merge-strategy.md) | 配置合并策略 |
| [Learning Path](concepts/learning-path.md) | 知识体系导航 |

## Comparisons
| Page | Summary |
|---|---|

## Queries
| Page | Summary | Date |
|---|---|---|