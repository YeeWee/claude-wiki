---
title: "SkillTool"
type: entity
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-17-其他工具精选.md]
related: []
---

SkillTool 是 Claude Code 中用于调用技能（Skills）的核心工具，支持斜杠命令执行。

## 核心功能

- 技能发现和验证
- Inline 和 Fork 两种执行模式
- 三层权限控制策略

## 配置

| 配置项 | 值 |
|--------|-----|
| name | 'Skill' |
| maxResultSizeChars | 100_000 |

## 输入 Schema

```typescript
{
  skill: string,     // 技能名称，如 "commit", "review-pr"
  args?: string      // 可选参数
}
```

## 输出 Schema

**Inline 模式**：
```typescript
{
  success: boolean,
  commandName: string,
  allowedTools?: string[],
  model?: string,
  status: 'inline'
}
```

**Forked 模式**：
```typescript
{
  success: boolean,
  commandName: string,
  status: 'forked',
  agentId: string,
  result: string
}
```

## 技能来源

| 来源 | 说明 |
|------|------|
| bundled | 内置技能 |
| local | 项目级技能 |
| plugin | 插件提供 |
| mcp | MCP 服务器提供 |

## 输入验证

1. 格式检查：非空字符串
2. 规范化：移除前导斜杠
3. 命令存在检查
4. 类型检查：必须是 prompt 类型

## 权限控制（三层策略）

1. **拒绝规则检查**：匹配则 deny
2. **允许规则检查**：匹配则 allow
3. **安全属性白名单**：SAFE_SKILL_PROPERTIES

## 执行模式

### Inline 模式

- 内容注入当前对话
- 修改权限上下文允许技能指定的工具
- 返回技能元数据

### Forked 模式

- 在独立子 Agent 中执行
- 返回执行结果

## Prompt 生成

技能列表通过 `formatCommandsWithinBudget()` 生成，受字符预算限制（1%上下文窗口）。

## Connections

- [PromptCommand](prompt-command.md)
- [技能系统](../concepts/skill-system.md)

## Open Questions

- 如何处理技能执行超时？

## Sources

- `chapters/chapter-17-其他工具精选.md`