---
title: "React Integration"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-06-状态管理AppState.md]
related: [../concepts/store-pattern.md, ../concepts/selector-pattern.md, ../entities/react-ink.md]
---

React Integration 是 Claude Code 状态管理与 React 组件的同步机制，通过 React 18 的 useSyncExternalStore hook 实现外部 Store 与 React 渲染周期的安全同步，支持并发渲染和 SSR 场景。

## useSyncExternalStore

### Hook 签名

```typescript
useSyncExternalStore(
  subscribe: (listener) => () => void,
  getSnapshot: () => T,
  getServerSnapshot?: () => T,
): T
```

### 参数说明

| 参数 | 用途 |
|------|------|
| subscribe | 注册监听器，返回取消订阅函数 |
| getSnapshot | 获取当前状态值（客户端） |
| getServerSnapshot | 获取当前状态值（服务端） |

### 在 useAppState 中的应用

```typescript
return useSyncExternalStore(store.subscribe, get, get)
```

`get` 函数同时用于客户端和服务端快照。

## 同步机制

### 渲染前读取

React 在每次渲染前调用 getSnapshot：

1. 获取当前 Store 状态
2. 执行 selector 选择器
3. Object.is 比较新旧值
4. 决定是否触发重渲染

### 批量更新

Store 状态变化时：

1. setState 更新状态
2. 触发所有 listener
3. React 批量处理更新请求
4. 统一触发重渲染

## 并发渲染兼容

### 解决的问题

传统外部 Store 的并发渲染问题：

- 渲染阶段状态不一致
- Tear 现象（读取到中间状态）

### useSyncExternalStore 解决

- 在渲染阶段安全读取 Store
- 确保 getSnapshot 返回一致值
- 支持 React 18 的并发特性

## AppStateProvider

### Provider 结构

```typescript
type Props = {
  children: React.ReactNode
  initialState?: AppState
  onChangeAppState?: OnChange<AppState>
}
```

### 职责

1. 创建唯一 Store 实例（useState 保持稳定）
2. Context 向子组件提供 Store
3. 监听外部设置变更
4. 处理权限模式初始化

### 嵌套保护

```typescript
const hasAppStateContext = useContext(HasAppStateContext)
if (hasAppStateContext) {
  throw new Error("AppStateProvider cannot be nested")
}
```

确保全局只有一个 AppState，避免状态碎片化。

## Connections

- [Store Pattern](../concepts/store-pattern.md) - Store 设计
- [Selector Pattern](../concepts/selector-pattern.md) - 选择器订阅
- [React/Ink](../entities/react-ink.md) - 终端 UI 框架

## Open Questions

- SSR 场景的具体应用
- React Strict Mode 下的行为差异

## Sources

- `chapters/chapter-06-状态管理AppState.md`