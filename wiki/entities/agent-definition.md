---
title: "Agent Definition"
type: entity
created: 2026-04-15
updated: 2026-04-15
sources: [chapters/chapter-12-agenttool多agent机制.md]
related: [../entities/agent-tool.md]
---

# Agent Definition

Agent Definition 定义了 Agent 的类型、配置和行为，分为三类。

## 类型定义

### BuiltInAgentDefinition

内置 Agent，定义在 `src/tools/AgentTool/builtInAgents.ts`：
- source: 'built-in'
- baseDir: 'built-in'
- getSystemPrompt: (params) => string

### CustomAgentDefinition

用户/项目/策略设置定义的 Agent：
- source: SettingSource
- filename?: string
- baseDir?: string

### PluginAgentDefinition

插件提供的 Agent：
- source: 'plugin'
- plugin: string

## 共享字段（BaseAgentDefinition）

| 字段 | 类型 | 说明 |
|------|------|------|
| `agentType` | `string` | Agent 类型标识 |
| `whenToUse` | `string` | 使用场景描述 |
| `disallowedTools` | `string[]?` | 禁用工具列表 |
| `model` | `string?` | 模型选择 |
| `omitClaudeMd` | `boolean?` | 是否省略 CLAUDE.md |
| `permissionMode` | `string?` | 权限模式 |
| `maxTurns` | `number?` | 最大轮数 |
| `isolation` | `string?` | 隔离模式 |

## Connections

- [AgentTool](../entities/agent-tool.md) - 使用定义启动 Agent

## Sources

- [第十二章：AgentTool 多 Agent 机制](../sources/2026-04-15-agenttool多agent机制.md)