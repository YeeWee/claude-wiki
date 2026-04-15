---
title: "LRU 缓存"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: [chapters/chapter-10-文件操作工具集.md]
related: [../entities/file-state-cache.md]
---

# LRU 缓存

Claude Code 的 FileStateCache 使用 LRU（Least Recently Used）缓存策略，实现双重限制。

## 双重限制

- 条目数限制（默认 100）
- 总大小限制（默认 25MB）

## 大小计算

使用 `Buffer.byteLength` 计算内容大小：
```typescript
sizeCalculation: value => Math.max(1, Buffer.byteLength(value.content))
```

## 路径规范化

所有路径通过 `normalize()` 处理，确保不同格式路径产生相同缓存键：
- `/foo/../bar` → `/bar`
- `./file.ts` → `file.ts`

## 缓存效果

- 最近访问的文件状态保留
- 大文件自动淘汰旧条目
- 路径格式差异不影响缓存

## Connections

- [FileStateCache](../entities/file-state-cache.md) - 应用 LRU 缓存

## Sources

- [第十章：文件操作工具集](../sources/2026-04-15-文件操作工具集.md)