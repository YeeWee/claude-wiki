---
title: "WebFetchTool"
type: entity
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-13-web工具集.md]
related: []
---

WebFetchTool 是 Claude Code 的网页抓取工具，从指定 URL 获取内容并转换为 Markdown 格式。

## 核心功能

- URL 内容抓取和 HTML 转 Markdown
- 使用 Haiku 模型处理和总结内容
- 多层权限检查机制
- LRU 缓存优化响应速度

## 配置

| 配置项 | 值 |
|--------|-----|
| name | 'WebFetch' |
| maxResultSizeChars | 100_000 |
| shouldDefer | true |
| isConcurrencySafe | true |
| isReadOnly | true |

## 输入 Schema

```typescript
{
  url: string.url(),  // 必须是有效 URL
  prompt: string      // 内容处理提示词
}
```

## 输出 Schema

```typescript
{
  bytes: number,      // 内容大小
  code: number,       // HTTP 响应码
  codeText: string,   // HTTP 响应文本
  result: string,     // 处理后的结果
  durationMs: number, // 执行时间
  url: string         // 实际获取的 URL
}
```

## 安全机制

### 预批准域名

自动允许的官方文档站点，包括：
- Anthropic: platform.claude.com, modelcontextprotocol.io
- 编程语言: docs.python.org, go.dev, doc.rust-lang.org
- Web 框架: react.dev, vuejs.org, nextjs.org
- 云服务: docs.aws.amazon.com, cloud.google.com

### 重定向限制

禁止跨域自动跟随重定向：
- 允许: example.com → www.example.com
- 禁止: 跨域、协议变化、端口变化、认证信息

### 资源限制

- URL 最大长度: 2000 字符
- HTTP 内容最大: 10MB
- 请求超时: 60 秒

## 缓存配置

- TTL: 15 分钟
- 最大大小: 50MB
- 类型: LRU

## Connections

- [WebSearchTool](web-search-tool.md)
- [预批准域名机制](../concepts/preapproved-domains.md)

## Open Questions

- 预批准域名列表的维护策略是什么？

## Sources

- `chapters/chapter-13-web工具集.md`