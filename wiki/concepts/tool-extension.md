---
title: "工具扩展"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-48-可扩展性设计.md]
related: [../concepts/tool-extensibility.md, ../concepts/fail-closed-principle.md]
---

工具扩展是 Claude Code 可扩展性设计的高复杂度方式，通过 TypeScript 代码添加新操作能力。

## 适用场景

- 新操作能力
- 复杂逻辑处理
- 需要完整权限控制

## 注册流程

| 步骤 | 操作 |
|------|------|
| 1 | 创建 ToolDef 并导出 |
| 2 | 在 tools.ts 中添加 import |
| 3 | 在 getAllBaseTools() 中添加 |
| 4 | 测试验证 |

## 条件加载机制

```typescript
// Feature Flag 控制
const SleepTool = feature('PROACTIVE') ? require('./SleepTool.js') : null

// 环境变量控制
const REPLTool = process.env.USER_TYPE === 'ant' ? require('./REPLTool.js') : null

// 运行时条件控制
...(isWorktreeModeEnabled() ? [EnterWorktreeTool] : [])
```

## Connections

- [Tool-based Extensibility](../concepts/tool-extensibility.md)
- [Fail-Closed Principle](../concepts/fail-closed-principle.md)

## Open Questions

- 工具扩展的安全边界？

## Sources

- `chapters/chapter-48-可扩展性设计.md`