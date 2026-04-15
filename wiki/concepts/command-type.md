---
title: "Command Type"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-05-类型系统基础.md]
related: [../concepts/tool-type.md, ../concepts/taskstate-type.md]
---

Command Type 是 Claude Code 斜杠命令的类型定义，采用联合类型设计 `Command = CommandBase & (PromptCommand | LocalCommand | LocalJSXCommand)`，支持三种命令形态：提示注入、本地执行、React 组件返回。

## 联合类型结构

### 类型定义

```typescript
type Command = CommandBase &
  (PromptCommand | LocalCommand | LocalJSXCommand)
```

### 三种命令类型

| 类型 | 说明 | 执行方式 |
|------|------|----------|
| PromptCommand | 提示命令 | 生成提示内容注入对话 |
| LocalCommand | 本地命令 | 懒加载模块执行逻辑 |
| LocalJSXCommand | JSX 命令 | 懒加载模块返回 React 组件 |

## CommandBase 共享字段

```typescript
type CommandBase = {
  name: string,               // 命令名称
  description: string,        // 命令描述
  aliases?: string[],         // 命令别名
  availability?: string[],    // 可用性限制
  isEnabled?: () => boolean,  // 是否启用
  isHidden?: boolean,         // 是否隐藏
  argumentHint?: string,      // 参数提示
  whenToUse?: string,         // 使用场景
  isMcp?: boolean,            // 是否 MCP 命令
  loadedFrom?: string,        // 加载来源
}
```

## PromptCommand

### 定义

```typescript
type PromptCommand = {
  type: 'prompt',
  progressMessage: string,    // 进度消息
  contentLength: number,      // 内容长度（token 估算）
  getPromptForCommand(args, context): Promise<ContentBlockParam[]>,
  context?: 'inline' | 'fork',  // 执行模式
  paths?: string[],           // 路径匹配模式
}
```

### 执行模式

- **inline**：在当前对话展开，直接注入提示
- **fork**：作为子 Agent 运行，独立上下文

### 路径匹配

`paths` 字段定义技能可见的路径模式，模型触及匹配文件后才显示命令。

## LocalCommand

### 定义

```typescript
type LocalCommand = {
  type: 'local',
  supportsNonInteractive: boolean,
  load: () => Promise<LocalCommandModule>,
}

type LocalCommandModule = {
  call: (args, context) => Promise<LocalCommandResult>
}
```

### 懒加载设计

- `load()` 函数延迟加载模块
- 避免启动时加载所有命令模块

## LocalJSXCommand

### 定义

```typescript
type LocalJSXCommand = {
  type: 'local-jsx',
  load: () => Promise<LocalJSXCommandModule>,
}

type LocalJSXCommandModule = {
  call: (onDone, context, args) => Promise<React.ReactNode>
}
```

### React 组件返回

返回 React 组件用于 UI 渲染，如对话选择器、表单等。

## LocalCommandResult

```typescript
type LocalCommandResult =
  | { type: 'text', value: string }
  | { type: 'compact', compactionResult, displayText }
  | { type: 'skip' }
```

## Connections

- [Tool Type](../concepts/tool-type.md) - 工具类型
- [TaskState Type](../concepts/taskstate-type.md) - 任务状态

## Open Questions

- CommandAvailability 的完整定义
- fork 模式的 Agent 创建流程

## Sources

- `chapters/chapter-05-类型系统基础.md`