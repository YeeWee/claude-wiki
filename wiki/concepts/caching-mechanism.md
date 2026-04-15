---
title: "缓存机制"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-13-web工具集.md]
related: []
---

Claude Web 工具使用 LRU 缓存优化响应速度和减少重复请求。

## WebFetchTool 缓存

### 配置

| 参数 | 值 |
|------|-----|
| TTL | 15 分钟 |
| 最大大小 | 50MB |
| 类型 | LRU |

### 缓存结构

```typescript
type CacheEntry = {
  bytes: number
  code: number
  codeText: string
  content: string
  contentType: string
  persistedPath?: string
  persistedSize?: number
}
```

## 域名检查缓存

| 参数 | 值 |
|------|-----|
| max | 128 |
| TTL | 5 分钟 |

只缓存 `allowed` 状态，`blocked` 和 `failed` 每次重新检查。

## Turndown 惰性加载

Turndown + Domino 模块约 1.4MB，首次 HTML 抓取时加载，之后复用实例。

## 缓存命中处理

命中时直接返回缓存内容，跳过 HTTP 请求和内容处理。

## Connections

- [WebFetchTool](../entities/web-fetch-tool.md)

## Open Questions

- 如何处理缓存失效场景？

## Sources

- `chapters/chapter-13-web工具集.md`