---
title: "Bridge系统"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: ["chapter-29-bridge系统架构.md", "chapter-30-repl-bridge.md"]
related: []
---

Bridge 系统是 Claude Code 远程控制功能的核心架构，实现"本地执行、远程控制"的创新模式。

## 设计理念

建立持久化服务端点，作为本地环境与云端服务的桥梁，打破传统 CLI 的物理边界。

## 核心场景

- 跨设备连续性：多设备无缝切换
- 协作共享：团队会话进度查看
- 移动办公：远程审批无需 SSH

## 架构原则

- 本地优先执行：敏感操作在本地
- 云端控制面：用户界面托管云端
- 会话隔离：worktree 或目录隔离
- 优雅降级：网络中断时继续运行

## 会话启动模式

| 模式 | 特点 |
|------|------|
| single-session | 单会话，结束时退出 |
| same-dir | 多会话共享目录 |
| worktree | 每会话独立 worktree |

决策优先级：恢复会话 > 命令行标志 > 项目配置 > 功能开关默认值。

## Work 机制

轮询获取任务：
- session: 启动本地会话进程
- healthcheck: 健康检查确认活跃

Work Secret 解码获取会话凭证（JWT）。

## 心跳机制

关键作用：
- 延长租约防止超时回收
- 认证检测触发 token 刷新
- 健康监控验证可用性

## 重连机制

两阶段策略：
1. 原地重连：tryReconnectInPlace
2. 新建会话：archiveSession + createSession

崩溃恢复指针（bridgePointer）支持进程崩溃后恢复。

## 优雅关闭

步骤：
1. SIGTERM → SIGKILL 终止子进程
2. 清理 Worktree
3. 停止 Work 通知服务端
4. 归档会话
5. 注销环境

## Connections

- [WebSocket传输](../concepts/websocket-transport.md)
- [SSE传输](../concepts/sse-transport.md)
- [会话管理](../concepts/session-management.md)
- [REPL Bridge](../concepts/repl-bridge.md)
- [Bridge系统架构](../sources/2026-04-15-bridge系统架构.md)
- [REPL Bridge远程控制集成](../sources/2026-04-15-repl-bridge.md)

## Open Questions

- Bridge 网络稳定性优化？
- 多会话资源调度策略？

## Sources

- `chapters/chapter-29-bridge系统架构.md`
- `chapters/chapter-30-repl-bridge.md`