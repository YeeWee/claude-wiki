---
title: "历史与回放"
type: source
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-44-历史与回放.md]
related: [../concepts/session-history.md, ../concepts/jsonl-format.md, ../concepts/resume-mechanism.md]
---

命令历史是 CLI 应用的核心用户体验之一。Claude Code 的历史系统不仅支持用户通过上下键快速访问之前的命令，还提供了 Ctrl+R 搜索功能，让用户能够快速定位和复用历史输入。

## 核心设计特点

1. **跨项目共享**：历史记录存储在全局配置目录，所有项目共享同一份历史
2. **项目过滤**：读取历史时按当前项目过滤，避免不同项目的命令混淆
3. **会话优先**：当前会话的命令优先显示，其他会话的命令排在后面
4. **粘贴内容处理**：支持大段粘贴内容的存储和恢复
5. **原子写入**：通过锁文件确保并发写入的安全性

## 存储结构

历史记录存储在 Claude 配置目录：

```
~/.claude/
├── history.jsonl           # 全局命令历史
├── paste-store/             # 大文本粘贴内容存储
│   ├── a1b2c3d4...         # 以 hash 命名的文件
│   └── ...
└── ...
```

## LogEntry 数据结构

```typescript
type LogEntry = {
  display: string                              // 显示文本
  pastedContents: Record<number, StoredPastedContent>  // 粘贴内容
  timestamp: number                            // 时间戳
  project: string                              // 项目路径
  sessionId?: string                           // 会话 ID
}
```

## 智能存储策略

- 小段粘贴文本（<= 1024 字符）直接内联在历史记录中
- 大段文本通过 hash 引用外部文件，避免历史文件膨胀
- 图片内容由 image-cache 单独管理

## Resume 机制

- 内存优先读取：`pendingEntries` 是最新的，还未刷盘
- 反向文件读取：从文件末尾向前读，确保新条目先被返回
- 跳过已移除：检查 `skippedTimestamps` 集合

## Connections

- [会话历史](../concepts/session-history.md)
- [JSONL 格式](../concepts/jsonl-format.md)
- [Resume 机制](../concepts/resume-mechanism.md)

## Open Questions

- 如何处理极长会话的历史管理？
- 历史记录的去重策略是什么？

## Sources

- `chapters/chapter-44-历史与回放.md`