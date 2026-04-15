---
title: "CLI Architecture"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-01-项目概述与开发环境.md, chapter-02-入口点与启动流程.md]
related: [../entities/anthropic.md, ../entities/bun.md, ../concepts/fast-path.md]
---

CLI 架构是 Claude Code 的整体设计模式，采用分层模块化结构，通过快速路径、动态导入和延迟加载优化启动性能，实现终端原生的 AI 交互体验。

## 架构层次

### 三层入口设计

```
cli.tsx (命令路由 + 快速路径)
    ↓
init.ts (核心初始化序列)
    ↓
main.tsx (主应用 + REPL 启动)
```

### 分层职责

| 层次 | 职责 | 特点 |
|------|------|------|
| cli.tsx | 命令路由、快速路径检测 | 最小化模块导入 |
| init.ts | 配置、网络、安全、遥测 | memoize 包装，单次执行 |
| main.tsx | Commander 解析、状态初始化 | REPL 启动、延迟预取 |

## 模块化设计

### 核心目录结构

```
src/
├── entrypoints/    CLI 入口点
├── components/     React UI 组件 (140+)
├── tools/          Tool 实现 (40+)
├── commands/       斜杠命令 (100+)
├── services/       服务层 (API, MCP, 分析)
├── hooks/          React Hooks (85+)
├── utils/          工具函数 (300+)
├── state/          AppState 状态管理
└── constants/      常量定义 (提示词、配置)
```

### 模块边界

- **tools/**：定义 AI 可执行操作
- **commands/**：用户斜杠命令
- **services/**：外部系统交互（API、MCP、遥测）
- **components/**：终端 UI 渲染

## 性能优化策略

### 快速路径

- `--version`：零模块导入，毫秒级响应
- `daemon/bridge/mcp`：轻量初始化，提前退出

### 延迟加载

- 遥测模块延迟到信任对话框之后
- 用户信息、云凭证在 REPL 渲染后预取

### 并行 I/O

- MDM 子进程在模块导入阶段启动
- keychain 预读取与主模块导入并行

## Connections

- [Anthropic](../entities/anthropic.md) - 开发公司
- [Bun](../entities/bun.md) - 运行时
- [Fast Path](../concepts/fast-path.md) - 快速路径机制

## Open Questions

- 不同模块间的依赖关系图
- 模块边界的具体定义和演进历史

## Sources

- `chapters/chapter-01-项目概述与开发环境.md`
- `chapters/chapter-02-入口点与启动流程.md`