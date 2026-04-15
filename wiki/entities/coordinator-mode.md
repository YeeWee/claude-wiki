---
title: "CoordinatorMode"
type: entity
created: 2026-04-15
updated: 2026-04-15
sources:
  - ../sources/2026-04-15-coordinator-mode.md
related:
  - ../concepts/multi-agent-orchestration.md
  - ../entities/scratchpad.md
---

# CoordinatorMode

One-paragraph summary: CoordinatorMode is Claude Code's multi-agent orchestration pattern where a main agent coordinates tasks by spawning Worker agents for parallel execution, synthesizing research results, and directing implementation while maintaining user communication.

## Enable Conditions

```typescript
isCoordinatorMode(): boolean {
  if (feature('COORDINATOR_MODE')) {
    return isEnvTruthy(process.env.CLAUDE_CODE_COORDINATOR_MODE)
  }
  return false
}
```

Requires both feature gate and environment variable.

## Tool Restrictions

### Coordinator Tools

Only 4 tools available:
- `AgentTool` - Spawn workers
- `TaskStop` - Stop wrong-direction workers
- `SendMessage` - Continue workers
- `SyntheticOutput` - Output to user

### Worker Tools

Workers use `ASYNC_AGENT_ALLOWED_TOOLS` (file ops, web, shell, task mgmt) but cannot use:
- AgentTool (prevent recursive spawning)
- TaskStopTool
- SendMessageTool
- AskUserQuestionTool

## Task Phases

| Phase | Owner | Action |
|-------|-------|--------|
| Research | Workers | Parallel investigation |
| Synthesis | Coordinator | Understand findings, craft specs |
| Implementation | Workers | Targeted changes per spec |
| Verification | Workers | Test validity |

## System Prompt Principles

- Coordinator messages go to user (Worker results are internal signals)
- Must synthesize research before directing implementation
- Never write "based on your findings" - demonstrate understanding with specific file paths and line numbers
- Parallelism is superpower: launch independent workers concurrently

## Connections

- [Multi-Agent Orchestration](../concepts/multi-agent-orchestration.md)
- [Scratchpad](../entities/scratchpad.md) - Cross-worker knowledge sharing

## Open Questions

- How does Coordinator handle conflicting Worker recommendations?
- What metrics track Coordinator efficiency?

## Sources

- `/src/coordinator/coordinatorMode.ts`
- `/src/constants/tools.ts`