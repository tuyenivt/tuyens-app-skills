---
name: unity-accessibility
description: Make Unity 2D games playable for all - colour-blind-safe signalling, touch targets, contrast, text scaling, reduced motion, remappable input.
metadata:
  category: mobile
  tags: [unity, accessibility, colour-blind, contrast, touch-target, reduced-motion, screen-reader]
user-invocable: false
---

# Unity Accessibility

> This skill owns **whether a player can perceive, reach, and act on the game**. UXML/USS structure and the screen stack belong to `unity-ui-patterns`; palette and sprite authoring belong to `unity-2d-rendering`; translated strings and text expansion belong to `unity-i18n`; input action maps belong to `unity-2d-physics-input`.

## When to Use

- Any UI screen, HUD, or board where state is communicated visually
- Gameplay distinguishes pieces, gems, tiles, or cells by colour
- Adding animation, screen shake, flashing, or timed pressure
- Reviewing a diff that adds an interactive element or a new signal
- Deciding what accessibility can actually be claimed for a store listing

## Rules

- **Colour is never the only carrier of information.** Every colour-coded distinction carries a redundant shape, symbol, pattern, label, or position. In a Match-3 or colour-matching game this is a correctness requirement, not a courtesy - roughly 8% of male players cannot reliably separate the hues a default gem palette uses, and for them the game is unplayable rather than merely uncomfortable
- Interactive touch targets are at least **48dp x 48dp** with at least 8dp between adjacent targets, regardless of the visual element's size. A 24px icon gets a 48dp hit area
- Body and UI text meets **4.5:1** contrast against its background; large text (>= 18pt, or >= 14pt bold) and meaningful icons meet **3:1**. Contrast is measured against the actual background, including gameplay art behind a transparent panel
- Text scales without clipping or overlap. Layouts reflow; they do not truncate the sentence
- Every animation, shake, flash, and particle burst has a reduced-motion path that removes or dampens it while preserving the information the motion carried
- Every input binding is remappable, and no interaction requires multi-touch, a drag held for a duration, or a gesture with no tap equivalent
- **Do not claim screen-reader support that has not been verified on device.** Unity's accessibility surface is limited compared to native app frameworks; state what was tested, on which platform, at which engine version

## Patterns

### Colour-blind-safe signalling

The three common types constrain the palette differently: deuteranopia and protanopia (red/green, the large majority) and tritanopia (blue/yellow, rare). Red-versus-green is the worst possible pairing; blue-versus-orange is the most robust.

```csharp
// Bad - the gem type IS the colour; two gems are identical to a deuteranope
gem.color = gemType switch { Red => Color.red, Green => Color.green, ... };

// Good - colour plus an intrinsic shape; readable in greyscale
gem.sprite = gemSprites[gemType];   // circle, diamond, star, hexagon, teardrop
gem.color  = gemPalette[gemType];   // colour remains, as reinforcement
```

**The test: render the screen in greyscale.** If two game-relevant states become indistinguishable, the design is broken and a palette swap will not fix it. Shape or symbol redundancy is the fix; a "colour-blind palette" toggle alone is a partial mitigation that still fails at high gem counts, because five to seven simultaneously distinguishable hues do not exist for a deuteranope.

Beyond the board, the same rule applies to every signal: correct/incorrect answer feedback, valid/invalid move highlights, enemy team identity, health and cooldown states, and required-versus-optional form fields.

| Signal | Colour-only (bad) | Redundant (good) |
| --- | --- | --- |
| Quiz answer result | Green / red fill | Check / cross icon + fill + result text |
| Legal move | Green cell tint | Tinted cell + dot marker + outline |
| Chess piece side | Light / dark tint | Distinct silhouette or side marker in addition to tint |
| Low health | Bar turns red | Bar turns red + pulses + numeric value |

Where a palette toggle is offered, it changes the palette *in addition to* the shape redundancy, and it applies to gameplay art and UI together - a toggle that recolours the HUD but not the board is worse than none.

### Touch targets and spacing

```csharp
// Bad - the hit area is the 24px glyph
<ui:Button class="icon-close" style="width: 24px; height: 24px;" />

// Good - visual glyph inside a compliant target
<ui:Button class="icon-close" />   /* USS: min-width: 48px; min-height: 48px; */
```

Express minimums in USS as `min-width`/`min-height` so they survive scaling. In a panel using Scale With Screen Size, verify the resulting physical size at the smallest supported screen - a target that is 48 reference units may be well under 48dp after scaling (`unity-ui-patterns`).

