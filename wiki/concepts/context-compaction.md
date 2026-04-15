---
title: "上下文压缩"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: ["chapter-26-上下文压缩服务.md"]
related: []
---

上下文压缩是 Claude Code 为解决长对话历史超出 API 上下文窗口限制而设计的核心机制。

## 压缩层级

多层渐进式策略，从轻量级到重量级：

| 层级 | 功能 | 触发条件 |
|------|------|----------|
| 微压缩 | 清理旧工具结果 | 缓存 TTL 过期 |
| 自动压缩 | 监控阈值触发 | token 达阈值 |
| Session Memory 压缩 | 使用预提取记忆 | Session Memory 可用 |
| 主压缩 | API 生成摘要 | 其他方式不足 |

## 触发机制

三种触发方式：

1. **自动触发**: token 使用量达到阈值（约 200K - 13K buffer）
2. **时间触发**: 距上次助手消息超过 API 缓存 TTL（默认 60 分钟）
3. **手动触发**: 用户执行 `/compact` 命令

## Token 预算

多级阈值：
- 警告阈值: threshold - 20K
- 错误阈值: threshold - 20K
- 阻塞限制: contextWindow - 3K

## 摘要结构

九部分模板：
1. Primary Request and Intent
2. Key Technical Concepts
3. Files and Code Sections
4. Errors and fixes
5. Problem Solving
6. All user messages
7. Pending Tasks
8. Current Work
9. Optional Next Step

## 工具结果处理

可压缩工具：FileRead, Shell, Grep, Glob, WebSearch, WebFetch, Edit, Write

两种路径：
- 时间微压缩：固定标记替换
- 缓存编辑路径：API cache_edits

## 历史裁剪

按 API 轮次分组，确保工具配对完整。Prompt-Too-Long 时裁剪最老组。

## 边界标记

压缩完成后生成边界标记，标识压缩发生点，携带已发现工具状态和保留片段元数据。

## Connections

- [Token预算](../concepts/token-budget.md)
- [Session Memory](../concepts/session-memory.md)
- [上下文压缩服务](../sources/2026-04-15-上下文压缩服务.md)

## Open Questions

- 智能信息保留策略的优化？
- 压缩摘要质量评估方法？

## Sources

- `chapters/chapter-26-上下文压缩服务.md`