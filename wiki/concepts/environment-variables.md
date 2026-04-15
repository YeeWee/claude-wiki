---
title: "Environment Variables"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-04-配置系统.md]
related: [../concepts/settings-layers.md, ../concepts/schema-validation.md]
---

Environment Variables 处理是 Claude Code 配置系统的安全分级机制，将环境变量分为安全（可在信任前应用）和危险（需信任确认）两类，确保敏感配置不会在未经授权的情况下生效。

## 安全分级

### 安全变量

可在信任对话框确认前应用：

```typescript
export const SAFE_ENV_VARS = new Set([
  'ANTHROPIC_CUSTOM_HEADERS',
  'ANTHROPIC_MODEL',
  'AWS_REGION',
  'CLAUDE_CODE_ENABLE_TELEMETRY',
  'OTEL_EXPORTER_OTLP_PROTOCOL',
])
```

包括：

- Claude Code 特定设置
- 遥测配置
- AWS/Vertex 区域配置

### 危险变量

需信任确认后才能应用：

| 变量 | 风险 |
|------|------|
| `ANTHROPIC_BASE_URL` | 重定向到恶意服务器 |
| `HTTP_PROXY/HTTPS_PROXY` | 拦截流量 |
| `NODE_EXTRA_CA_CERTS` | 信任恶意证书 |
| `PATH/LD_PRELOAD` | 执行恶意代码 |

## 应用流程

### 分阶段应用

```typescript
// 信任前：仅安全变量
export function applySafeConfigEnvironmentVariables(): void {
  // 1. 全局配置的环境变量
  // 2. 可信来源的所有环境变量
  // 3. 项目级来源的安全环境变量
}

// 信任后：所有变量
export function applyConfigEnvironmentVariables(): void {
  Object.assign(process.env, filterSettingsEnv(getSettings()?.env))
}
```

## Host-Managed Provider 保护

当宿主进程设置 `CLAUDE_CODE_PROVIDER_MANAGED_BY_HOST` 时：

```typescript
function withoutHostManagedProviderVars(env) {
  // 剥离提供商路由相关环境变量
  if (!isEnvTruthy(process.env.CLAUDE_CODE_PROVIDER_MANAGED_BY_HOST)) {
    return env
  }
  // 过滤掉提供商管理的变量
}
```

确保宿主配置的提供商路由不被用户配置覆盖。

## Connections

- [Settings Layers](../concepts/settings-layers.md) - 配置层次
- [Schema Validation](../concepts/schema-validation.md) - 配置验证

## Open Questions

- SAFE_ENV_VARS 的完整列表
- Provider Managed 变量的具体定义

## Sources

- `chapters/chapter-04-配置系统.md`