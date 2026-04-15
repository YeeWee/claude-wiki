---
title: "Tool-based Extensibility"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-46-架构设计原则总结.md, chapter-48-可扩展性设计.md]
related: [../concepts/tool-extension.md, ../concepts/fail-closed-principle.md]
---

Tool-based Extensibility 是 Claude Code 的核心设计理念："一切皆工具"。

## 统一 Tool 接口

无论是文件操作、网络请求、MCP 服务还是技能调用，都通过统一的 `Tool` 接口实现。

## 接口核心要素

```typescript
type Tool = {
  name: string                    // 工具唯一名称
  inputSchema: Input              // Zod 输入验证
  call(args, context)             // 核心执行方法
  checkPermissions()              // 权限检查
  isConcurrencySafe()             // 是否可并发
  isReadOnly()                    // 是否只读
  renderToolUseMessage()          // UI 渲染
}
```

## 优势

| 优势 | 说明 |
|------|------|
| 权限一致性 | 所有操作通过相同权限系统检查 |
| 并发安全性 | 工具可声明是否安全并发执行 |
| UI 统一性 | 工具执行过程有统一渲染机制 |
| MCP 集成 | MCP 工具自动适配内置工具接口 |

## 工具类型体系

- 内置工具（Built-in Tools）
- MCP 工具（MCP Tools）
- 技能工具（Skill Tools）

## Connections

- [工具扩展](../concepts/tool-extension.md)
- [Fail-Closed Principle](../concepts/fail-closed-principle.md)

## Open Questions

- 工具接口的演进方向？

## Sources

- `chapters/chapter-46-架构设计原则总结.md`
- `chapters/chapter-48-可扩展性设计.md`