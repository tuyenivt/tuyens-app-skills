---
name: unity-ui-patterns
description: Build Unity 2D game UI with UI Toolkit - UXML/USS structure, query caching, PanelSettings scaling, screen stack, safe area, aspect-ratio adaptivity.
metadata:
  category: mobile
  tags: [unity, ui-toolkit, uxml, uss, visualelement, panelsettings, safe-area, adaptivity]
user-invocable: false
---

# Unity UI Patterns

> This skill owns **UI Toolkit structure, layout, and navigation**. Repaint and layout cost budgets belong to `unity-performance`; sprites, atlases, and sorting belong to `unity-2d-rendering`; contrast, touch-target minimums, and screen readers belong to `unity-accessibility`; user-facing strings belong to `unity-i18n`.

## When to Use

- Building or reviewing any runtime UI screen, HUD, popup, or menu
- Deciding UXML/USS structure and where styling lives
- UI breaks on a different aspect ratio, notch, or orientation
- Navigation, back handling, or modal stacking is ad hoc

## Rules

- **UI Toolkit only.** uGUI (`Canvas`, `RectTransform`, `UnityEngine.UI`) is out of scope for this plugin. A project on uGUI is reported as out of scope, not reviewed against these rules and not rewritten
- Structure in UXML, styling in USS, behaviour in C#. Inline styles set from C# for anything a class could express are unreviewable and untraceable
- **Cache every `Q`/`Query` result at bind time.** A query is a tree walk; running one per frame or per event is the single most common UI Toolkit cost trap
- One `UIDocument` per logical screen or overlay, with a shared `PanelSettings` asset per resolution strategy. Do not scatter panel settings per document
- Layout adapts by rule, not by pixel constants: flexbox growth, percentage widths, and `min-width`/`max-width` breakpoints. Hardcoded pixel positions break on the next device
- Safe area is applied to a single root container per panel, sourced from `Screen.safeArea`, and re-applied on resolution or orientation change
- Never assume 16:9. Portrait phone, tall phone, tablet, and desktop windows must all be handled by the same USS, verified at the extremes
- UI Toolkit's runtime API surface moves between package versions. State the package version when citing a runtime binding, transition, or input API rather than asserting it flatly

## Patterns

### Query caching

```csharp
// Bad - a tree walk every frame
void Update() { root.Q<Label>("score").text = score.ToString(); }

// Good - resolved once, updated on change
Label _score;
void OnEnable() { _score = GetComponent<UIDocument>().rootVisualElement.Q<Label>("score"); }
public void SetScore(int v) => _score.text = v.ToString();
```

Resolve in `OnEnable` and not `Awake`: `rootVisualElement` is not reliably populated until the document is enabled. Re-resolve after any `visualTreeAsset` swap, since the old references point at a discarded tree.

For lists, `ListView` with `makeItem`/`bindItem` recycles elements. Building one element per row of a 500-question bank is both an allocation spike and a layout stall. Cache each row's own `Q` results once in `makeItem` (stash them on `userData`) rather than re-querying in `bindItem`, which runs on every scroll recycle.

Set `virtualizationMethod` deliberately: `FixedHeight` is cheaper and correct only when every row is genuinely the same height; a row holding localized text is not, so variable content needs `DynamicHeight`. Guessing `FixedHeight` clips the first long translation.

### Text that grows

Labels change size with locale, and with the player's system font scale. A box authored around one English string clips the first German or Japanese one that arrives.

```css
/* Bad - fixed box, so a longer translation clips instead of growing */
.stat-label { width: 220px; height: 44px; }

/* Good - content drives the box, bounded rather than fixed */
.stat-label { max-width: 320px; min-width: 0; white-space: normal; }
```

Three rules carry most of it: never set a fixed `width`/`height` on a text-bearing element, use `min-width`/`max-width` instead; put `min-width: 0` on flex text children so they can actually shrink and wrap rather than overflowing their row; and set `flex-shrink: 0` on the value side of a label/value pair so a long label never squeezes the number out. Raise the whole type ramp through one class on a root when the system font scale is large, so the screen scales coherently instead of one label outgrowing its row. Unity exposes no cross-platform system font scale - the value comes from the platform layer (Android `Configuration.fontScale`, iOS `preferredContentSizeCategory`) or from an in-game text-size setting, so treat it as supplied and default it to 1. Which strings need this, and their font assets and fallback chains, are `unity-i18n`.

