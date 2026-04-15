---
title: "Focus Management"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources:
  - ../sources/2026-04-15-focus-and-interaction.md
related:
  - ../entities/focus-manager.md
---

# Focus Management

One-paragraph summary: Focus Management in Ink implements DOM-like focus handling for terminal UI, supporting Tab navigation, click focus, automatic focus restoration, and event dispatch for focus/blur notifications.

## Core Model

### activeElement

Like browser's `document.activeElement`:
- Currently focused element
- Receives keyboard events
- Only one activeElement at a time

### Focus Stack

History of previously focused elements:
- Max 32 elements
- Deduped on push (prevents infinite growth from Tab cycles)
- Used for restoration when activeElement removed

## Navigation Mechanisms

### Tab Navigation

`collectTabbable(root)` finds all `tabIndex >= 0` elements in DOM order:
- `focusNext`: Move forward in tab order
- `focusPrevious`: Move backward
- Wrap-around at boundaries

### Click Focus

`handleClickFocus(target)`:
- Find nearest focusable ancestor (tabIndex defined)
- Focus that element
- Clicks on non-focusable elements focus their focusable ancestor

### Auto Focus

Elements with `autoFocus` attribute:
- Focus on mount if no other focus exists
- Only first autoFocus element gets focus

## Event Dispatch

### FocusEvent

```typescript
class FocusEvent extends TerminalEvent {
  type: 'focus' | 'blur'
  relatedTarget: DOMElement | null  // Other side of transition
  bubbles: true
  cancelable: false
}
```

- For focus: relatedTarget = element losing focus
- For blur: relatedTarget = element gaining focus

### Dispatch Flow

1. Capture phase: root → target
2. Target phase: at target element
3. Bubble phase: target → root

Parents can observe child focus changes via bubble phase.

## Restoration on Removal

`handleNodeRemoved`:
1. Filter focusStack (remove deleted nodes)
2. If activeElement was removed, dispatch blur
3. Pop from stack until finding node still in tree
4. Focus restored element, dispatch focus

## Integration Points

- Box component: `tabIndex`, `autoFocus`, `onFocus`, `onBlur`
- Dispatcher: Event dispatch priority
- React reconciler: Focus state preservation

## Connections

- [FocusManager](../entities/focus-manager.md) - Implementation entity

## Open Questions

- How does focus interact with terminal resize?
- What about focus in scrollable containers?

## Sources

- `/src/ink/focus.ts`
- `/src/ink/events/focus-event.ts`