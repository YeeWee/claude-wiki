---
title: "FileStateCache"
type: entity
created: 2026-04-15
updated: 2026-04-15
sources: [chapters/chapter-10-文件操作工具集.md]
related: [../concepts/read-before-write.md, ../concepts/lru-cache.md]
---

# FileStateCache

FileStateCache 是文件操作安全的核心机制，实现"先读后写"约束和并发冲突检测。

## 数据结构

```typescript
export type FileState = {
  content: string
  timestamp: number      // 文件修改时间（毫秒）
  offset: number | undefined   // 读取起始行
  limit: number | undefined    // 读取行数限制
  isPartialView?: boolean      // 是否为部分视图
}
```

## LRU 缓存实现

双重限制：
- 条目数限制（默认 100）
- 总大小限制（默认 25MB）

路径规范化确保不同格式路径（如 `/foo/../bar`）产生相同缓存键。

## 生命周期

1. FileReadTool 读取时设置
2. FileEditTool 编辑后更新
3. FileWriteTool 写入后更新

编辑和写入后，offset/limit 设为 undefined，表示完整文件视图。

## Connections

- [先读后写机制](../concepts/read-before-write.md) - 安全约束
- [LRU 缓存](../concepts/lru-cache.md) - 缓存策略
- [FileReadTool](../entities/file-read-tool.md) - 读取时设置缓存

## Sources

- [第十章：文件操作工具集](../sources/2026-04-15-文件操作工具集.md)