---
title: "React Context 体系"
type: source
created: 2026-04-15
updated: 2026-04-15
sources: [chapters/chapter-07-react-context体系.md]
related: []
---

# React Context 体系

Claude Code 使用 React Context 作为全局状态管理的核心机制，采用分层设计，每个 Context 负责特定领域的状态管理。

## 核心内容

### Context 层级架构

- **AppStoreContext**：顶层状态容器，管理 Notifications 和 ActiveOverlays
- **StatsContext**：性能统计追踪
- **ModalContext**：模态对话框布局管理
- **OverlayContext**：Overlay UI 状态和 ESC 键协调

### 设计模式

1. **Store-Provider-Hook 模式**：StatsContext 为代表，通过 TypeScript 接口定义 Store 方法，支持外部注入或内部创建
2. **派生 Hook 模式**：useCounter、useGauge、useTimer 等简化特定场景使用
3. **条件检测 Hook 模式**：useIsOverlayActive、useIsInsideModal 检测运行环境

### Notification 系统

通知系统实现了优先级队列、折叠合并和失效机制：
- 四级优先级：immediate、high、medium、low
- 折叠机制：相同 key 的通知合并
- 失效机制：通知可声明使其他通知无效
- immediate 优先级会立即抢占显示

### Stats 统计系统

性能指标收集和持久化机制：
- 简单计数/数值（metrics）
- 分布统计（histograms）- 使用蓄水池采样
- 字符串集合（sets）
- 导出 p50、p95、p99 百分位数

### Overlay 管理

解决多层 Overlay 环境下 ESC 键协调问题：
- 自动生命周期管理（mount 注册，unmount 注销）
- 区分模态与非模态 Overlay
- 布局强制重绘机制

## 关键文件

| 文件路径 | 主要内容 |
|---------|---------|
| `src/context/notifications.tsx` | 通知队列系统 |
| `src/context/modalContext.tsx` | 模态对话框管理 |
| `src/context/overlayContext.tsx` | Overlay 状态管理 |
| `src/context/stats.tsx` | 性能统计追踪 |
| `src/context.ts` | 系统上下文（Git 状态等） |

## 设计启示

- 分层职责避免状态混杂
- 完整 TypeScript 类型约束
- useEffect 自动管理注册/注销
- memoize 缓存优化性能

## Open Questions

- StatsContext 的持久化时机是否可优化？
- Notification 系统是否需要支持国际化？

## Sources

- [第七章：React Context 体系](../chapters/chapter-07-react-context体系.md)