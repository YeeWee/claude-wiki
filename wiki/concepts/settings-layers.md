---
title: "Settings Layers"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-04-配置系统.md]
related: [../concepts/merge-strategy.md, ../concepts/environment-variables.md]
---

Settings Layers 是 Claude Code 配置系统的层次结构设计，通过六层配置来源按优先级合并，实现用户、项目和企业灵活定制运行时行为。

## 层次结构

按优先级从低到高：

```text
Plugin Settings → User Settings → Project Settings → Local Settings → Flag Settings → Policy Settings
```

### 配置来源详解

| 来源 | 文件路径 | 说明 |
|------|---------|------|
| pluginSettings | 插件基础层 | 插件提供的默认配置 |
| userSettings | `~/.claude/settings.json` | 用户全局配置 |
| projectSettings | `$PROJECT/.claude/settings.json` | 项目共享配置，可提交 Git |
| localSettings | `$PROJECT/.claude/settings.local.json` | 项目本地配置，自动 gitignored |
| flagSettings | `--settings` 参数 | CLI 参数传入 |
| policySettings | 企业策略配置 | IT 管理员管控 |

## Policy Settings 子层次

采用"首个来源优先"策略：

```text
Remote API → MDM/HKLM → File → HKCU
```

| 来源 | 说明 |
|------|------|
| Remote API | 最高优先级，远程管理 |
| MDM/HKLM | macOS plist / Windows HKLM |
| File | managed-settings.json + drop-ins |
| HKCU | Windows 用户注册表，最低优先级 |

### Drop-in 目录

```text
managed-settings.d/
├── 10-security.json
├── 20-otel.json
└── 30-mcp.json
```

按字母顺序合并配置片段。

## 项目配置文件对比

| 属性 | settings.json | settings.local.json |
|------|---------------|---------------------|
| Git | 可提交 | 自动 gitignored |
| 用途 | 团队共享 | 个人本地偏好 |
| 示例 | 权限规则、MCP 服务器 | 个人 API 密钥 |

## Connections

- [Merge Strategy](../concepts/merge-strategy.md) - 配置合并规则
- [Environment Variables](../concepts/environment-variables.md) - 环境变量处理
- [Init Sequence](../concepts/init-sequence.md) - 配置加载时序

## Open Questions

- Policy Settings 的远程 API 实现细节
- Drop-in 目录的冲突解决策略

## Sources

- `chapters/chapter-04-配置系统.md`