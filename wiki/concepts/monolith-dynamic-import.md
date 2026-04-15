---
title: "Monolith + Dynamic Import"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-46-架构设计原则总结.md]
related: [../concepts/fast-path.md, ../concepts/lazy-loading.md]
---

Monolith + Dynamic Import 是 Claude Code 的核心架构策略，在单体架构基础上通过动态导入实现启动优化。

## 设计动机

### 单体优势

- 单进程启动，无网络开销
- 单二进制文件，零依赖
- 本地调试便捷
- 共享内存，效率高

### 动态导入解决

- 启动时加载所有模块，延迟高
- 内存占用随功能增长膨胀
- 无法灵活控制功能可见性

## 模块分层

| Layer | 描述 | 加载时机 |
|-------|------|---------|
| L0 | 立即加载（零延迟） | 入口点 |
| L1 | 快速路径导入 | 按需 |
| L2 | 主应用导入 | 正常启动 |
| L3 | 延迟预取 | REPL 渲染后 |

## 效果

- 版本查询 <10ms（零模块加载）
- 完整启动 ~300ms

## Connections

- [快速路径机制](../concepts/fast-path.md)
- [Lazy Loading](../concepts/lazy-loading.md)

## Open Questions

- 如何平衡功能丰富性与启动速度？

## Sources

- `chapters/chapter-46-架构设计原则总结.md`