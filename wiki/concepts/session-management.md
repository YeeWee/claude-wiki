---
title: "会话管理"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: ["chapter-29-bridge系统架构.md", "chapter-30-repl-bridge.md"]
related: []
---

会话管理是 Bridge 系统的核心功能，负责会话创建、监控、恢复和终止。

## 环境注册

Bridge 向云端注册环境，发送：
- 机器名称
- 工作目录
- Git 分支
- 最大会话数

返回：environment_id + environment_secret

## 会话创建

通过 createSession 函数创建会话：
- environmentId
- title
- gitRepoUrl
- branch

返回：session_id

## 崩溃恢复指针

writeBridgePointer 写入崩溃恢复信息：
- sessionId
- environmentId
- source

支持进程崩溃后的恢复。

## UUID 去重

BoundedUUIDSet（容量 2000）防止消息重复：
- recentPostedUUIDs: 已发送消息
- recentInboundUUIDs: 已接收消息

## 会话启动参数

子进程命令行参数：
- --print
- --sdk-url
- --session-id
- --input-format stream-json
- --output-format stream-json
- --replay-user-messages

环境变量：
- CLAUDE_CODE_ENVIRONMENT_KIND: 'bridge'
- CLAUDE_CODE_SESSION_ACCESS_TOKEN
- CLAUDE_CODE_USE_CCR_V2

## 会话状态

SessionHandle 状态：
- done: 完成状态 Promise
- activities: 活动列表
- accessToken: 访问令牌
- currentActivity: 当前活动

## 会话终止

终止状态：
- interrupted: SIGTERM/SIGINT
- completed: code === 0
- failed: 其他情况

## 环境注销

teardown 时：
- stopWork 通知服务端
- deregisterEnvironment 删除注册
- 服务端显示离线

## Connections

- [Bridge系统](../concepts/bridge-system.md)
- [REPL Bridge](../concepts/repl-bridge.md)
- [Bridge系统架构](../sources/2026-04-15-bridge系统架构.md)
- [REPL Bridge远程控制集成](../sources/2026-04-15-repl-bridge.md)

## Open Questions

- 会话状态的持久化策略？
- 多会话并发时的资源管理？

## Sources

- `chapters/chapter-29-bridge系统架构.md`
- `chapters/chapter-30-repl-bridge.md`