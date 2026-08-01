---
name: task-flutter-review-perf
description: Flutter / Dart perf review - jank and frame budget, rebuild scoping, list virtualization, image decode, isolates, startup, app size, leaks.
agent: flutter-performance-engineer
metadata:
  category: mobile
  tags: [flutter, dart, performance, jank, rebuild, isolate, app-size, memory-leak, workflow]
  type: workflow
user-invocable: true
---

> **Behavioral directive:** Load `Use skill: behavioral-principles` before executing this workflow.

# Flutter Performance Review

Client-side perf review naming the Flutter idiom: work inside `build`, unscoped rebuilds, non-lazy list constructors, full-resolution image decode, UI-isolate blocking, startup work before first frame, installed size growth, and undisposed controllers or subscriptions. Every finding states user-visible impact and labels its evidence as measured or estimated.

Performance review for Flutter projects.

## When to Use

- Flutter perf regression review of a change set
- Jank, dropped frames, or stutter on a specific screen
- Slow cold start or slow time to first meaningful frame
- Memory that climbs across a session or never returns after a route pops
- Installed app size growth
- Pre-release perf pass on scroll paths, image-heavy screens, or a new isolate

**Not for:** general review (`task-flutter-review`), security (`task-flutter-review-security`), pre-implementation design (`task-flutter-implement`).

Perceived slowness that is actually a missing loading state or an unhandled offline path is not a perf finding.

## Depth

| Depth | When | Runs |
|-------|------|------|
| `standard` | Default | All steps |
| `deep` | Profiling data supplied (DevTools timeline, startup trace, size analysis), or a perf-critical release | All + a `### Device & Measurement Plan` subsection under Recommendations |

## Measurement Discipline

- **Profile mode on a physical device is the only valid timing source.** Debug mode runs unoptimized with asserts enabled; its numbers overstate cost and are not evidence. Simulator and emulator timings are not device timings.
- **State the frame budget the target device actually runs at** - roughly 16ms at 60Hz, 8ms at 120Hz.
- **Separate UI-thread jank from raster-thread jank.** They have different fixes: UI thread points at build and layout cost, raster points at painting, clipping, opacity layers, and shader compilation.
- **Impact, not adjective.** "Rebuilds the full 200-row list on every keystroke" is a finding. "This is slow" is not.

## Generated Code

Generated files are build output, not review surface. Exclude from findings: `*.g.dart`, `*.freezed.dart`, `*.gr.dart`, `*.config.dart`, `*.mocks.dart`, and generated localization output. When a generated file carries the cost, review the source that produces it and cite that source's `file:line`.

## Invocation

| Form | Meaning |
|------|---------|
| `/task-flutter-review-perf` | Review the working tree (unstaged + staged + untracked) |
| `/task-flutter-review-perf --staged` | Review staged changes only |

When the working tree is clean, `review-precondition-check` falls back to `last-commit` mode on its own.

**Never modify the working tree.** Read with `git diff` only; uncommitted work is the review subject.

When invoked as a subagent (e.g. by `task-flutter-review`), the parent supplies the precondition handle, the pre-read diff, the depth level, the detected project shape, and the generated-file exclusion list: Step 3 is skipped, no git is re-run, and Step 12 returns findings instead of writing - the parent owns the report.

## Workflow

### Step 1 - Behavioral Principles

Use skill: `behavioral-principles`. Accept the parent's confirmation if invoked as a subagent.

### Step 2 - Stack and Project Shape

Accept the project shape from the parent when invoked as a subagent. Otherwise read `pubspec.yaml`; if it is absent or declares no `flutter` dependency, stop - this workflow reviews Flutter projects only.

Record: state management (Riverpod / Bloc / Provider / GetX / none), navigation, networking client, persistence store, image loading approach, and the platform targets present.

If state management is not Riverpod, record it and note in the Summary: `Detected <X>; Riverpod-specific guidance does not apply.` Review rebuild scoping against that library's own selector or listener mechanism.

### Step 3 - Resolve the Change Set

Use skill: `review-precondition-check`. Forward `--staged` if passed. **Skip entirely** when the parent supplied the handle plus the pre-read diff. Surface any fail-fast verbatim.

