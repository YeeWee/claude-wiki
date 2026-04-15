---
title: "WebSearchTool"
type: entity
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-13-web工具集.md]
related: []
---

WebSearchTool 使用 Anthropic API 内置搜索能力获取最新网络信息。

## 核心功能

- 网络搜索获取最新信息
- 支持域名过滤
- 流式进度反馈
- 仅对特定 API 提供商启用

## 配置

| 配置项 | 值 |
|--------|-----|
| name | 'WebSearch' |
| maxResultSizeChars | 100_000 |
| shouldDefer | true |
| isConcurrencySafe | true |
| isReadOnly | true |

## 输入 Schema

```typescript
{
  query: string.min(2),           // 搜索查询
  allowed_domains?: string[],     // 仅包含这些域名
  blocked_domains?: string[]      // 排除这些域名
}
```

## 输出 Schema

```typescript
{
  query: string,                  // 执行的搜索查询
  results: SearchResult[] | string[], // 搜索结果或文本评论
  durationSeconds: number         // 执行时间
}
```

## 启用条件

仅对以下用户启用：
- firstParty Anthropic API 用户
- Vertex AI 用户
- Foundry 用户

## Anthropic API 搜索工具 Schema

```typescript
{
  type: 'web_search_20250305',
  name: 'web_search',
  allowed_domains: input.allowed_domains,
  blocked_domains: input.blocked_domains,
  max_uses: 8  // 最大搜索次数
}
```

## Prompt 关键要求

必须在回答后包含 Sources 部分，列出所有相关 URL。

## Connections

- [WebFetchTool](web-fetch-tool.md)
- [Anthropic API](../concepts/anthropic-api.md)

## Open Questions

- 搜索结果的质量如何评估？

## Sources

- `chapters/chapter-13-web工具集.md`