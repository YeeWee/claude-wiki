---
title: "入口点与启动流程"
type: source
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-02-入口点与启动流程.md]
related: [../concepts/fast-path.md, ../concepts/daemon-mode.md, ../concepts/bridge-mode.md, ../concepts/init-sequence.md]
---

Claude Code 的启动流程采用三层架构设计：cli.tsx 负责快速路径检测和命令路由，init.ts 处理核心初始化序列，main.tsx 执行主应用启动。通过动态导入和条件判断最小化启动延迟，确保 `--version` 等常见操作响应时间在毫秒级别。

## cli.tsx 入口

cli.tsx 是最外层入口点，核心设计理念是**最小化启动延迟**。

### 环境预处理

文件开头的预处理代码在模块加载时立即执行：

- 修复 corepack 自动 pinning 问题
- CCR 环境下设置子进程最大堆内存
- Ablation baseline 实验环境配置

### 快速路径架构

快速路径检测是核心机制，每个分支处理特定命令模式并提前退出：

| 快速路径 | 触发条件 | 用途 |
|---------|---------|------|
| `--version` | 单参数 `-v/--version` | 版本查询，零导入 |
| `--dump-system-prompt` | 特定参数 | 输出系统提示词 |
| `--claude-in-chrome-mcp` | Chrome MCP 服务器模式 | 浏览器扩展集成 |
| `daemon` | Daemon 主进程 | 长驻后台监管进程 |
| `remote-control/bridge` | Bridge 远程控制模式 | 本地机器作为远程环境 |

`--version` 是最极致的快速路径：**零模块导入**，直接输出构建时内联的版本号。

## init.ts 初始化

init 函数使用 memoize 包装，确保只执行一次，核心初始化序列包括：

1. **配置系统启用**：`enableConfigs()`
2. **安全环境变量应用**：`applySafeConfigEnvironmentVariables()`
3. **CA 证书配置**：必须在任何 TLS 连接之前
4. **优雅退出设置**：`setupGracefulShutdown()`
5. **遥测初始化**：延迟加载避免启动时加载重型模块
6. **OAuth 信息填充**：`populateOAuthAccountInfoIfNeeded()`
7. **网络配置**：mTLS 和代理配置
8. **API 预连接**：并行执行减少首次请求延迟

### 性能优化策略

- **并行 I/O 启动**：MDM 子进程和 keychain 读取在模块导入阶段并行执行
- **延迟加载**：遥测模块延迟到信任对话框之后
- **预连接**：API 预连接与初始化并行

## main.tsx 主应用

主应用入口负责命令解析、状态初始化和 REPL 启动。

### 模块导入优化

预读取操作在模块导入阶段执行：

- `startMdmRawRead()` - 启动 MDM 子进程读取
- `startKeychainPrefetch()` - 启动 macOS keychain 预读取

### 条件导入

使用 `require()` 配合 `feature()` 门控实现动态导入和死代码消除。

### 延迟预取

`startDeferredPrefetches()` 在 REPL 渲染后执行：

- 用户信息预取
- 云提供商凭证预取
- 文件计数预取
- 分析和功能门控初始化

## Bridge 模式

Bridge 模式是远程控制功能，检查顺序体现安全设计原则：

1. 认证优先：OAuth 检查必须在功能门控之前
2. 功能门控：检查运行时功能开关
3. 版本兼容：确保客户端版本满足最低要求
4. 组织策略：最终的安全护栏

## Daemon 模式

Daemon 是长驻后台监管进程：

- 初始化更轻量：只调用 `initSinks()` 而非完整 `init()`
- 工作进程分离：`--daemon-worker` 用于子进程

## Connections

- [Fast Path](../concepts/fast-path.md) - 快速路径检测机制
- [Daemon Mode](../concepts/daemon-mode.md) - 后台监管进程
- [Bridge Mode](../concepts/bridge-mode.md) - 远程控制模式
- [Init Sequence](../concepts/init-sequence.md) - 初始化序列详解

## Open Questions

- MDM 子进程读取的具体实现细节
- keychain 预取如何与 macOS 安全框架交互
- 深度链接（cc:// 协议）的完整处理流程

## Sources

- `chapters/chapter-02-入口点与启动流程.md`