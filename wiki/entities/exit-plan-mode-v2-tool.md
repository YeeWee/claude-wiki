---
title: "ExitPlanModeV2Tool"
type: entity
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-15-计划模式工具.md]
related: []
---

ExitPlanModeV2Tool 是计划模式的出口工具，提交计划给用户审批并退出计划模式。

## 核心功能

- 提交计划给用户审批
- 处理 teammate leader 审批流程
- 恢复原权限模式
- 支持语义化权限请求

## 输入 Schema

```typescript
{
  allowedPrompts?: {
    tool: 'Bash',
    prompt: string  // 语义描述，如 "run tests"
  }[]
}
```

## 输出 Schema

```typescript
{
  plan: string | null,
  isAgent: boolean,
  filePath?: string,
  hasTaskTool?: boolean,
  planWasEdited?: boolean,
  awaitingLeaderApproval?: boolean,
  requestId?: string
}
```

## 权限检查

- Teammate：跳过权限 UI
- 普通用户：需要确认 "Exit plan mode?"

## 执行逻辑

1. 获取计划内容（从输入或磁盘）
2. Teammate 模式：发送审批请求到 leader mailbox
3. 普通用户：更新 AppState 退出计划模式
4. 恢复到 prePlanMode 保存的模式
5. 处理 auto-mode 断路器
6. 恢复权限规则

## Teammate Leader 审批

```typescript
{
  type: 'plan_approval_request',
  from: agentName,
  timestamp: string,
  planFilePath: string,
  planContent: string,
  requestId: string
}
```

发送到 `team-lead` mailbox，等待审批响应。

## 工具结果映射

**Leader 审批等待**：
```
Your plan has been submitted to the team lead for approval.
Plan file: ${filePath}
Request ID: ${requestId}
```

**用户批准**：
```
User has approved your plan. You can now start coding.
Your plan has been saved to: ${filePath}
## Approved Plan:
${plan}
```

## Prompt 关键要点

- 工具不接受计划内容参数，从文件读取
- 仅用于实现任务，不用于纯研究
- 有未解决问题时先使用 AskUserQuestion

## Connections

- [EnterPlanModeTool](enter-plan-mode-tool.md)
- [计划模式](../concepts/plan-mode.md)
- [Teammate协作](../concepts/teammate-collaboration.md)

## Open Questions

- 如何处理计划审批超时？

## Sources

- `chapters/chapter-15-计划模式工具.md`