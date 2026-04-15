---
title: "Coordinator Mode"
type: source
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-32-coordinator模式.md]
related:
  - ../concepts/multi-agent-orchestration.md
  - ../concepts/async-worker-execution.md
  - ../entities/coordinator-mode.md
  - ../entities/scratchpad.md
---

# Coordinator Mode

One-paragraph summary: Coordinator mode is Claude Code's multi-agent orchestration mechanism where a main agent coordinates tasks by spawning Worker agents for parallel execution, synthesizing results, and communicating with users while Workers handle specific research, implementation, and verification tasks.

## Core Architecture

### Role Definition

The Coordinator is an orchestrator that:
- Helps users achieve goals
- Directs workers to research, implement, and verify code changes
- Synthesizes results and communicates with users
- Answers questions directly when possible (doesn't delegate trivial work)

Every Coordinator message goes to the user. Worker results are internal signals, not conversation partners.

### Coordinator Tool Set

Only 4 tools are available in Coordinator mode:

| Tool | Purpose |
|------|---------|
| AgentTool | Spawn Worker agents |
| TaskStop | Stop workers in wrong direction |
| SendMessage | Continue stopped/running workers |
| SyntheticOutput | Output results to user |

### Worker Tool Set

Workers use `ASYNC_AGENT_ALLOWED_TOOLS` which includes:
- File operations: Read, Edit, Write, Glob, Grep
- Web: WebSearch, WebFetch
- Shell: Bash, PowerShell
- Task management: TodoWrite, TaskCreate/Update/List/Get
- Skills: Skill invocation

Workers cannot use: AgentTool (prevent recursive spawning), TaskStopTool, SendMessageTool, AskUserQuestionTool.

## Task Workflow

| Phase | Who | Purpose |
|-------|-----|---------|
| Research | Workers (parallel) | Investigate codebase, find files, understand problem |
| Synthesis | **Coordinator** | Read findings, understand problem, craft implementation specs |
| Implementation | Workers | Make targeted changes per spec, commit |
| Verification | Workers | Test changes work |

## Scratchpad Knowledge Sharing

Scratchpad directory (`/tmp/claude-{uid}/{sanitized-cwd}/{sessionId}/scratchpad/`) provides cross-worker knowledge persistence:
- Research results storage
- Intermediate state persistence for long tasks
- Cross-worker coordination (shared files, discoveries, configs)
- No permission prompts for reads/writes in Scratchpad

## task-notification Format

Worker results arrive as XML notifications:

```xml
<task-notification>
<task-id>{agentId}</task-id>
<status>completed|failed|killed</status>
<summary>{human-readable status summary}</summary>
<result>{agent's final text response}</result>
<usage>
  <total_tokens>N</total_tokens>
  <tool_uses>N</tool_uses>
  <duration_ms>N</duration_ms>
</usage>
</task-notification>
```

## Continue vs Spawn Decision

| Situation | Mechanism | Reason |
|-----------|-----------|--------|
| Research exactly matches edit file | Continue (SendMessage) | Worker has file context, clear plan |
| Research broad, implementation narrow | Spawn fresh | Avoid exploration noise, focused context |
| Fix failure or recent work continuation | Continue | Worker has error context |
| Verify different worker's code | Spawn fresh | Fresh perspective, no implementation assumptions |
| First implementation direction wrong | Spawn fresh | Wrong context pollutes retry |

## Connections

- [Multi-Agent Orchestration](../concepts/multi-agent-orchestration.md)
- [Async Worker Execution](../concepts/async-worker-execution.md)
- [Permission System](../sources/2026-04-15-permission-system.md) - Worker tool restrictions

## Open Questions

- How does Coordinator handle conflicting Worker results?
- What triggers mode switching during session resume?

## Sources

- `chapters/chapter-32-coordinator模式.md`
- `/src/coordinator/coordinatorMode.ts`
- `/src/constants/tools.ts`
- `/src/tools/SendMessageTool/SendMessageTool.ts`
- `/src/tools/AgentTool/resumeAgent.ts`