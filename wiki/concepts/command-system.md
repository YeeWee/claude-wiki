---
title: "命令系统"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-18-命令系统架构.md]
related: []
---

命令系统是 Claude Code 与用户交互的核心入口点，采用分层设计：类型层、注册层、路由层。

## 三种命令类型

| 类型 | 说明 | 执行方式 |
|------|------|----------|
| PromptCommand | AI 驱动 | 生成提示发送给模型 |
| LocalCommand | 本地执行 | TypeScript 代码执行 |
| LocalJSXCommand | UI 渲染 | React 组件渲染 |

## CommandBase 通用属性

| 属性 | 说明 |
|------|------|
| name | 命令名称 |
| description | 命令描述 |
| availability | 可用性限制 |
| isEnabled | 动态启用判断 |
| loadedFrom | 加载来源 |
| aliases | 别名列表 |

## 命令来源优先级

1. Bundled skills
2. Builtin plugin skills
3. Skill directory commands
4. Workflow commands
5. Plugin commands
6. Plugin skills
7. Built-in commands

## 延迟加载

所有命令通过 `load()` 函数动态导入：
- 减少启动时间
- 降低内存占用
- 支持条件编译

## 可用性过滤

- **claude-ai**：仅限 claude.ai OAuth 订阅用户
- **console**：仅限直接 API 用户

## 命令查找

匹配顺序：
1. 命令名称（name）
2. 显示名称（userFacingName()）
3. 别名列表（aliases）

## 远程模式安全

`REMOTE_SAFE_COMMANDS`：仅影响本地 TUI 状态
`BRIDGE_SAFE_COMMANDS`：移动端可用命令

## Skill 加载

`createSkillCommand` 将 Markdown 文件转换为 PromptCommand：
1. 读取 SKILL.md
2. 解析 YAML frontmatter
3. 替换参数占位符
4. 执行内嵌 shell 命令
5. 返回处理后的文本

## Connections

- [PromptCommand](../entities/prompt-command.md)
- [LocalCommand](../entities/local-command.md)
- [LocalJSXCommand](../entities/local-jsx-command.md)
- [SkillTool](../entities/skill-tool.md)

## Open Questions

- 如何处理命令名称冲突？

## Sources

- `chapters/chapter-18-命令系统架构.md`