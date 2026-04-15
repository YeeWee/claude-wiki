---
title: "LSP协议"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: ["chapter-27-lsp服务.md"]
related: []
---

Language Server Protocol (LSP) 是微软开发的开放协议，用于建立编辑器与语言服务器之间的标准化通信。

## 协议基础

基于 JSON-RPC，定义客户端与服务器之间的通信规范。

## 核心能力

服务器可声明支持的能力：
- textDocumentSync: 文档同步
- hover: 悬停提示
- definition: 定义跳转
- references: 引用查找
- documentSymbol: 文档符号
- publishDiagnostics: 诊断发布
- callHierarchy: 调用层级

## 初始化握手

流程：
1. 客户端发送 initialize 请求 + 客户端能力
2. 服务器返回 InitializeResult + 服务器能力
3. 客户端发送 initialized 通知

## 客户端能力

Claude Code 声明：
- textDocument.synchronization: didSave 支持
- textDocument.publishDiagnostics: relatedInformation, tagSupport
- textDocument.hover: markdown/plaintext 格式
- textDocument.definition: linkSupport
- textDocument.documentSymbol: hierarchicalDocumentSymbolSupport

## 文件同步通知

- textDocument/didOpen: 打开文件
- textDocument/didChange: 内容变更
- textDocument/didSave: 保存文件
- textDocument/didClose: 关闭文件

## 诊断发布

服务器通过 `textDocument/publishDiagnostics` 通知推送诊断信息。包含：
- uri: 文件 URI
- diagnostics: 诊断数组
- version: 文档版本

严重程度：Error(1)、Warning(2)、Info(3)、Hint(4)

## 传输方式

- stdio: 标准输入/输出流（最常用）
- socket: TCP socket

## Connections

- [Language Server](../entities/language-server.md)
- [代码诊断系统](../concepts/code-diagnosis.md)
- [LSP服务](../sources/2026-04-15-lsp服务.md)

## Open Questions

- LSP 3.17+ 新特性的支持计划？
- 多语言服务器的协调机制？

## Sources

- `chapters/chapter-27-lsp服务.md`