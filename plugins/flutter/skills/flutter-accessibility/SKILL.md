---
name: flutter-accessibility
description: "Review Flutter accessibility: Semantics labels, TalkBack/VoiceOver, colour contrast, touch-target size, text scaling, focus order, announcements."
metadata:
  category: mobile
  tags: [flutter, dart, accessibility, a11y, semantics, screen-reader, contrast, touch-target, text-scaling, focus]
user-invocable: false
---

# Flutter Accessibility

> Platform tiers are defined in `flutter-adaptive-responsive`; this skill states the tier each pattern applies to. Localized strings are `flutter-i18n`'s - a semantic label is user-facing text and follows the same no-hardcoding rule.

## When to Use

- Building or reviewing any interactive surface: buttons, icon buttons, form fields, list tiles, custom gesture targets, dialogs
- Adding a custom-painted or gesture-based control that has no built-in semantics
- Auditing an existing screen against screen reader, contrast, target size, or text-scale requirements
- A report that a control is unlabelled, unreachable by keyboard, invisible to TalkBack or VoiceOver, or overflows when the user raises the system font size

## Rules

- Every interactive element exposes a label that names its **action or destination**, not its icon. If a screen reader cannot say what the control does, it is broken
- Labels are localized strings, never literals in the widget tree
- Decorative images and icons are hidden from the semantics tree; meaningful ones carry a text alternative. An image that carries information the surrounding text does not repeat is meaningful
- Minimum interactive target is 48x48 dp on Android and 44x44 pt on iOS. Small visuals are padded up to the target, not shrunk to fit
- Text contrast meets WCAG AA: 4.5:1 for body text, 3:1 for large text (>= 18pt, or >= 14pt bold) and for UI component boundaries and meaningful graphics
- No widget assigns a fixed height, width, or aspect ratio to a box whose only content is text. Text size is user-controlled and grows without bound
- Text scaling is supported to at least 200%. Clamping is a last resort, applied at the narrowest possible scope, never app-wide
- Focus order follows visual reading order, and every action reachable by pointer is reachable by keyboard on any tier where a keyboard exists
- A state change the user did not directly cause, or whose result is off-screen, is announced. A silent success is a failure for a screen reader user
- Colour is never the only carrier of meaning: pair it with text, an icon, or a shape

## Patterns

### Labelling interactive elements

*Tiers: All.*

```dart
// Bad - TalkBack announces "button"; VoiceOver announces nothing useful
IconButton(icon: const Icon(Icons.delete), onPressed: _delete)

// Good - tooltip feeds the semantics tree and gives sighted users a hover hint too
IconButton(
  icon: const Icon(Icons.delete),
  tooltip: l10n.deleteOrder,
  onPressed: _delete,
)
```

Icon-only controls are the single most common defect: the icon has no text, so the semantics node has no label. `tooltip:` on `IconButton` is the shortest correct fix. For controls with no tooltip parameter, wrap:

```dart
Semantics(button: true, label: l10n.playTrack, child: _customPlayControl())
```

Label the action, not the glyph. `"delete order"` is a label; `"trash can icon"` is not. Do not include the role in the label - the framework already announces "button", so `label: 'Delete button'` is read as "delete button, button".

A gesture handler with no semantics is invisible to assistive tech:

```dart
// Bad - GestureDetector produces no semantics node at all
GestureDetector(onTap: _openProfile, child: _avatar())

// Good - either use a real button widget, or declare the semantics
Semantics(button: true, label: l10n.openProfile, child: GestureDetector(onTap: _openProfile, child: _avatar()))
```

Prefer the real widget (`InkWell`, `TextButton`, `ListTile`) over declaring semantics by hand; the framework's own controls already carry role, state, and focus behaviour.

### What the screen reader actually says

*Tiers: Mobile primary (TalkBack, VoiceOver); desktop uses the OS reader; web builds a separate DOM semantics tree and must be verified on the real web build.*

```dart
// Bad - three separate nodes; the user swipes three times to hear one row
Row(children: [Icon(Icons.check), Text(order.id), Text(order.total)])

// Good - one node, one coherent announcement
MergeSemantics(
  child: Row(children: [const Icon(Icons.check), Text(order.id), Text(order.total)]),
)
```

The semantics tree, not the widget tree, is what gets read. Three `Text` widgets in a row are three stops in swipe navigation unless merged. Conversely, one node holding a whole screen is unnavigable.

| Widget | Effect |
|--------|--------|
| `Semantics` | creates or annotates a node (label, hint, value, role flags) |
| `MergeSemantics` | collapses descendants into one node so the row is read as one item |
| `ExcludeSemantics` | drops the subtree from the tree entirely (decoration) |
| `BlockSemantics` | hides everything painted beneath, for a modal overlay |

Use `hint:` for what happens on activation when the label alone does not imply it (`label: order.id`, `hint: l10n.opensOrderDetail`). Use `value:` for a control's current setting, which is what makes a slider or toggle intelligible.