### UXML structure and USS styling

```xml
<!-- Bad - identity, layout, and look all inline -->
<ui:Button name="play" style="width: 320px; height: 88px; background-color: #2A8;" />

<!-- Good - name for lookup, class for style -->
<ui:Button name="play" class="btn btn--primary" text="Play" />
```

Name attributes are for `Q` lookup and tests. Classes carry style. A USS class edited once restyles every button; an inline style must be hunted per file.

Keep the tree shallow. Each nesting level costs a layout pass, and a deeply wrapped hierarchy makes flexbox growth behaviour hard to predict.

### PanelSettings and scale mode

`PanelSettings` decides how the panel maps to the screen. The choice is the whole resolution-independence strategy:

| Scale mode | Behaviour | Use for |
| --- | --- | --- |
| Constant Pixel Size | 1 UI unit = 1 screen pixel | Desktop tools; wrong for mobile - UI shrinks on high-DPI |
| Scale With Screen Size | Scales against a reference resolution | Default for mobile 2D games |
| Constant Physical Size | Scales against DPI to a physical target | Touch-target-critical UI where physical size matters |

With Scale With Screen Size, the reference resolution plus the match-width-or-height factor determines what happens off-ratio. Match on width for portrait games so vertical extra space becomes headroom rather than shrinking everything; match on height for landscape. A single reference resolution with no breakpoints still produces a stretched tablet layout - scaling is not adaptivity.

### Screen and popup stack

```csharp
// Bad - screens hide each other by direct reference; back button guesses
menuDoc.SetActive(false); gameDoc.SetActive(true);

// Good - one owner, explicit stack, back pops
public void Push(ScreenId id);   // hides current, shows id
public bool Pop();               // returns false when the stack is empty -> quit prompt
```

One navigator owns the stack. Screens do not know each other. Android back and desktop Escape both route into `Pop`, and a modal that captures input must be the top of the stack rather than an element that happens to render last.

Hiding a screen: `display: none` removes it from layout and skips its layout/paint work; `visibility: hidden` and zero opacity keep it in layout and keep costing. Use `display` for screens, opacity only for transitions.

### Safe area and notch

```csharp
// Bad - a magic top pad that is wrong on every other device
root.style.paddingTop = 48;

// Good - derived from the runtime safe area, in panel coordinates
var sa = Screen.safeArea;
root.style.paddingTop = Screen.height - sa.yMax;
root.style.paddingBottom = sa.yMin;
```

`Screen.safeArea` is in screen pixels; if the panel scales, convert before applying, or apply the padding to a root that the panel scales as a whole. Re-apply on orientation change and on desktop window resize - safe area is not constant for the process lifetime. `GeometryChangedEvent` on the root is what detects both; polling `Screen.orientation` from `Update` is the wrong shape. The same event drives the root class swap for a layout restructure. Gesture bars at the bottom matter as much as the notch at the top: a Play button under the home indicator is unreachable.

### Aspect-ratio adaptivity

| Form factor | Typical ratio | Layout rule |
| --- | --- | --- |
| Phone portrait | 9:19.5 to 9:16 | Single column; board centred; controls anchored to the bottom safe area |
| Phone landscape | 19.5:9 | Board centred; HUD moves to the side gutters, not above/below |
| Tablet | 4:3 to 3:2 | Cap board `max-width`; let gutters grow rather than the board |
| Desktop window | arbitrary, resizable | Same as tablet, plus a `min-width` floor and a resize re-layout |

Drive these with USS `max-width`/`min-width` and flex growth on the gutters, so one stylesheet covers the range. Where a genuine restructure is needed (HUD above versus beside the board), swap a root class on an orientation change rather than maintaining two UXML trees.

For a fixed-aspect board (Sudoku, Chess, 2048), constrain the board container by the smaller dimension and let surrounding space absorb the difference. Stretching a square board to fill an arbitrary ratio is a correctness bug, not a cosmetic one.

### Events and pointer input

```csharp
// Bad - polling a click flag from Update
void Update() { if (_btn.HasFocus() && Input.GetMouseButtonDown(0)) Play(); }

// Good - registered callback, unregistered with the screen
_btn.RegisterCallback<ClickEvent>(OnPlay);
void OnDisable() => _btn.UnregisterCallback<ClickEvent>(OnPlay);
```

Unregister on disable. Callbacks captured against a screen that is pushed and popped repeatedly leak both delegates and their captured state.

