---
title: "SandboxManager"
type: entity
created: 2026-04-15
updated: 2026-04-15
sources:
  - chapters/chapter-09-bashtool深度解析.md
  - chapters/chapter-34-沙箱安全.md
related:
  - ../concepts/sandbox-isolation.md
  - ../concepts/fail-closed-principle.md
---

# SandboxManager

SandboxManager 是 Claude Code 沙箱系统的 CLI 适配层，基于 `@anthropic-ai/sandbox-runtime` 包，提供操作系统级别的隔离。

## 接口定义

```typescript
interface ISandboxManager {
  initialize(sandboxAskCallback?: SandboxAskCallback): Promise<void>
  isSupportedPlatform(): boolean
  isSandboxingEnabled(): boolean
  areUnsandboxedCommandsAllowed(): boolean
  wrapWithSandbox(command: string, binShell?: string, ...): Promise<string>
  cleanupAfterCommand(): void
  getFsReadConfig(): FsReadRestrictionConfig
  getFsWriteConfig(): FsWriteRestrictionConfig
  getNetworkRestrictionConfig(): NetworkRestrictionConfig
}
```

## 核心功能

### 平台支持

| 平台 | 实现机制 |
|------|---------|
| macOS | Seatbelt (sandbox-exec) |
| Linux | bubblewrap (bwrap) |
| WSL2 | bubblewrap |
| WSL1 | 不支持 |

### 配置转换

将 Claude Code 设置转换为 SandboxRuntimeConfig：
- 文件系统隔离：读写路径的 allow/deny 配置
- 网络隔离：域名白名单、Unix socket 控制
- 命令排除：用户配置的排除命令列表

### 关键方法

| 方法 | 说明 |
|------|------|
| `initialize` | 初始化沙箱，检测 worktree 主仓库，订阅配置变更 |
| `isSandboxingEnabled` | 检查平台支持、依赖、策略设置 |
| `wrapWithSandbox` | 包装命令，调用底层 sandbox-runtime |
| `cleanupAfterCommand` | 清理可能植入的 bare git repo 文件 |

### 安全配置

始终拒绝写入：
- settings.json 文件（防止沙箱逃逸）
- .claude/skills 目录
- 检测到的 bare git repo 文件（HEAD, objects, refs, hooks）

## Bare Git Repo 攻击防护

攻击场景：攻击者在沙箱命令执行期间植入 HEAD + objects/ + refs/ 等文件，伪造 bare repo，利用 git config core.fsmonitor 在后续非沙箱 git 执行时逃逸沙箱。

防御：`scrubBareGitRepoFiles()` 在每次沙箱命令完成后清理这些潜在的危险文件。

## 配置来源

- 用户设置 (`sandbox.enabled`, `sandbox.autoAllowBashIfSandboxed`)
- 项目设置（路径特定规则）
- 策略设置（企业锁定，最高优先级）
- Feature flags（平台启用/禁用列表）

## Connections

- [Sandbox 隔离机制](../concepts/sandbox-isolation.md) - 安全架构
- [BashTool](../entities/bash-tool.md) - 使用沙箱的工具

## Sources

- [第九章：BashTool 深度解析](../sources/2026-04-15-bashtool深度解析.md)