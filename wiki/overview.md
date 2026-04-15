---
title: "Claude Code Source Code Learning Wiki Overview"
type: overview
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-01 to chapter-50]
related:
  - entities/query-engine.md
  - entities/tool-interface.md
  - entities/agent-tool.md
  - entities/mcp.md
  - concepts/cli-architecture.md
  - concepts/fail-closed-principle.md
  - concepts/async-generator-pattern.md
  - concepts/multi-agent-orchestration.md
  - concepts/tool-extensibility.md
---

# Claude Code Source Code Learning Wiki

## Summary
This wiki documents the architecture, design patterns, and implementation details of Claude Code, Anthropic's official CLI tool. Based on 50 chapters of source code analysis, it covers the complete technical landscape from entry points to advanced features, creating a comprehensive reference for understanding how a production-grade AI CLI is built.

## Central Thesis
Claude Code represents a sophisticated CLI application that combines React-based UI (via Ink), complex state management (Zustand + React Context), a powerful tool system (50+ tools with unified interface), MCP integration for extensibility, and multi-agent coordination (Coordinator mode). The architecture emphasizes three core principles: **Monolith + Dynamic Import** (simple deployment with lazy loading), **Fail-Closed** (reject when uncertain), and **Tool-based Extensibility** (everything is a tool).

## Architecture Layers

### 1. Entry Points Layer
- **CLI Entry**: Main command-line interface with argument parsing
- **Bridge Entry**: Remote control via WebSocket/SSE
- **Daemon Entry**: Background process for persistent operations
- **Fast Path Entry**: Quick startup for frequent commands

### 2. Configuration Layer
- **Four-layer merge**: Global → User → Project → Local
- **Feature Flags**: GrowthBook integration for gradual rollout
- **Build Variants**: DCE for platform-specific optimization

### 3. State Management Layer
- **AppState**: Zustand store with selectors pattern
- **React Context**: Notification queue, stats, theme
- **Priority Queue**: Notification ordering and folding

### 4. Tool System Layer
- **Core Interface**: `Tool<TInput, TOutput>` with AsyncGenerator
- **buildTool Factory**: Safe defaults, simplified definition
- **50+ Built-in Tools**: Bash, File, Search, Web, Agent, etc.
- **Permission Integration**: Five-layer check flow

### 5. Extension Layer
- **Skills**: YAML frontmatter + prompt templates
- **Plugins**: Builtin + user-defined hooks
- **MCP**: Model Context Protocol servers
- **Hooks**: Event-driven customization

### 6. Communication Layer
- **API Client**: Stream processing, token tracking
- **MCP Transport**: WebSocket, SSE, stdio
- **Bridge System**: Remote session management
- **REPL Bridge**: Message flow with FlushGate

### 7. UI Layer
- **Ink Framework**: React reconciler + Yoga layout
- **FocusManager**: DOM-like focus stack
- **Text Selection**: anchor/focus model
- **ANSI Diff**: Screen buffer optimization

## Key Entities
- [Anthropic](entities/anthropic.md) - Developer company
- [Bun](entities/bun.md) - JavaScript runtime
- [React/Ink](entities/react-ink.md) - Terminal UI framework
- [MCP](entities/mcp.md) - Extensibility protocol
- [GrowthBook](entities/growthbook.md) - Feature flag platform

## Key Concepts
- [CLI Architecture](concepts/cli-architecture.md) - Multi-entry design
- [Fail-Closed Principle](concepts/fail-closed-principle.md) - Security philosophy
- [AsyncGenerator Pattern](concepts/async-generator-pattern.md) - Streaming execution
- [Multi-agent Orchestration](concepts/multi-agent-orchestration.md) - Coordinator + Workers
- [Sandbox Isolation](concepts/sandbox-isolation.md) - OS-level security

## Design Principles

### Monolith + Dynamic Import
Single bundle deployment with lazy loading for optional features. Avoids microservices complexity while maintaining startup performance.

### Fail-Closed Security
When uncertain, reject operations. Sandbox isolation, permission checks, and bare git repo attack prevention.

### Tool-based Extensibility
Everything users interact with is a tool. Skills, commands, and MCP calls all use the unified Tool interface.

### Layered Configuration
Cascade merge from global defaults through project-specific overrides. Schema validation ensures correctness.

## Open Questions
- How does QueryEngine orchestrate complex multi-turn conversations?
- What are the performance implications of the React+Ink UI layer?
- How does the Coordinator mode scale with many Workers?
- What security vulnerabilities remain after Fail-Closed implementation?

## Sources
All 50 chapters in `chapters/` directory have been processed. See [Index](index.md) for complete source listing.