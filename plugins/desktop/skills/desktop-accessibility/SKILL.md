---
name: desktop-accessibility
description: Make Avalonia apps accessible - AutomationProperties names, custom peers, NVDA and Narrator smoke tests, keyboard reach, focus, contrast, scaling.
metadata:
  category: desktop
  tags: [accessibility, a11y, avalonia, automation-peer, uia, nsaccessibility, screen-reader, keyboard-navigation, focus, contrast, text-scaling]
user-invocable: false
---

# Desktop Accessibility

> Confirm the UI framework is Avalonia 11 or later before applying this skill - the guidance assumes its UI Automation and NSAccessibility bridges exist, and it names the specific places where they still fray.
>
> This skill owns **usability without a mouse, without perfect vision, and with a screen reader**. Control composition and layout belong to `avalonia-control-patterns`; command and focus-state plumbing to `avalonia-mvvm-patterns`; translated strings and text expansion to `desktop-i18n`; OS-level shortcuts to `desktop-platform-integration`; destructive-flow preview and undo semantics to `desktop-batch-operations`.

## When to Use

- Adding or reviewing any interactive control, dialog, or modal
- Writing a custom-drawn control, or a screen built on DataGrid or custom text editing
- Choosing colours that carry meaning (status, validation, diff, selection)
- Sizing text, buttons, list rows, or icon-only controls
- Reviewing a screen for keyboard or screen-reader usability

## Rules