Events propagate through the tree (trickle down, bubble up). Use `evt.StopPropagation()` on a modal backdrop so a tap does not reach the gameplay elements behind it. For drag interactions (Match-3 swaps, tile drags), use `PointerDownEvent`/`PointerMoveEvent`/`PointerUpEvent` with pointer capture rather than reconstructing drags from mouse polling - pointer events carry touch identity, mouse events do not.

Gameplay input that is not UI stays in the Input System (`unity-2d-physics-input`). Routing board taps through UI elements couples the board to the panel's hit-testing and scale mode.

### Transitions and juice

USS transitions animate style properties on the UI thread without per-frame C#:

```css
.btn { transition: scale 120ms ease-out; }
.btn:hover, .btn:active { scale: 1.06; }
```

Animate `translate`, `scale`, `rotate`, and `opacity`. USS has no `transform` shorthand - the transform is split into the four standalone properties `translate`, `scale`, `rotate`, and `transform-origin`, each animated by name. Animating `width`, `height`, `margin`, or `padding` forces a layout pass on every frame of the transition; on a full screen that is a measurable stall.

Every transition needs a reduced-motion path - see `unity-accessibility`.

## Output Format

Two modes, chosen by whether the request supplies code to judge or asks for code to be produced.

**Authoring mode** - the request is to write or design something. Emit the code or design, then any `Deferred:` lines. No finding blocks, no severity, no status line: nothing was reviewed, so a not-run line would misdescribe the work.

**Review mode** - source, a diff, or a symptom report was supplied. Emit one block per finding.

```
### [Severity] {file:line | symbol or type.member, when source was supplied without paths | asset path | symptom, when no source was supplied}

- Category: {Structure | QueryCost | Scaling | Navigation | SafeArea | Adaptivity | Events | Transitions | OutOfScopeUI}
- Evidence: {source | inferred (state what was not seen)}
- Code: {one-line citation, or `not supplied` when the finding is inferred}
- Impact: {what breaks and where - "board stretches on 4:3 tablet", "query runs 60x/sec"}
- Fix: {concrete change}
```

`Severity: {Critical | High | Medium | Low}` - Critical = UI unreachable or unusable on a supported form factor (control under the notch or gesture bar, board clipped off-screen). High = per-frame query or layout-animating transition on a hot screen, or a navigation stack that cannot return to a previous screen. Medium = inline styling, uncached query on a cold path, or a hardcoded pixel constant with a working fallback. Low = structural nit with no current cost.

Severity that does not fit a listed band: assign the nearest lower band and state why in `Impact`. `Category` takes exactly one value - where a defect fits two, pick the one the `Fix` addresses and name the other in `Impact`; where it fits none, pick the closest and name the real concern in `Impact`.

`Evidence: inferred` is required whenever the source was not read. It bounds the header at High: a Critical-band defect is written High, and `Impact` names the uncapped band. It never raises a block - a Medium defect stays Medium. Among blocks sharing a band, order by what the reader must fix first: root cause before the symptoms it produces.

A defect owned by a sibling named in the ownership blockquote is not emitted as a finding. Write those after the findings, one per line, as `Deferred: {defect} -> {owning skill}`, so the workflow routes rather than drops them. Omit entirely when there are none.

In review mode, close with exactly one status line, after any `Deferred:` lines:

| Condition | Line |
| --- | --- |
| The project uses uGUI | `uGUI project - UI Toolkit review out of scope.` and no findings |
| One or more findings emitted | none - the findings are the output |
| No findings, and a symptom or report was available to reason from | `No UI findings.` |
| No source, diff, symptom, or report of any kind was supplied | `UI check not run: no source supplied.` |

A symptom-only report (a QA ticket, a verbal description) is checkable input: emit `Evidence: inferred` findings from it rather than the not-run line.

## Avoid

- `Q`/`Query` called from `Update`, `OnGUI`, or an event handler that fires per frame
- Inline `style` assignments from C# for anything a USS class expresses
- Hardcoded pixel positions, sizes, or safe-area pads
- A single reference resolution treated as adaptivity
- Screens toggling each other directly with no navigation owner
- Callbacks registered without a matching unregister
- Transitions animating `width`, `height`, `margin`, or `padding`
- Board or gameplay input routed through UI elements
- Version-sensitive UI Toolkit runtime API cited without naming the package version
