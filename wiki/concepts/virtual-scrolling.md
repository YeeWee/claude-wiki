---
title: "虚拟滚动"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-47-性能优化策略.md]
related: []
---

虚拟滚动是 Claude Code 的渲染性能优化机制，避免渲染所有历史消息。

## 关键参数

| 参数 | 值 | 说明 |
|------|-----|------|
| DEFAULT_ESTIMATE | 3 | 未测量项估计高度 |
| OVERSCAN_ROWS | 80 | 上下缓冲区 |
| COLD_START_COUNT | 30 | 冷启动渲染数量 |
| MAX_MOUNTED_ITEMS | 300 | 最大挂载项数 |
| SCROLL_QUANTUM | 40 | scrollTop 量化阈值 |

## 优化策略

| 策略 | 效果 |
|------|------|
| 高度缓存 | 减少 ~50x stringWidth 调用 |
| 滚动量化 | 减少 CPU 峰值 |
| Overscan | 消除滚动白屏 |
| 最大挂载限制 | 防止 fiber 爆炸 |

## 行宽缓存

字符串宽度缓存避免重复测量：

```typescript
const cache = new Map<string, number>()
const MAX_CACHE_SIZE = 4096
```

## Connections

- 无

## Open Questions

- 极端场景下的性能表现？

## Sources

- `chapters/chapter-47-性能优化策略.md`