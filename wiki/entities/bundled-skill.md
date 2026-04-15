---
title: "Bundled Skill"
type: entity
created: 2026-04-15
updated: 2026-04-15
sources:
  - ../sources/2026-04-15-chapter-20-技能系统架构.md
  - ../sources/2026-04-15-chapter-21-技能实战解析.md
related:
  - ../concepts/skill-system.md
  - ../concepts/frontmatter.md
---

# Bundled Skill

## 概要

Bundled Skill（内置技能）是 Claude Code 预置的技能，编译进 CLI 二进制文件中，对所有用户可用。这些技能覆盖了配置管理、调试、代码审查等核心功能。

## BundledSkillDefinition 结构

```typescript
export type BundledSkillDefinition = {
  name: string                    // 技能名称
  description: string             // 技能描述
  aliases?: string[]              // 可选别名
  whenToUse?: string              // 自动触发条件说明
  argumentHint?: string           // 参数提示
  allowedTools?: string[]         // 允许的工具列表
  model?: string                  // 可选模型指定
  disableModelInvocation?: boolean // 禁止自动调用
  userInvocable?: boolean         // 是否用户可调用
  isEnabled?: () => boolean       // 动态启用条件
  hooks?: HooksSettings           // 钩子配置
  context?: 'inline' | 'fork'     // 执行上下文
  agent?: string                  // 代理类型
  files?: Record<string, string>  // 附加文件
  getPromptForCommand: (args, context) => Promise<ContentBlockParam[]>
}
```

## 核心内置技能

| 技能名称 | 功能 | 关键特性 |
|---------|------|---------|
| `update-config` | 管理 settings.json 配置 | 动态生成 JSON Schema |
| `debug` | 启用调试日志，诊断问题 | 自动读取日志尾部 |
| `simplify` | 代码多维审查 | 并行启动三个审查代理 |
| `skillify` | 会话转技能 | 通过 AskUserQuestion 采访用户 |
| `keybindings-help` | 键盘快捷键定制 | - |
| `verify` | 验证功能 | - |
| `lorem-ipsum` | 测试内容生成 | - |
| `remember` | 记忆审查 | 仅对特定用户类型可用 |
| `batch` | 批量操作 | - |
| `stuck` | 调试卡住状态 | - |

## 注册机制

内置技能通过 `registerBundledSkill()` 函数程序化注册：

```typescript
export function registerBundledSkill(definition: BundledSkillDefinition): void {
  // 处理内嵌文件：首次调用时提取到磁盘
  const command: Command = {
    type: 'prompt',
    name: definition.name,
    source: 'bundled',
    loadedFrom: 'bundled',
    // ...
  }
  bundledSkills.push(command)
}
```

## 内嵌文件安全提取

内置技能可携带参考文件，首次调用时安全提取到磁盘：
- 进程级 nonce 目录：防止预创建符号链接攻击
- `O_EXCL | O_NOFOLLOW`：禁止覆盖已存在文件
- `0o700/0o600` 模式：仅限所有者访问
- 路径遍历检测：拒绝 `..` 和绝对路径

## Connections

- [技能系统](../concepts/skill-system.md) - 技能的核心概念
- [Frontmatter](../concepts/frontmatter.md) - 元数据定义机制
- [Fork Context](../concepts/fork-context.md) - 子 Agent 执行

## Open Questions

- 内置技能与用户技能的功能边界如何划分？
- 功能标志控制（如 `AGENT_TRIGGERS`）的启用条件？

## Sources

- `src/skills/bundledSkills.ts`
- `src/skills/bundled/index.ts`