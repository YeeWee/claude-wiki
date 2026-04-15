---
title: "权限上下文"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-15-计划模式工具.md]
related: []
---

权限上下文是 Claude Code 权限系统的状态管理机制，支持模式切换和恢复。

## 核心字段

| 字段 | 说明 |
|------|------|
| mode | 当前权限模式 |
| prePlanMode | 进入计划模式前的原始模式 |
| strippedDangerousRules | 剥离的危险权限规则 |

## 模式类型

- default
- plan
- auto
- bypassPermissions

## 模式切换

### 进入计划模式

`prepareContextForPlanMode` 函数：
1. 保存原始模式到 `prePlanMode`
2. 设置 mode 为 'plan'
3. 可能启用 auto 模式（如果配置）

### 退出计划模式

恢复到 `prePlanMode` 保存的模式：
1. 检查 auto-mode 断路器
2. 恢复剥离的危险权限
3. 清除 prePlanMode 字段

## auto-mode 断路器

从 auto 模式退出时，检查 `isAutoModeGateEnabled()`：
- 启用时恢复 auto 模式
- 禁用时恢复 default 模式

## 危险权限剥离

进入 auto 模式时可能剥离危险权限：
- `stripDangerousPermissionsForAutoMode`
- `restoreDangerousPermissions`

## Connections

- [计划模式](plan-mode.md)
- [权限系统](permission-system.md)

## Open Questions

- 如何处理权限恢复失败？

## Sources

- `chapters/chapter-15-计划模式工具.md`