---
title: "FileReadTool"
type: entity
created: 2026-04-15
updated: 2026-04-15
sources: [chapters/chapter-10-文件操作工具集.md]
related: [../entities/file-state-cache.md, ../entities/file-edit-tool.md]
---

# FileReadTool

FileReadTool 是 Claude Code 功能最丰富的文件工具，支持多格式文件读取。

## 核心参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `file_path` | `string` | 必须是绝对路径 |
| `offset` | `number?` | 起始行号（从 1 开始） |
| `limit` | `number?` | 读取行数限制 |
| `pages` | `string?` | PDF 文件页面范围 |

## 多格式支持

- 文本文件：使用 cat -n 格式添加行号
- 图像文件：智能压缩，Token 预算检查
- PDF 文件：完整读取或页面提取
- Jupyter Notebook：解析 cells 结构

## 安全机制

- 文件大小限制：Token 限制验证
- 二进制文件检测：阻止读取二进制文件
- 设备文件阻塞检测：防止读取会无限输出的设备

## 去重机制

避免重复发送相同内容：
- 检查缓存条目是否存在
- 读取范围完全匹配
- 文件修改时间未变

返回 `file_unchanged` 类型提示模型引用之前结果。

## Connections

- [FileStateCache](../entities/file-state-cache.md) - 读取时设置缓存
- [FileEditTool](../entities/file-edit-tool.md) - 需要先读取
- [先读后写机制](../concepts/read-before-write.md) - 安全约束

## Sources

- [第十章：文件操作工具集](../sources/2026-04-15-文件操作工具集.md)