---
name: iced-widget-patterns
description: Iced 0.14 widgets - virtualized lists for 100k rows, the native table widget, Element lifetimes, styling, keyboard focus, empty and error states.
metadata:
  category: desktop
  tags: [rust, iced, widgets, virtualization, table, lifetimes, styling, keyboard, focus, empty-state]
user-invocable: false
---

# Iced Widget Patterns

> **Iced is pre-1.0, and this plugin's projects track latest rather than pinning a minor.** Widget modules, style APIs, and constructor signatures move between minor versions, so **read the resolved version from `Cargo.lock`** - `Cargo.toml` holds a range and cannot answer whether a widget exists - and verify every widget signature against that version's docs before writing or accepting one.
>
> This skill owns **what is on screen and how it is built**. The `Message` enum, `update`, and state placement belong to `iced-architecture-patterns`; progress streaming and cancellation to `iced-async-patterns`; measured frame cost and profiling to `desktop-performance`; keyboard and contrast requirements as a policy to `desktop-accessibility`; translated strings to `desktop-i18n`.

## When to Use

- Building or reviewing a view function
- A result set is large enough that rendering every row is in question
- Adding a table, a custom widget, or a style
- Rendering the states around the happy path: scanning, empty, error, partial failure

## Rules

- **A list whose length is driven by user data is virtualized.** A dedup run returns 100k rows; building 100k `Element`s per frame is a freeze, not a slow frame
- `view` builds `Element`s from `&self` only. No allocation of the underlying data, no I/O, no mutation
- **Iced has no screen-reader or accessibility-tree support** (iced-rs/iced issue #552, open since October 2020). Keyboard operability and visual affordances are the only accessibility this stack delivers; never claim or imply screen-reader support
- Every interactive control is reachable and operable by keyboard, and focus order follows visual order
- A view renders four states explicitly: loading/scanning, empty, error, and populated. A view handling only the populated case is incomplete
- A custom widget is written only when composition of existing widgets cannot express it. Implementing `Widget` means owning layout, draw, and event handling by hand
- Styling goes through the theme and style functions, not per-widget hardcoded colours
- `Element<'a, Message>` borrows from the state it was built from; a widget holding data with a shorter lifetime than the returned `Element` will not compile, and cloning it into the widget to escape that is the wrong fix

## Patterns

### Virtualized lists

The core requirement for this app class. `scrollable` over a `column` of 100k rows constructs 100k widgets on every frame, whether or not they are visible.

```rust
// Bad - 100k Elements built per frame; the app locks up on a large dedup result
scrollable(column(groups.iter().map(row_view).collect::<Vec<_>>()))

// Good - only the visible window is built
let (first, count) = visible_window(scroll_offset, viewport_height, ROW_HEIGHT, groups.len());
let spacer_above = Space::with_height(first as f32 * ROW_HEIGHT);
let spacer_below = Space::with_height((groups.len() - first - count) as f32 * ROW_HEIGHT);
scrollable(column![spacer_above, column(groups[first..first + count].iter().map(row_view)), spacer_below])
    .on_scroll(Message::Scrolled)
```

This requires a fixed row height so offsets are computable without measuring. Variable-height rows need a prefix-sum index of heights, which is worth it only when the design demands them. Two rules that make the simple version correct: keep row height a constant the layout also uses, and clamp the window to the data length so a stale scroll offset after a rescan does not index out of bounds.

Before hand-rolling this, check whether the pinned version's `table` or `scrollable` already offers a lazy or windowed variant - Iced's widget set gains capabilities each minor, and reimplementing one is avoidable cost.

### The native `table` widget

Iced 0.14 ships `iced::widget::table` natively, along with `markdown`, `text_editor`, and `highlighter`. Use the native table for tabular result sets rather than hand-building a grid of rows and columns, which is the pattern earlier Iced versions forced.

```rust
// Bad - a grid assembled from rows and columns; column alignment drifts and
// resizing is reimplemented from scratch
column(rows.iter().map(|r| row![text(&r.name).width(300), text(r.size), text(&r.path)]))

// Good - the native widget, with columns declared once
use iced::widget::table;
table(columns, &self.rows)
```

Verify the exact `table` constructor, column-definition type, and whether it virtualizes internally against the pinned 0.14 docs - if it does not virtualize, the windowing above still applies to its row source. Do not assume a hand-rolled table from an older codebase should stay; migrating removes code.

### Element lifetimes

```rust
// Bad - builds a String per row per frame, then borrows it into the Element
fn row_view(entry: &FileEntry) -> Element<'_, Message> {
    let label = format!("{} ({} bytes)", entry.name, entry.size);
    text(&label).into()   // does not compile: label dropped at end of fn
}

// Good - the widget owns what it must, and borrows what outlives it
fn row_view(entry: &FileEntry) -> Element<'_, Message> {
    row![text(entry.name.as_str()), text(entry.size.to_string())].into()
}
```

The signal to read here: a compile error about a lifetime in `view` usually means a value is being computed during rendering that should have been computed in `update` and stored. Formatting 100k labels per frame is the cost even when the borrow is made to work - precompute display strings into the model when they are stable, and format only the visible window when they are not.

### Custom widgets

Reach for composition first. Implementing `Widget` means writing `size`, `layout`, `draw`, and `on_event` yourself and keeping them consistent across version bumps.

| Need | Approach |
| --- | --- |
| A labelled control, a card, a toolbar | Compose existing widgets in a helper `fn -> Element` |
| A one-off visual (histogram, thumbnail grid cell) | `canvas` |
| Reusable behaviour over existing widgets | Wrap in a helper, or `mouse_area` / `responsive` |
| Genuinely new layout and hit-testing | Implement `Widget` |

A helper function returning `Element` gives reuse at zero framework cost and is the answer in most cases.

### Styling

```rust
// Bad - a literal colour per widget; the theme is bypassed and dark mode breaks
container(content).style(|_| container::Style { background: Some(Color::WHITE.into()), ..Default::default() })

// Good - derived from the active theme
container(content).style(|theme: &Theme| {
    let palette = theme.extended_palette();
    container::Style { background: Some(palette.background.weak.color.into()), ..Default::default() }
})
```

Hardcoded colours are the reason an app looks correct in light mode and unreadable in dark. Derive from the palette, and keep shared styles in one module so a variant is defined once.

### Keyboard, focus, and the accessibility ceiling

Focus order follows construction order, so build the tree in the order the user should traverse it rather than reordering visually afterwards. Wire the conventional bindings explicitly - Enter to confirm, Escape to dismiss, arrows to move selection in a list - via a keyboard subscription or `on_press` handlers; Iced does not supply them.

State the ceiling plainly when accessibility is discussed: Iced exposes no accessibility tree, so a screen reader announces nothing about this UI. That is a stack limitation, not a project defect, and it is not closed by widget choices. What is in scope: keyboard operability, visible focus indicators, sufficient contrast from the theme palette, and text that is not the only carrier of a state (pair colour with a label or icon).

### The states around the happy path

```rust
// Bad - renders only the populated case; an empty or failed scan shows a blank pane
scrollable(column(self.groups.iter().map(row_view)))

// Good - each state is a branch with its own affordance
match &self.scan {
    ScanState::Idle => empty_prompt("Choose a folder to scan"),
    ScanState::Running { done, total } => progress_view(*done, *total),   // with a Cancel button
    ScanState::Failed(err) => error_view(err),                            // message + Retry
    ScanState::Done(groups) if groups.is_empty() => empty_result("No duplicates found"),
    ScanState::Done(groups) => results_view(groups),
}
```

Model these as one enum rather than a set of independent booleans, so impossible combinations (`is_loading && has_error`) cannot be represented. A partial-failure result is a populated state with a banner, not an error state - the successes are still shown.

## Output Format

When this skill produces a finding:

```
[Must|Recommend] <file:line>
Category: <virtualization | table-widget | element-lifetime | custom-widget | styling | keyboard-focus | missing-state | per-frame-work>
Issue: <the defect, named>
Consequence: <what the user sees - "UI locks for ~8s on a 100k-row result", "unreadable in dark theme">
Fix: <the concrete change>
```

When designing a view rather than reviewing:

```
Iced version: <the resolved version from Cargo.lock | UNRESOLVED - read before proceeding>
Widgets: <the composition, native widgets named>
Large lists: <virtualized, with row height and window source | N/A - bounded at <n> rows>
States rendered: <idle | loading | empty | error | partial | populated - list every branch>
Keyboard: <focus order and the bindings wired>
Accessibility ceiling: <screen readers unsupported by Iced (issue #552); keyboard and contrast covered>
Styling: <theme-derived | hardcoded values, and why>
```

Any signature stated for an Iced widget carries `verified against <version>` or `UNVERIFIED - confirm against the pinned version`. No Iced signature is asserted without one of the two.

## Avoid

- Building an `Element` per item for a list whose length is user-data-driven
- `scrollable` over a `column` of every row in a large result set
- Hand-building a grid of rows and columns where the native `table` fits
- `format!`, `to_string`, or any per-row string construction inside `view` for off-screen rows
- Cloning data into a widget to silence a lifetime error
- Computing derived values in `view` that `update` could store
- Implementing `Widget` where a helper `fn -> Element` or `canvas` suffices
- Hardcoded `Color` literals instead of the theme palette
- Interactive controls with no keyboard path, or focus order that contradicts visual order
- Claiming or implying screen-reader support for an Iced UI
- Colour as the only signal of a state
- A view that renders the populated case and leaves loading, empty, and error blank
- Independent booleans for view state where one enum makes bad combinations unrepresentable
- Reproducing a widget signature from memory instead of the pinned version's docs