**Reference units are not dp.** The rules above are in dp (a physical size); UXML and USS are authored in reference px against the panel's reference resolution. They coincide only when the panel renders 1:1. Under Scale With Screen Size the scale factor is roughly `actual screen width / reference width`, so a 48px element on a 1080-wide reference shrinks below 48dp on any narrower device. Convert before judging a target compliant, and when the reference resolution is not stated in the diff, say which reference you assumed rather than reporting a bare pass or fail.

Board cells are exempt from the 48dp minimum only where the board's own geometry sets the size (a 9x9 Sudoku grid on a phone) - but then the input handling must tolerate imprecision: snap to the nearest cell, and confirm destructive actions rather than committing on the first ambiguous tap.

Adjacent targets need spacing. Two 48dp buttons flush against each other still produce mistaps at the seam.

### Contrast

Measure against what is actually behind the text. The frequent failure is a HUD label that passes on the design mock's flat background and fails over the bright gameplay art it ships against. Fixes, in order of robustness: an opaque or heavily tinted backing plate, then a text outline or shadow, then a colour change. A pure alpha reduction on the backing plate reintroduces the problem on the brightest levels.

Disabled states are commonly authored at very low opacity and fall below 3:1. A disabled control still has to be readable, or the player cannot tell what is unavailable versus what is absent.

### Text scaling and reflow

```css
/* Bad - fixed height clips the second line the moment text grows */
.answer-btn { height: 56px; overflow: hidden; }

/* Good - grows with content, floor preserved */
.answer-btn { min-height: 56px; white-space: normal; }
```

Support at least 200% text scale. A quiz game is text; a fixed-height answer button that clips at 130% makes questions ungradeable.

Where the platform exposes a system font-scale preference, honouring it is the correct default; where it does not, ship an in-game text-size setting. Interaction with translation is real: German and Russian run roughly 30% longer than English before any scaling is applied, so the worst case is longest-locale plus maximum scale, and that is the case to verify (`unity-i18n`).

### Screen readers - what is actually available

Be precise here rather than optimistic:

- Unity's runtime accessibility exposure to platform screen readers is provided by the **Accessibility module** (`com.unity.modules.accessibility`, a built-in engine module enabled by default, not an external package) via the `UnityEngine.Accessibility` `AccessibilityHierarchy` / `AssistiveSupport` API. At 6000.3 it covers TalkBack, VoiceOver, and - new in 6.3 - Narrator on Windows and VoiceOver on macOS. It remains narrower than native UIKit or Android View accessibility, and it does not make an arbitrary rendered game board readable
- **UI Toolkit runtime elements are not automatically exposed to platform screen readers** the way native controls are. The hierarchy is a separate node tree that you build and keep in sync with your visual elements in your own code; nothing is exposed that you did not add a node for. Verify on device before claiming it
- The dependable path for a casual 2D game is **in-game accessibility**: a self-voicing option using platform TTS for quiz prompts and answers, high-contrast and large-text modes, and full keyboard/gamepad navigability with a visible focus indicator. These are under the game's control and testable
- Focus order and a visible focus ring are worth getting right regardless: they serve switch access, keyboard, and gamepad players, and they are pure UI Toolkit work

Statements about screen-reader behaviour must name the Unity version and the platform they were verified on - the module ships with the engine, so the editor version is what pins its surface. An unverified claim in a store listing is a compliance problem, not just a docs problem.

### Reduced motion

```csharp
// Bad - unconditional shake; a vestibular trigger with no opt-out
StartCoroutine(ShakeCamera(0.4f, 30f));

// Good - the signal survives; the motion does not
if (!Settings.ReduceMotion) StartCoroutine(ShakeCamera(0.4f, 30f));
else FlashBorderOnce();
```

Reduced motion removes the motion, not the feedback. A match that was communicated only by a particle burst still needs a communicated result - a static highlight, a score tick, a sound.

Cover screen shake, parallax, full-screen particle bursts, rapid flashing, spinning or looping backgrounds, and large transition animations. Anything flashing faster than roughly 3 Hz across a large area is a seizure risk and should not be present at all, not merely gated behind a toggle.

Where a UI transition is skipped, the end state must still be reached - a screen that only becomes visible via a transition disappears entirely under reduced motion.

### Remappable input and timing

Every action rebindable, with a reset-to-default. The Input System supports interactive rebinding and persisted overrides; expose it rather than hardcoding a scheme (`unity-2d-physics-input`).

