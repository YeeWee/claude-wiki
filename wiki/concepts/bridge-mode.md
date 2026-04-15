---
title: "Bridge Mode"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-02-入口点与启动流程.md, chapter-03-feature-flag与构建变体.md]
related: [../concepts/fast-path.md, ../concepts/daemon-mode.md, ../entities/growthbook.md]
---

Bridge 模式是 Claude Code 的远程控制功能，允许将本地机器作为远程开发环境，让用户通过 claude.ai Web 界面操作本地 CLI。通过严格的检查顺序（认证 → 功能门控 → 版本 → 组织策略）确保安全。

## 设计目的

### 远程控制能力

将本地 CLI 作为远程环境的执行端：

- 用户在 claude.ai Web 界面发起请求
- 本地 CLI 执行实际操作（文件读写、命令执行）
- 结果通过 WebSocket 同步到 Web 界面

### 安全设计

检查顺序体现安全优先原则：

1. **认证优先**：OAuth 检查必须在 GrowthBook 之前
2. **功能门控**：检查运行时功能开关
3. **版本兼容**：确保客户端版本满足最低要求
4. **组织策略**：最终的安全护栏

## 初始化流程

```typescript
// cli.tsx
if (feature('BRIDGE_MODE') && args[0] === 'remote-control') {
  profileCheckpoint('cli_bridge_path');
  
  enableConfigs();
  
  // 1. OAuth 认证检查（必须在 GrowthBook 之前）
  if (!getClaudeAIOAuthTokens()?.accessToken) {
    exitWithError(BRIDGE_LOGIN_ERROR);
  }
  
  // 2. GrowthBook 功能门控
  const disabledReason = await getBridgeDisabledReason();
  if (disabledReason) { exitWithError(disabledReason); }
  
  // 3. 版本检查
  const versionError = checkBridgeMinVersion();
  if (versionError) { exitWithError(versionError); }
  
  // 4. 组织策略检查
  await waitForPolicyLimitsToLoad();
  if (!isPolicyAllowed('allow_remote_control')) {
    exitWithError("Error: Remote Control is disabled by policy.");
  }
  
  await bridgeMain(args.slice(1));
  return;
}
```

## 功能门控

### 构建时 + 运行时双重门控

```typescript
export function isBridgeEnabled(): boolean {
  return feature('BRIDGE_MODE')  // 构建时门控
    ? isClaudeAISubscriber() &&
        getFeatureValue_CACHED_MAY_BE_STALE('tengu_ccr_bridge', false)  // 运行时门控
    : false
}
```

- `feature('BRIDGE_MODE')`：构建时，external 构建返回 false
- `getFeatureValue_CACHED_MAY_BE_STALE()`：运行时，GrowthBook 动态配置

## Connections

- [Fast Path](../concepts/fast-path.md) - Bridge 作为快速路径之一
- [Daemon Mode](../concepts/daemon-mode.md) - 对比另一种后台模式
- [GrowthBook](../entities/growthbook.md) - 运行时功能门控

## Open Questions

- WebSocket 连接的稳定性和重连机制
- Bridge 模式下的权限管理策略

## Sources

- `chapters/chapter-02-入口点与启动流程.md`
- `chapters/chapter-03-feature-flag与构建变体.md`