---
title: "Bun"
type: entity
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-01-项目概述与开发环境.md, chapter-03-feature-flag与构建变体.md]
related: [../concepts/feature-flag.md, ../concepts/dead-code-elimination.md]
---

Bun 是 Claude Code 的运行时和打包工具，由 Jarred Sumner 开发，以极速启动和内置打包能力著称。Claude Code 选择 Bun 作为核心技术栈，充分利用其 TypeScript 原生支持和 `bun:bundle` feature flag 机制。

## 核心特性

### 极速启动

Bun 的启动速度远超 Node.js，这对 CLI 工具至关重要：

- 原生 TypeScript 支持，无需预编译
- 快速模块解析和加载
- 低延迟的首字节响应

### bun:bundle feature flag

`bun:bundle` 模块提供构建时 feature flag 支持：

```typescript
import { feature } from 'bun:bundle'

if (feature('DUMP_SYSTEM_PROMPT')) {
  // 此代码块在 external 构建中被移除
}
```

工作原理：

1. 构建时读取 feature flag 配置
2. 将 `feature('FLAG_NAME')` 替换为布尔常量
3. 执行死代码消除（DCE）

### DCE 限制

Bun bundler 的 `feature()` 评估器有每个函数的复杂度预算限制：

- 导入别名计入复杂度预算
- 超过阈值时无法证明 feature() 是常量
- 解决方案：将导入别名移到顶层 `const`

## 在 Claude Code 中的应用

### 快速路径设计

利用 Bun 的快速启动实现快速路径：

- `--version` 路径：零模块导入，毫秒级响应
- 动态导入避免加载重型模块

### 构建变体

通过 Bun 的构建能力区分 ant 和 external 版本：

- ant 构建包含所有内部功能
- external 构建移除敏感代码

## Connections

- [Feature Flag](../concepts/feature-flag.md) - 功能开关机制
- [Dead Code Elimination](../concepts/dead-code-elimination.md) - DCE 技术
- [Project Overview](../sources/2026-04-15-project-overview.md) - 技术栈详解

## Open Questions

- Bun 与 Node.js 在 CLI 工具场景的具体性能对比数据
- bun:bundle 的完整配置选项和限制

## Sources

- `chapters/chapter-01-项目概述与开发环境.md`
- `chapters/chapter-03-feature-flag与构建变体.md`