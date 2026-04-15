---
title: "Feature Flag"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources:
  - chapter-01-项目概述与开发环境.md
  - chapter-03-feature-flag与构建变体.md
  - chapter-28-分析与服务.md
related:
  - ../entities/bun.md
  - ../entities/growthbook.md
  - ../concepts/dead-code-elimination.md
  - ../concepts/event-logging.md
---

Feature Flag 是 Claude Code 的功能开关机制，分为构建时（bun:bundle）和运行时（GrowthBook）两种类型。构建时 feature flag 通过静态替换和 DCE 移除未启用代码，运行时 feature flag 通过动态配置支持灰度发布。

## 构建时 Feature Flag

### bun:bundle feature 函数

```typescript
import { feature } from 'bun:bundle'

if (feature('DUMP_SYSTEM_PROMPT')) {
  // ant 构建保留，external 构建移除
}
```

工作原理：

1. 构建时注入：Bun bundler 读取构建配置
2. 静态替换：`feature('FLAG')` → `true` 或 `false`
3. DCE 优化：基于常量消除死代码

### 正面模式（推荐）

```typescript
return feature('BRIDGE_MODE')
  ? enabledCode()
  : defaultValue
```

优势：未启用时，整个表达式和字符串字面量都被消除。

### 构建变体字符串替换

```typescript
// 构建时 "external" 被替换为实际的 USER_TYPE
if ("external" === 'ant') {
  // ant 构建：保留；external 构建：移除
}
```

## 运行时 Feature Flag

### GrowthBook 动态配置

```typescript
// 非阻塞获取，可能使用缓存
const enabled = getFeatureValue_CACHED_MAY_BE_STALE('feature_name', false)
```

#### 远程评估模式

`remoteEval: true` 使服务端预计算特性值，避免客户端重复评估，减少计算开销和延迟。

#### 多层缓存优先级

1. 环境变量覆盖（测试/调试用）
2. 配置覆盖（本地开发）
3. 内存 payload 缓存（运行时）
4. 磁盘缓存（持久化）

#### 定向投放属性

通过用户属性实现精准投放：
- 设备 ID / 会话 ID
- 平台类型
- 组织/账户 UUID
- 用户类型 / 订阅类型
- 速率限制层级

#### 定期刷新机制

长时间运行的会话需要定期刷新获取最新配置。刷新完成后触发信号通知订阅者重建依赖对象。

#### 特性值获取方法

| 方法 | 说明 |
|------|------|
| `getFeatureValue_CACHED_MAY_BE_STALE` | 立即返回缓存，适合启动路径 |
| `getFeatureValue_DEPRECATED` | 阻塞等待，已弃用 |
| `checkSecurityRestrictionGate` | 等待重初始化，适合安全检查 |

#### 熔断配置示例

动态配置名：
- `tengu_event_sampling_config`: 事件采样率
- `tengu_frond_boric`: Sink 熔断开关

适用场景：

- 灰度发布
- 用户级功能开关
- A/B 测试

### ant 配置覆盖

ant 用户可通过环境变量或 `/config` 命令覆盖 GrowthBook 配置。

## Feature Flag 类型

| 类型 | 机制 | 用途 |
|------|------|------|
| 构建 feature() | 静态替换 + DCE | 移除内部功能 |
| 运行 GrowthBook | 动态配置 + 缓存 | 灰度发布 |

## Connections

- [Bun](../entities/bun.md) - bun:bundle 提供者
- [GrowthBook](../entities/growthbook.md) - 运行时配置平台
- [Dead Code Elimination](../concepts/dead-code-elimination.md) - DCE 技术
- [Event Logging](../concepts/event-logging.md) - 熔断与采样配置

## Open Questions

- 完整的 feature flag 列表及其用途
- GrowthBook 缓存过期策略
- 特性开关的审计追踪？
- 配置冲突解决策略？

## Sources

- `chapters/chapter-01-项目概述与开发环境.md`
- `chapters/chapter-03-feature-flag与构建变体.md`
- `chapters/chapter-28-分析与服务.md`