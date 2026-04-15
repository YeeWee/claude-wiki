---
title: "技能扩展"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-48-可扩展性设计.md]
related: []
---

技能扩展是 Claude Code 可扩展性设计的低复杂度方式，通过 Markdown 文件注入领域知识。

## 适用场景

- 领域知识注入
- 提示模板
- 工作流程封装

## Frontmatter 字段

| 字段 | 类型 | 说明 |
|------|------|------|
| name | string | 技能名称 |
| description | string | 技能描述 |
| when_to_use | string | 使用场景提示 |
| allowed-tools | string[] | 允许工具列表 |
| context | inline \| fork | 执行上下文 |
| paths | string[] | 条件激活路径 |

## 加载来源优先级

| 来源 | 位置 | 优先级 |
|------|------|--------|
| Managed | 企业策略目录 | 最高 |
| User | ~/.claude/skills | 高 |
| Project | .claude/skills | 中 |
| Additional | --add-dir | 低 |
| Legacy | .claude/commands | 最低 |

## 条件技能激活

通过 `paths` 字段定义条件技能，当用户操作匹配路径时自动激活。

## Connections

- 无

## Open Questions

- 技能模板如何标准化？

## Sources

- `chapters/chapter-48-可扩展性设计.md`