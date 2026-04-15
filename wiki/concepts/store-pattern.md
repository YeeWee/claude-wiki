---
title: "Store Pattern"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-06-状态管理AppState.md]
related: [../concepts/selector-pattern.md, ../concepts/react-integration.md, ../entities/react-ink.md]
---

Store Pattern 是 Claude Code 状态管理的设计模式，采用极简实现（仅 35 行代码），提供 getState、setState、subscribe 三个核心方法。通过不可变更新、引用比较和 onChange 回调实现高效的状态流转。

## 极简实现

### 核心接口

```typescript
type Store<T> = {
  getState: () => T
  setState: (updater: (prev: T) => T) => void
  subscribe: (listener: Listener) => () => void
}

function createStore<T>(initialState: T, onChange?: OnChange<T>): Store<T>
```

### 实现特点

| 特性 | 说明 |
|------|------|
| 不可变更新 | setState 接收 updater 函数，返回新对象 |
| 引用比较 | Object.is 判断是否触发更新 |
| Set 存储 Listener | 自动去重，快速删除 |
| onChange 回调 | 状态变化时的额外处理钩子 |

## setState 工作流程

```typescript
setState: (updater) => {
  const prev = state
  const next = updater(prev)
  if (Object.is(next, prev)) return  // 相同引用跳过
  state = next
  onChange?.({ newState: next, oldState: prev })
  for (const listener of listeners) listener()
}
```

步骤：

1. 保存旧状态
2. 计算 new 状态
3. 引用比较（跳过未变）
4. 更新状态
5. 触发 onChange 回调
6. 通知所有 listener

## 与其他方案对比

| 方案 | 代码量 | 学习曲线 | 适用场景 |
|------|--------|----------|----------|
| Redux | 中等 | 高 | 大型 Web，严格流转 |
| Zustand | 低 | 中 | React 应用，selector |
| Claude Code Store | 极低 | 极低 | CLI/TUI，useSyncExternalStore |

灵感来自 Zustand，但更加精简：

- 无 actions、dispatch 抽象
- 直接 setState 函数式更新
- 与 React 18 的 useSyncExternalStore 配合

## onChange 回调用途

AppStateProvider 创建 Store 时传入：

- 同步状态到远程 daemon
- 触发持久化操作
- 记录状态变更日志

## Connections

- [Selector Pattern](../concepts/selector-pattern.md) - 选择器订阅
- [React Integration](../concepts/react-integration.md) - React 同步
- [React/Ink](../entities/react-ink.md) - 终端 UI 框架

## Open Questions

- onChange 回调的具体实现
- 与 React 并发渲染的兼容性

## Sources

- `chapters/chapter-06-状态管理AppState.md`