---
title: "Forked Agent"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: ["chapter-42-提示建议服务.md"]
related: ["prompt-suggestion", "speculation"]
---

Forked Agent 是 Claude Code 的子 Agent 执行框架，核心优势是复用主线程的 Prompt Cache，避免重复处理系统提示和工具定义。

## CacheSafeParams 类型

```typescript
type CacheSafeParams = {
  systemPrompt: SystemPrompt      // 必须与父请求匹配
  userContext: { [k: string]: string }
  systemContext: { [k: string]: string }
  toolUseContext: ToolUseContext
  forkContextMessages: Message[]  // 父上下文消息
}
```

## 缓存复用约束

关键约束：不发送与主线程不同的 API 参数，否则会破坏缓存。

| 参数 | 处理方式 | 原因 |
|------|----------|------|
| tools | 通过 canUseTool 拒绝 | tools:[] 会破坏缓存 |
| thinking config | 不覆盖 | 是缓存键的一部分 |
| maxOutputTokens | 不设置 | 会间接改变 budget_tokens |

## 保存时机

在 Stop Hooks 中保存：
```typescript
if (querySource === 'repl_main_thread' || querySource === 'sdk') {
  saveCacheSafeParams(createCacheSafeParams(stopHookContext))
}
```

## 执行入口

`runForkedAgent()` 函数执行 Forked Agent：
- `promptMessages` - Fork 专用提示消息
- `cacheSafeParams` - 复用缓存参数
- `canUseTool` - 工具权限控制（建议生成时拒绝所有工具）
- `querySource` - 来源标记
- `forkLabel` - Fork 标签用于日志
- `skipTranscript` - 跳过会话记录
- `skipCacheWrite` - 跳过缓存写入

## 上下文消息传递

通过 `forkContextMessages` 继承主线程对话历史，与 Fork 专用提示拼接形成完整输入上下文。

## Connections

- [Prompt Suggestion](../concepts/prompt-suggestion.md) - 建议生成使用 Forked Agent
- [Speculation](../concepts/speculation.md) - 推测执行使用 Forked Agent

## Open Questions

- Forked Agent 的最大并行数量限制如何设计？
- 缓存失效时 Fork 请求的成本如何控制？

## Sources

- `chapters/chapter-42-提示建议服务.md`