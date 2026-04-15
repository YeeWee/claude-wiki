---
title: "AskUserQuestionTool"
type: entity
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-17-其他工具精选.md]
related: []
---

AskUserQuestionTool 实现多选项用户问答交互，支持结构化的偏好收集。

## 核心功能

- 多选项问答
- Preview 功能展示可视化内容
- 唯一性验证

## 配置

| 配置项 | 值 |
|--------|-----|
| name | 'AskUserQuestion' |
| maxResultSizeChars | 100_000 |
| shouldDefer | true |
| isConcurrencySafe | true |
| isReadOnly | true |
| requiresUserInteraction | true |

## 输入 Schema

```typescript
{
  questions: [{
    question: string,         // 问题文本
    header: string,           // 短标签（最多 12 字符）
    options: [{
      label: string,          // 选项显示文本
      description: string,    // 选项说明
      preview?: string        // 可选预览内容
    }],                       // 2-4 个选项
    multiSelect?: boolean     // 允许多选
  }]
}
```

## 唯一性验证

- 问题文本必须唯一
- 每个问题内选项标签必须唯一

## Preview 功能

支持展示可视化内容：
- ASCII mockups
- Code snippets
- Diagram variations
- HTML fragments

**HTML 验证**：
- 禁止完整 HTML 文档
- 禁止 script/style 标签
- 必须包含 HTML 标签

## 执行逻辑

```typescript
async call({ questions, answers = {}, annotations }) {
  return {
    data: { questions, answers, ...(annotations && { annotations }) }
  }
}
```

实际用户交互在权限组件中完成。

## 结果格式化

```
User has answered your questions:
"question text"="answer"
selected preview: {preview}
user notes: {notes}
```

## Prompt 定义

使用场景：
1. 收集用户偏好
2. 澄清模糊指令
3. 获取实现决策
4. 提供方向选择

**计划模式注意**：在计划模式中使用此工具澄清需求，不要用它问"我的计划准备好了吗"。

## Connections

- [用户交互](../concepts/user-interaction.md)

## Open Questions

- 如何处理用户选择"Other"的情况？

## Sources

- `chapters/chapter-17-其他工具精选.md`