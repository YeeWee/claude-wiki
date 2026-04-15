---
title: "Component Architecture"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: ["chapter-37-组件架构.md"]
related: ["query-engine", "theme-provider"]
---

Claude Code 的前端组件架构采用 React + Ink 框架实现终端用户界面，遵循分层设计原则。

## 分层结构

从顶层到底层：
1. **顶层包装器** - App.tsx，提供全局 Context Provider
2. **Context Providers** - FpsMetricsProvider -> StatsProvider -> AppStateProvider -> ThemeProvider
3. **核心渲染组件** - Messages.tsx、Message.tsx、VirtualMessageList.tsx
4. **消息类型组件** - UserTextMessage、AssistantTextMessage、AssistantToolUseMessage 等
5. **工具交互组件** - ToolUseLoader、MessageResponse、PermissionRequest
6. **状态显示组件** - StatusLine、AgentProgressLine、Spinner
7. **设计系统** - ThemedBox、ThemedText、Dialog、ProgressBar

## 类型分发机制

Message.tsx 根据 `message.type` 属性动态选择渲染子组件：
- `attachment` -> AttachmentMessage
- `assistant` -> AssistantMessageBlock（进一步细分：text/tool_use/thinking）
- `user` -> UserMessage
- `system` -> SystemTextMessage

## 性能优化策略

- **React Compiler**：使用 `_c()` 编译缓存减少重渲染
- **虚拟列表**：VirtualMessageList 处理大量消息
- **Memo 缓存**：组件结果缓存避免重复计算
- **Ref 持久化**：状态数据通过 Ref 保持稳定引用

## 目录结构

`src/components/` 包含约 146 个组件文件，按功能分组：messages/、design-system/、permissions/、PromptInput/、shell/ 等。

## Connections

- [QueryEngine](../entities/query-engine.md) - 对话状态管理
- [ThemeProvider](../entities/theme-provider.md) - 主题上下文

## Open Questions

- 虚拟列表在大消息量场景的性能边界是什么？
- React Compiler 编译缓存对复杂组件树的实际收益如何测量？

## Sources

- `chapters/chapter-37-组件架构.md`