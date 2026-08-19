# Quality Guidelines

> Code quality standards for backend development.

---

## Overview

<!--
Document your project's quality standards here.

Questions to answer:
- What patterns are forbidden?
- What linting rules do you enforce?
- What are your testing requirements?
- What code review standards apply?
-->

(To be filled by the team)

## Shared Core: Shift-Click Selection

The shared terminal interaction code in `src/Surface.zig` owns mouse selection
state for both macOS and GTK. Changes to Shift-click extension must preserve the
following executable contracts:

### Behavior Contract

- `mouse-shift-click-extend` defaults to `true`. When disabled, the flow must
  retain the v1.3.1 behavior: only an existing selection can be extended, and
  the previous click pin remains the fixed anchor.
- The Shift-click fast-path must yield to double/triple-click handling only
  when both conditions hold: elapsed time is within `mouse-interval` and the
  physical distance from `left_click_xpos/left_click_ypos` is at most one cell
  width. A distant click must extend even when it happens quickly.
- A tracked click pin may be used only on the same active screen. If the
  primary/alternate screen changed, consume the extension attempt without
  changing or untracking the existing pin.
- With no selection, use `mouse.left_click_pin` as the anchor. With a
  selection, keep the endpoint farther from the click and move the nearer
  endpoint, using `Selection.topLeft` / `bottomRight` and linear screen-cell
  distance.
- When redirecting an anchor, track the new pin before untracking the old pin.
  Set `left_click_xpos` to the cell's left edge for a left anchor and to its
  right edge for a right anchor so `mouseSelection` preserves the 60% threshold
  rule.

### Required Tests

The `Surface: shift click endpoint logic` test must cover clicks before,
after, and inside a selection, reverse and rectangle selections, single-cell
selections, cross-row distances, and both anchor pixel-edge cases. Runtime
changes to the click flow also require checking mouse reporting, copy-on-select,
screen switching, and the disabled configuration path.

### Common Mistakes

- Do not use `hasSelection()` as the only gate; a plain click intentionally
  clears the selection before the Shift-click target is chosen.
- Do not untrack the old pin before `trackPin` succeeds, or an allocation error
  can lose the user's anchor.
- Do not compare only click time; that makes a fast click far away look like a
  double-click and breaks the core interaction.

---

## Forbidden Patterns

<!-- Patterns that should never be used and why -->

(To be filled by the team)

---

## Required Patterns

<!-- Patterns that must always be used -->

(To be filled by the team)

---

## Testing Requirements

<!-- What level of testing is expected -->

(To be filled by the team)

---

## Code Review Checklist

<!-- What reviewers should check -->

(To be filled by the team)
