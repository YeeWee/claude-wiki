---
title: "Focus and Interaction"
type: source
created: 2026-04-15
updated: 2026-04-15
sources: [chapter-36-焦点与交互.md]
related:
  - ../concepts/focus-management.md
  - ../concepts/text-selection.md
  - ../entities/focus-manager.md
---

# Focus and Interaction

One-paragraph summary: Focus and interaction systems in Ink implement DOM-like focus management, mouse text selection, and keyboard event handling for terminal UI, supporting Tab navigation, click focus, smart word/line selection, and multi-protocol key parsing.

## Focus Management Architecture

FocusManager stored on root DOM element, accessible via parentNode chain:

```typescript
class FocusManager {
  activeElement: DOMElement | null  // Current focused element
  focusStack: DOMElement[]          // History stack (max 32)
  enabled: boolean                  // Focus operations enabled
}
```

### Focus Operations

- `focus(node)`: Switch focus, dispatch blur/focus events
- `focusNext/Previous(root)`: Tab navigation via `collectTabbable`
- `handleNodeRemoved`: Restore focus from stack when element removed
- `handleClickFocus`: Click triggers focus on nearest focusable ancestor

### Focus Events

`FocusEvent` follows DOM spec:
- `relatedTarget`: Lost/gained element reference
- `bubbles: true`: Parent can observe child focus changes
- `cancelable: false`: Cannot cancel focus events

## Text Selection System

### SelectionState Model

anchor/focus model (like browser Selection API):
- `anchor`: Fixed endpoint (mouse press position)
- `focus`: Active endpoint (current drag position)
- `anchorSpan`: Word/line selection extension reference
- `scrolledOffAbove/Below`: Cache for scrolled content

### Selection Operations

| Operation | Function |
|-----------|----------|
| Start | `startSelection(col, row)` |
| Update | `updateSelection(col, row)` |
| Finish | `finishSelection()` |
| Word select | `selectWordAt(col, row)` |
| Line select | `selectLineAt(row)` |

### Word Character Classification

Matches iTerm2 defaults: `/[\p{L}\p{N}_/.\-+~\\]/u`
- Paths like `/usr/bin/bash` or `~/.claude/config.json` selected completely on double-click.

### Scroll Tracking

- `shiftAnchor`: Move anchor during drag-scroll
- `shiftSelection`: Move entire selection during keyboard scroll
- Soft wrap handling: Join lines correctly when copying

### Selection Rendering

`applySelectionOverlay`: Directly modify screen buffer styles with solid background color (not SGR-7 inverse), preserving syntax highlighting foreground colors.

## Keyboard Event Handling

### KeyboardEvent Class

```typescript
class KeyboardEvent extends TerminalEvent {
  key: string         // 'a', 'down', 'return', 'f1'
  ctrl: boolean
  shift: boolean
  meta: boolean       // Alt/Option
  superKey: boolean   // Cmd/Win
  fn: boolean
}
```

Printable check: `e.key.length === 1`

### Supported Protocols

| Protocol | Format |
|----------|--------|
| CSI u (Kitty) | `ESC[codepoint[;modifier]u` |
| modifyOtherKeys | `ESC[27;modifier;keycode~` |
| X10/SGR Mouse | Mouse event encoding |
| Application keyboard | `ESC O letter` (numpad) |

### Modifier Decoding

```typescript
decodeModifier(modifier) {
  m = modifier - 1  // Encoding offset
  return {
    shift: m & 1,    // Bit 0
    meta: m & 2,     // Bit 1 (Alt)
    ctrl: m & 4,     // Bit 2
    super: m & 8,    // Bit 3 (Cmd)
  }
}
```

## Event Dispatch

### Dispatcher Class

```typescript
class Dispatcher {
  currentEvent: TerminalEvent | null
  currentUpdatePriority: number
  discreteUpdates: Function
}
```

### Priority Mapping

- **Discrete** (sync): keydown, click, focus, blur, paste
- **Continuous** (batch): resize, scroll, mousemove

### Capture/Bubble Flow

`collectListeners` order: `[root-cap, ..., parent-cap, target-cap, target-bub, parent-bub, ..., root-bub]`

## Button Component

```typescript
type ButtonState = { focused: boolean, hovered: boolean, active: boolean }

<Button onAction={handler} tabIndex={0}>
  {(state) => <Text color={state.focused ? 'cyan' : 'white'}>Click</Text>}
</Button>
```

Features:
- No default styles (render function pattern)
- Multi-trigger: Enter, Space, Click
- Active state: 100ms visual feedback

## Connections

- [Focus Management](../concepts/focus-management.md)
- [Text Selection](../concepts/text-selection.md)
- [Ink UI Framework](../sources/2026-04-15-ink-ui-framework.md) - Core rendering system

## Open Questions

- How does selection handle terminal resize?
- What about accessibility considerations?

## Sources

- `chapters/chapter-36-焦点与交互.md`
- `/src/ink/focus.ts`
- `/src/ink/selection.ts`
- `/src/ink/events/dispatcher.ts`
- `/src/ink/parse-keypress.ts`
- `/src/ink/components/Button.tsx`