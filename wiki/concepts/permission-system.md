---
title: "权限系统"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-13-web工具集.md, chapter-17-其他工具精选.md]
related: []
---

权限系统是 Claude Code 安全机制的核心，控制工具执行和用户确认流程。

## 权限检查流程

### WebFetchTool 示例

1. 预批准域名检查
2. 拒绝规则检查
3. 询问规则检查
4. 允许规则检查
5. 默认询问用户

### SkillTool 三层策略

1. 拒绝规则检查：匹配则 deny
2. 允许规则检查：匹配则 allow
3. 安全属性白名单：SAFE_SKILL_PROPERTIES

## 权限结果类型

| 行为 | 说明 |
|------|------|
| allow | 自动允许执行 |
| deny | 拒绝执行 |
| ask | 需要用户确认 |
| passthrough | 需要进一步确认 |

## 预批准域名

自动允许的官方文档站点，减少常见文档站点的权限请求。

### 匹配模式

- 完整主机名匹配（如 docs.python.org）
- 路径前缀匹配（如 github.com/anthropics）

## 权限规则

规则内容格式：`domain:hostname` 或工具特定格式。

## 安全属性白名单

SkillTool 定义 SAFE_SKILL_PROPERTIES，只有包含安全属性的技能才自动允许。

## Connections

- [WebFetchTool](../entities/web-fetch-tool.md)
- [SkillTool](../entities/skill-tool.md)
- [预批准域名](preapproved-domains.md)

## Open Questions

- 如何平衡安全性和用户体验？

## Sources

- `chapters/chapter-13-web工具集.md`
- `chapters/chapter-17-其他工具精选.md`