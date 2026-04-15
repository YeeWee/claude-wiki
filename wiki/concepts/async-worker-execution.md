---
title: "Async Worker Execution"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources:
  - ../sources/2026-04-15-coordinator-mode.md
related:
  - ../entities/coordinator-mode.md
  - ../concepts/multi-agent-orchestration.md
---

# Async Worker Execution

One-paragraph summary: Async Worker Execution is Coordinator mode's core design where Workers run independently without blocking the Coordinator, enabling parallel task processing and result notification via task-notification XML messages.

## Execution Model

### Forced Async

In Coordinator mode, Workers are **always async**:
- `shouldRunAsync` includes `isCoordinator()` check
- Coordinator does not wait for Worker completion
- Workers run in background with own abort controllers

### Registration

```typescript
const agentBackgroundTask = registerAsyncAgent({
  agentId,
  abortController,
  appState,
  ...
})
```

### Lifecycle

```typescript
runAsyncAgentLifecycle({
  taskId,
  abortController,
  makeStream: onCacheSafeParams => runAgent({ ... }),
  onComplete: result => finalizeAgentTool(...),
})
```

## Notification Mechanism

### task-notification Format

Workers communicate results via XML:

```xml
<task-notification>
<task-id>{agentId}</task-id>
<status>completed|failed|killed</status>
<summary>{human-readable summary}</summary>
<result>{agent's final text}</result>
<usage>
  <total_tokens>N</total_tokens>
  <tool_uses>N</tool_uses>
  <duration_ms>N</duration_ms>
</usage>
</task-notification>
```

Appears as user-role message in Coordinator's conversation (but is internal signal, not user message).

## Worker Continuation

### SendMessage

Resume stopped/running Workers:

```typescript
if (task.status === 'running') {
  queuePendingMessage(agentId, message)  // Queue for next tool call
} else {
  resumeAgentBackground({ agentId, prompt: message })  // Resume directly
}
```

### Resume Process

`resumeAgentBackground`:
1. Load transcript from disk
2. Filter valid messages (remove orphaned tool uses)
3. Reconstruct content replacement state
4. Append new message to history
5. Register as background task
6. Start lifecycle

## TaskStop

Stop wrong-direction workers:
- Coordinator realizes approach incorrect
- User changes requirements
- Worker taking too long

Stopped workers can be continued with SendMessage with corrected instructions.

## Parallel Execution Benefits

| Benefit | Mechanism |
|---------|-----------|
| Research coverage | Multiple workers, multiple angles |
| Time efficiency | Concurrent execution |
| Context isolation | Independent worker contexts |
| Error recovery | Stop wrong direction, restart fresh |

## Connections

- [Multi-Agent Orchestration](../concepts/multi-agent-orchestration.md) - Overall pattern
- [CoordinatorMode](../entities/coordinator-mode.md) - Entity

## Open Questions

- How are resource limits enforced across parallel workers?
- What happens if Coordinator crashes while Workers running?

## Sources

- `/src/tools/AgentTool/`
- `/src/tools/SendMessageTool/resumeAgent.ts`