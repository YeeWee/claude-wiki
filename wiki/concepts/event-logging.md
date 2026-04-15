---
title: "事件日志"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: ["chapter-28-分析与服务.md"]
related: []
---

事件日志是 Claude Code 数据收集和分析的核心机制，基于 OpenTelemetry 实现多通道导出。

## 双通道导出

- **Datadog**: 面向外部用户的日志分析
- **第一方事件**: Anthropic 内部分析平台

## 事件采样

通过 GrowthBook 动态配置 `tengu_event_sampling_config` 控制。未配置事件 100% 记录，采样率 0 丢弃，采样率 1 全记录。

## 元数据丰富化

自动附加三类元数据：
- core_metadata: 模型、会话、用户类型、beta 特性
- user_metadata: 账户 UUID、组织 UUID、邮箱
- envContext: 平台、架构、终端类型、CI 状态

## 基数控制

Datadog 专用：
- MCP 工具名归一化
- 模型名缩短
- 版本号截断
- 用户分桶（30 个）

## 熔断机制

单 Sink 独立控制，配置名 `tengu_frond_boric`。熔断时第一方事件写入磁盘 JSONL 文件持久化。

## 失败处理

第一方导出器特性：
- 二次方退避重试
- 认证回退（401 时尝试无认证）
- 批次分块

## 主要事件类型

- 工具使用: tengus_tool_use_success/error
- API: tengus_api_success/error
- 会话: tengus_init/started/exit/cancel
- OAuth: tengus_oauth_success/error
- 异常: tengus_uncaught_exception/unhandled_rejection

## Connections

- [GrowthBook](../entities/growthbook.md)
- [Datadog](../entities/datadog.md)
- [OpenTelemetry](../entities/opentelemetry.md)
- [Feature Flag](../concepts/feature-flag.md)
- [分析与服务](../sources/2026-04-15-分析与服务.md)

## Open Questions

- 事件归档策略？
- 跨通道事件关联？

## Sources

- `chapters/chapter-28-分析与服务.md`