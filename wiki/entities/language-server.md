---
title: "Language Server"
type: entity
created: 2026-04-15
updated: 2026-04-15
sources: ["chapter-27-lsp服务.md"]
related: []
---

Language Server 是实现代码智能功能的后端进程，通过 LSP 协议与客户端通信。

## 核心功能

提供代码智能功能：
- 实时代码诊断（错误、警告）
- 定义跳转
- 引用查找
- 代码补全
- 文档符号

## 生命周期

状态流转：
- stopped → starting → running
- running → stopping → stopped
- 任意状态 → error
- error → starting（重试）

## 崩溃恢复

崩溃恢复限制防止无限重启：
- 默认 maxRestarts = 3
- crashRecoveryCount 超过限制时拒绝启动

## 瞬态错误处理

自动重试机制：
- 错误码 -32801 (content modified)
- 指数退避：500ms → 1000ms → 2000ms
- 最大重试次数：3

## 传输方式

- stdio: 标准输入/输出流（最常用）
- socket: TCP socket

## 配置来源

仅通过插件提供，不支持用户/项目直接配置。

配置 Schema 包含：
- command: LSP 服务器命令
- args: 命令行参数
- extensionToLanguage: 文件扩展名映射
- initializationOptions: 初始化选项
- startupTimeout: 启动超时
- maxRestarts: 最大重启次数

## Connections

- [LSP协议](../concepts/lsp-protocol.md)
- [代码诊断系统](../concepts/code-diagnosis.md)
- [LSP服务](../sources/2026-04-15-lsp服务.md)

## Open Questions

- 如何支持更多 Language Server？
- 诊断信息的智能聚合？

## Sources

- `chapters/chapter-27-lsp服务.md`