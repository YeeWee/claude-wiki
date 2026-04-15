---
title: "GlobTool"
type: entity
created: 2026-04-15
updated: 2026-04-15
sources: [chapters/chapter-11-搜索工具集.md]
related: [../entities/grep-tool.md, ../entities/ripgrep.md]
---

# GlobTool

GlobTool 基于 ripgrep `--files` 模式实现快速文件发现，用于基于 glob 模式匹配文件名。

## 核心参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `pattern` | `string` | Glob 匹配模式（如 `**/*.ts`） |
| `path` | `string?` | 搜索目录，默认当前工作目录 |

## 输出结构

```typescript
{
  durationMs: number,    // 执行耗时（毫秒）
  numFiles: number,      // 找到的文件总数
  filenames: string[],   // 匹配的文件路径数组
  truncated: boolean,    // 结果是否被截断
}
```

## 执行特性

- 结果按修改时间排序（最新的在前）
- 默认限制 100 个文件
- 相对路径转换节省 token
- 支持绝对路径和相对路径模式

## 环境变量控制

| 环境变量 | 默认值 | 说明 |
|----------|--------|------|
| `CLAUDE_CODE_GLOB_NO_IGNORE` | `true` | 是否忽略 .gitignore |
| `CLAUDE_CODE_GLOB_HIDDEN` | `true` | 是否包含隐藏文件 |
| `CLAUDE_CODE_GLOB_TIMEOUT_SECONDS` | - | 超时时间 |

## Connections

- [GrepTool](../entities/grep-tool.md) - 内容搜索工具
- [ripgrep](../entities/ripgrep.md) - 底层引擎
- [Tool 接口](../entities/tool-interface.md) - 继承结构

## Sources

- [第十一章：搜索工具集](../sources/2026-04-15-搜索工具集.md)