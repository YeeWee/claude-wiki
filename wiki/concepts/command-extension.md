---
title: "命令扩展"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-48-可扩展性设计.md]
related: []
---

命令扩展是 Claude Code 可扩展性设计的中等复杂度方式，添加新的斜杠命令。

## 命令类型选择

| 类型 | 执行方式 | 适用场景 |
|------|---------|---------|
| PromptCommand | AI 模型执行 | 需要 AI 判断的复杂任务 |
| LocalCommand | TypeScript 直接执行 | 确定性操作、信息查询 |
| LocalJSXCommand | React 组件渲染 | 交互式 UI、配置面板 |

## 注册流程

| 步骤 | 操作 |
|------|------|
| 1 | 创建命令目录 `commands/myCommand/` |
| 2 | 创建 index.ts 导出命令定义 |
| 3 | 创建实现文件 |
| 4 | 在 commands.ts 中导入并注册 |

## Connections

- 无

## Open Questions

- 如何处理命令冲突？

## Sources

- `chapters/chapter-48-可扩展性设计.md`