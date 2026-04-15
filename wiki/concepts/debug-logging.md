---
title: "调试日志"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-45-调试与诊断.md]
related: [../concepts/doctor-command.md, ../concepts/error-log-sink.md]
---

调试日志系统是 Claude Code 运行时追踪和问题定位的核心机制。

## 启用方式

| 方式 | 命令/环境变量 | 输出目标 |
|-----|-------------|---------|
| 命令行标志 | `--debug` 或 `-d` | 日志文件 |
| 环境变量 | `DEBUG=true` | 日志文件 |
| stderr 输出 | `--debug-to-stderr` | stderr |
| 自定义文件 | `--debug-file=/path` | 自定义路径 |
| 过滤模式 | `--debug=api,hooks` | 仅指定类别 |

## 日志级别

```typescript
type DebugLogLevel = 'verbose' | 'debug' | 'info' | 'warn' | 'error'
```

级别顺序：verbose (最详细) → debug → info → warn → error

## 存储位置

- 调试日志：`~/.claude/debug/<session-id>.txt`
- 最新日志：`~/.claude/debug/latest`（符号链接）

## Connections

- [/doctor 命令](../concepts/doctor-command.md)
- [错误日志 Sink](../concepts/error-log-sink.md)

## Open Questions

- 日志文件大小限制？

## Sources

- `chapters/chapter-45-调试与诊断.md`