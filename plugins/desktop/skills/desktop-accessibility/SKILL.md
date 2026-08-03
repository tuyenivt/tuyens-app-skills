---
name: desktop-accessibility
description: Make Iced desktop apps usable without a mouse - keyboard reach, focus order and indicators, contrast, OS text scaling, non-colour meaning.
metadata:
  category: desktop
  tags: [accessibility, a11y, iced, keyboard-navigation, focus, contrast, text-scaling, screen-reader-gap, accesskit]
user-invocable: false
---

# Desktop Accessibility

> Confirm the UI framework is Iced before applying this skill - its central constraint is an Iced-specific platform gap, and the guidance is scoped to what remains achievable under it.
>
> This skill owns **usability without a mouse and without perfect vision**. Widget composition and layout belong to `iced-widget-patterns`; message and focus state plumbing to `iced-architecture-patterns`; translated strings and text expansion to `desktop-i18n`; keyboard-driven OS shortcuts to `desktop-platform-integration`.

## When to Use

- Adding or reviewing any interactive widget, dialog, or modal
- Choosing colours that carry meaning (status, validation, diff, selection)
- Sizing text, buttons, list rows, or icon-only controls
- Designing a flow whose only affordance is click or hover
- Reviewing a screen for keyboard reachability

## Rules

- **Iced has no screen-reader support, and this cannot be fixed in application code.** Iced does not consume AccessKit; there is no accessibility tree, no role, no name, and no announcement. Issue #552 has been open since October 2020. Every recommendation in this skill is a mitigation within that constraint
- **Do not assert that Iced 0.14 added accessibility hooks.** Several third-party 2026 articles claim this. It is not in the release notes and no such API exists. Do not restate the claim and do not build on it
- **Do not propose an AccessKit integration.** The framework has no consumption point for it. Proposing one produces work that cannot compile against the framework. `egui` and `slint` do consume AccessKit - naming them is how a framework-swap tradeoff is stated, not a fix to apply downstream
- **Every action is reachable and operable by keyboard alone.** An action available only via click, hover, right-click, or drag is inaccessible with no workaround in this stack
- **Focus is always visible.** A focus indicator that relies on a subtle background shift, or is drawn only on hover, is not an indicator
- Colour never carries meaning alone. Every colour-coded state also carries text, an icon, or a shape
- Text scales with the OS setting rather than being pinned to a fixed pixel size, and the layout holds at the larger size
- A modal traps focus for its lifetime and returns focus to the invoking control on dismiss. `Esc` always dismisses

## Patterns

### State the platform gap, once, accurately

When accessibility scope is discussed, the honest statement is:

> Iced exposes no accessibility tree. Screen readers (NVDA, JAWS, VoiceOver) see an empty window. This is a framework-level exclusion, tracked upstream since 2020, and is not addressable from application code. The app targets keyboard, contrast, scaling, and non-colour affordances; blind-user support requires a different UI framework.

That is a scope declaration, not a defence. It belongs in the project's accessibility statement so the exclusion is a known decision rather than an unnoticed one. Repeating it as a header on every finding is noise - say it once.

### Keyboard reachability

```rust
// Bad - the only way to remove a row; no keyboard path at all
mouse_area(row).on_right_press(Message::Remove(id))

// Good - a focusable control plus a key binding on the focused row
row![ label, button(icon::trash()).on_press(Message::Remove(id)) ]
// and, in subscription:
keyboard::on_key_press(|key, _| match key {
    Key::Named(Named::Delete) => Some(Message::RemoveFocused),
    _ => None,
})
```

The checklist per screen: Tab reaches every actionable control; Shift-Tab reverses in the same order; Enter or Space activates the focused control; `Esc` leaves the current context; arrow keys move within a list or grid; and no control is reachable only by hover or drag.

Iced's `widget::focus_next` / `focus_previous` operate on the widget tree order, so **DOM-equivalent order is layout order**. A control positioned visually first but constructed last receives focus last, which is a real defect the eye does not catch - verify by tabbing, not by reading the code.

### Focus indicators

```rust
// Bad - focus differs from rest by a 4% background lift; invisible on a laptop panel
fn focused(t: &Theme) -> Style { Style { background: t.palette().background.scale(1.04).into(), ..base(t) } }

// Good - a border that changes width and colour, so it survives low contrast and colour blindness
fn focused(t: &Theme) -> Style {
    Style { border: Border { width: 2.0, color: t.palette().primary, radius: 4.0.into() }, ..base(t) }
}
```

The indicator must be visible in both light and dark themes and must not be the same treatment as hover - a user tabbing needs to distinguish "focused" from "the mouse happens to be here".

### Contrast and non-colour meaning

| Element | Minimum ratio against its background |
| --- | --- |
| Body and label text | 4.5:1 |
| Text 18pt+ or 14pt bold | 3:1 |
| Icons, focus rings, control borders | 3:1 |
| Disabled text | Exempt from the ratio, but must still read as disabled |

