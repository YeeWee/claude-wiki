---
title: "GrowthBook"
type: entity
created: 2026-04-15
updated: 2026-04-15
sources: ["chapter-28-分析与服务.md"]
related: []
---

GrowthBook 是 Claude Code 的特性开关管理平台，通过远程评估实现动态配置下发。

## 客户端配置

关键配置参数：
- `remoteEval: true`: 服务端预计算特性值，避免客户端重复评估
- `cacheKeyAttributes`: ['id', 'organizationUUID'] - 当用户 ID 或组织变更时触发重新拉取
- `apiHostRequestHeaders`: 认证头用于私有特性开关访问

## 用户属性

用于特性开关的定向投放：
- id: 设备 ID
- sessionId: 会话 ID
- platform: win32/darwin/linux
- organizationUUID: 组织 UUID
- accountUUID: 账户 UUID
- userType: 用户类型
- subscriptionType: 订阅类型
- rateLimitTier: 速率限制层级

## 缓存策略

缓存读取优先级：
1. 环境变量覆盖 (`CLAUDE_INTERNAL_FC_OVERRIDES`)
2. 配置覆盖 (`growthBookOverrides`)
3. 内存 payload 缓存 (`remoteEvalFeatureValues`)
4. 砬盘缓存 (`cachedGrowthBookFeatures`)

## 定期刷新

支持定期刷新特性值，确保长时间运行的会话能获取最新配置。刷新完成后触发 `onGrowthBookRefresh` 信号。

## 特性值获取方法

| 方法 | 特点 | 适用场景 |
|------|------|----------|
| `getFeatureValue_CACHED_MAY_BE_STALE` | 立即返回磁盘缓存 | 启动关键路径 |
| `checkSecurityRestrictionGate` | 等待重初始化完成 | 安全关键检查 |

## Connections

- [Feature Flag](../concepts/feature-flag.md)
- [分析与服务](../sources/2026-04-15-分析与服务.md)

## Open Questions

- 如何处理特性开关的网络故障？
- 特性开关的回滚机制？

## Sources

- `chapters/chapter-28-分析与服务.md`