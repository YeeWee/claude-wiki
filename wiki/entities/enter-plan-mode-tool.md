---
title: "EnterPlanModeTool"
type: entity
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-15-计划模式工具.md]
related: []
---

EnterPlanModeTool 是触发计划模式的入口工具，请求用户批准进入只读探索阶段。

## 核心功能

- 请求用户批准进入计划模式
- 切换权限上下文
- 保存原始模式供恢复

## 配置

| 配置项 | 值 |
|--------|-----|
| name | 'EnterPlanMode' |
| maxResultSizeChars | 100_000 |
| shouldDefer | true |
| isConcurrencySafe | true |
| isReadOnly | true |

## 输入 Schema

无参数。

## 执行逻辑

1. 检查 Agent 上下文（禁止在子 Agent 中使用）
2. 处理模式转换
3. 准备权限上下文（保存 prePlanMode）
4. 更新 AppState 切换到 plan 模式

## 权限上下文准备

`prepareContextForPlanMode` 函数：
- 保存原始模式到 `prePlanMode` 字段
- 支持 auto 模式在计划模式中保持激活
- 从 bypassPermissions 进入时不启用 auto 模式

## Prompt 版本差异

**外部用户版本**：倾向于主动使用，鼓励用于实现任务
**Ant 用户版本**：更加谨慎，仅在真正模糊情况下使用

## 工具结果映射

发送指令：
```
In plan mode, you should:
1. Thoroughly explore the codebase
2. Identify similar features and patterns
3. Consider multiple approaches
4. Use AskUserQuestion if needed
5. Design implementation strategy
6. Use ExitPlanMode to present plan
```

## Connections

- [ExitPlanModeV2Tool](exit-plan-mode-v2-tool.md)
- [计划模式](../concepts/plan-mode.md)
- [权限上下文](../concepts/permission-context.md)

## Open Questions

- 如何避免在简单任务中误触发计划模式？

## Sources

- `chapters/chapter-15-计划模式工具.md`