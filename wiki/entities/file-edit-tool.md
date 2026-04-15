---
title: "FileEditTool"
type: entity
created: 2026-04-15
updated: 2026-04-15
sources: [chapters/chapter-10-文件操作工具集.md]
related: [../entities/file-read-tool.md, ../entities/file-write-tool.md]
---

# FileEditTool

FileEditTool 实现精确字符串替换编辑，核心设计是原子读取-编辑-写入流程。

## 核心参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `file_path` | `string` | 文件绝对路径 |
| `old_string` | `string` | 要替换的文本 |
| `new_string` | `string` | 替换后的文本 |
| `replace_all` | `boolean?` | 全局替换标志 |

## 验证流程（8 重检查）

1. 检查新旧字符串是否相同
2. 检查权限拒绝规则
3. 文件大小限制（防止 OOM）
4. 文件存在性检查
5. 读取状态检查（核心安全机制）
6. 文件修改时间检查（并发冲突检测）
7. 字符串匹配检查
8. 多次匹配检查

## 引号规范化

处理弯引号和直引号差异：
- 先尝试精确匹配
- 再尝试引号规范化匹配
- 编辑后保持文件的原始引号风格

## 原子执行

从加载文件到写入磁盘之间避免任何 async 操作，防止并发编辑交错。

自动检测并保持：
- 文件编码（UTF-8、UTF-16LE）
- 行尾风格（CRLF 或 LF）

## Connections

- [FileReadTool](../entities/file-read-tool.md) - 需要先读取
- [FileWriteTool](../entities/file-write-tool.md) - 替代方案
- [FileStateCache](../entities/file-state-cache.md) - 验证缓存状态

## Sources

- [第十章：文件操作工具集](../sources/2026-04-15-文件操作工具集.md)