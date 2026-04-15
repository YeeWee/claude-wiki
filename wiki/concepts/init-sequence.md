---
title: "Init Sequence"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-02-入口点与启动流程.md]
related: [../concepts/cli-architecture.md, ../concepts/settings-layers.md]
---

Init Sequence 是 Claude Code 的核心初始化序列，由 init.ts 文件实现，使用 memoize 包装确保只执行一次。序列包括配置系统启用、环境变量应用、网络配置、遥测初始化等关键步骤，通过精心编排的顺序确保各组件正确初始化。

## 设计原则

### Memoize 包装

```typescript
export const init = memoize(async (): Promise<void> => {
  // 单次执行，后续调用直接返回
})
```

确保初始化只执行一次，避免重复加载。

### 错误处理

```typescript
try {
  // 初始化序列
} catch (error) {
  if (error instanceof ConfigParseError) {
    if (getIsNonInteractiveSession()) {
      process.stderr.write(`Configuration error: ${error.message}\n`);
      gracefulShutdownSync(1);
    } else {
      return import('../components/InvalidConfigDialog.js').then(m =>
        m.showInvalidConfigDialog({ error })
      );
    }
  }
  throw error;
}
```

区分交互和非交互模式的错误处理。

## 初始化步骤

### 核心序列

| 步序 | 操作 | 说明 |
|------|------|------|
| 1 | enableConfigs() | 启用配置系统 |
| 2 | applySafeConfigEnvironmentVariables() | 安全环境变量 |
| 3 | applyExtraCACertsFromConfig() | CA 证书（TLS 前必须） |
| 4 | setupGracefulShutdown() | 优雅退出 |
| 5 | initialize1PEventLogging() | 遥测初始化 |
| 6 | populateOAuthAccountInfoIfNeeded() | OAuth 信息 |
| 7 | initJetBrainsDetection() | JetBrains IDE 检测 |
| 8 | configureGlobalMTLS() | mTLS 配置 |
| 9 | configureGlobalAgents() | 代理配置 |
| 10 | preconnectAnthropicApi() | API 预连接 |
| 11 | registerCleanup() | 清理注册 |

### 网络配置顺序

```
CA 证书 → mTLS → 代理 → API 预连接
```

`applyExtraCACertsFromConfig()` 必须在任何 TLS 连接之前执行。

### 预连接优化

`preconnectAnthropicApi()` 与后续初始化并行执行，减少首次 API 请求延迟。

## 遥测初始化

### 延迟到信任对话框之后

```typescript
export function initializeTelemetryAfterTrust(): void {
  if (isEligibleForRemoteManagedSettings()) {
    void waitForRemoteManagedSettingsToLoad()
      .then(async () => {
        applyConfigEnvironmentVariables();  // 包含远程设置
        await doInitializeTelemetry();
      });
  } else {
    void doInitializeTelemetry();
  }
}
```

延迟加载避免启动时加载 ~400KB 的 OpenTelemetry 模块。

## Connections

- [CLI Architecture](../concepts/cli-architecture.md) - 整体架构
- [Settings Layers](../concepts/settings-layers.md) - 配置系统

## Open Questions

- 各 checkpoint 的具体耗时数据
- init 函数的完整依赖图

## Sources

- `chapters/chapter-02-入口点与启动流程.md`