The handle supplies `mode`, `base`, `current_branch`, `reviewable`, `counts`, and `notes`. Review only the paths in `reviewable`; `counts.binary` and `counts.generated` are excluded already.

Read the content once and reuse: `git diff <base>` for the change body, `git diff --name-status <base>` for the per-file status. Untracked files in `reviewable` do not appear in `git diff <base>`; read those files directly and treat their full contents as added lines.

### Step 4 - Read the Performance Surface

Cite real `file:line`. Open:

- Every changed widget with a `build` method, plus the `State` classes around them
- Every changed state holder (notifier, bloc, controller) and its provider or dependency wiring
- Scrollable constructors and their `itemBuilder` bodies
- Image call sites and any caching layer around them
- `main()`, `runApp`, bootstrap, and anything initialized before the first frame
- `pubspec.yaml` for added dependencies and assets
- Isolate entry points (`compute`, `Isolate.run`, `Isolate.spawn`) and what crosses the port

If the diff is small but ripples into unchanged code - a new screen mounting an existing widget that rebuilds the world, or a new call site into an existing unbounded cache - read the unchanged file. The regression lives there.

### Step 5 - Build-Path Cost and Rebuild Scoping

Use skill: `flutter-performance` for the pattern bank. Use skill: `flutter-widget-patterns` for `const`, keys, and lifecycle. Use skill: `flutter-riverpod-patterns` for provider scope and rebuild blast radius.

- [ ] **No work in `build`:** no I/O, parsing, sorting, `RegExp` compilation, or list construction. `build` runs whenever an ancestor rebuilds, not when the data changes
- [ ] **`const` on invariant subtrees** - a `const` widget is canonicalized and its subtree is skipped rather than rebuilt
- [ ] **State pushed down:** `setState` on a route-level `State` rebuilds the route. Hold state at the smallest widget that needs it
- [ ] **Selector-scoped watches:** `ref.watch(provider.select((s) => s.field))` over watching the whole object; `ref.read` in callbacks, never `ref.watch`
- [ ] **Builder scope:** a `Consumer`, `BlocBuilder`, or `ValueListenableBuilder` wrapping the whole screen rebuilds the whole screen. Wrap only the reactive part
- [ ] **`MediaQuery.of(context)` in `build`** subscribes to every `MediaQuery` change - keyboard, orientation, text scale, padding. When only one property is read, use the property-scoped accessor (`MediaQuery.sizeOf(context)` and siblings)
- [ ] **`child:` on `AnimatedBuilder` / `AnimatedWidget`** so the invariant subtree is built once instead of every tick
- [ ] **`RepaintBoundary` around an independently-animating subtree** so its repaints do not propagate to ancestors
- [ ] **Layer-forcing effects:** the `Opacity`, `ClipPath`, and `BackdropFilter` widgets force an offscreen layer. Prefer alpha applied directly to a color or image, a `borderRadius` on the decoration, and `FadeTransition` / `AnimatedOpacity` which animate at the render-object layer without rebuilding the subtree

**Bad / good.**

```dart
// Bad - sorts the whole list on every frame an ancestor rebuilds
Widget build(BuildContext context) {
  final sorted = items.toList()..sort((a, b) => a.name.compareTo(b.name));
  return Column(children: sorted.map(Row.new).toList());
}

// Good - sorting belongs to the state holder; build only renders
Widget build(BuildContext context) {
  final sorted = ref.watch(sortedItemsProvider);
  return Column(children: sorted.map(Row.new).toList());
}
```

### Step 6 - List and Scroll Virtualization

- [ ] **Lazy constructors for dynamic or unbounded collections:** `ListView.builder` / `.separated`, `GridView.builder`, or `SliverChildBuilderDelegate`. The default `ListView(children: [...])` constructor builds every child up front, on the frame the list appears
- [ ] **`itemExtent` or `prototypeItem`** when rows are uniform, so the viewport does not measure children to lay out the scroll extent
- [ ] **No `shrinkWrap: true` + `NeverScrollableScrollPhysics` nested lists** - that lays out every child regardless of visibility. Compose with slivers in one `CustomScrollView` instead
- [ ] **Bounded per-item work:** no date formatting, `RegExp` compile, sort, or `firstWhere` scan over another collection inside `itemBuilder`
- [ ] **Windowed or paginated loading** for server-backed lists; an in-memory list that only grows is a leak with a scroll bar
- [ ] **Stable keys** (`ValueKey(item.id)`) so element and `State` are reused across reorder rather than rebuilt
- [ ] **`addAutomaticKeepAlives` / `AutomaticKeepAliveClientMixin` used deliberately** - keeping offscreen items alive defeats virtualization's memory benefit
- [ ] List children already get repaint boundaries by default (`addRepaintBoundaries`); adding another per row is noise, not a fix

