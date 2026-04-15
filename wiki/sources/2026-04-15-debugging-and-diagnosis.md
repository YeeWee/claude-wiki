---
title: "调试与诊断"
type: source
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-45-调试与诊断.md]
related: [../concepts/doctor-command.md, ../concepts/debug-logging.md, ../concepts/error-log-sink.md]
---

调试与诊断工具是开发者的"健康体检系统"。Claude Code 提供了完整的诊断机制，从 `/doctor` 命令到多层次的日志系统，帮助开发者快速定位和解决问题。

## 核心能力维度

| 能力维度 | 核心功能 | 使用场景 |
|---------|---------|---------|
| 安装诊断 | `/doctor` 命令 | 安装验证、配置检查、环境问题 |
| 运行调试 | 调试日志系统 | 实时追踪、问题定位、性能分析 |
| 错误追踪 | 错误日志持久化 | Bug 报告、历史回溯、MCP 问题 |

## `/doctor` 命令

诊断信息收集内容包括：

- 安装类型检测：npm-global、npm-local、native、package-manager、development
- 版本信息
- 多安装检测
- 配置警告检测
- ripgrep 状态

## 上下文警告检测

系统检测可能导致上下文过载的配置问题：

- CLAUDE.md 文件过大警告
- Agent 描述过长警告
- MCP 工具上下文过大警告
- 不可达权限规则警告

## 调试日志系统

多种启用方式：

| 方式 | 命令/环境变量 | 输出目标 |
|-----|-------------|---------|
| 命令行标志 | `--debug` 或 `-d` | 日志文件 |
| 环境变量 | `DEBUG=true` | 日志文件 |
| stderr 输出 | `--debug-to-stderr` | stderr |
| 自定义文件 | `--debug-file=/path/to/file.log` | 自定义路径 |
| 过滤模式 | `--debug=api,hooks` | 仅指定类别 |

## 日志文件位置

| 日志类型 | 存储路径 | 内容说明 |
|---------|---------|---------|
| 调试日志 | `~/.claude/debug/<session-id>.txt` | 运行时调试信息 |
| 最新日志 | `~/.claude/debug/latest` | 符号链接指向最新日志 |
| 错误日志 | `~/.claude/errors/<date>.jsonl` | 错误记录 |
| MCP 日志 | `~/.claude/mcp-logs/<server>/<date>.jsonl` | MCP 服务器日志 |

## Connections

- [/doctor 命令](../concepts/doctor-command.md)
- [调试日志](../concepts/debug-logging.md)
- [错误日志 Sink](../concepts/error-log-sink.md)

## Open Questions

- 如何自动化诊断流程？
- 错误日志的保留策略是什么？

## Sources

- `chapters/chapter-45-调试与诊断.md`