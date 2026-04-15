---
title: "Dead Code Elimination"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-03-feature-flag与构建变体.md]
related: [../entities/bun.md, ../concepts/feature-flag.md, ../concepts/build-variants.md]
---

Dead Code Elimination（DCE）是 Bun bundler 的优化技术，通过静态分析移除未使用的代码分支。配合 feature flag 机制，实现 ant 和 external 构建变体的功能隔离，确保公开发布版本不包含内部功能代码。

## 工作原理

### 静态替换

构建时，`feature()` 函数被替换为布尔常量：

```typescript
// 构建前
if (feature('DUMP_SYSTEM_PROMPT')) { ... }

// ant 构建后
if (true) { ... }  // 保留

// external 构建后
if (false) { ... }  // 整个分支被移除
```

### 构建变体字符串替换

```typescript
// 构建时 "external" 被替换为实际的 USER_TYPE 值
if ("external" === 'ant') { ... }
```

- ant 构建：`"ant" === 'ant'` → true，保留代码
- external 构建：`"external" === 'ant'` → false，移除代码

## DCE 模式

### 条件导入

```typescript
const REPLTool = process.env.USER_TYPE === 'ant'
  ? require('./REPLTool.js').REPLTool
  : null

const tools = [
  BaseTools,
  ...(REPLTool ? [REPLTool] : []),
]
```

### 动态导入敏感模块

```typescript
if (feature('COMMIT_ATTRIBUTION')) {
  void import('../utils/gitAttribution.js').then(module => {
    module.setupGitAttribution()
  })
}
```

完全隔离敏感字符串，不在构建产物中出现。

## DCE 的限制

### 复杂度预算

Bun bundler 有每个函数的复杂度预算限制：

```typescript
// DCE cliff: Bun's feature() evaluator has a per-function complexity budget.
// 导入别名计入预算，超过阈值时 feature() 无法被证明为常量
```

解决方案：将导入别名移到顶层 `const`。

### 正面模式优先

反面模式可能导致字符串字面量保留：

```typescript
// 不推荐：字符串 'BRIDGE_MODE' 可能保留
if (!feature('BRIDGE_MODE')) { return }
```

## Connections

- [Bun](../entities/bun.md) - DCE 提供者
- [Feature Flag](../concepts/feature-flag.md) - 触发 DCE 的机制
- [Build Variants](../concepts/build-variants.md) - DCE 的应用场景

## Open Questions

- 复杂度预算的具体阈值
- DCE 对构建产物大小的影响量化

## Sources

- `chapters/chapter-03-feature-flag与构建变体.md`