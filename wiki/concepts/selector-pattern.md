---
title: "Selector Pattern"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-06-状态管理AppState.md]
related: [../concepts/store-pattern.md, ../concepts/react-integration.md]
---

Selector Pattern 是 Claude Code 状态订阅的设计模式，通过 useAppState hook 接收选择器函数，精准控制组件重渲染。核心原理是 Object.is 比较选择器返回值，即使 Store 状态变化，若选择器结果未变，组件不会重渲染。

## useAppState 实现

### Hook 签名

```typescript
function useAppState<T>(selector: (state: AppState) => T): T
```

### 实现原理

```typescript
function useAppState<T>(selector: (state: AppState) => T): T {
  const store = useAppStore()

  const get = () => {
    const state = store.getState()
    return selector(state)
  }

  return useSyncExternalStore(store.subscribe, get, get)
}
```

关键：React 18 的 useSyncExternalStore 让外部 Store 与渲染周期同步。

## 最佳实践

### 正确用法

```typescript
// 选择单个字段 - 推荐
const verbose = useAppState(s => s.verbose)

// 选择现有子对象 - 推荐（引用稳定）
const { text, promptId } = useAppState(s => s.promptSuggestion)

// 多次调用选择多个字段 - 推荐
const verbose = useAppState(s => s.verbose)
const model = useAppState(s => s.mainLoopModel)
```

### 错误用法

```typescript
// 返回新对象 - 每次调用创建新引用
const { text, promptId } = useAppState(s => ({
  text: s.promptSuggestion.text,
  promptId: s.promptSuggestion.promptId
}))
// Object.is 比较永远 false，不必要重渲染
```

## 性能优化原理

### 执行流程

1. Store 状态变化
2. 调用所有 listener
3. Hook 执行 selector(state)
4. Object.is 比较新旧值
5. 若值变化，触发重渲染
6. 若值未变，跳过重渲染

### 精准重渲染

```typescript
// 其他字段变更，verbose 值未变
store.setState(s => ({ ...s, settings: newSettings }))
// verbose 选择器返回 false
// Object.is(false, false) = true
// 组件不重渲染
```

## useSetAppState

仅获取更新函数，不订阅状态：

```typescript
function useSetAppState() {
  return useAppStore().setState
}
```

优势：

- **稳定引用**：setState 永不变化
- **无订阅开销**：组件不因状态变化重渲染
- **适合事件处理**：按钮点击等场景

## Connections

- [Store Pattern](../concepts/store-pattern.md) - Store 设计
- [React Integration](../concepts/react-integration.md) - React 同步机制

## Open Questions

- 选择器在复杂嵌套对象场景的性能
- 多选择器调用的优化策略

## Sources

- `chapters/chapter-06-状态管理AppState.md`