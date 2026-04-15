---
title: "配置系统"
type: source
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-04-配置系统.md]
related: [../concepts/settings-layers.md, ../concepts/environment-variables.md, ../concepts/schema-validation.md, ../concepts/merge-strategy.md]
---

Claude Code 的配置系统采用六层配置来源按优先级合并的架构，支持用户、项目和企业灵活定制运行时行为。通过分层覆盖、安全优先、向后兼容和性能优化等设计原则，实现配置的精细化管理和安全边界控制。

## 配置层次结构

按优先级从低到高排列：

```text
Plugin Settings → User Settings → Project Settings → Local Settings → Flag Settings → Policy Settings
```

### 配置来源详解

| 来源 | 文件路径 | 说明 |
|------|---------|------|
| userSettings | `~/.claude/settings.json` | 用户全局配置，跨项目生效 |
| projectSettings | `$PROJECT/.claude/settings.json` | 项目共享配置，可提交 Git |
| localSettings | `$PROJECT/.claude/settings.local.json` | 项目本地配置，自动 gitignored |
| flagSettings | `--settings` 参数指定 | CLI 参数传入的配置文件 |
| policySettings | 企业策略配置 | IT 管理员统一管控 |

### 企业策略配置路径

| 平台 | 路径 |
|------|------|
| macOS | `/Library/Application Support/ClaudeCode/managed-settings.json` |
| Windows | `C:\Program Files\ClaudeCode\managed-settings.json` |
| Linux | `/etc/claude-code/managed-settings.json` |

支持 drop-in 目录：`managed-settings.d/` 按字母顺序合并配置片段。

## Settings Schema

核心字段定义：

- **模型配置**：model、availableModels
- **权限配置**：permissions（allow、deny、defaultMode）
- **环境变量**：env 对象
- **Hook 配置**：hooks（PreToolUse、PostToolUse）
- **沙箱配置**：sandbox
- **MCP 配置**：allowedMcpServers、deniedMcpServers
- **插件配置**：enabledPlugins、strictKnownMarketplaces

## 合并策略

使用 lodash 的 `mergeWith` 配合自定义合并器：

| 字段类型 | 合并行为 |
|---------|---------|
| 数组 | 合并 + 去重 |
| 对象 | 深度合并 |
| 基本类型 | 覆盖 |
| undefined | 删除字段 |

### Policy Settings 特殊处理

采用"首个来源优先"策略，优先级：

1. Remote API（最高优先级）
2. MDM/HKLM（macOS plist / Windows HKLM）
3. File（managed-settings.json + drop-ins）
4. HKCU（最低优先级）

## 环境变量处理

### 安全分级

**安全变量**：可在信任对话框确认前应用
- Claude Code 特定设置、遥测配置、AWS/Vertex 区域配置

**危险变量**：需信任确认后才能应用
- `ANTHROPIC_BASE_URL`、`HTTP_PROXY/HTTPS_PROXY`、`NODE_EXTRA_CA_CERTS`、`PATH/LD_PRELOAD`

### Host-Managed Provider 保护

当 `CLAUDE_CODE_PROVIDER_MANAGED_BY_HOST` 设置时，剥离提供商路由相关环境变量。

## 配置验证

使用 Zod Schema 验证，向后兼容原则：

- 允许：添加新可选字段、新枚举值、新属性
- 禁止：删除字段、删除枚举值、将可选字段改为必需

验证失败时，无效字段保留在文件中供用户修复。

## 缓存机制

三级缓存避免重复 I/O：

- **会话级缓存**：合并后的最终配置
- **来源级缓存**：每个来源的配置
- **文件级缓存**：解析后的文件内容

缓存失效时机：写入配置、切换项目目录、插件初始化完成、Hook 刷新触发。

## Connections

- [Settings Layers](../concepts/settings-layers.md) - 配置层次详解
- [Environment Variables](../concepts/environment-variables.md) - 环境变量处理
- [Schema Validation](../concepts/schema-validation.md) - 配置验证机制
- [Merge Strategy](../concepts/merge-strategy.md) - 配置合并策略

## Open Questions

- MDM 策略读取的跨平台实现差异
- 远程设置轮询检测的具体实现
- drop-in 目录合并时的冲突解决策略

## Sources

- `chapters/chapter-04-配置系统.md`