### Step 7 - Images, Assets, and Decode Cost

- [ ] **Decode resolution bounded** via `cacheWidth` / `cacheHeight` or `ResizeImage`, sized to the box the image renders into. A decoded bitmap costs roughly `width * height * 4` bytes regardless of display size - a 4000x3000 photo in a 96px avatar is about 48 MB of pixels held for one thumbnail
- [ ] **Remote images go through a caching layer** rather than re-fetching per rebuild
- [ ] **`ImageCache` limits are a decision, not a default** - raising `maximumSize` / `maximumSizeBytes` trades memory for fewer decodes and needs a stated reason
- [ ] **No re-decode per build:** `Image.memory` fed a freshly computed `Uint8List` inside `build` decodes again every frame. Hoist the bytes, or cache the decoded image
- [ ] **`precacheImage`** for images that would otherwise pop in on first paint
- [ ] **Resolution variants shipped** (1x / 2x / 3x) rather than one oversized asset scaled at runtime

### Step 8 - Off-Thread Work and Isolates

Use skill: `dart-language-patterns` for isolate and async mechanics.

- [ ] **CPU-bound work moved off the UI isolate** via `compute` or `Isolate.run`: large JSON parse, crypto, image transform, big sort or filter
- [ ] **`async` does not move work off the isolate.** A synchronous loop blocks the UI whether or not the enclosing function is `async` - only an isolate or a genuine `await` on I/O yields
- [ ] **Threshold reasoning:** spawning an isolate and copying its message is not free. Recommend one when the work is tens of milliseconds, not microseconds
- [ ] **Large binary payloads** transferred with `TransferableTypedData` rather than copied
- [ ] **Repeated background work uses a long-lived isolate with ports**, not one spawn per item
- [ ] **Platform-channel calls are already asynchronous** and the native work does not run on the Dart isolate - do not wrap one in `compute`. A background isolate that needs channels requires its binary messenger to be initialized explicitly for that isolate

### Step 9 - Startup Time and App Size

- [ ] **`main()` before `runApp` does the minimum:** no blocking network call, no full database open and migrate, no large parse. Defer past the first frame or behind an explicit splash state
- [ ] **Initialization is lazy** - a provider or service never read should never be constructed
- [ ] **Startup measured from a profile-mode startup trace** (`flutter run --profile --trace-startup`), not from stopwatch feel
- [ ] **Size measured from a build-size analysis** (`flutter build <target> --analyze-size`, reviewed in the DevTools app-size tool). Report the delta this change introduces, not the absolute total
- [ ] **New dependency justified against its size contribution** - a package pulled in for one helper is a size finding
- [ ] **Asset hygiene:** uncompressed images, unused fonts, whole icon sets, and bundled test fixtures shipping to production
- [ ] **Android ships an app bundle or split-per-ABI builds** rather than one universal APK
- [ ] **Icon and font tree-shaking left enabled in release** - a non-constant `IconData` defeats it and pulls the whole icon font into the binary

### Step 10 - Memory, Disposal, and Leaks

- [ ] **Everything created in a `State` is released in `dispose`:** `AnimationController`, `Ticker`, `TextEditingController`, `ScrollController`, `PageController`, `TabController`, `FocusNode`, `StreamSubscription`, `StreamController`, `Timer`, and any platform-channel or `EventChannel` subscription
- [ ] **Every `addListener` has a matching `removeListener`** - a listener closure captures its `State` and pins the element tree it belongs to
- [ ] **Riverpod:** screen-scoped providers are `autoDispose` or explicitly invalidated; `ref.onDispose` closes whatever the provider opened; a `keepAlive` is a deliberate cache with a stated eviction path
- [ ] **Global caches, static maps, and singletons are bounded and evicted** - unbounded growth reads as a slow leak
- [ ] **Large retained objects released when the route pops** (decoded images, full result sets, buffered streams)
- [ ] **Leak suspicion is confirmed, not asserted:** navigate away, force a GC, and check the retaining path in the DevTools memory view before filing a leak as measured

