---
title: "REPL Bridge"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: ["chapter-30-repl-bridge.md"]
related: []
---

REPL Bridge 是 Claude Code 远程控制功能的核心实现，提供完整的生命周期管理。

## 核心模块

- replBridge.ts: 核心逻辑
- sessionRunner.ts: 会话进程管理

## ReplBridgeHandle 接口

对外暴露的功能：
- writeMessages(messages): 写入消息数组
- writeSdkMessages(messages): 写入 SDK 消息
- sendControlRequest(request): 发送控制请求
- sendControlResponse(response): 发送控制响应
- sendControlCancelRequest(requestId): 取消请求
- sendResult(): 发送结果
- teardown(): 清理销毁

## BridgeState 状态机

四种状态：
- ready: 初始化完成，等待连接
- connected: Transport 连接成功
- reconnecting: 连接中断，正在重连
- failed: 连接失败，需用户干预

## BridgeCoreParams

依赖注入模式，不读取 bootstrap state：
- createSession: 会话创建函数
- archiveSession: 会话归档函数
- onAuth401: OAuth 401 处理
- getPollIntervalConfig: 轮询配置
- initialHistoryCap: 历史消息上限
- onInboundMessage: 入站消息回调

## 消息流转

入站（服务器 → CLI）：
- SSE 推送消息
- handleIngressMessage 处理
- writeMessages 传递

出站（CLI → 服务器）：
- stdout NDJSON 输出
- extractActivities 解析
- Transport POST 写入

## FlushGate 机制

防止消息顺序错乱：
- 初始刷新期间新消息排队
- 刷新完成后统一发送

## 性能优化

- UUID 去重缓冲区（2000）
- 历史消息上限
- 活动环形缓冲（10 条）
- stderr 环形缓冲（10 行）

## 安全设计

- OAuth token 清除：子进程不继承父进程 token
- Session Access Token: 专用会话令牌
- 权限请求转发：敏感操作需云端确认

## Connections

- [Bridge系统](../concepts/bridge-system.md)
- [会话管理](../concepts/session-management.md)
- [传输层](../concepts/transport-layer.md)
- [REPL Bridge远程控制集成](../sources/2026-04-15-repl-bridge.md)

## Open Questions

- 重连成功率的优化？
- 子进程资源监控？

## Sources

- `chapters/chapter-30-repl-bridge.md`