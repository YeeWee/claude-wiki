---
title: "Permission Modes"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources:
  - ../sources/2026-04-15-permission-system.md
related:
  - ../entities/permission-rule.md
  - ../concepts/rule-matching.md
---

# Permission Modes

One-paragraph summary: Permission Modes define Claude Code's overall behavior strategy for handling AI resource access requests, ranging from interactive confirmation (default) to automated decision-making (auto, acceptEdits) to restrictive blocking (dontAsk, bypassPermissions).

## Mode Types

| Mode | Title | Behavior | Use Case |
|------|-------|----------|----------|
| `default` | Default | Ask for sensitive ops | Normal interactive use |
| `plan` | Plan Mode | Read-only, planning | Non-destructive analysis |
| `acceptEdits` | Accept edits | Auto-accept in working dir | Trusted project work |
| `bypassPermissions` | Bypass | Skip all checks | Emergency/debug (dangerous) |
| `dontAsk` | Don't Ask | Ask → Deny | Testing/restricted env |
| `auto` | Auto | AI classifier decides | Automation (ant-only) |

## Mode Configuration

```typescript
type PermissionModeConfig = {
  title: string       // Display name
  shortTitle: string  // Compact name
  symbol: string      // Icon (⏸, ⏵⏵)
  color: string       // Theme color
  external: ExternalPermissionMode  // API representation
}
```

## Mode Cycling

Shift+Tab cycles through modes with user-type-specific paths:

**External Users:**
```
default → acceptEdits → plan → default
```

**Ant Users:**
```
default → bypassPermissions → auto → default
```
(Skips acceptEdits, plan; adds auto)

## Mode Effects

### default
- Sensitive operations require user confirmation
- UI shows permission prompt dialog
- User can allow/deny/add rule

### plan
- Only Read, Glob, Grep, LSP tools active
- No write operations allowed
- Useful for code review without modification

### acceptEdits
- Auto-allow edits within configured working directories
- Still ask for operations outside work dirs
- Still ask for dangerous operations

### bypassPermissions
- Skip all permission checks
- Tools execute without prompts
- Dangerous: No safety layer

### dontAsk
- Convert all `ask` decisions to `deny`
- Used in testing environments
- Prevents blocking on prompts

### auto
- AI classifier evaluates each operation
- Returns allow/deny based on risk assessment
- Iron Gate: Fallback to deny if classifier fails
- Consecutive denial tracking triggers fallback to prompting

## Connections

- [PermissionRule](../entities/permission-rule.md) - Rule structure
- [Rule Matching](../concepts/rule-matching.md) - Match mechanism

## Open Questions

- How does mode interact with policy settings?
- What telemetry tracks mode usage patterns?

## Sources

- `/src/utils/permissions/PermissionMode.ts`
- `/src/utils/permissions/getNextPermissionMode.ts`