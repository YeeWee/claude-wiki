---
title: "优先级队列"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: [chapters/chapter-07-react-context体系.md]
related: []
---

# 优先级队列

Claude Code 的 Notification 系统使用优先级队列管理通知显示顺序。

## 优先级定义

```typescript
const PRIORITIES: Record<Priority, number> = {
  immediate: 0,  // 最高优先级
  high: 1,
  medium: 2,
  low: 3
};
```

## 获取下一个通知

```typescript
export function getNext(queue: Notification[]): Notification | undefined {
  if (queue.length === 0) return undefined;
  return queue.reduce((min, n) =>
    PRIORITIES[n.priority] < PRIORITIES[min.priority] ? n : min
  );
}
```

## 简化实现

队列本身无序，每次取出时遍历找最小优先级值。对于小型队列（通常 5-10 条通知），O(n) 查询是合理的。

## immediate 特殊处理

immediate 优先级的通知会立即抢占显示：
- 清除现有 timeout
- 将当前非 immediate 通知重新入队
- 立即设置新的 timeout

## 应用场景

- 错误/警告：immediate 优先级
- 进度更新：medium 优先级
- 一般提示：low 优先级

## Sources

- [第七章：React Context 体系](../sources/2026-04-15-react-context体系.md)