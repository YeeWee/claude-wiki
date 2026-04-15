---
title: "MCP Skill Builder"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources:
  - ../sources/2026-04-15-chapter-21-技能实战解析.md
related:
  - ../concepts/skill-system.md
---

# MCP Skill Builder

## 概要

MCP Skill Builder 将 MCP 服务器提供的工具转换为 Claude Code 技能，使 MCP 工具能够以技能的形式被调用。

## 注册机制

为避免循环依赖，系统使用写入一次的注册模式：

```typescript
export type MCPSkillBuilders = {
  createSkillCommand: typeof createSkillCommand
  parseSkillFrontmatterFields: typeof parseSkillFrontmatterFields
}

let builders: MCPSkillBuilders | null = null

export function registerMCPSkillBuilders(b: MCPSkillBuilders): void {
  builders = b
}

export function getMCPSkillBuilders(): MCPSkillBuilders {
  if (!builders) {
    throw new Error('MCP skill builders not registered')
  }
  return builders
}
```

## 注册时机

注册在 `loadSkillsDir.ts` 模块初始化时完成：

```typescript
// src/skills/loadSkillsDir.ts (末尾)
registerMCPSkillBuilders({
  createSkillCommand,
  parseSkillFrontmatterFields,
})
```

## 设计目的

这种设计确保：
- `mcpSkillBuilders.ts` 是依赖图的叶子节点
- `mcpSkills.ts` 可以依赖它而不形成循环
- 在 Bun 打包的二进制文件中正确运行

## 安全考虑

MCP 技能来自远程服务器，被视为不可信。系统禁止执行其 Markdown 中的 shell 命令：

```typescript
// createSkillCommand 中的安全处理
if (loadedFrom !== 'mcp') {
  finalContent = await executeShellCommandsInPrompt(finalContent, ...)
}
// MCP 技能跳过 shell 命令执行
```

## Connections

- [Skill System](../concepts/skill-system.md) - 技能的核心概念
- [Bundled Skill](../entities/bundled-skill.md) - 内置技能实体

## Sources

- `src/skills/mcpSkillBuilders.ts`
- `src/skills/loadSkillsDir.ts`