**Impact heuristic.** A leaked controller survives navigation, so the symptom is never the first leak - it is the tenth. Phrase impact as accumulation across a session ("about 40 MB retained per visit to this screen, never released"), ending in an OS kill the user experiences as the app closing itself.

### Step 11 - Evidence and Impact

Label every finding's evidence. Never present an estimate as a measurement, and never cite a debug-mode number at all.

| Evidence | Use when | Example phrasing |
|----------|----------|------------------|
| `measured` | A profile-mode trace, startup trace, or size analysis was supplied | `raster thread 24ms/frame on the feed scroll, Pixel 6, profile mode` |
| `estimated` | The pattern is unambiguous but no trace exists | `rebuilds all 200 rows on every keystroke` |
| `unverified` | Cost depends on data only the author has (collection size, image dimensions, device tier) | Raise as `[Recommend]` and name the measurement to run |

`flutter-performance` emits only `measured` and `estimated`; `unverified` is this workflow's third bucket for findings that stay `[Recommend]` pending measurement. With no profile data supplied, cap the finding at High Impact - the atomic applies the same cap.

**Severity mapping.** `flutter-performance` grades findings `Critical | High | Medium | Low`; this report groups by impact. Map `Critical` and `High` to **High Impact**, `Medium` to Medium, `Low` to Low. A `Critical`-origin finding leads the High Impact section and keeps the atomic's rationale (sustained dropped frames or unbounded growth on a primary path) in its impact line - do not silently flatten it into an ordinary High.

Confirm that a hot path introduced here is observable at all; if it is not, raise Low.

### Step 12 - Write Report

Standalone only. Subagent runs return findings in the Output Format to the parent, which writes the single merged report.

Write the assembled report to `review-perf-<branch>.md` in the current working directory, overwriting the file if it already exists.

Derive `<branch>` from the handle's `current_branch`, sanitized for a filename: replace `/` and any character outside `[A-Za-z0-9_-]` with `-`, collapse consecutive `-` into one, strip leading and trailing `-`.

The file is this YAML frontmatter followed by the report body (raw Markdown, unfenced):

```yaml
---
branch: <branch>
scope_mode: working-tree | staged-only | last-commit
files: <N>
scope: core-only | +perf | +sec | full
depth: standard | deep
generated_at: <ISO 8601 UTC timestamp>
---
```

Field sources: `branch` = the handle's `current_branch` (unsanitized), `scope_mode` = the handle's `mode`, `files` = the handle's `counts.reviewable`, `scope` = `+perf`, `depth` = the depth resolved from the Depth table, `generated_at` = the current UTC time in ISO 8601.

After writing, print exactly one confirmation line:

```
Report written to review-perf-<branch>.md (<N> files, scope: <scope>)
```

## Output Format

The fence below delimits the template for display only - it is not part of the report. Emit the report body as raw Markdown so headings, tables, and lists render; never wrap the whole report in a code fence.

```markdown
## Flutter Performance Review Summary

**Stack Detected:** Flutter <version> / Dart <version>
**State Management:** Riverpod | Bloc | Provider | GetX | none
**Platform Targets:** <list>
**Measurement Basis:** profile-mode trace on <device> | estimated from code (no trace supplied)
**Scope:** Client (Flutter)
**Overall:** Clean | Issues Found - [count by impact]

## Findings

### High Impact

- **Location:** [file:line]
- **Issue:** [name the Flutter idiom: sort inside `build`, non-lazy `ListView` over a dynamic collection, full-resolution decode into a thumbnail, sync parse on the UI isolate, undisposed `AnimationController`]
- **User-Visible Impact:** [what the user experiences: "dropped frames while typing", "adds ~1.2s to cold start", "+3.1 MB installed size", "about 40 MB retained per visit, never released"]
- **Evidence:** measured (<source>) | estimated | unverified
- **Thread:** UI | raster | startup | memory | size
- **Fix:** [concrete Dart change with code]

### Medium Impact / Low Impact

[Same structure]

_Omit empty sections._

## Recommendations

[Structural improvements not tied to a single finding]

### Device & Measurement Plan _(deep depth only)_

[Which device tiers to measure on, which trace to capture, and the number that decides whether the fix worked]

## Next Steps

Each tagged `[Implement]` or `[Delegate]`. Order: Must > Recommend.
Impact maps to intent: High -> [Must]; Medium / Low -> [Recommend].

1. **[Implement]** [Must] file:line - [one-line action]
2. **[Delegate]** [Recommend] [scope: server contract] - [one-line action]

_Omit if no actionable findings._
```

