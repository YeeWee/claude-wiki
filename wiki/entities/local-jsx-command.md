---
title: "LocalJSXCommand"
type: entity
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-18-命令系统架构.md]
related: []
---

LocalJSXCommand 返回 React 组件，在终端中渲染交互式 UI，是 Claude Code 终端界面丰富体验的关键。

## 核心特征

- 返回 React 组件
- 使用 Ink 框架渲染
- 通过 onDone 回调关闭 UI

## 类型定义

```typescript
type LocalJSXCommand = {
  type: 'local-jsx'
  load: () => Promise<LocalJSXCommandModule>
}

type LocalJSXCommandModule = {
  call: LocalJSXCommandCall
}

type LocalJSXCommandCall = (
  onDone: LocalJSXCommandOnDone,
  context: ToolUseContext & LocalJSXCommandContext,
  args: string
) => Promise<React.ReactNode>

type LocalJSXCommandOnDone = (
  result?: string,
  options?: {
    display?: 'skip' | 'system' | 'user'
    shouldQuery?: boolean
    metaMessages?: string[]
    nextInput?: string
    submitNextInput?: boolean
  }
) => void
```

## 完成回调参数

| 参数 | 说明 |
|------|------|
| display | 结果显示方式：'user'（默认）、'system'、'skip' |
| shouldQuery | 是否发送给模型 |
| nextInput | 下一个输入 |
| submitNextInput | 自动提交 |

## 典型实现：/help 命令

```typescript
const help = {
  type: 'local-jsx',
  name: 'help',
  description: 'Show help and available commands',
  load: () => import('./help.js')
}

export const call = async (onDone, { options: { commands } }) => {
  return <HelpV2 commands={commands} onClose={onDone} />;
};
```

## 典型实现：/config 命令

```typescript
const config = {
  aliases: ['settings'],
  type: 'local-jsx',
  name: 'config',
  description: 'Open config panel',
  load: () => import('./config.js')
}

export const call = async (onDone, context) => {
  return <Settings onClose={onDone} context={context} defaultTab="Config" />;
};
```

## 动态属性

```typescript
export default {
  type: 'local-jsx',
  name: 'model',
  get description() {
    return `Set the AI model (currently ${renderModelName(getMainLoopModel())})`
  },
  argumentHint: '[model]',
  get immediate() {
    return shouldInferenceConfigCommandBeImmediate()
  },
  load: () => import('./model.js')
}
```

## 执行模式

- 接收 onDone 回调传递给组件
- 返回 React 组件由 Ink 渲染
- 组件内部通过 onClose 触发关闭

## Connections

- [PromptCommand](prompt-command.md)
- [LocalCommand](local-command.md)
- [Ink框架](../concepts/ink-framework.md)

## Open Questions

- 如何处理复杂 UI 的状态管理？

## Sources

- `chapters/chapter-18-命令系统架构.md`