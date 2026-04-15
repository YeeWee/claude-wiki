---
title: "ThemeProvider"
type: entity
created: 2026-04-15
updated: 2026-04-15
sources: ["chapter-37-组件架构.md"]
related: []
---

ThemeProvider 是 Claude Code 设计系统的主题上下文提供者，支持 dark/light/auto 三种主题模式。

## 核心职责

- 提供主题设置上下文
- 管理系统主题跟踪（用于 auto 模式）
- 支持主题预览和保存

## ThemeContextValue 结构

```typescript
type ThemeContextValue = {
  themeSetting: ThemeSetting     // 用户偏好 ('auto' | 'dark' | 'light')
  setThemeSetting: (setting: ThemeSetting) => void
  setPreviewTheme: (setting: ThemeSetting) => void
  savePreview: () => void
  cancelPreview: () => void
  currentTheme: ThemeName        // 解析后的主题 (never 'auto')
}
```

## 实现特点

- 使用 `watchSystemTheme` 监听终端主题变化
- auto 模式下动态跟随系统主题
- 支持预览模式（不保存，可取消）

## useTheme Hook

返回 `[ThemeName, (setting: ThemeSetting) => void]`，用于获取当前主题和设置函数。

## Connections

- [Component Architecture](../concepts/component-architecture.md) - 设计系统层

## Open Questions

- 终端主题变化检测的兼容性问题如何处理？
- 自定义主题的扩展机制是否需要设计？

## Sources

- `chapters/chapter-37-组件架构.md`