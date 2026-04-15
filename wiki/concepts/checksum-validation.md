---
title: "Checksum 校验"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-43-远程设置同步.md]
related: [../concepts/etag-cache.md]
---

Checksum 校验是远程设置同步中用于验证配置一致性的机制，基于 SHA256 哈希实现。

## 实现细节

```typescript
function computeChecksumFromSettings(settings: SettingsJson): string {
  const sorted = sortKeysDeep(settings)
  const normalized = jsonStringify(sorted)
  const hash = createHash('sha256').update(normalized).digest('hex')
  return `sha256:${hash}`
}
```

## 关键设计

- 递归排序所有键，匹配 Python 的 `sort_keys=True`
- 使用紧凑 JSON 格式，匹配 Python 的 `separators=(",", ":")`
- SHA256 哈希前缀 `sha256:` 用于标识算法版本

## HTTP ETag 缓存

Checksum 用于 HTTP ETag 缓存：

- 请求头：`If-None-Match: "${checksum}"`
- 304 响应：缓存有效，无需重新下载
- 200 响应：新配置，需要应用

## Connections

- [ETag 缓存](../concepts/etag-cache.md)

## Open Questions

- 如何处理算法版本升级？

## Sources

- `chapters/chapter-43-远程设置同步.md`