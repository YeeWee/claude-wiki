---
title: "React/Ink"
type: entity
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-01-项目概述与开发环境.md]
related: [../concepts/store-pattern.md, ../concepts/react-integration.md]
---

React 和 Ink 是 Claude Code 的终端 UI 渲染框架。Ink 将 React 的组件化思维带入终端环境，通过 Yoga 布局引擎实现 Flexbox 式的声明式界面构建，让开发者用 JSX 描述终端 UI 而非手动控制 ANSI 转义序列。

## Ink 框架特性

### React 组件模型

Ink 提供完整的 React 开发体验：

- 状态管理（useState、useEffect）
- 生命周期控制
- 组件复用和组合
- Context 跨组件传递

### Flexbox 布局

通过 Yoga 布局引擎实现类似 CSS Flexbox 的终端布局：

- `flexDirection`：column/row 方向
- `justifyContent`：主轴对齐
- `alignItems`：交叉轴对齐
- `flexWrap`：换行控制

### 声明式 UI

用 JSX 描述终端界面：

```typescript
<Box flexDirection="column">
  <Header />
  <MessageList />
  <TextInput />
  <StatusBar />
</Box>
```

## Claude Code 的 Ink 定制

### ink/ 目录

Claude Code 对 Ink 进行了深度定制，`src/ink/` 目录包含 50+ 文件：

- 渲染优化
- 键盘事件处理
- 终端能力检测
- 组件扩展

### React Hooks

85+ 个 React Hooks 用于状态订阅和副作用处理：

- `useAppState`：AppState 选择器订阅
- `useSetAppState`：仅获取更新函数
- `useToolJSX`：工具 JSX 渲染

## 终端能力处理

### ANSI 转义序列

Ink 自动处理：

- 颜色渲染（chalk 集成）
- 光标控制
- 屏幕清除
- 文本样式

### 终端兼容性

处理不同终端的差异：

- VT100 基础支持
- 现代 ANSI 扩展
- Windows cmd 兼容

## Connections

- [Store Pattern](../concepts/store-pattern.md) - 状态管理设计
- [React Integration](../concepts/react-integration.md) - React 与 AppState 同步
- [Project Overview](../sources/2026-04-15-project-overview.md) - UI 层详解

## Open Questions

- Ink 在低性能终端环境下的降级策略
- Yoga 布局引擎在终端场景的特殊处理

## Sources

- `chapters/chapter-01-项目概述与开发环境.md`