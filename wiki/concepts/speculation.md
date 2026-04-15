---
title: "Speculation"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: ["chapter-42-提示建议服务.md"]
related: ["forked-agent", "prompt-suggestion"]
---

Speculation（推测执行）是 Claude Code 提示建议服务的高级功能，当用户接受建议时，系统可能已提前执行了预测的操作。

## Overlay 文件隔离

推测执行使用 Copy-on-Write 机制隔离文件操作：
- Overlay 路径：`getClaudeTempDir()/speculation/{pid}/{id}`
- 写入前复制原文件到 overlay
- 写入重定向到 overlay 路径
- 用户拒绝时 overlay 被丢弃

## 执行边界控制

边界类型：
```typescript
type CompletionBoundary =
  | { type: 'bash'; command: string }
  | { type: 'edit'; toolName: string; filePath: string }
  | { type: 'denied_tool'; toolName: string; detail: string }
  | { type: 'complete'; outputTokens: number }
```

边界触发条件：
- 文件编辑：需要权限确认（除非 acceptEdits/bypassPermissions 模式）
- Bash 命令：非只读命令停止
- 权限拒绝：用户拒绝工具时停止

## 工具分类

```typescript
const WRITE_TOOLS = new Set(['Edit', 'Write', 'NotebookEdit'])
const SAFE_READ_ONLY_TOOLS = new Set(['Read', 'Glob', 'Grep', 'ToolSearch', 'LSP', 'TaskGet', 'TaskList'])
```

## Pipeline 建议

推测完成时生成下一个建议，形成连续预测链：
```typescript
void generatePipelinedSuggestion(
  contextRef.current,
  suggestionText,
  messagesRef.current,  // 包含已推测的消息
  setAppState,
  abortController,
)
```

## 接受/拒绝流程

- **acceptSpeculation()**：合并 overlay 文件到实际路径
- **abortSpeculation()**：丢弃 overlay，中止执行

## Connections

- [Forked Agent](../concepts/forked-agent.md) - 推测执行使用 Forked Agent
- [Prompt Suggestion](../concepts/prompt-suggestion.md) - 建议触发推测执行

## Open Questions

- Overlay 文件的清理时机如何设计？
- Pipeline 连续预测的深度上限如何确定？
- 推测执行的资源消耗如何监控？

## Sources

- `chapters/chapter-42-提示建议服务.md`