### Images and icons

*Tiers: All.*

```dart
// Bad - a purely decorative divider icon becomes a swipe stop
Icon(Icons.chevron_right)

// Good
ExcludeSemantics(child: Icon(Icons.chevron_right))
```

```dart
// Bad - the chart is the only place the trend appears, and it is silent
Image.asset('assets/revenue_chart.png')

// Good
Image.asset('assets/revenue_chart.png', semanticLabel: l10n.revenueUpPercent(12))
```

`Image` and `Icon` both take `semanticLabel`. The test is whether removing the image loses information: a chevron next to a labelled row loses nothing (exclude it); a chart, a status badge, or an avatar that identifies a person carries information (label it).

### Touch targets

*Tiers: Mobile primary; pointer tiers benefit but are less constrained.*

```dart
// Bad - a 20x20 tap target; usable only with a precise touch
GestureDetector(onTap: _close, child: const Icon(Icons.close, size: 20))

// Good - the icon stays 20dp, the target grows to the minimum
InkWell(
  onTap: _close,
  child: const SizedBox(
    width: kMinInteractiveDimension,   // 48.0
    height: kMinInteractiveDimension,
    child: Icon(Icons.close, size: 20),
  ),
)
```

Visual size and target size are independent. `IconButton` already sizes to the minimum; the defect appears when a raw `GestureDetector`, a small `InkWell`, or `MaterialTapTargetSize.shrinkWrap` bypasses it. Adjacent targets also need separation - two 48dp targets touching each other still produce mistaps at the boundary.

Assert it rather than eyeballing it:

```dart
testWidgets('meets tap target guidelines', (tester) async {
  await tester.pumpWidget(const App());
  await expectLater(tester, meetsGuideline(androidTapTargetGuideline));
  await expectLater(tester, meetsGuideline(iOSTapTargetGuideline));
});
```

### Text scaling

*Tiers: All.*

```dart
// Bad - at 200% font scale the text is clipped inside a box that cannot grow
SizedBox(height: 48, child: Text(l10n.orderTotal))

// Good - the box follows the text
ConstrainedBox(
  constraints: const BoxConstraints(minHeight: 48),
  child: Text(l10n.orderTotal),
)
```

A fixed height is a promise about text size that the user is entitled to break. The same defect appears as a fixed-height `Container`, a `Row` of chips with a hardcoded height, a bottom bar sized in pixels, or an `AspectRatio` wrapping text. Use `minHeight`, intrinsic sizing, or let the content scroll.

Read the current scale through `MediaQuery.textScalerOf(context)` and scale non-text elements that must track it (an icon beside a label, a badge around a number) with `TextScaler.scale`. Do not multiply font sizes yourself - Flutter already applies the scale to `TextStyle.fontSize`.

Clamping is occasionally unavoidable for a control whose geometry genuinely cannot flex:

```dart
// Acceptable only at the narrowest scope, never around MaterialApp
MediaQuery.withClampedTextScaling(maxScaleFactor: 1.5, child: _fixedGeometryChip())
```

Clamping the whole app silently discards the user's system setting and is a defect, not a policy.

### Contrast

*Tiers: All.*

```dart
// Bad - grey on white measures roughly 2.3:1
Text(l10n.caption, style: const TextStyle(color: Color(0xFF9E9E9E)))

// Good - the theme's onSurfaceVariant is contrast-checked once, centrally
Text(l10n.caption, style: Theme.of(context).textTheme.bodySmall)
```

Contrast is a property of the colour pair, so it belongs in the theme where the pair is defined, not at each call site. Per-widget colour literals are how a design system silently loses contrast.

Check both `Brightness.light` and `Brightness.dark`, and check disabled and placeholder states - those are where palettes usually fall below 4.5:1. Non-text elements are not exempt: an icon that conveys status, a focus ring, and an input border need 3:1 against their background.

```dart
await expectLater(tester, meetsGuideline(textContrastGuideline));
await expectLater(tester, meetsGuideline(labeledTapTargetGuideline));
```

`meetsGuideline` catches the systematic cases in CI; a gradient or image background still needs a manual check because the effective background varies per pixel.

### Focus order and keyboard traversal

*Tiers: Desktop and Web required; Mobile whenever a hardware keyboard or switch-access device is attached.*

```dart
// Bad - the visual order is title, field, submit; the tree order sends focus to submit first
Stack(children: [_submitButton(), _titleField(), _bodyField()])

// Good - group the subtree and state the order explicitly
FocusTraversalGroup(
  policy: OrderedTraversalPolicy(),
  child: Column(children: [
    FocusTraversalOrder(order: const NumericFocusOrder(1), child: _titleField()),
    FocusTraversalOrder(order: const NumericFocusOrder(2), child: _bodyField()),
    FocusTraversalOrder(order: const NumericFocusOrder(3), child: _submitButton()),
  ]),
)
```

