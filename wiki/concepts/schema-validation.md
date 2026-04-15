---
title: "Schema Validation"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-04-配置系统.md]
related: [../concepts/settings-layers.md, ../concepts/merge-strategy.md]
---

Schema Validation 是 Claude Code 配置系统的验证机制，使用 Zod Schema 定义配置结构，确保向后兼容和类型安全。验证失败时保留原始数据供用户修复，而非直接覆盖。

## Zod Schema 定义

### SettingsSchema 核心字段

```typescript
export const SettingsSchema = lazySchema(() =>
  z.object({
    $schema: z.literal(CLAUDE_CODE_SETTINGS_SCHEMA_URL).optional(),
    model: z.string().optional(),
    permissions: PermissionsSchema().optional(),
    env: EnvironmentVariablesSchema().optional(),
    hooks: HooksSchema().optional(),
    sandbox: SandboxSettingsSchema().optional(),
    allowedMcpServers: z.array(...).optional(),
    enabledPlugins: z.record(...).optional(),
  })
)
```

## 向后兼容原则

### 允许的变更

```typescript
/**
 * ✅ ALLOWED:
 * - 添加新可选字段（必须 .optional()）
 * - 添加新枚举值（保留旧值）
 * - 添加对象新属性
 * - 放宽验证规则
 * - 使用 union 类型渐进迁移
 */
```

### 禁止的变更

```typescript
/**
 * ❌ BREAKING:
 * - 删除字段（标记 deprecated 替代）
 * - 删除枚举值
 * - 将可选字段改为必需
 * - 加强验证规则
 * - 重命名字段（保留旧名）
 */
```

## 验证流程

### 解析和验证

```typescript
function parseSettingsFileUncached(path: string) {
  // 1. JSON 解析
  const data = safeParseJSON(content)
  
  // 2. 过滤无效权限规则（防止污染整个文件）
  const ruleWarnings = filterInvalidPermissionRules(data, path)
  
  // 3. Schema 验证
  const result = SettingsSchema().safeParse(data)
  
  if (!result.success) {
    // 返回验证错误，不覆盖文件
    return { settings: null, errors: formatZodError(result.error) }
  }
  
  return { settings: result.data, errors: ruleWarnings }
}
```

### 失败处理

验证失败时：

- 保留原始文件内容
- 输出格式化的错误信息
- 用户可手动修复

## Connections

- [Settings Layers](../concepts/settings-layers.md) - 配置层次
- [Merge Strategy](../concepts/merge-strategy.md) - 合并规则

## Open Questions

- Zod Schema 的完整定义
- 错误格式化的具体实现

## Sources

- `chapters/chapter-04-配置系统.md`