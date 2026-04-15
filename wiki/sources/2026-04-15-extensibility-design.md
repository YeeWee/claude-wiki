---
title: "可扩展性设计"
type: source
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-48-可扩展性设计.md]
related: [../concepts/tool-extension.md, ../concepts/command-extension.md, ../concepts/skill-extension.md, ../concepts/mcp-extension.md]
---

Claude Code 的可扩展性设计是其作为"agentic coding tool"的核心优势。通过统一的扩展架构，用户可以从多个维度增强 CLI 的能力。

## 四种扩展方式

| 扩展方式 | 定义位置 | 复杂度 | 适用场景 | 特权级别 |
|---------|---------|--------|---------|---------|
| **工具扩展** | TypeScript 代码 | 高 | 新操作能力、复杂逻辑 | 编译时 |
| **命令扩展** | TypeScript 代码 | 中 | 用户交互入口、UI 组件 | 编译时 |
| **技能扩展** | Markdown 文件 | 低 | 领域知识注入、提示模板 | 运行时 |
| **MCP 扩展** | JSON 配置 + 外部服务 | 可变 | 外部系统集成、动态发现 | 运行时 |

## 工具扩展核心要素

Tool 接口核心定义：

- 身份标识：name, aliases
- 输入验证：inputSchema, inputJSONSchema
- 执行方法：call(), description()
- 权限控制：isEnabled(), isConcurrencySafe(), isReadOnly(), isDestructive()
- UI 渲染：renderToolUseMessage(), renderToolResultMessage()

## 命令类型选择

| 类型 | 执行方式 | 适用场景 | 示例 |
|------|---------|---------|------|
| `PromptCommand` | AI 模型执行 | 需要 AI 判断的复杂任务 | `/commit`, `/review` |
| `LocalCommand` | TypeScript 直接执行 | 确定性操作、信息查询 | `/cost`, `/version` |
| `LocalJSXCommand` | React 组件渲染 | 交互式 UI、配置面板 | `/config`, `/help` |

## 技能扩展 Frontmatter 字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `name` | string | 技能名称 |
| `description` | string | 技能描述 |
| `when_to_use` | string | 使用场景提示 |
| `allowed-tools` | string[] | 允许工具列表 |
| `context` | `inline` \| `fork` | 执行上下文 |
| `paths` | string[] | 条件激活路径 |

## MCP 配置优先级

| 来源 | 优先级 | 配置文件 |
|------|--------|---------|
| Enterprise | 最高 | 企业策略配置 |
| Local | 高 | 项目私有配置 |
| Project | 中 | `.mcp.json` |
| User | 低 | `~/.claude/settings.json` |
| Plugin | 最低 | 插件内嵌配置 |

## Connections

- [工具扩展](../concepts/tool-extension.md)
- [命令扩展](../concepts/command-extension.md)
- [技能扩展](../concepts/skill-extension.md)
- [MCP 扩展](../concepts/mcp-extension.md)

## Open Questions

- 如何处理扩展之间的冲突？
- 扩展的安全边界如何界定？

## Sources

- `chapters/chapter-48-可扩展性设计.md`