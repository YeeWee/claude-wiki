---
title: "其他工具精选"
type: source
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-17-其他工具精选.md]
related: []
---

SkillTool、MCPTool、ConfigTool 和 AskUserQuestionTool 构成了 Claude Code 的"软性交互层"，负责技能执行、MCP 调用、配置管理和用户问答。

## 核心要点

1. **SkillTool**：技能执行，支持 Inline 和 Fork 两种模式
2. **MCPTool**：MCP 协议客户端，动态适配任意 MCP 工具
3. **ConfigTool**：配置读写，支持全局和项目设置
4. **AskUserQuestionTool**：多选项用户问答，支持 Preview 功能

## 工具分层架构

- **硬性操作层**：BashTool、FileReadTool、FileWriteTool、GrepTool
- **软性交互层**：SkillTool、MCPTool、ConfigTool、AskUserQuestionTool

## SkillTool

### 技能来源

| 来源 | 说明 |
|------|------|
| bundled | 内置技能 |
| local | 项目级技能 |
| plugin | 插件提供 |
| mcp | MCP 服务器提供 |

### 权限控制（三层策略）

1. 拒绝规则检查
2. 允许规则检查
3. 安全属性白名单（SAFE_SKILL_PROPERTIES）

### 执行模式

- **Inline**：内容注入当前对话，修改权限上下文允许技能指定的工具
- **Forked**：在独立子 Agent 中执行

## MCPTool

### Schema 设计

使用 `passthrough()` Schema 允许任意输入对象，实际 Schema 由 MCP 服务器定义。

### 与 MCP 客户端集成

实际调用逻辑在 `mcpClient.ts` 中：
1. MCP 客户端读取工具定义
2. 动态创建 MCPTool 实例
3. 注册到工具集合

## ConfigTool

### 支持的设置项

**全局设置**（`~/.claude.json`）：
- theme、editorMode、verbose
- autoCompactEnabled、fileCheckpointingEnabled
- todoFeatureEnabled

**项目设置**（`settings.json`）：
- model、autoMemoryEnabled
- permissions.defaultMode、language

### 执行逻辑

- **GET**：读取自动允许
- **SET**：写入需要确认，包含类型转换、选项验证、异步验证

### 即时 UI 效果

通过 `appStateKey` 机制实现配置变更的立即 UI 效果。

## AskUserQuestionTool

### Schema 设计

- 问题选项：label、description、preview
- 问题：question、header、options（2-4个）、multiSelect
- 唯一性验证：问题文本唯一、选项标签唯一

### Preview 功能

支持展示可视化内容：
- ASCII mockups
- Code snippets
- Diagram variations
- HTML fragments（禁止完整文档、script/style 标签）

### 权限交互

实际用户交互在权限组件中完成，答案回填到 `answers` 字段。

## Agent 工具可用性

**所有 Agent 禁用的工具**：
- TASK_OUTPUT_TOOL_NAME
- EXIT_PLAN_MODE_V2_TOOL_NAME
- ENTER_PLAN_MODE_TOOL_NAME
- AGENT_TOOL_NAME（防递归）
- ASK_USER_QUESTION_TOOL_NAME

**异步 Agent 允许的工具**：包括 SkillTool、文件工具、搜索工具等。

## Connections

- [SkillTool](../entities/skill-tool.md)
- [MCPTool](../entities/mcp-tool.md)
- [ConfigTool](../entities/config-tool.md)
- [AskUserQuestionTool](../entities/ask-user-question-tool.md)
- [MCP协议](../concepts/mcp-protocol.md)

## Open Questions

- 如何优化技能发现机制以处理大量技能？
- Preview 功能的最佳使用场景是什么？

## Sources

- `chapters/chapter-17-其他工具精选.md`