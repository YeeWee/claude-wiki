---
title: "计划模式工具"
type: source
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-15-计划模式工具.md]
related: []
---

计划模式是 Claude Code 的"规划-执行"工作流机制，在执行复杂任务前进行充分的探索和规划，获得用户确认后再动手。

## 核心要点

1. **降低风险**：复杂任务先获得用户确认再执行
2. **权限上下文切换**：进入计划模式时保存原始模式，退出时恢复
3. **计划文件管理**：通过 slug 生成唯一路径，支持会话恢复
4. **Teammate 审批流程**：leader 通过 mailbox 审批 teammate 的计划

## 工具概览

| 工具 | 功能 | 特性 |
|------|------|------|
| EnterPlanModeTool | 进入计划模式 | 请求用户批准，切换权限上下文 |
| ExitPlanModeV2Tool | 退出计划模式 | 提交计划审批，恢复原模式 |

## 使用场景

**适用场景**：
- 新功能实现
- 多种实现方案可选
- 多文件修改
- 架构决策
- 需求不明确需探索

**不适用场景**：
- 单行修复
- 添加单一函数
- 用户给出详细指令
- 纯研究任务

## 执行流程

```
用户请求 → EnterPlanModeTool → 权限确认 → 只读探索阶段
    → 编写计划文件 → ExitPlanModeV2Tool → 用户审批
    → 退出计划模式 → 开始实现
```

## 权限上下文准备

`prepareContextForPlanMode` 函数处理模式切换：
- `prePlanMode` 字段保存原始模式
- 支持 auto 模式在计划模式中保持激活
- 从 bypassPermissions 模式进入时不启用 auto 模式

## 计划文件管理

计划文件存储在 `.claude/plans/` 目录：
- 主会话：`{slug}.md`
- 子 Agent：`{slug}-agent-{agentId}.md`

Slug 使用单词组合生成，确保可读性和唯一性。

## 五阶段工作流

1. **Phase 1: Initial Understanding** - 使用 explore agent 理解请求
2. **Phase 2: Problem Discovery** - 深入理解问题空间
3. **Phase 3: Solution Design** - 收敛解决方案
4. **Phase 4: Final Plan** - 编写计划文件
5. **Phase 5: Present Plan** - 提交审批

## Teammate 审批流程

对于设置了 `planModeRequired` 的 teammate：
1. 计划发送到 team leader mailbox
2. Leader 审批或拒绝
3. 响应发送到 teammate inbox

## Connections

- [EnterPlanModeTool](../entities/enter-plan-mode-tool.md)
- [ExitPlanModeV2Tool](../entities/exit-plan-mode-tool.md)
- [计划模式](../concepts/plan-mode.md)
- [权限上下文](../concepts/permission-context.md)

## Open Questions

- Interview Phase 的最佳实践是什么？
- 计划文件结构实验的结果如何？

## Sources

- `chapters/chapter-15-计划模式工具.md`