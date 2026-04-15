---
title: "REPL Bridge远程控制集成"
type: source
created: 2026-04-15
updated: 2026-04-15
sources: ["chapter-30-repl-bridge.md"]
related: []
---

REPL Bridge 是 Claude Code 实现远程控制功能的核心组件，允许用户通过 claude.ai 网站远程控制本地运行的 CLI 会话。

## 核心模块

- replBridge.ts: 核心逻辑，完整的生命周期管理
- sessionRunner.ts: 会话进程管理

## ReplBridgeHandle 接口

提供消息写入、控制请求发送和清理功能：
- writeMessages: 写入消息数组
- writeSdkMessages: 写入 SDK 消息
- sendControlRequest: 发送控制请求
- sendControlResponse: 发送控制响应
- sendResult: 发送结果
- teardown: 清理销毁

## BridgeState 状态机

- ready: 初始化完成，等待连接
- connected: Transport 连接成功，可双向通信
- reconnecting: 连接中断，正在重连
- failed: 连接失败，需用户干预

## BridgeCoreParams 配置

采用依赖注入模式，所有上下文信息通过参数传入，不读取 bootstrap state，确保可测试性和独立性。

关键参数：
- createSession: 会话创建函数
- archiveSession: 会话归档函数
- onAuth401: OAuth 401 处理
- getPollIntervalConfig: 轮询配置

## 重连机制

两阶段策略：
1. 原地重连：tryReconnectInPlace
2. 新建会话：archiveSession + createSession

崩溃恢复指针（bridgePointer）支持进程崩溃后恢复。

## Transport 连接管理

两种传输模式：
- v1 (Session-Ingress): HybridTransport，OAuth 或 JWT 认证
- v2 (CCR): SSETransport + CCRClient，强制 JWT 认证

## FlushGate 机制

防止消息顺序错乱，初始刷新期间新消息被排队，等刷新完成后统一发送。

## SessionRunner

子进程管理：
- 使用 spawn 创建子进程
- NDJSON 输出解析
- stderr 环形缓冲（10 行）
- 活动环形缓冲（10 条）

工具动词映射提供可读的活动描述。

## 安全考虑

- OAuth token 清除：子进程不继承父进程 token
- Session Access Token：专用会话令牌
- 权限请求转发：敏感操作需用户在 claude.ai 确认

## Connections

- [REPL Bridge](../concepts/repl-bridge.md)
- [Transport层](../concepts/transport-layer.md)
- [会话Runner](../concepts/session-runner.md)

## Open Questions

- 如何进一步优化重连成功率？
- 子进程资源监控和限制机制？

## Sources

- `chapters/chapter-30-repl-bridge.md`