---
title: "ripgrep"
type: entity
created: 2026-04-15
updated: 2026-04-15
sources: [chapters/chapter-11-搜索工具集.md]
related: [../entities/glob-tool.md, ../entities/grep-tool.md]
---

# ripgrep

ripgrep 是 Claude Code 搜索工具的底层引擎，通过三种运行模式适配不同分发场景。

## 运行模式

| 模式 | 命令来源 | 适用场景 |
|------|----------|----------|
| `system` | 系统安装的 `rg` | 用户偏好系统版本 |
| `embedded` | bun-internal 内嵌 | 捆绑分发模式 |
| `builtin` | vendor 目录二进制 | npm 包分发 |

## 配置发现

`getRipgrepConfig()` 使用 memoize 缓存，避免重复检测。

## 特殊处理

### EAGAIN 错误

在资源受限环境（Docker、CI）中可能触发 EAGAIN 错误，自动重试单线程模式（`-j 1`）。

### 超时机制

| 平台 | 默认超时 |
|------|----------|
| WSL | 60 秒 |
| 其他 | 20 秒 |

### macOS 代码签名

捆绑的 ripgrep 二进制可能需要自动代码签名和 quarantine 属性移除。

## 流式处理

`ripGrepStream()` 支持实时结果输出：
- 结果即时可用
- 支持用户提前中止
- 内存占用更低

## Connections

- [GlobTool](../entities/glob-tool.md) - 使用 ripgrep --files
- [GrepTool](../entities/grep-tool.md) - 使用 ripgrep 搜索
- [搜索工具集](../sources/2026-04-15-搜索工具集.md) - 集成文档

## Sources

- [第十一章：搜索工具集](../sources/2026-04-15-搜索工具集.md)