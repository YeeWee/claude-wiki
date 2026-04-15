---
title: "Fast Path"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-02-入口点与启动流程.md]
related: [../concepts/cli-architecture.md, ../concepts/daemon-mode.md, ../concepts/bridge-mode.md]
---

Fast Path（快速路径）是 Claude Code 的启动优化机制，通过在 cli.tsx 入口处检测特定命令模式并提前退出，避免加载不必要的重型模块，将版本查询等常见操作的响应时间降至毫秒级别。

## 设计理念

### 最小化启动延迟

CLI 工具的启动速度直接影响用户体验：

- 用户期望版本查询等简单操作快速响应
- 避免加载 React、Ink、API 客户端等重型模块
- 利用 Bun 的快速启动特性

### 分层检测

在 cli.tsx 入口处按优先级检测：

1. 最高优先级：零模块导入的快速路径
2. 中等优先级：轻量初始化的特殊模式
3. 默认路径：完整初始化和 REPL 启动

## 快速路径列表

| 快速路径 | 触发条件 | 初始化级别 |
|---------|---------|-----------|
| `--version` | `-v/--version` | 零导入 |
| `--dump-system-prompt` | 特定参数 | feature() 门控 |
| `daemon` | Daemon 主进程 | enableConfigs + initSinks |
| `remote-control/bridge` | Bridge 模式 | enableConfigs + OAuth + GrowthBook |
| `--claude-in-chrome-mcp` | Chrome MCP | enableConfigs + MCP 初始化 |
| `ps/logs/attach/kill` | 后台会话管理 | enableConfigs |

## --version 极致优化

```typescript
// cli.tsx
if (args.length === 1 && args[0] === '--version') {
  console.log(`${MACRO.VERSION} (Claude Code)`);  // 构建时内联
  return;  // 直接退出，不加载任何其他模块
}
```

特点：

- **零导入**：不加载任何模块
- **构建时内联**：MACRO.VERSION 在构建时被替换为版本字符串
- **毫秒级响应**：启动延迟 ~10ms

## Bridge 模式初始化

```typescript
if (feature('BRIDGE_MODE') && args[0] === 'remote-control') {
  enableConfigs();
  
  // OAuth 检查必须在 GrowthBook 之前
  if (!getClaudeAIOAuthTokens()?.accessToken) {
    exitWithError(BRIDGE_LOGIN_ERROR);
  }
  
  // GrowthBook 功能门控
  const disabledReason = await getBridgeDisabledReason();
  if (disabledReason) { exitWithError(disabledReason); }
  
  await bridgeMain(args.slice(1));
  return;
}
```

检查顺序体现安全设计：认证 → 功能门控 → 版本 → 组织策略。

## Connections

- [CLI Architecture](../concepts/cli-architecture.md) - 整体架构设计
- [Daemon Mode](../concepts/daemon-mode.md) - Daemon 快速路径
- [Bridge Mode](../concepts/bridge-mode.md) - Bridge 快速路径

## Open Questions

- 快速路径的完整列表和触发条件
- startCapturingEarlyInput() 的具体实现

## Sources

- `chapters/chapter-02-入口点与启动流程.md`