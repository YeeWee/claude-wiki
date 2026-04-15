---
title: "Merge Strategy"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-04-配置系统.md]
related: [../concepts/settings-layers.md, ../concepts/schema-validation.md]
---

Merge Strategy 是 Claude Code 配置系统的合并规则，使用 lodash 的 `mergeWith` 配合自定义合并器实现层次合并。数组合并去重，对象深度合并，基本类型覆盖，undefined 表示删除字段。

## 合并规则

### 自定义合并器

```typescript
export function settingsMergeCustomizer(objValue, srcValue) {
  if (Array.isArray(objValue) && Array.isArray(srcValue)) {
    return mergeArrays(objValue, srcValue)  // 合并 + 去重
  }
  return undefined  // 使用 lodash 默认深度合并
}
```

### 字段类型合并行为

| 字段类型 | 合并行为 | 示例 |
|---------|---------|------|
| 数组 | 合并 + 去重 | `["A", "B"] + ["B", "C"] → ["A", "B", "C"]` |
| 对象 | 深度合并 | 递归合并嵌套属性 |
| 基本类型 | 覆盖 | `model: "opus" 覆盖 "sonnet"` |
| undefined | 删除字段 | `{ key: undefined } 删除 key` |

## Policy Settings 特殊处理

### 首个来源优先

Policy Settings 采用与普通配置相反的策略：

```typescript
export function getPolicySettingsOrigin(): 'remote' | 'plist' | 'hklm' | 'file' | 'hkcu' | null {
  // 检查 Remote API（最高优先级）
  // 检查 MDM/HKLM
  // 检查 File
  // 检查 HKCU（最低优先级）
  
  // 返回首个存在的来源
}
```

一旦高优先级来源存在配置，就不会被低优先级来源覆盖。

## 字段删除语义

### 使用 undefined 删除

```typescript
updateSettingsForSource('userSettings', {
  model: undefined,  // 删除 model 字段
  permissions: {
    allow: undefined  // 删除 permissions.allow
  }
})
```

### 合并时检测删除意图

```typescript
mergeWith(existingSettings, updates, (objValue, srcValue, key, object) => {
  if (srcValue === undefined && typeof key === 'string') {
    delete object[key]  // 删除字段
    return undefined
  }
  if (Array.isArray(srcValue)) {
    return srcValue  // 数组替换而非合并
  }
})
```

## Connections

- [Settings Layers](../concepts/settings-layers.md) - 配置层次
- [Schema Validation](../concepts/schema-validation.md) - 配置验证

## Open Questions

- 深度合并的循环引用处理
- Drop-in 目录合并的冲突解决

## Sources

- `chapters/chapter-04-配置系统.md`