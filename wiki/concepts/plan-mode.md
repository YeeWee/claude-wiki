---
title: "计划模式"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-15-计划模式工具.md]
related: []
---

计划模式是 Claude Code 的"规划-执行"工作流机制，在执行复杂任务前进行探索和规划。

## 核心价值

1. **降低风险**：复杂任务先获得用户确认再执行
2. **提高效率**：避免因方向错误导致的返工
3. **增强透明度**：用户参与决策过程
4. **文档化**：计划文件作为实现过程的记录

## 适用场景

- 新功能实现
- 多种实现方案可选
- 多文件修改
- 架构决策
- 需求不明确需探索

## 不适用场景

- 单行修复
- 添加单一函数
- 用户给出详细指令
- 纯研究任务

## 工具支持

| 工具 | 功能 |
|------|------|
| EnterPlanModeTool | 进入计划模式 |
| ExitPlanModeV2Tool | 退出计划模式，提交审批 |

## 执行流程

```
用户请求 → EnterPlanModeTool → 权限确认 → 只读探索
    → 编写计划文件 → ExitPlanModeV2Tool → 用户审批
    → 退出计划模式 → 开始实现
```

## 五阶段工作流

1. **Phase 1: Initial Understanding** - 使用 explore agent 理解请求
2. **Phase 2: Problem Discovery** - 深入理解问题空间
3. **Phase 3: Solution Design** - 收敛解决方案
4. **Phase 4: Final Plan** - 编写计划文件
5. **Phase 5: Present Plan** - 提交审批

## 权限上下文

- `prePlanMode` 保存原始模式
- 支持 auto 模式在计划模式中保持激活
- 退出时恢复原模式

## 计划文件

存储在 `.claude/plans/` 目录：
- 主会话：`{slug}.md`
- 子 Agent：`{slug}-agent-{agentId}.md`

## Teammate 审批

设置 `planModeRequired` 时，计划发送到 team leader mailbox 等待审批。

## Connections

- [EnterPlanModeTool](../entities/enter-plan-mode-tool.md)
- [ExitPlanModeV2Tool](../entities/exit-plan-mode-v2-tool.md)
- [权限上下文](permission-context.md)

## Open Questions

- Interview Phase 的最佳实践是什么？

## Sources

- `chapters/chapter-15-计划模式工具.md`