## Self-Check

- [ ] `behavioral-principles` loaded (or accepted from parent)
- [ ] Stack confirmed; state management, image approach, and platform targets recorded; non-Riverpod state management surfaced rather than flagged
- [ ] `review-precondition-check` ran (or parent-supplied handle and diff reused); diff read once against the handle's `base` and reused; untracked reviewable files read directly
- [ ] Performance surface read directly (widgets and `State`, state holders, scrollables, image call sites, bootstrap, `pubspec.yaml`, isolate entry points)
- [ ] `flutter-performance`, `flutter-widget-patterns`, `flutter-riverpod-patterns` consulted; work in `build`, `const`, rebuild scope, `MediaQuery` subscription, layer-forcing effects checked
- [ ] List virtualization checked: lazy constructor, extent hint, no `shrinkWrap` nesting, bounded `itemBuilder`, stable keys, pagination
- [ ] Image decode bounds, caching layer, `ImageCache` limits, and per-build re-decode checked when the diff touches images
- [ ] `dart-language-patterns` consulted; UI-isolate blocking, isolate threshold, and payload transfer assessed when the diff adds computation
- [ ] Startup path and size delta assessed when the diff touches bootstrap, dependencies, or assets
- [ ] Disposal completeness audited for every controller, subscription, timer, ticker, and listener introduced; provider disposal scope checked
- [ ] Every finding labelled `measured` / `estimated` / `unverified`; no debug-mode timing cited as evidence
- [ ] Every finding states user-visible impact, not an adjective
- [ ] Generated files excluded from findings; the producing source cited instead
- [ ] Findings ordered by impact; quick wins separated from structural changes
- [ ] Depth honored: `standard` ran all steps; `deep` added the Device & Measurement Plan
- [ ] Next Steps produced with `[Implement]` / `[Delegate]` tags, ordered Must > Recommend
- [ ] Report written to `review-perf-<branch>.md` with the sanitized branch name and complete frontmatter (`branch`, `scope_mode`, `files`, `scope`, `depth`, `generated_at`), confirmation line printed (standalone only; subagent runs return findings to the parent)

## Avoid

- State-changing git from this workflow (fetch/checkout/merge/pull/rebase/stash) - uncommitted work is the review subject and must not be disturbed
- Writing a report when invoked as a subagent - the parent owns it
- Citing a debug-mode timing, an emulator timing, or a hot-reload observation as evidence
- Reporting cost without user-visible impact ("this rebuilds a lot" vs "rebuilds all 200 rows on every keystroke, dropping frames while typing")
- Generic frontend advice where a Flutter idiom exists ("scope the rebuild with `select`", not "reduce re-renders")
- Raising findings against `*.g.dart`, `*.freezed.dart`, `*.gr.dart`, `*.config.dart`, `*.mocks.dart`, or generated localization output
- Prescribing `const` on a widget whose subtree actually varies
- Recommending an isolate for work too small to pay back its spawn and copy cost
- Recommending `RepaintBoundary` per list row where the framework already inserts one
- Treating `async` as a way to move CPU work off the UI isolate
- Blaming the client for latency that belongs to the server - route it to the owning service
- Filing a missing loading state or unhandled offline path as a perf finding
- Optimizing without a measurement plan that would show the fix worked
- Emitting `[Question]`, `[Suggestion]`, `[Consider]`, `[Nit]`, `[Nitpick]`, or `[Praise]` labels - if it isn't `[Must]` or `[Recommend]`, don't write it down.
