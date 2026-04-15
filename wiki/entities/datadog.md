---
title: "Datadog"
type: entity
created: 2026-04-15
updated: 2026-04-15
sources: ["chapter-28-分析与服务.md"]
related: []
---

Datadog 是面向外部用户的日志分析平台，用于监控 Claude Code 的错误和性能指标。

## 事件过滤

只记录预定义的事件类型，包括：
- chrome_bridge_connection_succeeded/failed
- tengus_api_error/success
- tengus_init/exit
- tengus_uncaught_exception/unhandled_rejection
- 工具使用事件

## 基数控制

为降低存储成本，实施基数控制：

1. **MCP 工具名归一化**: 所有 MCP 工具名映射为 `mcp`
2. **模型名缩短**: 外部用户的模型名映射为短名或 `other`
3. **版本号截断**: 开发版本只保留基础版本和日期部分

## 用户分桶

通过 SHA256 哈希将用户 ID 分配到 30 个固定桶中，用于估算受影响用户数量，同时保护用户隐私。

## 端点

日志发送到 `logs.us5.datadoghq.com`。

## 熔断机制

可通过 GrowthBook 动态配置 `tengu_frond_boric` 独立禁用 Datadog 导出。

## Connections

- [事件日志](../concepts/event-logging.md)
- [分析与服务](../sources/2026-04-15-分析与服务.md)

## Open Questions

- 基数控制的粒度如何进一步优化？
- Datadog 集成的可扩展性？

## Sources

- `chapters/chapter-28-分析与服务.md`