---
title: "FileWriteTool"
type: entity
created: 2026-04-15
updated: 2026-04-15
sources: [chapters/chapter-10-文件操作工具集.md]
related: [../entities/file-edit-tool.md, ../entities/file-read-tool.md]
---

# FileWriteTool

FileWriteTool 用于创建新文件或完整覆盖现有文件，区分 create 和 update 两种模式。

## 核心参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `file_path` | `string` | 文件绝对路径 |
| `content` | `string` | 写入内容 |

## 输出模式

- `create`：文件不存在时创建
- `update`：文件存在时完整覆盖

返回 structuredPatch（完整 diff）和 originalFile。

## 验证流程

1. 检查 secrets
2. 检查权限拒绝规则
3. 检查文件存在性
4. 读取状态检查
5. 修改时间检查

## 行尾处理

与 FileEditTool 的关键差异：
- Write 使用 LF（模型意图）
- Edit 保持原行尾风格

## Connections

- [FileEditTool](../entities/file-edit-tool.md) - 精确替换替代方案
- [FileReadTool](../entities/file-read-tool.md) - 需要先读取
- [FileStateCache](../entities/file-state-cache.md) - 验证缓存状态

## Sources

- [第十章：文件操作工具集](../sources/2026-04-15-文件操作工具集.md)