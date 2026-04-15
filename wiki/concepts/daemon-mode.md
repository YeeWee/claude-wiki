---
title: "Daemon Mode"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-02-入口点与启动流程.md]
related: [../concepts/fast-path.md, ../concepts/bridge-mode.md]
---

Daemon 模式是 Claude Code 的长驻后台监管进程，负责监控和管理后台任务。通过轻量初始化（仅调用 initSinks 而非完整 init）实现快速启动，与工作进程分离架构确保稳定性。

## 设计目的

### 长驻后台管理

Daemon 进程作为监管者：

- 监控后台任务状态
- 管理 daemon worker 工作进程
- 处理跨会话协调

### 与 Bridge 模式的区别

| 模式 | 初始化 | 用途 |
|------|--------|------|
| Daemon | initSinks() | 本地后台管理 |
| Bridge | enableConfigs + OAuth + GrowthBook | 远程控制 |

## 初始化流程

```typescript
// cli.tsx
if (feature('DAEMON') && args[0] === 'daemon') {
  profileCheckpoint('cli_daemon_path');
  enableConfigs();  // 仅启用配置
  initSinks();      // 仅初始化分析 sinks，而非完整 init()
  await daemonMain(args.slice(1));
  return;
}
```

### 轻量初始化

只调用 `initSinks()` 而非完整的 `init()`：

- 不加载遥测模块
- 不初始化网络配置
- 不执行 OAuth 信息填充

## 工作进程分离

### daemon-worker 路径

```typescript
if (feature('DAEMON') && args[0] === 'daemon-worker') {
  // 工作进程：不启用配置和分析
  await daemonWorkerMain(args.slice(1));
  return;
}
```

分离设计优势：

- 主进程负责监管和协调
- 工作进程负责实际任务执行
- 故障隔离：工作进程崩溃不影响主进程

## Connections

- [Fast Path](../concepts/fast-path.md) - Daemon 作为快速路径之一
- [Bridge Mode](../concepts/bridge-mode.md) - 对比另一种后台模式

## Open Questions

- Daemon 与工作进程的通信机制
- Daemon 的任务调度和状态管理

## Sources

- `chapters/chapter-02-入口点与启动流程.md`