Verify against the actual theme palette in both light and dark, not against a design mock.

```rust
// Bad - the only signal that a rename will fail is a red row
row.style(if item.conflicts { red } else { normal })

// Good - colour plus an icon plus text; readable in greyscale
row![ if item.conflicts { icon::warning() } else { icon::blank() },
      text(&item.name),
      text(if item.conflicts { "name taken" } else { "" }) ]
```

Roughly 8% of men have a colour vision deficiency. For a batch rename tool the colour-only case is specifically dangerous: the user confirms a destructive apply without perceiving the warning.

### Text scaling and hit targets

Read the OS text scale at startup and apply it as a factor over the base size rather than hardcoding `size(14)` everywhere. Then verify the layout at 200%: fixed-height rows clip descenders, fixed-width buttons truncate labels, and single-line containers hide content entirely. Combine with the longest locale (`desktop-i18n`) - that is the real worst case, not either axis alone.

Hit targets: 24x24 logical pixels minimum for any clickable control, 32x32 for primary actions and icon-only buttons. Icon-only controls also need a tooltip, since with no accessibility tree the tooltip is the only name the control has - and a tooltip that appears on hover only is unreachable by keyboard, so the label must also exist somewhere the keyboard user can read.

### Dialogs and focus traps

A modal takes focus on open, cycles Tab within itself, and restores focus to the control that opened it on close. **The one thing worse than no trap is a permanent trap**: a dialog that captures Tab but offers no keyboard-reachable dismiss strands the user with no way out but killing the process. `Esc` dismisses, always, and the destructive-confirm dialog defaults focus to the safe choice (Cancel), not to Apply.

## Output Format

Two modes, chosen by whether the request supplies code to judge or asks for code to be produced.

**Authoring mode** - the request is to write or design something. Emit the code or design, then any `Deferred:` lines. No finding blocks, no severity, no status line.

**Review mode** - source, a diff, or a symptom report was supplied. Emit one block per finding.

```
### [Severity] {file:line | symbol, when source was supplied without paths | symptom, when no source was supplied}

- Category: {KeyboardReach | FocusOrder | FocusIndicator | FocusTrap | Contrast | ColourOnly | TextScaling | HitTarget | MissingLabel}
- Evidence: {source | inferred (state what was not seen)}
- Code: {one-line citation, or `not supplied` when the finding is inferred}
- Barrier: {who is blocked and from what - "keyboard-only user cannot remove a queued file"}
- Fix: {the concrete change}
```

`Severity: {Critical | High | Medium | Low}` - Critical = an action cannot be performed without a mouse, or a modal traps focus with no keyboard dismiss. High = a destructive or warning state carried by colour alone, focus that is invisible, or focus order that does not match visual order. Medium = contrast below the ratio table, layout breaking at 200% text scale, or a hit target under 24x24. Low = a missing tooltip on a labelled control, or an indicator that is visible but weak.

Severity that does not fit a listed band: assign the nearest lower band and state why in `Barrier`. `Category` takes exactly one value - where a defect fits two, pick the one the `Fix` addresses and name the other in `Barrier`.

`Evidence: inferred` is required whenever the source was not read. It bounds the header at High and never raises a block.

When the report's scope includes screen-reader support, emit exactly one scope line before the findings: `Screen-reader support is out of scope - Iced exposes no accessibility tree (upstream issue #552, open since 2020) and this is not addressable in application code.` Never emit a finding whose fix is "add screen-reader support" or "integrate AccessKit".

A defect owned by a sibling named in the ownership blockquote is written after the findings as `Deferred: {defect} -> {owning skill}`, one per line. Omit when there are none.

In review mode, close with exactly one status line, after any `Deferred:` lines:

| Condition | Line |
| --- | --- |
| One or more findings emitted | none - the findings are the output |
| Source or a symptom report was available, nothing found | `No accessibility findings.` |
| No source, diff, or symptom supplied | `Accessibility check not run: no source supplied.` |

## Avoid

- Stating or implying that Iced supports screen readers
- Repeating the third-party claim that Iced 0.14 added accessibility hooks
- Proposing an AccessKit integration for Iced
- Filing a finding whose fix the framework cannot accept
- An action reachable only by click, hover, right-click, or drag
- Focus order that follows visual position rather than construction order
- A focus indicator distinguished only by a background tint, or identical to hover
- Colour as the sole carrier of a warning, error, or conflict state
- Hardcoded text sizes that ignore the OS scale setting
- Fixed-height rows or fixed-width buttons verified only at 100% scale
- Icon-only controls whose only name is a hover tooltip
- Hit targets under 24x24 logical pixels
- A modal that traps Tab but has no keyboard dismiss
- A destructive confirm dialog with initial focus on the destructive action
