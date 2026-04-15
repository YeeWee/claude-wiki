---
title: "GrepTool"
type: entity
created: 2026-04-15
updated: 2026-04-15
sources: [chapters/chapter-11-搜索工具集.md]
related: [../entities/glob-tool.md, ../entities/ripgrep.md]
---

# GrepTool

GrepTool 基于 ripgrep 实现高效内容搜索，支持正则表达式搜索和多种输出模式。

## 核心参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `pattern` | `string` | 正则表达式搜索模式 |
| `path` | `string?` | 搜索路径 |
| `glob` | `string?` | 文件类型过滤 |
| `type` | `string?` | 文件类型（js、py、rust 等） |
| `output_mode` | `enum?` | 输出模式 |
| `head_limit` | `number?` | 结果限制（默认 250） |
| `offset` | `number?` | 偏移量（分页） |

## 三种输出模式

### files_with_matches（默认）

返回匹配的文件列表，按修改时间排序。

### content

返回匹配的行内容，包含行号和上下文。

### count

返回每个文件的匹配次数。

## VCS 目录排除

自动排除版本控制目录：`.git`、`.svn`、`.hg`、`.bzr`、`.jj`、`.sl`

## 分页机制

使用 `offset` 和 `head_limit` 进行分页：
- 第一次：`{ pattern: "function", head_limit: 100 }`
- 第二次：`{ pattern: "function", head_limit: 100, offset: 100 }`

## Connections

- [GlobTool](../entities/glob-tool.md) - 文件匹配工具
- [ripgrep](../entities/ripgrep.md) - 底层引擎
- [Tool 接口](../entities/tool-interface.md) - 继承结构

## Sources

- [第十一章：搜索工具集](../sources/2026-04-15-搜索工具集.md)