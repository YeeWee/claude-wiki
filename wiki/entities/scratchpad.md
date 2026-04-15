---
title: "Scratchpad"
type: entity
created: 2026-04-15
updated: 2026-04-15
sources:
  - ../sources/2026-04-15-coordinator-mode.md
related:
  - ../entities/coordinator-mode.md
  - ../concepts/multi-agent-orchestration.md
---

# Scratchpad

One-paragraph summary: Scratchpad is a cross-worker knowledge sharing directory in Coordinator mode, providing a shared filesystem space where Worker agents can read and write without permission prompts for durable knowledge persistence across parallel task execution.

## Path Format

```typescript
getScratchpadDir(): string {
  return join(getProjectTempDir(), getSessionId(), 'scratchpad')
}
// Format: /tmp/claude-{uid}/{sanitized-cwd}/{sessionId}/scratchpad/
```

## Enable Check

```typescript
isScratchpadEnabled(): boolean {
  return checkStatsigFeatureGate('tengu_scratch')
}
```

## Creation

```typescript
async ensureScratchpadDir(): Promise<string> {
  const scratchpadDir = getScratchpadDir()
  await fs.mkdir(scratchpadDir, { mode: 0o700, recursive: true })
  return scratchpadDir
}
```

Created with `0o700` mode (owner-only access) for security.

## Security Validation

```typescript
isScratchpadPath(absolutePath: string): boolean {
  const normalizedPath = normalize(absolutePath)
  return (
    normalizedPath === scratchpadDir ||
    normalizedPath.startsWith(scratchpadDir + sep)
  )
}
```

Path normalization prevents traversal attacks (`../` escape).

## Use Cases

1. **Research results storage**: Research workers write findings for implementation workers
2. **Intermediate state**: Long tasks persist progress to avoid loss
3. **Cross-worker coordination**: Shared configs, file structures, discoveries

## Permission Exemption

Workers in Scratchpad directory get automatic permission exemption:
```typescript
// filesystem.ts: checkEditableInternalPath()
// Scratchpad writes allowed without prompts
```

## Connections

- [Coordinator Mode](../entities/coordinator-mode.md) - Scratchpad users
- [Multi-Agent Orchestration](../concepts/multi-agent-orchestration.md)

## Open Questions

- How is Scratchpad cleanup handled on session end?
- What size limits apply to Scratchpad?

## Sources

- `/src/utils/permissions/filesystem.ts`