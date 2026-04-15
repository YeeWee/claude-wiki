---
title: "命令系统架构"
type: source
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-18-命令系统架构.md]
related: []
---

命令系统是 Claude Code 与用户交互的核心入口点，采用分层设计：类型层、注册层、路由层。

## 核心要点

1. **三种命令类型**：PromptCommand、LocalCommand、LocalJSXCommand
2. **延迟加载**：通过 `load()` 函数动态导入
3. **来源管理**：统一的 `loadedFrom` 字段追踪
4. **安全控制**：availability 限制用户范围，allowedTools 限制 AI 工具集

## 命令类型分类

### PromptCommand

不直接执行逻辑，而是生成发送给 AI 模型的提示内容。

关键方法：`getPromptForCommand(args, context)` → `ContentBlockParam[]`

典型实现：`/commit` 命令

### LocalCommand

直接在本地执行 TypeScript 代码，不涉及 AI 模型。

典型实现：`/cost`、`/version` 命令

### LocalJSXCommand

返回 React 组件，在终端中渲染交互式 UI。

典型实现：`/help`、`/config`、`/model` 命令

完成回调：`LocalJSXCommandOnDone(result, options)`
- `display`：'skip' | 'system' | 'user'
- `shouldQuery`：是否发送给模型

## CommandBase 通用属性

| 属性 | 说明 |
|------|------|
| availability | 可用性限制 |
| description | 命令描述 |
| isEnabled | 动态启用判断 |
| loadedFrom | 加载来源 |
| aliases | 别名列表 |
| argumentHint | 参数提示 |
| whenToUse | 使用场景 |
| userInvocable | 用户可调用 |
| immediate | 立即执行 |

## 命令注册与路由

### 命令来源优先级

1. Bundled skills
2. Builtin plugin skills
3. Skill directory commands
4. Workflow commands
5. Plugin commands
6. Plugin skills
7. Built-in commands

### 可用性过滤

- **claude-ai**：仅限 claude.ai OAuth 订阅用户
- **console**：仅限直接 API 用户

### 命令查找

1. 命令名称（name）
2. 显示名称（userFacingName()）
3. 别名列表（aliases）

### 远程模式安全

`REMOTE_SAFE_COMMANDS` 定义远程模式可用命令，仅影响本地 TUI 状态。

`BRIDGE_SAFE_COMMANDS` 定义移动端可用命令。

## Skill 加载机制

`createSkillCommand` 函数将 Markdown 文件转换为 PromptCommand：
1. 读取 SKILL.md 文件
2. 解析 YAML frontmatter
3. 替换参数占位符 `${arg}`
4. 执行内嵌 shell 命令 `!`command``
5. 返回处理后的文本

## 设计原则

**类型分离**：AI 驱动、本地执行、UI 渲染三种类型
**延迟加载**：减少启动开销，按需加载
**来源管理**：自动去重和优先级处理
**安全控制**：多层权限过滤

## Connections

- [PromptCommand](../entities/prompt-command.md)
- [LocalCommand](../entities/local-command.md)
- [LocalJSXCommand](../entities/local-jsx-command.md)
- [命令系统](../concepts/command-system.md)
- [Skill加载](../concepts/skill-loading.md)

## Open Questions

- 如何处理命令名称冲突？
- 远程模式的安全边界是否足够？

## Sources

- `chapters/chapter-18-命令系统架构.md`