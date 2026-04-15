---
title: "MCP 扩展"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-48-可扩展性设计.md]
related: [../entities/mcp.md]
---

MCP 扩展是 Claude Code 可扩展性设计的可变复杂度方式，通过 MCP 协议连接外部服务。

## 适用场景

- 外部系统集成
- 动态发现工具
- 现有 MCP 生态利用

## 配置优先级

| 来源 | 优先级 | 配置文件 |
|------|--------|---------|
| Enterprise | 最高 | 企业策略配置 |
| Local | 高 | 项目私有配置 |
| Project | 中 | .mcp.json |
| User | 低 | ~/.claude/settings.json |
| Plugin | 最低 | 插件内嵌配置 |

## 环境变量替换

支持变量：

- `${ENV_VAR}`：环境变量替换
- `${ENV_VAR:-default}`：带默认值
- `${PROJECT_ROOT}`：项目根目录
- `${CLAUDE_CONFIG_DIR}`：Claude 配置目录

## Connections

- [MCP](../entities/mcp.md)

## Open Questions

- MCP 服务器安全边界？

## Sources

- `chapters/chapter-48-可扩展性设计.md`