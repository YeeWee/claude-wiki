---
title: "折叠机制"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: [chapters/chapter-07-react-context体系.md]
related: []
---

# 折叠机制

Notification 系统的折叠机制允许相同 key 的通知合并，类似 Array.reduce 的语义。

## 折叠函数

```typescript
fold?: (accumulator: Notification, incoming: Notification) => Notification
```

## 折叠位置

### 折叠到当前显示

- 检查当前通知 key 是否匹配
- 执行 fold 函数合并
- 重置 timeout

### 折叠到队列

- 在队列中查找相同 key
- 执行 fold 函数合并
- 更新队列项

## 应用场景

- **进度更新**：将多个进度通知合并为一条带百分比的通知
- **计数累积**：多条相同类型的通知合并显示总数
- **时间延长**：重复通知延长显示时间而非重复排队

## 示例

进度通知折叠：
```
初始: "正在处理 1/10"
折叠: "正在处理 2/10" → "正在处理 1-2/10"
折叠: "正在处理 3/10" → "正在处理 1-3/10"
```

## Sources

- [第七章：React Context 体系](../sources/2026-04-15-react-context体系.md)