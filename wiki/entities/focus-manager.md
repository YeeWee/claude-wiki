---
title: "FocusManager"
type: entity
created: 2026-04-15
updated: 2026-04-15
sources:
  - ../sources/2026-04-15-focus-and-interaction.md
related:
  - ../concepts/focus-management.md
---

# FocusManager

One-paragraph summary: FocusManager implements DOM-like focus management for terminal UI, tracking active element, managing focus history stack, and handling Tab navigation, click focus, and automatic focus restoration when elements are removed.

## Class Structure

```typescript
class FocusManager {
  activeElement: DOMElement | null = null
  private focusStack: DOMElement[] = []  // Max 32 elements
  private enabled = true
}
```

## Core Operations

### focus(node)

Switch focus to new element:
1. Check if already focused (skip)
2. Check if enabled (skip)
3. Push previous activeElement to stack (dedupe to prevent infinite growth)
4. Dispatch `blur` event to previous element
5. Update `activeElement`
6. Dispatch `focus` event to new element

### focusNext/focusPrevious

Tab navigation:
- `collectTabbable(root)` gathers all `tabIndex >= 0` elements
- Calculate next index with wrap-around
- `moveFocus(direction)` handles directional navigation

### handleNodeRemoved

When focused element is removed:
1. Filter stack to remove deleted nodes
2. If activeElement was removed, dispatch `blur`
3. Restore from stack: pop until finding node still in tree
4. Dispatch `focus` to restored element

## Integration

Stored on root DOM element, accessible via:
- `getRootNode(node)` - Traverse parentNode to root
- `getFocusManager(node)` - Get manager from root

## Connections

- [Focus Management](../concepts/focus-management.md)
- [Ink UI Framework](../sources/2026-04-15-ink-ui-framework.md) - Rendering host

## Open Questions

- What happens when all focusable elements are removed?
- How does FocusManager handle dynamic tabIndex changes?

## Sources

- `/src/ink/focus.ts`