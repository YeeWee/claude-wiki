---
title: "远程管理设置"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-43-远程设置同步.md]
related: [../sources/2026-04-15-remote-managed-settings-sync.md]
---

远程管理设置（Remote Managed Settings）是 Claude Code 为企业用户提供的一项核心功能，允许企业管理员通过远程 API 统一管理和分发配置。

## 核心特性

- **中心化管理**：企业可通过 API 统一推送配置
- **Checksum 校验**：基于 SHA256 的 ETag 缓存机制
- **优雅降级**：网络故障时自动使用本地缓存
- **安全审计**：危险配置变更时弹出确认对话框
- **后台轮询**：每小时自动检查更新

## 资格判定

符合资格的用户类型：

- Console 用户（API Key）
- OAuth Enterprise/Team 用户
- 特定条件的 OAuth 用户

不符合资格：

- 第三方 Provider 用户
- 自定义 Base URL 用户
- Cowork 环境用户

## 安全机制

危险设置包括：
- Shell 设置（apiKeyHelper, mcpServers）
- 非安全列表的环境变量
- Hooks 配置

## Connections

- [远程设置同步源页面](../sources/2026-04-15-remote-managed-settings-sync.md)

## Open Questions

- 多设备并发修改如何处理？
- 远程与本地设置的优先级？

## Sources

- `chapters/chapter-43-远程设置同步.md`