Default traversal follows the widget tree and the geometry Flutter infers from it, which diverges from reading order whenever a `Stack`, an absolute position, or a reordered layout is involved. `FocusTraversalGroup` also stops focus from escaping a dialog or a bottom sheet into the page behind it.

Verify by tabbing the screen end to end: focus must be visible at every stop, must never land on a non-interactive element, and must never leave a modal.

### Announcing dynamic state

*Tiers: All.*

```dart
// Bad - the item is removed and the screen reader says nothing
setState(() => _items.remove(item));

// Good
setState(() => _items.remove(item));
SemanticsService.announce(l10n.itemRemoved(item.name), TextDirection.ltr);
```

For a region whose content updates in place (a validation message, a live count, a status line), mark it instead of announcing manually:

```dart
Semantics(liveRegion: true, child: Text(_errorText))
```

`liveRegion` re-announces the node when its content changes; `SemanticsService.announce` is for one-off events with no persistent node. Pass the direction from `Directionality.of(context)` when the app supports RTL. Announce meaningful transitions only - loading finished, save succeeded, validation failed - not every rebuild.

### Gesture-only actions need a semantic alternative

*Tiers: All.*

A swipe, drag, or long-press is invisible to a screen-reader user - swipe gestures are captured for navigation. Every gesture-only action gets an equivalent on the node:

```dart
// Swipe-to-delete row: expose the action to assistive tech
Semantics(
  customSemanticsActions: {CustomSemanticsAction(label: l10n.deleteOrder): _delete},
  child: Dismissible(...),
)
```

Widgets like `Dismissible` announce nothing about the hidden action by themselves. The same applies to drag-to-reorder (expose move up/down actions) and long-press menus (expose the menu as an action or a visible affordance).

## Output Format

When invoked from an implementation workflow, emit the accessibility plan:

```
| Element | Semantics | Target | Contrast pair | Focus order | Announces |
|---------|-----------|--------|---------------|-------------|-----------|
| Delete icon button | tooltip: l10n.deleteOrder | 48dp (IconButton) | onSurface / surface | 4 | itemRemoved |
| Order row | MergeSemantics, hint: opensDetail | 56dp | onSurface / surface | 2 | - |
| Status dot | ExcludeSemantics (label on row) | n/a | 3:1 vs surface | - | - |
```

Record text-scaling decisions (minHeight vs fixed, clamp scope) and gesture-alternative actions in the Semantics column of the affected row.

When invoked from a review workflow, emit one block per finding:

```
### [Blocker | High | Medium | Low] file:line

- Area: Accessibility
- Check: {Semantic-Label | Screen-Reader-Output | Image-Alternative | Touch-Target | Text-Scaling | Contrast | Focus-Order | State-Announcement | Gesture-Alternative}
- Tier: {Mobile | Desktop | Web | All}
- Code: {one-line citation}
- Impact: {what a user relying on the affected assistive path cannot do}
- Fix: {concrete edit}
```

Close with one coverage line: `Checks clean: {comma-separated Check values with zero findings | none}`.

**Severity calibration.** `Blocker` = an action is impossible via an assistive path (unlabelled control on a required flow, content clipped at the supported text scale, focus trap, keyboard-unreachable action on a keyboard tier). `High` = the action is possible but the announcement is wrong or misleading, or contrast is below AA on body text. `Medium` = extra friction (unmerged rows, decorative nodes in traversal, colour-only meaning with a redundant cue nearby). `Low` = wording of a label or hint.

**Label mapping for the umbrella review:** `Blocker`, `High` -> `[Must]`; `Medium`, `Low` -> `[Recommend]`.

Contrast findings that cannot be computed from source (image or gradient backgrounds, runtime-derived colours) are reported as `Check: Contrast` with `Impact: unverifiable from source - manual check required` rather than being dropped or guessed.

## Avoid

- `IconButton` with no `tooltip`, and any `GestureDetector` acting as a button with no `Semantics`
- Labels that name the icon, restate the role ("Delete button"), or are hardcoded English literals
- Decorative icons and images left in the semantics tree; meaningful ones left without `semanticLabel`
- Tap targets under 48x48 dp / 44x44 pt, and `MaterialTapTargetSize.shrinkWrap` used to tighten a layout
- Fixed `height`, `width`, or `AspectRatio` on a box containing text
- `MediaQuery.withClampedTextScaling` above a screen, and manual multiplication of `fontSize` by the text scale
- Colour literals at widget call sites instead of contrast-checked theme roles
- Contrast checked in light mode only, or with disabled and placeholder states skipped
- Colour as the sole indicator of state, error, or selection
- Focus order left to tree order inside a `Stack` or a reordered layout; a dialog with no `FocusTraversalGroup`
- Silent success and silent removal - announce the outcome
- `liveRegion: true` on a node that changes every frame
- Treating a `meetsGuideline` pass as full coverage - it does not test screen reader phrasing or focus order
