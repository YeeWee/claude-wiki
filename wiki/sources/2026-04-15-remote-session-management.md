---
title: "Remote Session Management"
type: source
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-31-远程会话管理.md]
related:
  - ../concepts/env-less-architecture.md
  - ../concepts/jwt-refresh.md
  - ../entities/remote-bridge-core.md
---

# Remote Session Management

One-paragraph summary: Remote session management is the core component enabling Claude Code's Remote Control feature, allowing users to control local CLI sessions via Web interfaces through a simplified "Env-less" architecture that directly connects to session-ingress layers.

## Key Architecture

### Env-less Design

The `remoteBridgeCore.ts` adopts an "Env-less" architecture that simplifies connection by directly connecting to session-ingress layer, bypassing the Environments API poll/dispatch middleware:

| Feature | Env-based (v1) | Env-less (v2) |
|---------|---------------|---------------|
| Session creation | POST /environments/{id}/sessions | POST /v1/code/sessions |
| Credential fetch | poll → dispatch → register | POST /v1/code/sessions/{id}/bridge |
| Complexity | ~2400 lines | ~1000 lines |

### Core API Flow

1. `POST /v1/code/sessions` - Create session, get `session.id`
2. `POST /v1/code/sessions/{id}/bridge` - Get `{worker_jwt, expires_in, api_base_url, worker_epoch}`
3. `createV2ReplTransport()` - Establish SSE + CCRClient transport channel
4. `createTokenRefreshScheduler()` - Periodically refresh JWT
5. Auto rebuild on 401 (carrying old seq-num)

## Session Lifecycle

The session follows a state machine from initializing → ready → connected → reconnecting/idle → torn_down:

- **Initializing**: Create session, fetch credentials, create transport
- **Ready**: SSE connection + history flush
- **Connected**: Active communication, write messages, send control requests
- **Reconnecting**: JWT refresh + rebuild on 401
- **Torn down**: Cleanup, archive session, close transport

## Authentication Recovery

Two mechanisms ensure connection stability:

1. **Proactive refresh**: Scheduled JWT refresh before expiration
2. **401 recovery**: On SSE 401 event, refresh OAuth, re-fetch credentials, rebuild transport

Key design: `getLastSequenceNum()` preserves SSE sequence number high watermark, avoiding server replay of full history. `flushGate` queues writes during rebuild to prevent message loss.

## SDK Support

The `codeSessionApi.ts` is extracted as an independent module for SDK integration, allowing external SDKs to use `createCodeSession` and `fetchRemoteCredentials` without bundling heavy CLI dependencies like analytics and transport.

## Connections

- [Env-less Architecture](../concepts/env-less-architecture.md)
- [JWT Refresh](../concepts/jwt-refresh.md)
- [Sandbox Security](../sources/2026-04-15-sandbox-security.md) - Security layer for command execution

## Open Questions

- How does the session handle network interruptions beyond 401?
- What are the telemetry patterns for session failures?

## Sources

- `chapters/chapter-31-远程会话管理.md`
- `/src/bridge/remoteBridgeCore.ts` (1009 lines)
- `/src/bridge/codeSessionApi.ts` (169 lines)
- `/src/bridge/replBridgeTransport.ts` (371 lines)