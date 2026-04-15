---
title: "Lazy Loading"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-46-架构设计原则总结.md, chapter-47-性能优化策略.md]
related: [../concepts/monolith-dynamic-import.md, ../concepts/fast-path.md]
---

Lazy Loading 是 Claude Code 的延迟加载策略，重型模块在需要时才加载，减少启动延迟。

## 多级延迟加载

```
cli.tsx → init.ts → main.tsx → REPL → 延迟预取
```

## 延迟预取内容

| 优先级 | 内容 | 失败影响 |
|--------|------|---------|
| 高 | 用户信息 | 影响功能门控 |
| 高 | 系统上下文 | 影响提示词 |
| 中 | 云凭证 | 影响 BYOC |
| 低 | 文件计数 | 仅影响显示 |
| 低 | 变化检测器 | 影响热重载 |
| 低 | 遥测 | 仅影响分析 |

## 并行 I/O

慢速 I/O 操作在模块导入阶段并行启动：

- MDM 配置子进程读取
- macOS keychain 预读取

## 早期输入捕获

`startCapturingEarlyInput()` 在模块加载期间预先捕获用户输入，减少首次交互感知延迟。

## Connections

- [Monolith + Dynamic Import](../concepts/monolith-dynamic-import.md)
- [快速路径机制](../concepts/fast-path.md)

## Open Questions

- 如何确定延迟加载边界？

## Sources

- `chapters/chapter-46-架构设计原则总结.md`
- `chapters/chapter-47-性能优化策略.md`