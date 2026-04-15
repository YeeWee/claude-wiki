---
title: "LocalCommand"
type: entity
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-18-命令系统架构.md]
related: []
---

LocalCommand 直接在本地执行 TypeScript 代码，不涉及 AI 模型，适用于简单确定性操作。

## 核心特征

- 本地代码执行
- 不涉及 AI 模型
- 延迟加载实现代码

## 类型定义

```typescript
type LocalCommand = {
  type: 'local'
  supportsNonInteractive: boolean   // 支持非交互模式
  load: () => Promise<LocalCommandModule>
}

type LocalCommandModule = {
  call: (args: string, context: LocalJSXCommandContext) => Promise<LocalCommandResult>
}

type LocalCommandResult =
  | { type: 'text'; value: string }
  | { type: 'compact'; compactionResult: CompactionResult }
  | { type: 'skip' }
```

## 典型实现：/cost 命令

```typescript
const cost = {
  type: 'local',
  name: 'cost',
  description: 'Show the total cost and duration',
  supportsNonInteractive: true,
  load: () => import('./cost.js')
}
```

## 典型实现：/version 命令

```typescript
const version = {
  type: 'local',
  name: 'version',
  description: 'Print the version...',
  isEnabled: () => process.env.USER_TYPE === 'ant',
  supportsNonInteractive: true,
  load: () => Promise.resolve({ call })
}
```

使用 `Promise.resolve` 直接返回模块（无需动态导入）。

## 延迟加载设计

```typescript
load: () => import('./clear.js')
```

好处：
- 减少启动时间
- 降低内存占用
- 支持条件编译

## 动态属性

```typescript
get isHidden() {
  if (process.env.USER_TYPE === 'ant') return false
  return isClaudeAISubscriber()
}
```

通过 getter 实现动态属性。

## Connections

- [PromptCommand](prompt-command.md)
- [LocalJSXCommand](local-jsx-command.md)

## Open Questions

- 如何处理执行失败的错误恢复？

## Sources

- `chapters/chapter-18-命令系统架构.md`