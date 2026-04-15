---
title: "远程设置同步"
type: source
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-43-远程设置同步.md]
related: [../concepts/remote-managed-settings.md, ../concepts/checksum-validation.md, ../concepts/graceful-degradation.md]
---

远程设置同步（Remote Managed Settings）是 Claude Code 为企业用户提供的一项核心功能，允许企业管理员通过远程 API 统一管理和分发配置。该系统实现了设置的中心化管理，支持企业级部署场景下的配置同步和安全管控。

## 核心特性

- **中心化管理**：企业可通过 API 统一推送配置，无需逐台设备手动配置
- **Checksum 校验**：基于 SHA256 的 ETag 缓存机制，减少不必要的网络请求
- **优雅降级**：网络故障时自动使用本地缓存，确保系统可用性
- **安全审计**：危险配置变更时弹出确认对话框，防止恶意配置注入
- **后台轮询**：每小时自动检查更新，支持会话期间的实时同步

## 架构设计

系统将缓存状态拆分为两个模块以打破循环依赖：

- `syncCacheState.ts`：叶子模块，无 auth 导入，负责缓存状态管理
- `syncCache.ts`：资格检查模块，依赖 auth

## 资格判定规则

| 用户类型 | 订阅类型 | 资格 |
|---------|---------|-----|
| Console（API Key） | 任意 | 符合 |
| OAuth | Enterprise/Team | 符合 |
| OAuth | subscriptionType=null | 符合（API 决定） |
| OAuth | 其他订阅 | 不符合 |
| 第三方 Provider | 任意 | 不符合 |
| 自定义 Base URL | 任意 | 不符合 |

## 同步流程

同步流程遵循 HTTP ETag 缓存机制：

1. 检查用户资格
2. 从本地缓存加载设置
3. 向远程 API 发送请求（带 If-None-Match 头）
4. 处理响应：304 使用缓存，200 应用新设置
5. 安全检查危险配置变更
6. 后台轮询每小时检查更新

## 安全机制

危险设置包括：
- Shell 设置（如 `apiKeyHelper`, `mcpServers`）
- 非安全列表的环境变量
- Hooks 配置

## Connections

- [远程管理设置](../concepts/remote-managed-settings.md)
- [Checksum 校验](../concepts/checksum-validation.md)
- [优雅降级](../concepts/graceful-degradation.md)

## Open Questions

- 如何处理多设备并发修改冲突？
- 远程设置与本地设置的优先级如何确定？

## Sources

- `chapters/chapter-43-远程设置同步.md`