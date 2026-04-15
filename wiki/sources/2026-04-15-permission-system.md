---
title: "Permission System"
type: source
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-33-权限系统.md]
related:
  - ../concepts/permission-modes.md
  - ../concepts/rule-matching.md
  - ../entities/permission-rule.md
---

# Permission System

One-paragraph summary: The permission system is Claude Code's security foundation, controlling AI access to system resources through multi-layer defense including permission modes, rule matching, safety checks, and AI classifiers for auto decision-making.

## Permission Modes

| Mode | Title | Symbol | Description |
|------|-------|--------|-------------|
| `default` | Default | none | Requires user confirmation for sensitive operations |
| `plan` | Plan Mode | ⏸ | Read-only, planning only |
| `acceptEdits` | Accept edits | ⏵⏵ | Auto-accept edits within working directory |
| `bypassPermissions` | Bypass | ⏵⏵ | Skip all permission checks (dangerous) |
| `dontAsk` | Don't Ask | ⏵⏵ | Convert all asks to denials |
| `auto` | Auto | ⏵⏵ | AI classifier auto-decision (ant-only) |

Mode cycling via Shift+Tab follows specific paths based on user type (ant users have different cycle order).

## Rule Matching Mechanism

### Rule Structure

```typescript
type PermissionRule = {
  source: PermissionRuleSource  // userSettings, projectSettings, policySettings, etc.
  ruleBehavior: PermissionBehavior  // 'allow' | 'deny' | 'ask'
  ruleValue: {
    toolName: string  // 'Bash', 'Read', etc.
    ruleContent?: string  // 'npm:*', '/path/**'
  }
}
```

### Shell Rule Types

| Type | Pattern | Example |
|------|---------|---------|
| exact | Precise match | `git status` |
| prefix | Prefix match (legacy `:*`) | `npm:*` |
| wildcard | Wildcard pattern | `git*` |

### Rule Priority

1. Deny rules first (any matching deny = immediate rejection)
2. Internal editable paths (plan files, scratchpad)
3. Safety checks (dangerous files/directories)
4. Working directory checks (acceptEdits mode)
5. Allow rules
6. Default behavior based on mode

## Safety Checks

### Dangerous Files

Protected files that block auto-editing:
- `.gitconfig`, `.gitmodules`
- `.bashrc`, `.bash_profile`, `.zshrc`, `.zprofile`
- `.ripgreprc`, `.mcp.json`, `.claude.json`

### Dangerous Directories

- `.git`, `.vscode`, `.idea`, `.claude`

### Dangerous Bash Patterns

Cross-platform code execution entry points:
- `python`, `python3`, `node`, `deno`, `ruby`, `perl`, `php`, `lua`
- `npx`, `bunx`, `npm run`, `yarn run`, `pnpm run`, `bun run`
- `bash`, `sh`, `ssh`
- `eval`, `exec`, `env`, `xargs`, `sudo`, `zsh`, `fish`

## Auto Mode Classifier

In `auto` mode, an AI classifier evaluates unknown operations:
- Safe tool whitelist bypasses classifier (Read, Grep, Glob, Task tools, etc.)
- acceptEdits fast-path check first
- Classifier returns allow/deny/ask
- Consecutive denial tracking (max 3 consecutive, max 20 total) triggers fallback to prompting

## Permission Updates

Update types:
- `addRules` / `replaceRules` / `removeRules`
- `setMode`
- `addDirectories` / `removeDirectories`

Updates can be persisted to settings files based on destination.

## Connections

- [Permission Modes](../concepts/permission-modes.md)
- [Rule Matching](../concepts/rule-matching.md)
- [Sandbox Security](../sources/2026-04-15-sandbox-security.md) - Additional security layer

## Open Questions

- How does classifier handle novel command patterns?
- What triggers policy setting overrides?

## Sources

- `chapters/chapter-33-权限系统.md`
- `/src/utils/permissions/permissions.ts`
- `/src/utils/permissions/PermissionMode.ts`
- `/src/utils/permissions/shellRuleMatching.ts`
- `/src/types/permissions.ts`