- **Avalonia has real screen-reader support; hold the app to it.** AutomationPeers map to UI Automation on Windows and NSAccessibility on macOS, shipped since v11 (July 2023; AT-SPI2 on Linux since v12). NVDA, Narrator, and VoiceOver work with Avalonia apps - never scope screen readers out of an accessibility review on this stack
- **Every interactive control has an accessible name**: `AutomationProperties.Name`, a `LabeledBy` reference, or text content the peer already surfaces. An icon-only button without one announces as "button"
- **A custom-drawn control overrides `OnCreateAutomationPeer`** or it does not exist to assistive technology: no role, no name, no state
- **Peers existing does not imply usability - smoke test with a real screen reader.** Known gaps: TextBox caret movement is not announced while arrowing through text (issue #9770), and DataGrid keyboard accessibility is weak (discussion #10175). Composite controls are where support frays; buttons, lists, checkboxes, and menus are solid
- **Every action is reachable and operable by keyboard alone.** An action available only via click, hover, right-click, or drag is inaccessible
- **Focus is always visible**, and focus order matches visual order
- Colour never carries meaning alone. Every colour-coded state also carries text, an icon, or a shape
- Text scales with the OS setting rather than being pinned to fixed pixel sizes, and the layout holds at 200%
- A modal traps focus for its lifetime, `Esc` always dismisses, and focus returns to the invoking control on close

## Patterns

### The support statement, stated accurately

When accessibility scope is discussed, the honest statement is:

> Avalonia exposes an accessibility tree: AutomationPeers map to UI Automation on Windows and NSAccessibility on macOS. NVDA, Narrator, and VoiceOver announce standard controls. Known gaps: TextBox caret movement is not announced while arrowing (#9770), and DataGrid keyboard access is weak (#10175). Standard controls are solid; composite and custom controls are verified by testing, not assumed.

Both misclaims are defects: asserting no support (true of some desktop GUI stacks, stale here) scopes out users the stack can serve; asserting parity with mature native toolkits hides exactly the gaps the smoke test exists to catch.

### Names for assistive technology

```xml
<!-- Bad - Narrator announces "button"; the user hears nothing about what it does -->
<Button Command="{Binding RemoveCommand}">
  <PathIcon Data="{StaticResource TrashIcon}"/>
</Button>

<!-- Good - a spoken name for AT, a tooltip for sighted users -->
<Button Command="{Binding RemoveCommand}"
        AutomationProperties.Name="Remove file"
        ToolTip.Tip="Remove file">
  <PathIcon Data="{StaticResource TrashIcon}"/>
</Button>
```

- `AutomationProperties.Name` is what is spoken; `HelpText` adds a longer description read on demand; `LabeledBy` points a control at an existing visible label instead of duplicating the string
- `AutomationProperties.AutomationId` is for UI tests. It is never spoken and does not substitute for a `Name`
- Names are localized resources (`desktop-i18n`) - a translated UI with English automation names is half translated
- State the user must learn about (scan finished, conflicts found) needs a control that announces or can be focused and read - a repaint of a coloured region tells a screen reader nothing; verify the announcement in the smoke test

### Custom controls need a peer

```csharp
// Bad - custom-drawn thumbnail grid; assistive tech sees one opaque rectangle
public class ThumbnailGrid : Control { /* custom rendering */ }

// Good - the control describes its role, name, and children to AT
public class ThumbnailGrid : Control {
    protected override AutomationPeer OnCreateAutomationPeer()
        => new ThumbnailGridPeer(this);  // ControlAutomationPeer subclass: role, name, items, selection
}
```

Before writing a custom control at all, compose standard ones - `ListBox` and friends carry working peers and keyboard handling for free; the composition choice belongs to `avalonia-control-patterns`. A peer that lies (announces a list but exposes no items) fails the smoke test as surely as no peer.

### Keyboard reachability

```xml
<!-- Bad - the only way to remove a row is a pointer handler -->
<Border PointerReleased="OnRightClickRemove"> ... </Border>

<!-- Good - selection plus a key binding; the pointer path can stay -->
<ListBox ItemsSource="{Binding Files}" SelectionMode="Multiple">
  <ListBox.KeyBindings>
    <KeyBinding Gesture="Delete" Command="{Binding RemoveSelectedCommand}"/>
  </ListBox.KeyBindings>
</ListBox>
```

The checklist per screen: Tab reaches every actionable control; Shift-Tab reverses in the same order; Enter or Space activates the focused control; `Esc` leaves the current context; arrow keys move within a list or grid; and nothing is reachable only by hover or drag.

Default tab order follows the visual tree, so a control constructed out of visual order tabs out of order - fix the tree; `TabIndex` is the override of last resort. `IsTabStop="False"` removes stops that carry no action. Verify by tabbing, not by reading the XAML.

### Focus indicators

```xml
<!-- Bad - focus differs from rest by a 4% background lift; invisible on a laptop panel -->
<Style Selector="Button:focus">
  <Setter Property="Background" Value="#0A000000"/>
</Style>

<!-- Good - a border change that survives low contrast and colour blindness -->
<Style Selector="Button:focus-visible">
  <Setter Property="BorderBrush" Value="{DynamicResource SystemAccentColor}"/>
  <Setter Property="BorderThickness" Value="2"/>
</Style>
```

`:focus-visible` fires for keyboard focus, which is the case the indicator serves. It must be visible in both light and dark themes and must differ from hover - a user tabbing needs to distinguish "focused" from "the mouse happens to be here".

### Contrast and non-colour meaning

| Element | Minimum ratio against its background |
| --- | --- |
| Body and label text | 4.5:1 |
| Text 18pt+ or 14pt bold | 3:1 |
| Icons, focus rings, control borders | 3:1 |
| Disabled text | Exempt from the ratio, but must still read as disabled |

```xml
<!-- Bad - the only signal that a rename will fail is a red row -->
<TextBlock Text="{Binding Name}" Foreground="{Binding ConflictBrush}"/>

<!-- Good - icon plus text plus colour; survives greyscale and is announced -->
<StackPanel Orientation="Horizontal">
  <PathIcon IsVisible="{Binding HasConflict}" Data="{StaticResource WarnIcon}"/>
  <TextBlock Text="{Binding Name}"/>
  <TextBlock IsVisible="{Binding HasConflict}" Text="name taken"/>
</StackPanel>
```

Roughly 8% of men have a colour vision deficiency, and a colour-only warning is also invisible to a screen reader. For a batch rename tool this case is specifically dangerous: the user confirms a destructive apply without perceiving the warning.

### Text scaling and hit targets

Apply the OS text scale as a factor over base sizes rather than hardcoding `FontSize="12"` everywhere, then verify the layout at 200%: fixed-height rows clip descenders, fixed-width buttons truncate labels, single-line containers hide content entirely. Combine with the longest shipped locale (`desktop-i18n`) - that is the real worst case, not either axis alone.

Hit targets: 24x24 logical pixels minimum for any clickable control, 32x32 for primary actions and icon-only buttons. Icon-only controls carry both a tooltip and an `AutomationProperties.Name` - the tooltip serves the sighted mouse user, the name serves everyone else.

### Dialogs and focus traps

A modal takes focus on open, cycles Tab within itself, and restores focus to the control that opened it on close. **The one thing worse than no trap is a permanent trap**: a dialog that captures Tab but offers no keyboard-reachable dismiss strands the user. `Esc` dismisses, always, and the destructive-confirm dialog defaults focus to the safe choice (Cancel), never Apply.

### The screen-reader smoke test

Before a screen is called accessible, walk it with NVDA or Narrator on Windows (VoiceOver on macOS): every interactive control announces a role and a name; Tab order matches visual order; the state changes the flow depends on (preview ready, conflicts found, batch complete) are announced or reachable; and the whole rename or dedup flow completes without touching the pointer. Screens built on the known-gap surfaces - DataGrid, custom text editing - are tested first, because that is where peers exist and usability still fails.

## Output Format

Two modes, chosen by whether the request supplies code to judge or asks for code to be produced.

**Authoring mode** - the request is to write or design something. Emit the code or design, then any `Deferred:` lines. No finding blocks, no severity, no status line.

**Review mode** - source, a diff, or a symptom report was supplied. Emit one block per finding, ordered by severity, Critical first.

```
### [Severity] {file:line | file, when the line is unknown | symbol, when source was supplied without paths | symptom, when no source was supplied}

- Category: {MissingName | PeerGap | KeyboardReach | FocusOrder | FocusIndicator | FocusTrap | Contrast | ColourOnly | TextScaling | HitTarget}
- Evidence: {tested (name the screen reader and the screen walked - a named tester's reported session counts) | source (a prose description of the implementation counts; quote it in Code:) | inferred (state what was not seen)}
- Code: {one-line citation, or `not supplied` when the finding is tested or inferred}
- Barrier: {who is blocked and from what - "an NVDA user cannot tell which rows conflict"}
- Fix: {the concrete change}
```

`Severity: {Critical | High | Medium | Low}` - Critical = an action cannot be performed without a mouse, a modal traps focus with no keyboard dismiss, or a primary flow runs through a control invisible to assistive technology (no peer, no name, no alternative). High = a missing accessible name on an interactive control, a custom control without a peer off the primary flow, a destructive or warning state carried by colour alone, focus that is invisible, focus order diverging from visual order, or a known-gap surface (#9770, #10175) shipped without a smoke test. Medium = contrast below the ratio table, layout breaking at 200% text scale, a hit target under 24x24, or `AutomationId` standing in for a `Name`. Low = a missing tooltip on a named control, or an indicator that is visible but weak.

Severity that does not fit a listed band: assign the nearest lower band and state why in `Barrier`. `Category` takes exactly one value - where a defect fits two, pick the one the `Fix` addresses and name the other in `Barrier`. Repeated instances of the same defect in one file merge into one block with every location named. `PeerGap` covers both a custom control with no peer and a framework gap surface shipped untested. `FocusTrap` covers dialog focus discipline generally: trap lifecycle, keyboard dismiss, initial focus, and restore on dismiss.

`Evidence: inferred` is required whenever neither source nor a screen-reader run informed the finding. It bounds the header at High and never raises a block; when the cap lowers a would-be Critical, say so in `Barrier`. A claim that a screen *works* with a screen reader requires `tested` evidence - `source` proves peers exist, not that they are usable.

When the request's scope includes screen-reader support - in either mode - emit exactly one scope line before the findings or the authored output: `Screen-reader support is in scope - Avalonia maps AutomationPeers to UI Automation and NSAccessibility; known gaps: TextBox caret announcement (#9770), DataGrid keyboard access (#10175).` Never emit a finding claiming screen-reader support is impossible on this stack, and never close a screen as accessible on peer presence alone.

A defect - or, in authoring mode, out-of-scope work the deliverable depends on - owned by a sibling named in the ownership blockquote is written after the findings or the authored output as `Deferred: {item} -> {owning skill}`, one per line. Omit when there are none.

In review mode, after any `Deferred:` lines, close per this table - when findings were emitted there is no separate closing line:

| Condition | Line |
| --- | --- |
| One or more findings emitted | none - the findings are the output |
| Source or a symptom report was available, nothing found | `No accessibility findings.` |
| No source, diff, or symptom supplied | `Accessibility check not run: no source supplied.` |

## Avoid

- Stating that Avalonia lacks screen-reader support, or scoping assistive technology out of a review
- Calling a screen screen-reader usable because peers exist, without an NVDA or Narrator pass
- Icon-only controls with no `AutomationProperties.Name`
- `AutomationId` treated as a spoken name
- English automation names in a localized UI
- A custom-drawn control with no `AutomationPeer`
- Shipping DataGrid-heavy or custom-text-editing screens without testing the known gaps (#9770, #10175)
- An action reachable only by click, hover, right-click, or drag
- Construction order that diverges from visual order, so Tab jumps around the screen
- A focus indicator distinguished only by a background tint, or identical to hover
- Colour as the sole carrier of a warning, error, or conflict state
- Fixed-height rows or fixed-width buttons verified only at 100% scale
- Hit targets under 24x24 logical pixels
- A modal that traps Tab but has no keyboard dismiss
- A destructive confirm dialog with initial focus on the destructive action
