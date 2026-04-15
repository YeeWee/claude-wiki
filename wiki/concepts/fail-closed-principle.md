---
title: "Fail-Closed Principle"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources:
  - ../sources/2026-04-15-sandbox-security.md
  - ../sources/2026-04-15-permission-system.md
  - chapter-46-架构设计原则总结.md
  - chapter-50-附录.md
related:
  - ../entities/sandbox-manager.md
  - ../concepts/sandbox-isolation.md
  - ../concepts/tool-extensibility.md
---

# Fail-Closed Principle

One-paragraph summary: The Fail-Closed principle is Claude Code's core security design philosophy: when the system encounters uncertain or un-understood situations, it always chooses the safer path (reject/deny/ask) rather than allowing potentially dangerous operations.

## Definition

**Fail-Closed** means: "When uncertain, fail by closing (blocking) the operation."

Opposite of Fail-Open, which would allow operations when systems fail or are uncertain.

## Application in Claude Code

### AST Parsing (Bash Commands)

From `/src/utils/bash/ast.ts`:

```typescript
/**
 * This module's core design property is FAIL-CLOSED:
 * - We never interpret syntax structures we don't understand
 * - If tree-sitter produces node types not explicitly allowed
 * - We refuse to extract argv, requiring user confirmation
 * 
 * This is not a sandbox. It answers one question:
 * "Can we generate a trusted argv[] for this command?"
 * If not, ask user.
 */
```

Only explicitly allowed node types are processed. Un-understood syntax = rejection + user confirmation.

### Permission System

When classification fails or classifier unavailable:
- `auto` mode: If Iron Gate enabled, deny; otherwise, ask user
- `dontAsk` mode: Convert all asks to denials
- Unknown patterns: Always ask/deny before allowing

### Sandbox Settings

- Platform not supported: Sandbox disabled
- Dependencies missing: Sandbox disabled
- Policy settings override local: Cannot enable if policy disables
- Configuration parse error: Fall back to safest defaults

## Examples

| Situation | Fail-Closed Behavior |
|-----------|---------------------|
| Unknown Bash syntax | Ask user instead of auto-execute |
| Classifier unavailable | Deny or ask (not auto-allow) |
| Sandbox unavailable | Block sandboxed commands or ask |
| Path traversal attempt | Deny write immediately |
| Dangerous file edit | Require explicit user confirmation |

## Tool Default Values (Fail-Close Strategy)

Tool system applies Fail-Closed through conservative defaults:

```typescript
const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: () => false,    // 默认不安全
  isReadOnly: () => false,           // 默认写操作
  isDestructive: () => false,        // 默认非破坏性
}
```

- New tools default to "unsafe" - must explicitly declare `isConcurrencySafe: true` for concurrent execution
- Forces developers to think about security attributes upfront
- Prevents accidental dangerous operations from newly added tools

## Contrast with Fail-Open

| Principle | Failure Behavior | Claude Code Approach |
|-----------|-----------------|---------------------|
| Fail-Open | Allow operations when system fails | Not used (dangerous) |
| Fail-Closed | Block operations when uncertain | Default approach |

## Connections

- [Sandbox Isolation](../concepts/sandbox-isolation.md) - Physical implementation
- [SandboxManager](../entities/sandbox-manager.md) - Interface entity
- [Tool Extensibility](../concepts/tool-extensibility.md) - Extension system design

## Open Questions

- How does system handle legitimate user overrides?
- What logging captures fail-closed decisions?

## Sources

- `/src/utils/bash/ast.ts`
- `/src/utils/permissions/permissions.ts`
- `chapters/chapter-46-架构设计原则总结.md`
- `chapters/chapter-50-附录.md`