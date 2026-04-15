---
title: "Feature Flag 与构建变体"
type: source
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-03-feature-flag与构建变体.md]
related: [../entities/growthbook.md, ../concepts/feature-flag.md, ../concepts/dead-code-elimination.md, ../concepts/build-variants.md]
---

Claude Code 使用 Bun bundler 的 feature flag 机制实现构建时功能开关，配合死代码消除（DCE）技术区分 ant（内部）和 external（公开发布）两种构建变体。这套系统使工具能灵活管理内外版本的功能差异，同时保持构建产物的精简高效。

## Feature Flag 机制

### bun:bundle feature 函数

`feature()` 函数来自 Bun 的 bundle 系统，工作原理：

1. **构建时注入**：Bun bundler 在编译阶段读取构建配置
2. **静态替换**：将 `feature('FLAG_NAME')` 替换为布尔常量
3. **DCE 优化**：基于替换后的常量进行死代码消除

### 正面模式（推荐）

```typescript
// 正面模式：条件为真时执行，否则返回默认值
return feature('BRIDGE_MODE')
  ? enabledCode()
  : false
```

优势：当功能未启用时，整个表达式和相关字符串字面量都会被消除。

### 反面模式（不推荐）

```typescript
if (!feature('BRIDGE_MODE')) {
  return  // 字符串 'BRIDGE_MODE' 仍会保留在构建产物中
}
```

## Dead Code Elimination

### 构建变体字符串替换

使用巧妙的技术区分构建变体：

```typescript
// 构建时，"external" 被替换为实际的 USER_TYPE 值
if ("external" === 'ant') {
  // ant 构建：保留；external 构建：移除
}
```

### 条件导入模式

对于敏感模块使用条件 `require`：

```typescript
const REPLTool =
  process.env.USER_TYPE === 'ant'
    ? require('./tools/REPLTool.js').REPLTool
    : null
```

### DCE 的限制

Bun bundler 的 `feature()` 评估器有每个函数的复杂度预算限制。解决方案：将导入别名移到顶层 `const` 语句。

## 构建变体差异

### ant vs external

| 变体 | USER_TYPE | 说明 |
|------|-----------|------|
| ant | 'ant' | Anthropic 内部员工版本 |
| external | 'external' | 公开发布版本 |

### ant-only 功能

- 专属工具：ConfigTool、TungstenTool、REPLTool
- 环境变量权限：可访问更多安全环境变量
- 分析元数据：额外的调试和分析功能

## 条件加载模式

### 工具集条件加载

```typescript
return [
  BaseTools,
  ...(process.env.USER_TYPE === 'ant' ? [ConfigTool] : []),
  ...(feature('FLAG') ? [OptionalTool] : []),
]
```

### CLI 快速路径

```typescript
if (feature('DUMP_SYSTEM_PROMPT') && args[0] === '--dump-system-prompt') {
  // 整个分支在 external 构建中被移除
}
```

### 模块级条件加载

敏感模块使用动态 import 完全隔离：

```typescript
if (feature('COMMIT_ATTRIBUTION')) {
  void import('../utils/gitAttribution.js').then(module => {
    module.setupGitAttribution()
  })
}
```

## GrowthBook 动态配置

GrowthBook 提供运行时动态配置，支持缓存机制：

```typescript
// 非阻塞获取功能值，可能使用缓存
export function getFeatureValue_CACHED_MAY_BE_STALE<T>(
  feature: string,
  defaultValue: T,
): T
```

ant 用户可通过 `/config` 命令或环境变量覆盖配置。

## 最佳实践

### Feature Flag 使用规范

1. 使用正面模式：`feature('FLAG') ? enabledCode : defaultValue`
2. 保持 feature() 内联：不要将结果存储到变量
3. 动态导入敏感模块：使用 `await import()` 或 `require()`
4. 避免复杂导入别名：将别名移到顶层 `const`

### 构建变体区分规范

1. 使用 `"external" === 'ant'` 而非 `process.env.USER_TYPE === 'ant'`
2. 在注释中标注 `[ANT-ONLY]`
3. `process.env.USER_TYPE === 'ant'` 用于运行时判断

## Connections

- [GrowthBook](../entities/growthbook.md) - 动态配置平台
- [Feature Flag](../concepts/feature-flag.md) - 功能开关机制详解
- [Dead Code Elimination](../concepts/dead-code-elimination.md) - DCE 技术原理
- [Build Variants](../concepts/build-variants.md) - 构建变体系统

## Open Questions

- DCE 复杂度预算的具体限制阈值
- GrowthBook 缓存过期策略的实现细节
- 如何在 external 构建中完全隔离敏感字符串

## Sources

- `chapters/chapter-03-feature-flag与构建变体.md`