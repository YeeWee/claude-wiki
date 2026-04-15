---
title: "代码诊断系统"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: ["chapter-27-lsp服务.md"]
related: []
---

代码诊断系统是 Claude Code 通过 LSP 实现的代码智能功能核心组件。

## 诊断注册机制

遵循 AsyncHookRegistry 模式：
- `registerPendingLSPDiagnostic`: 存储异步诊断
- `checkForLSPDiagnostics`: 检索待处理诊断
- 诊断作为 Attachment 交付到对话

## 容量限制

| 限制 | 值 |
|------|-----|
| MAX_DIAGNOSTICS_PER_FILE | 10 |
| MAX_TOTAL_DIAGNOSTICS | 30 |
| MAX_DELIVERED_FILES | 500 |

## 去重机制

跨轮次去重防止重复交付：
- LRU 缓存 `deliveredDiagnostics`
- 诊断键生成：message + severity + range + source + code
- 已交付诊断跳过

## 严重程度映射

LSP → Claude 格式：
- 1 → Error
- 2 → Warning
- 3 → Info
- 4 → Hint
- default → Error

## 通知处理

`registerLSPNotificationHandlers` 注册处理器：
- 验证参数结构
- 转换 LSP 诊断格式
- 注册诊断
- 跟踪连续失败（超过 3 次记录警告）

## 诊断流程

1. LSP 服务器发送 publishDiagnostics 通知
2. 处理器验证并格式化
3. 注册到 pendingDiagnostics
4. 查询时检索并去重
5. 作为 Attachment 交付

## Connections

- [LSP协议](../concepts/lsp-protocol.md)
- [Language Server](../entities/language-server.md)
- [LSP服务](../sources/2026-04-15-lsp服务.md)

## Open Questions

- 诊断优先级排序策略？
- 诊断聚合和智能过滤？

## Sources

- `chapters/chapter-27-lsp服务.md`