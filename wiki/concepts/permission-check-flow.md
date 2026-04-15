---
title: "权限检查流程"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: [chapters/chapter-09-bashtool深度解析.md]
related: [../entities/bash-tool.md, ../concepts/sandbox-isolation.md]
---

# 权限检查流程

BashTool 的权限检查采用多层防护策略，从精确匹配到语义检查逐步放宽。

## 五层防护

### 第一层：精确匹配

检查命令是否完全匹配已定义规则：
- Deny 规则优先
- Ask 规则其次
- Allow 规则允许
- Passthrough 继续后续检查

### 第二层：规则匹配

匹配模式：
- exact：完全匹配
- prefix：前缀匹配（禁止匹配复合命令）
- wildcard：通配符匹配

### 第三层：路径约束

检查路径限制和 sed 编辑约束。

### 第四层：模式检查

检查 Permission Mode 和 ReadOnly 状态。

### 第五层：安全检查

检测危险模式：
- 命令替换：`$()`、`${}`
- 进程替换：`<()`、`()`
- Zsh 危险命令：zmodload、emulate 等

## 安全设计要点

### 复合命令防护

前缀规则不能匹配复合命令，防止 `Bash(cd:*)` 匹配 `cd /path && rm -rf /`

### 环境变量剥离

deny/ask 规需剥离所有环境变量，防止 `FOO=bar rm` 绕过 `Bash(rm:*)` deny 规则。

### 安全包装器剥离

允许 `Bash(npm:*)` 匹配 `timeout 10 npm install`。

### 安全环境变量白名单

SAFE_ENV_VARS 定义可安全剥离的变量（如 NODE_ENV、RUST_BACKTRACE），不包括危险变量（如 PATH、LD_PRELOAD）。

## Connections

- [BashTool](../entities/bash-tool.md) - 应用权限检查
- [Sandbox 隔离](../concepts/sandbox-isolation.md) - 双重防护

## Sources

- [第九章：BashTool 深度解析](../sources/2026-04-15-bashtool深度解析.md)