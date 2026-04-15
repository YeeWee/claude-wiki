---
title: "Skill System"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources:
  - ../sources/2026-04-15-chapter-20-技能系统架构.md
  - ../sources/2026-04-15-chapter-21-技能实战解析.md
related:
  - ../entities/bundled-skill.md
  - ../concepts/frontmatter.md
  - ../concepts/fork-context.md
---

# Skill System

## 概要

技能系统是 Claude Code 提供专业化能力的核心机制。通过技能，用户可以扩展 AI 能力边界、定义自动化流程、定制模型行为、隔离执行上下文。

## 技能来源

| 来源类型 | 说明 | 加载位置 |
|---------|------|---------|
| Bundled | 内置技能，编译进 CLI | `src/skills/bundled/` |
| User | 用户全局技能 | `~/.claude/skills/` |
| Project | 项目级技能 | `.claude/skills/` |
| Managed | 企业策略管理 | 策略配置目录 |
| MCP | MCP 服务器提供 | MCP 连接 |

## 技能文件结构

推荐使用目录格式：
```
.claude/skills/
├── my-skill/
│   └── SKILL.md        # 必须命名为 SKILL.md
│   └── examples/       # 附加文件（可选）
│       └── demo.md
```

## 加载层次

技能加载按优先级分层（从高到低）：
1. Managed Skills：企业策略目录
2. User Skills：用户全局配置
3. Project Skills：项目目录
4. Additional Dirs：`--add-dir` 指定
5. Legacy Commands：旧版 `.claude/commands`
6. Bundled Skills：CLI 内置技能

## 条件技能

具有 `paths` frontmatter 的技能在匹配路径时才激活：
```yaml
paths:
  - "src/**/*.ts"
  - "tests/**/*.ts"
```

使用 `ignore` 库进行 gitignore 风格的路径匹配。

## 动态发现

当操作深层目录中的文件时，系统向上遍历发现沿途的技能目录：
```typescript
while (currentDir.startsWith(resolvedCwd + pathSep)) {
  const skillDir = join(currentDir, '.claude', 'skills')
  // 检查目录是否存在，排除 gitignore 目录
}
```

## 命令表示

技能在系统中以 `Command` 类型表示：
```typescript
type Command = {
  type: 'prompt'
  name: string
  description: string
  allowedTools?: string[]
  model?: string
  context?: 'inline' | 'fork'
  hooks?: HooksSettings
  getPromptForCommand: (args, context) => Promise<ContentBlockParam[]>
}
```

## Connections

- [Bundled Skill](../entities/bundled-skill.md) - 内置技能实体
- [Frontmatter](../concepts/frontmatter.md) - 元数据定义
- [Fork Context](../concepts/fork-context.md) - 子 Agent 执行
- [MCP Skill Builder](../concepts/mcp-skill-builder.md) - MCP 工具转换

## Sources

- `src/skills/loadSkillsDir.ts`
- `src/skills/bundledSkills.ts`