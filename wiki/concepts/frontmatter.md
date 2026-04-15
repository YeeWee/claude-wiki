---
title: "Frontmatter"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources:
  - ../sources/2026-04-15-chapter-20-技能系统架构.md
related:
  - ../concepts/skill-system.md
  - ../entities/bundled-skill.md
---

# Frontmatter

## 概要

Frontmatter 是技能文件中使用的 YAML 元数据定义机制，位于 Markdown 文件的顶部，用于定义技能的名称、描述、行为配置等信息。

## 示例

```yaml
---
name: code-review
description: 执行代码审查流程
allowed-tools:
  - Read
  - Bash(git:*)
when_to_use: 当用户要求审查代码时调用
argument-hint: "[target branch]"
arguments:
  - target
context: fork
effort: high
paths:
  - "src/**/*.ts"
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "echo '即将执行命令'"
---
```

## 字段详解

| 字段名 | 类型 | 说明 |
|--------|------|------|
| `name` | `string` | 显示名称 |
| `description` | `string` | 技能描述 |
| `when_to_use` | `string` | 自动触发条件说明 |
| `allowed-tools` | `string[]` | 允许使用的工具列表 |
| `argument-hint` | `string` | 参数提示文本 |
| `arguments` | `string[]` | 命名参数列表 |
| `model` | `string` | 专属模型（"inherit" 继承默认） |
| `context` | `'inline' \| 'fork'` | 执行上下文类型 |
| `agent` | `string` | Fork 时使用的 Agent 类型 |
| `effort` | `string \| number` | 努力级别 |
| `user-invocable` | `boolean` | 是否用户可调用 |
| `disable-model-invocation` | `boolean` | 禁止模型自动调用 |
| `paths` | `string[]` | 路径匹配模式（条件技能） |
| `hooks` | `HooksSettings` | 注册的钩子配置 |
| `shell` | `string \| object` | Shell 执行配置 |

## 解析流程

`parseSkillFrontmatterFields()` 函数负责：
1. 验证并转换各字段类型
2. 处理默认值逻辑
3. 解析复杂字段（如 `model`、`hooks`）
4. 验证 `effort` 值的有效性

## 关键设计点

**`when_to_use` 字段**：对自动调用至关重要，帮助模型识别调用时机。

**`allowed-tools` 模式**：推荐使用精确模式如 `Bash(git:*)` 而非 `Bash`。

**`context: 'fork'`**：用于不需要用户中途干预的自包含任务。

## Connections

- [技能系统](../concepts/skill-system.md) - 技能的核心概念
- [Fork Context](../concepts/fork-context.md) - 子 Agent 执行
- [Hook 系统](../concepts/hook-system.md) - 钩子配置

## Sources

- `src/skills/loadSkillsDir.ts`