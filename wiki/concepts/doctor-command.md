---
title: "/doctor 命令"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-45-调试与诊断.md]
related: [../concepts/debug-logging.md]
---

/doctor 命令是 Claude Code 的诊断入口，用于检查安装和配置问题。

## 诊断内容

### 安装类型检测

- npm-global：npm 全局安装
- npm-local：npm 本地安装
- native：原生打包安装
- package-manager：Homebrew/apt/winget 等
- development：开发模式
- unknown：无法识别

### 其他诊断

- 版本信息
- 多安装检测
- 配置警告
- ripgrep 状态

## 上下文警告检测

- CLAUDE.md 文件过大
- Agent 描述过长
- MCP 工具上下文过大
- 不可达权限规则

## Connections

- [调试日志](../concepts/debug-logging.md)

## Open Questions

- 如何自动化诊断流程？

## Sources

- `chapters/chapter-45-调试与诊断.md`