---
title: "PermissionRule"
type: entity
created: 2026-04-15
updated: 2026-04-15
sources:
  - ../sources/2026-04-15-permission-system.md
related:
  - ../concepts/permission-modes.md
  - ../concepts/rule-matching.md
---

# PermissionRule

One-paragraph summary: PermissionRule defines a permission configuration entry specifying tool access rules with source origin, behavior type (allow/deny/ask), and target specification (tool name with optional content pattern).

## Structure Definition

```typescript
type PermissionRule = {
  source: PermissionRuleSource     // Where rule originated
  ruleBehavior: PermissionBehavior // 'allow' | 'deny' | 'ask'
  ruleValue: PermissionRuleValue   // Target specification
}

type PermissionRuleValue = {
  toolName: string       // 'Bash', 'Read', 'WebSearch'
  ruleContent?: string   // 'npm:*', '/path/**', 'git*'
}

type PermissionRuleSource =
  | 'userSettings'    // Global user configuration
  | 'projectSettings' // Project-level settings
  | 'localSettings'   // Local override
  | 'flagSettings'    // Command-line flags
  | 'policySettings'  // Enterprise policy
  | 'cliArg'          // CLI argument
```

## Rule Content Patterns

### Shell Command Rules

| Type | Syntax | Match Behavior |
|------|--------|----------------|
| exact | `git status` | Precise command match |
| prefix | `npm:*` (legacy) | Command starts with prefix |
| wildcard | `git*` | Pattern with `*` wildcard |

### File Path Rules

- `/path/**` - Directory wildcard
- Exact paths for specific file control

## Parsing

`permissionRuleValueFromString(ruleString)`:
- Format: `"ToolName"` or `"ToolName(content)"`
- Handles escaped parentheses `\(` and `\)`
- Empty or `*` content = tool-level rule

## Priority Rules

1. Deny rules first (any match = immediate deny)
2. Then check allow rules
3. Unmatched = mode-based default

## Connections

- [Permission Modes](../concepts/permission-modes.md)
- [Rule Matching](../concepts/rule-matching.md)

## Open Questions

- How are rules validated for syntax correctness?
- What happens when rules conflict across sources?

## Sources

- `/src/utils/permissions/permissionRuleParser.ts`
- `/src/types/permissions.ts`