No interaction may *require* a gesture: every drag, swipe, long-press, and pinch has a tap or button equivalent. A Match-3 swap works by tapping two adjacent gems as well as by dragging.

Timing accommodations, which for puzzle games are the accessibility feature that actually changes who can play:

| Pressure source | Accommodation |
| --- | --- |
| Countdown timer | Extend, or a no-timer / relaxed mode |
| Auto-advancing question | Advance on input, not on a timer |
| Cascade or resolution animation | Skippable, and never input-blocking beyond its duration |
| Toast or feedback that vanishes | Persist until dismissed, or long enough to read at slow reading speed |
| Double-tap or hold-to-confirm | Tap plus explicit confirm as an alternative |

Never gate progression or story on reaction speed in a casual puzzle game unless a relaxed mode reaches the same content.

## Output Format

Two modes, chosen by whether the request supplies code to judge or asks for code to be produced.

**Authoring mode** - the request is to write or design something. Emit the code or design, then any `Deferred:` lines. No finding blocks, no severity, no status line: nothing was reviewed, so a not-run line would misdescribe the work.

**Review mode** - source, a diff, or a symptom report was supplied. Emit one block per finding.

```
### [Severity] {file:line | symbol or type.member, when source was supplied without paths | asset path | symptom, when no source was supplied}

- Category: {ColourOnly | TouchTarget | Contrast | TextScaling | ScreenReader | ReducedMotion | InputRemapping | Timing | FocusOrder}
- Evidence: {source | inferred (state what was not seen)}
- Affects: {who is blocked - "deuteranopia, ~8% of male players", "200% text scale", "switch access"}
- Code: {one-line citation, or `not supplied` when the finding is inferred}
- Impact: {what the player cannot do - "cannot distinguish red and green gems, game unplayable"}
- Fix: {concrete change}
```

`Severity: {Critical | High | Medium | Low}` - Critical = a player group cannot play or cannot complete a core loop (colour-only board signalling, flashing above ~3 Hz, an action with no non-gesture path, progression gated on reaction speed with no relaxed mode). High = a core interaction is unreliable or unreadable for a known group (sub-48dp target on a primary control, body text below 4.5:1, text clipped at 200% scale, unconditional screen shake). Medium = a secondary or cosmetic surface with the same defect, or a missing remap for a non-essential action. Low = focus-order or labelling nit with a working alternative path.

Severity that does not fit a listed band: assign the nearest lower band and state why in `Impact`. `Category` takes exactly one value - where a defect fits two, pick the one the `Fix` addresses and name the other in `Impact`; where it fits none, pick the closest and name the real concern in `Impact`.

`Evidence: inferred` is required whenever the source was not read. It bounds the header at High: a Critical-band defect is written High, and `Impact` names the uncapped band. It never raises a block - a Medium defect stays Medium. Among blocks sharing a band, order by what the reader must fix first: root cause before the symptoms it produces. `Evidence` records whether the source was read; `Verified` below records whether the behaviour was exercised on a device.

Screen-reader findings state the Unity version and the platform the behaviour was verified against; where it was not verified on device, write `Verified: not tested` in `Impact` rather than asserting the behaviour.

A defect owned by a sibling named in the ownership blockquote is not emitted as a finding. Write those after the findings, one per line, as `Deferred: {defect} -> {owning skill}`, so the workflow routes rather than drops them. Omit entirely when there are none.

In review mode, close with exactly one status line, after any `Deferred:` lines:

| Condition | Line |
| --- | --- |
| One or more findings emitted | none - the findings are the output |
| No findings, and a symptom or report was available to reason from | `No accessibility findings.` |
| No source, diff, symptom, or report of any kind was supplied | `Accessibility check not run: no source supplied.` |

A symptom-only report (a QA ticket, a verbal description) is checkable input: emit `Evidence: inferred` findings from it rather than the not-run line.

## Avoid

- Hue as the sole distinguishing feature of a game piece, tile, cell, or answer state
- A colour-blind palette toggle offered as a substitute for shape or symbol redundancy
- A palette toggle that recolours the HUD but not the gameplay board
- Hit areas sized to the visual glyph
- Contrast measured against a design mock rather than the shipped background
- Fixed-height text containers with `overflow: hidden`
- Screen-reader support claimed without on-device verification, or without naming the Unity version and platform tested
- Screen shake, flashing, or particle bursts with no reduced-motion path
- Reduced motion that removes the feedback along with the motion
- Flashing faster than ~3 Hz across a large area, toggle or not
- A gesture with no tap or button equivalent
- Countdown pressure with no extended or relaxed mode
