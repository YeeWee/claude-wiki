---
title: "ConfigTool"
type: entity
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-17-其他工具精选.md]
related: []
---

ConfigTool 提供动态配置管理能力，支持读取和修改 Claude Code 设置。

## 核心功能

- GET/SET 配置操作
- 全局设置和项目设置分离
- 即时 UI 效果

## 配置

| 配置项 | 值 |
|--------|-----|
| name | 'Config' |
| maxResultSizeChars | 100_000 |
| shouldDefer | true |
| isConcurrencySafe | true |

## 输入 Schema

```typescript
{
  setting: string,        // 设置键，如 "theme", "model"
  value?: string | boolean | number  // 新值，省略则读取
}
```

## 输出 Schema

```typescript
{
  success: boolean,
  operation?: 'get' | 'set',
  setting?: string,
  value?: unknown,
  previousValue?: unknown,
  newValue?: unknown,
  error?: string
}
```

## 支持的设置项

### 全局设置（`~/.claude.json`）

| 设置 | 类型 | 说明 |
|------|------|------|
| theme | string | UI 颜色主题 |
| editorMode | string | 键绑定模式 |
| verbose | boolean | 详细调试输出 |
| autoCompactEnabled | boolean | 自动压缩上下文 |
| todoFeatureEnabled | boolean | Todo 功能开关 |

### 项目设置（`settings.json`）

| 设置 | 类型 | 说明 |
|------|------|------|
| model | string | 模型覆盖 |
| autoMemoryEnabled | boolean | 自动记忆 |
| permissions.defaultMode | string | 默认权限模式 |
| language | string | 响应语言 |

## 执行逻辑

### GET 操作

自动允许，读取配置并格式化。

### SET 操作

1. 类型强制转换（布尔值）
2. 选项验证
3. 异步验证（如模型 API 检查）
4. 写入存储
5. 同步 AppState（立即 UI 效果）

## 权限控制

- GET：自动允许
- SET：需要确认

## Prompt 生成

从设置注册表动态生成，包含每个设置的类型、选项和说明。

## Connections

- [设置系统](../concepts/settings-system.md)

## Open Questions

- 如何处理设置冲突？

## Sources

- `chapters/chapter-17-其他工具精选.md`