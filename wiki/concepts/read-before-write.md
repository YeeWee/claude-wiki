---
title: "先读后写机制"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: [chapters/chapter-10-文件操作工具集.md]
related: [../entities/file-state-cache.md, ../entities/file-edit-tool.md]
---

# 先读后写机制

"先读后写"是 Claude Code 文件操作的核心安全约束，通过 readFileState 缓存实现。

## 核心原则

模型必须先读取文件，才能进行编辑或写入操作。

## 实现机制

### 缓存验证

FileEditTool 和 FileWriteTool 的验证流程：
- 检查 readFileState 是否存在
- 检查是否为完整视图（isPartialView）
- 检查修改时间是否变化（并发冲突检测）

### 失败场景

- 文件未读取：返回错误，提示需先读取
- 部分视图：如 CLAUDE.md 注入，需显式读取
- 文件被修改：返回错误，提示文件已变更

### Windows 平台 fallback

修改时间检测在 Windows 平台可能不稳定，使用内容比对作为 fallback。

## 安全意义

1. **防止盲目修改**：确保模型了解文件内容
2. **并发冲突检测**：防止多个编辑交错
3. **操作可追溯**：缓存记录读取状态

## Connections

- [FileStateCache](../entities/file-state-cache.md) - 缓存实现
- [FileEditTool](../entities/file-edit-tool.md) - 应用此机制
- [FileWriteTool](../entities/file-write-tool.md) - 应用此机制

## Sources

- [第十章：文件操作工具集](../sources/2026-04-15-文件操作工具集.md)