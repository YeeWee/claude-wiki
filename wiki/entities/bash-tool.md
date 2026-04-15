---
title: "BashTool"
type: entity
created: 2026-04-15
updated: 2026-04-15
sources: [chapters/chapter-09-bashtool深度解析.md]
related: [../concepts/sandbox-isolation.md, ../concepts/async-generator-pattern.md]
---

# BashTool

BashTool 是 Claude Code 中使用频率最高的工具，负责执行 Shell 命令，扮演"执行引擎"角色。据统计，在典型开发场景中调用占比超过 60%。

## 核心职责

1. **命令执行**：将 Claude 生成的 Shell 命令转化为实际操作
2. **安全隔离**：通过沙箱机制防止恶意命令损害系统
3. **权限控制**：确保每条命令都在用户授权范围内执行
4. **输出处理**：实时捕获命令输出，为 Claude 提供反馈
5. **后台执行**：支持长时间运行任务的异步处理

## 关键特性

### 执行流程

采用 AsyncGenerator 模式，实现实时进度、可控中断、后台切换。

### 安全机制

双重防护：权限检查（应用层）+ 沙箱隔离（系统层）。

### 后台执行

三种触发方式：
- 显式请求：`run_in_background: true`
- 自动超时：命令超过 15 秒
- 用户手动：Ctrl+B 快捷键

## Connections

- [Sandbox 隔离机制](../concepts/sandbox-isolation.md) - 安全隔离核心
- [AsyncGenerator 模式](../concepts/async-generator-pattern.md) - 执行流程模式
- [Tool 接口](./tool-interface.md) - 继承自 Tool 类型

## Open Questions

- 沙箱排除命令是否需要更细粒度的控制？
- 自动后台化的阈值是否可配置？

## Sources

- [第九章：BashTool 深度解析](../sources/2026-04-15-bashtool深度解析.md)