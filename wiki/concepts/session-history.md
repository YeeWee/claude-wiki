---
title: "会话历史"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-44-历史与回放.md]
related: [../concepts/jsonl-format.md, ../concepts/resume-mechanism.md]
---

会话历史是 Claude CLI 的核心用户体验功能，支持命令历史记录和快速访问。

## 核心特点

1. **跨项目共享**：历史记录存储在全局配置目录
2. **项目过滤**：读取历史时按当前项目过滤
3. **会话优先**：当前会话的命令优先显示
4. **粘贴内容处理**：支持大段粘贴内容存储
5. **原子写入**：通过锁文件确保并发安全

## LogEntry 结构

```typescript
type LogEntry = {
  display: string                              // 显示文本
  pastedContents: Record<number, StoredPastedContent>  // 粘贴内容
  timestamp: number                            // 时间戳
  project: string                              // 项目路径
  sessionId?: string                           // 会话 ID
}
```

## 智能存储

- 小段文本（<= 1024 字符）内联存储
- 大段文本通过 hash 引用外部文件
- 图片由 image-cache 单独管理

## Connections

- [JSONL 格式](../concepts/jsonl-format.md)
- [Resume 机制](../concepts/resume-mechanism.md)

## Open Questions

- 极长会话的历史管理策略？
- 历史记录的去重机制？

## Sources

- `chapters/chapter-44-历史与回放.md`