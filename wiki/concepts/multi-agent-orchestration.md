---
title: "Multi-Agent Orchestration"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources:
  - ../sources/2026-04-15-coordinator-mode.md
related:
  - ../entities/coordinator-mode.md
  - ../entities/scratchpad.md
  - ../concepts/async-worker-execution.md
---

# Multi-Agent Orchestration

One-paragraph summary: Multi-Agent Orchestration is Claude Code's Coordinator mode pattern where a main agent directs multiple Worker agents for parallel task execution, combining research efficiency of parallel workers with synthesis capabilities of a central coordinator.

## Architecture

```
User Request → Coordinator (orchestrator)
                    ↓
    ┌───────────────┼───────────────┐
    ↓               ↓               ↓
Worker 1       Worker 2       Worker 3
(research)     (research)     (implement)
    ↓               ↓               ↓
    └───────────────┴───────────────┘
                    ↓
            task-notification
                    ↓
              Coordinator
                    ↓
            User Response
```

## Key Principles

### Parallelism Superpower

Workers are async. Launch independent workers concurrently:
- Research tasks: Run multiple workers covering different angles
- Implementation tasks: One at a time per file set (avoid conflicts)
- Verification: Can run alongside implementation (different file areas)

### Synthesis Responsibility

Coordinator must understand Worker findings before directing follow-up:
- Read findings, identify approach
- Write prompts with specific file paths and line numbers
- Never write "based on your findings" - demonstrate understanding

### Context Isolation

Workers have isolated contexts:
- Independent tool sets (no AgentTool, TaskStop, SendMessage)
- No cross-worker communication except via Scratchpad
- Clean context for each task

## Task Workflow Phases

| Phase | Actor | Goal |
|-------|-------|------|
| Research | Workers (parallel) | Investigate codebase |
| Synthesis | Coordinator | Understand, write specs |
| Implementation | Workers | Execute changes |
| Verification | Workers | Test validity |

## Knowledge Sharing

Scratchpad directory enables durable cross-worker knowledge:
- Research results stored for implementation workers
- No permission prompts for Scratchpad operations
- Session-scoped: `/tmp/claude-{uid}/{cwd}/{sessionId}/scratchpad/`

## Continue vs Spawn Decision

| Situation | Decision | Reason |
|-----------|----------|--------|
| Research file = edit file | Continue | Worker has context |
| Research broad, edit narrow | Spawn fresh | Clean context |
| Fix failure | Continue | Error context available |
| Verify other's code | Spawn fresh | Fresh perspective |
| Wrong direction attempt | Spawn fresh | Avoid polluted context |

## Connections

- [CoordinatorMode](../entities/coordinator-mode.md) - Implementation entity
- [Scratchpad](../entities/scratchpad.md) - Knowledge sharing entity
- [Async Worker Execution](../concepts/async-worker-execution.md)

## Open Questions

- How to handle conflicting Worker recommendations?
- What metrics evaluate orchestration efficiency?

## Sources

- `/src/coordinator/coordinatorMode.ts`
- `/src/tools/AgentTool/`