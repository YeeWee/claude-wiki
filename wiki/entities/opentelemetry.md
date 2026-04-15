---
title: "OpenTelemetry"
type: entity
created: 2026-04-15
updated: 2026-04-15
sources: ["chapter-28-分析与服务.md"]
related: []
---

OpenTelemetry 是 Claude Code 分析系统的核心技术框架，提供 Metrics、Logs、Traces 三种遥测信号支持。

## 三种信号

| 信号 | 环境变量 | 导出器选项 |
|------|----------|------------|
| Metrics | `OTEL_METRICS_EXPORTER` | OTLP, Prometheus, Console |
| Logs | `OTEL_LOGS_EXPORTER` | OTLP, Console |
| Traces | `OTEL_TRACES_EXPORTER` | OTLP, Console |

## OTLP 协议

支持三种传输方式：
- grpc: gRPC 协议
- http/json: HTTP JSON 协议
- http/protobuf: HTTP Protobuf 协议（默认）

## LoggerProvider

第一方事件日志使用 BatchLogRecordProcessor：
- scheduledDelayMillis: 批处理延迟
- maxExportBatchSize: 最大批次大小
- maxQueueSize: 队列容量

## BigQuery 指标导出

内部用户和 API 客户自动启用 BigQuery 指标导出。

## Beta 调试追踪

使用独立端点配置 `BETA_TRACING_ENDPOINT`，与主 OTLP 端点分离。

## Connections

- [事件日志](../concepts/event-logging.md)
- [分析与服务](../sources/2026-04-15-分析与服务.md)

## Open Questions

- 如何优化 OTLP 导出性能？
- 多信号关联分析的实现？

## Sources

- `chapters/chapter-28-分析与服务.md`