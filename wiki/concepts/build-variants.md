---
title: "Build Variants"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-03-feature-flag与构建变体.md]
related: [../concepts/feature-flag.md, ../concepts/dead-code-elimination.md, ../entities/anthropic.md]
---

Build Variants（构建变体）是 Claude Code 区分内部和公开发布版本的技术，通过 USER_TYPE 环境变量控制 feature flag 配置，配合 DCE 移除未启用代码，实现 ant（内部）和 external（公开发布）两种构建产物。

## 变体类型

### ant vs external

| 变体 | USER_TYPE | 说明 |
|------|-----------|------|
| ant | 'ant' | Anthropic 内部员工版本 |
| external | 'external' | 公开发布版本 |

### 功能差异

ant-only 功能：

- 专属工具：ConfigTool、TungstenTool、REPLTool
- 环境变量权限：ANT_ONLY_SAFE_ENV_VARS 集合
- 分析元数据：额外的调试和日志
- `/config` 命令：可覆盖 GrowthBook 配置

## 构建时区分

### 字符串替换技术

```typescript
// 构建时 "external" 被替换为实际的 USER_TYPE 值
if ("external" === 'ant') {
  // ant 构建："ant" === 'ant' → true，保留
  // external 构建："external" === 'ant' → false，移除
}
```

### 运行时判断

```typescript
// 运行时检查 USER_TYPE 环境变量
if (process.env.USER_TYPE === 'ant') {
  // ant-only 功能
}
```

## ant-only 功能示例

### 工具加载

```typescript
const tools = [
  ...(process.env.USER_TYPE === 'ant' ? [ConfigTool] : []),
  ...(process.env.USER_TYPE === 'ant' && REPLTool ? [REPLTool] : []),
]
```

### 环境变量权限

```typescript
const ANT_ONLY_SAFE_ENV_VARS = new Set([
  'ANTHROPIC_API_KEY',
  'CLAUDE_CODE_MODEL',
  // ...更多内部环境变量
])

if (process.env.USER_TYPE === 'ant' && ANT_ONLY_SAFE_ENV_VARS.has(varName)) {
  // ant 用户可访问
}
```

### 分析元数据

```typescript
if (process.env.USER_TYPE === 'ant') {
  logForDebugging(`[ANT-ONLY] 1P event: ${eventName}`)
}
```

## Connections

- [Feature Flag](../concepts/feature-flag.md) - 控制变体的机制
- [Dead Code Elimination](../concepts/dead-code-elimination.md) - 移除代码的技术
- [Anthropic](../entities/anthropic.md) - 内部版本用户

## Open Questions

- ant-only 功能的完整列表
- 如何防止 ant 功能意外泄露到 external 构建

## Sources

- `chapters/chapter-03-feature-flag与构建变体.md`