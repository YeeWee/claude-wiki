---
title: "Text Selection"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources:
  - ../sources/2026-04-15-focus-and-interaction.md
related:
---

# Text Selection

One-paragraph summary: Text Selection in Ink terminal UI supports mouse-based text selection with anchor/focus model, smart word and line selection, scroll tracking, and soft wrap handling for accurate copy operations.

## Selection Model

### anchor/focus

Like browser Selection API:
- **anchor**: Fixed endpoint (mouse press position)
- **focus**: Active endpoint (current mouse position)

Not start/end - anchor can be before or after focus depending on drag direction.

### SelectionState

```typescript
type SelectionState = {
  anchor: Point | null              // Start point
  focus: Point | null               // Current drag point
  isDragging: boolean               // Active drag
  anchorSpan: { lo, hi, kind }      // Word/line reference
  scrolledOffAbove: string[]        // Scroll cache above
  scrolledOffBelow: string[]        // Scroll cache below
  virtualAnchorRow?: number         // Track position beyond screen
}
```

## Selection Operations

### Basic

| Function | Action |
|----------|--------|
| `startSelection(col, row)` | Begin drag, set anchor |
| `updateSelection(col, row)` | Update focus during drag |
| `finishSelection()` | End drag, keep highlight |
| `clearSelection(s)` | Reset to empty |

### Smart

| Function | Trigger | Behavior |
|----------|---------|----------|
| `selectWordAt(col, row)` | Double-click | Select word at position |
| `selectLineAt(row)` | Triple-click | Select entire row |

### Word Boundary Detection

Character classification:
- 0: Whitespace (space, empty)
- 1: Word character - `/[\p{L}\p{N}_/.\-+~\\]/u`
- 2: Punctuation

Word boundaries: class change between adjacent characters.

Result: Paths like `/usr/bin/bash` or `~/.claude/config.json` selected completely.

## Scroll Handling

### Drag Scroll

`shiftAnchor`: Move anchor position when user drags beyond screen bounds:
- Track virtual row beyond screen
- Clamp to visible range
- Preserve position for return

### Keyboard Scroll

`shiftSelection`: Move entire selection with content:
- Update anchor and focus positions
- Handle cache of scrolled-off text
- Clear if both ends exceed same boundary

## Rendering

### Selection Overlay

`applySelectionOverlay`:
- Uses solid background color (not SGR-7 inverse)
- Preserves syntax highlighting foreground
- Respects `noSelect` markers (line numbers, diff markers)

### noSelect Areas

Some cells marked as non-selectable:
- Line numbers
- Diff +/- markers
- UI decorations

## Copy

`getSelectedText`:
1. Add scrolled-off cached lines
2. Extract in-screen selection
3. Handle soft wrap (join lines appropriately)

Soft wrap handling:
- `joinRows(lines, text, softWrap)`
- If softWrap true: concatenate to previous line
- Otherwise: new line entry

## Connections

- [Focus Management](../concepts/focus-management.md) - Related UI concept

## Open Questions

- How does selection handle multi-cell Unicode?
- What about accessibility for selection?

## Sources

- `/src/ink/selection.ts`
- `/src/ink/hooks/use-selection.ts`