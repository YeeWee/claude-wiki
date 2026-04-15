---
title: "Anthropic"
type: entity
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-01-项目概述与开发环境.md, chapter-03-feature-flag与构建变体.md, chapter-49-学习资源与社区.md, chapter-43-远程设置同步.md]
related: [../concepts/cli-architecture.md, ../sources/2026-04-15-project-overview.md, ../concepts/mcp-extension.md, ../concepts/learning-path.md]
---

Anthropic 是 Claude Code 的开发公司，专注于构建安全、有益的 AI 系统。Claude Code 作为 Anthropic 官方的命令行界面工具，直接连接到 Claude 大语言模型。

## 在 Claude Code 中的角色

### 产品定位

Claude Code 是 Anthropic 官方推出的 CLI 工具，与 Web Claude 和 API 集成方式并行：

- 官方维护和发布
- 直接使用 Anthropic API
- 内部版本（ant）和外部版本（external）的区分

### ant vs external 构建变体

Anthropic 内部员工使用 ant 构建，享有额外的功能和权限：

- 专属工具：ConfigTool、TungstenTool、REPLTool
- 环境变量权限：可访问更多安全环境变量
- 分析元数据：额外的调试和分析功能
- `/config` 命令：可覆盖 GrowthBook 配置

### API 集成

- 使用 `@anthropic-ai/sdk` 作为官方 SDK
- 支持 Claude Opus、Sonnet、Haiku 等模型
- 实现 prompt caching、thinking mode 等高级特性

## Connections

- [CLI Architecture](../concepts/cli-architecture.md) - Claude Code 的架构设计
- [Project Overview](../sources/2026-04-15-project-overview.md) - 项目概述
- [Feature Flags](../sources/2026-04-15-feature-flags.md) - 构建变体系统

## Open Questions

- Anthropic 内部的 Claude Code 使用场景和反馈
- ant 版本的完整功能列表

## Sources

- `chapters/chapter-01-项目概述与开发环境.md`
- `chapters/chapter-03-feature-flag与构建变体.md`