---
title: "PromptCommand"
type: entity
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-18-命令系统架构.md]
related: []
---

PromptCommand 是最特殊的命令类型——不直接执行逻辑，而是生成发送给 AI 模型的提示内容。

## 核心特征

- 生成 AI 提示而非执行代码
- 通过 `allowedTools` 限制模型可使用的工具
- 核心方法：`getPromptForCommand(args, context)`

## 类型定义

```typescript
type PromptCommand = {
  type: 'prompt'
  progressMessage: string          // 进度提示
  contentLength: number            // 内容长度（token 估算）
  argNames?: string[]              // 参数名称
  allowedTools?: string[]          // 允许的工具列表
  model?: string                   // 指定模型
  source: SettingSource | 'builtin' | 'mcp' | 'plugin' | 'bundled'
  hooks?: HooksSettings            // 钩子配置
  context?: 'inline' | 'fork'      // 执行上下文
  agent?: string                   // Agent 类型
  getPromptForCommand(args, context): Promise<ContentBlockParam[]>
}
```

## 典型实现：/commit 命令

```typescript
const command = {
  type: 'prompt',
  name: 'commit',
  description: 'Create a git commit',
  allowedTools: [
    'Bash(git add:*)',
    'Bash(git status:*)',
    'Bash(git commit:*)'
  ],
  contentLength: 0,
  progressMessage: 'creating commit',
  source: 'builtin',
  async getPromptForCommand(_args, context) {
    // 生成包含 git 状态、安全协议、任务指令的提示
    return [{ type: 'text', text: finalContent }]
  }
}
```

## 工具权限控制

`allowedTools` 限制模型可使用的工具，确保模型只能执行特定类别命令。

## Skill 加载机制

`createSkillCommand` 将 Markdown 文件转换为 PromptCommand：
1. 读取 SKILL.md 文件
2. 解析 YAML frontmatter
3. 替换参数占位符
4. 执行内嵌 shell 命令
5. 返回处理后的文本

## Connections

- [LocalCommand](local-command.md)
- [LocalJSXCommand](local-jsx-command.md)
- [SkillTool](skill-tool.md)

## Open Questions

- 如何优化提示生成以减少 token 使用？

## Sources

- `chapters/chapter-18-命令系统架构.md`