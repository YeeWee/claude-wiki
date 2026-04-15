---
title: "Sandbox 隔离机制"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: [chapters/chapter-09-bashtool深度解析.md]
related: [../entities/bash-tool.md, ../entities/sandbox-manager.md]
---

# Sandbox 隔离机制

Sandbox 隔离是 Claude Code 安全系统的核心，通过操作系统级别的隔离防止恶意命令损害用户系统。

## 架构概述

双重防护策略：
- 权限检查（应用层）：规则匹配、路径约束、模式检查
- 沙箱隔离（系统层）：文件系统隔离、网络隔离

## 平台实现

| 平台 | 技术 | 说明 |
|------|------|------|
| macOS | seatbelt | Apple 安全框架 |
| Linux/WSL2 | bubblewrap | 容器化隔离 |

## 启用判断流程

1. 检查全局启用状态
2. 检查是否明确禁用
3. 检查命令是否存在
4. 检查排除命令列表

## 排除命令机制

用户可配置不需要沙箱的命令：
- 检查命令子字符串匹配
- 检查命令开头匹配
- 剥离环境变量和安全包装器

**重要提示**：排除命令机制是用户便利功能，而非安全边界。

## 配置转换

- 文件系统路径：allowWrite、denyWrite、denyRead
- 网络域名：allowedDomains、deniedDomains
- 始终拒绝写入 settings.json（防止沙箱逃逸）

## Connections

- [BashTool](../entities/bash-tool.md) - 使用沙箱
- [SandboxManager](../entities/sandbox-manager.md) - 适配层实现

## Sources

- [第九章：BashTool 深度解析](../sources/2026-04-15-bashtool深度解析.md)