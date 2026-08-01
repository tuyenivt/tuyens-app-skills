---
name: flutter-performance
description: "Diagnose Flutter jank: frame budget, rebuild scoping, RepaintBoundary, list virtualization, image cache, isolates, startup, app size, leaks."
metadata:
  category: mobile
  tags: [flutter, dart, performance, jank, frame-budget, rebuild, images, isolates, app-size, memory-leak]
user-invocable: false
---

# Flutter Performance

> Confirm the project's target platforms first - they decide which of the app-size and rendering guidance applies. Widget composition and lifecycle mechanics live in `flutter-widget-patterns`; provider scope and disposal in `flutter-riverpod-patterns`. This skill owns **cost**: what a frame, a cold start, an installed byte, and a retained object cost the user.

## When to Use

- Investigating dropped frames, scroll stutter, or slow screen transitions
- Reviewing a diff for rebuild cost, list construction, image loading, or UI-isolate blocking
- Cutting cold-start time or installed app size
- Chasing memory that grows across repeated navigation

## Rules

- Measure in **profile mode on a physical device**. Debug-mode timings are meaningless (no AOT, assertions on, extra checks), and emulator GPU behaviour is not the user's
- Two threads share each frame: **UI** (build, layout, paint) and **raster** (GPU). Roughly 16ms at 60Hz, 8ms at 120Hz, per thread. Attribute the overrun to a thread before changing anything
- `build()` is a hot path - it can run every frame. No sorting, filtering, parsing, regex construction, or object-graph building inside it
- Rebuild the smallest subtree that changed; keep state as close to the widget that reads it as possible
- Any list or grid that can exceed one screen uses a `builder` constructor
- Images decode at display resolution, not source resolution. Decoded bytes, not file bytes, are what memory holds
- CPU-bound work beyond a few milliseconds moves off the UI isolate
- Every controller, subscription, ticker, timer, and listener has a matching release on the same lifecycle
- A fix without a before/after number is a guess. Report the number or report that you have none

## Patterns

### Frame budget and thread attribution

| Symptom | Likely thread | Typical cause |
| --- | --- | --- |
| Stutter scrolling a data list | UI | expensive per-item build or layout |
| Stutter on a blur / shadow / gradient screen | Raster | `saveLayer` from `Opacity`, `ShaderMask`, `BackdropFilter`, complex clips |
| Freeze on a tap, then a jump | UI (blocked) | sync JSON, crypto, or image work on the UI isolate |
| First-run-only hitch on an animation | Raster | shader compilation (engine-dependent, see below) |
| Fine at 60Hz, janky at 120Hz | either | budget halves to ~8ms; the same work no longer fits |

A frame is dropped when **either** thread misses the budget. The DevTools frame chart shows both bars per frame - fix the one that is over, not the one you assumed.

**Rendering engine caveat.** Which engine renders (Impeller or Skia) depends on platform and Flutter version, and it changes the first-run shader-jank picture: Impeller precompiles, and Skia-era warm-up workarounds do not transfer to it. Confirm what the project actually runs before prescribing an engine-specific fix. Report the symptom (a first-run-only raster spike on one animation) rather than a number tied to an engine version.

### `const` and rebuild scoping

```dart
// Bad - setState rebuilds the whole subtree, static header included
return Column(children: [ExpensiveHeader(), Text('$_count')]);

// Good - the const instance is identical across rebuilds, so the framework skips it
return Column(children: [const ExpensiveHeader(), Text('$_count')]);
```

Element update short-circuits when the new widget is *identical* to the old one. `const` gives that identity for free; a `const` constructor that cannot be used is a missed skip on every rebuild.

```dart
// Bad - one counter change rebuilds the chart too
Widget build(_) => Column(children: [Chart(data), Text('$count')]);

// Good - the child is built once and hoisted out of the rebuild closure
ValueListenableBuilder<int>(
  valueListenable: counter,
  child: Chart(data),
  builder: (_, v, child) => Column(children: [child!, Text('$v')]),
)
```

The `child` parameter exists on `AnimatedBuilder` and the `*Builder` family for exactly this - it is the cheapest available rebuild scoping.

```dart
// Bad - subscribes to every MediaQuery change; opening the keyboard rebuilds this widget
final w = MediaQuery.of(context).size.width;

// Good - rebuilds only when size changes
final w = MediaQuery.sizeOf(context).width;
```

```dart
// Bad - re-sorts on every frame of any animation in this subtree
Widget build(_) => ListView(children: (items..sort(byName)).map(tile).toList());

// Good - sort when the data changes, cache the result, build from it
```

### `RepaintBoundary`: when it helps and when it hurts

| Situation | Boundary? | Why |
| --- | --- | --- |
| Small animating widget over an expensive static background | Yes, around the animating part | confines the repaint to its own layer |
| Expensive static subtree beside a frequently repainting sibling | Yes, around the static part | stops it repainting for a neighbour's sake |
| Item inside a `ListView` / `GridView` builder | Already present | the child delegates add one per item by default |
| Whole-screen wrapper "just in case" | No | a layer with nothing isolated behind it |
| Widget that repaints on the same frames as its parent | No | extra layer, zero avoided work |

Each boundary is a composited layer with its own memory and compositing cost, so over-application moves cost from repaint to composite rather than removing it. Confirm with repaint highlighting in DevTools before and after - this is not a decision to reason your way to.

### List virtualization

```dart
// Bad - builds and lays out 5000 rows before the first frame
ListView(children: items.map(RowTile.new).toList())

// Good - builds roughly a viewport at a time
ListView.builder(itemCount: items.length, itemBuilder: (_, i) => RowTile(items[i]))
```

- `itemExtent` (or `prototypeItem`) when every row is the same height: scroll geometry is then computable without laying out children, which makes long jumps and scrollbar drags cheap. Omit both when heights genuinely vary - a wrong extent clips or overlaps content
- `shrinkWrap: true` defeats virtualization by laying out every child to measure itself. A list inside another scrollable wants `CustomScrollView` plus slivers instead
- Item builders are multiplied by every item crossing the viewport per second; keep them cheap and `const`-heavy
- Automatic keep-alives trade memory for scroll-back state. Apply per item that needs it, not list-wide

### Image decode and cache

```dart
// Bad - a 4000x3000 photo decodes to ~48 MB of RGBA to fill a 120px slot
Image.network(url)

// Good - decode at the size actually painted (device pixels, not logical)
Image.network(url, cacheWidth: 240)
```

- Decoded size is `width * height * 4` bytes and is independent of JPEG/PNG compression: a 300 KB file can be a 48 MB decode
- The in-memory cache is `PaintingBinding.instance.imageCache`, bounded by `maximumSize` (entry count) and `maximumSizeBytes` (decoded bytes). Defaults are tuned for a general app; on an image-heavy screen the fix is decoding smaller and evicting under memory pressure, not raising the ceiling
- That cache is per-process and dies with the app. Network images that should survive a restart need a disk-caching image package
- `precacheImage` removes a first-paint pop for an image you know is next; precaching a whole list undoes the virtualization win

### Moving work off the UI isolate

```dart
// Bad - a 2 MB parse on the UI isolate drops every frame it spans
final data = jsonDecode(body) as List;

// Good - runs on a worker isolate; the UI keeps rendering
final data = await Isolate.run(() => jsonDecode(body) as List);
```

- `Isolate.run` for one-shot work; `compute` is the Flutter-side equivalent and needs a top-level or static callback. Repeated work is cheaper on one long-lived spawned isolate with ports than on a fresh spawn per call
- Arguments and results are **copied** across the boundary. Heavy compute on a small payload wins; trivial compute on a huge payload loses
- Isolates have no widgets and no `BuildContext`. Plugin and platform-channel use from a background isolate needs explicit setup - check the plugin supports it before designing around it
- `async`/`await` does **not** move work off the UI isolate. Two synchronous CPU chunks with an `await` between them both still run there

### Startup to first meaningful frame

- Before `runApp`, do only what the first frame needs. Analytics, remote config, and non-critical plugin init move to a post-frame callback or a background path
- No synchronous file, database, or network read on the launch path - a blocking read before `runApp` is a blank screen the user watches
- The first route renders its own loading state instead of the whole app waiting behind one future
- Measure with `--trace-startup` in profile mode and compare runs. Track two separate numbers: time to first frame, and time to a screen with real content. A spinner is not a meaningful frame

### Installed app size

| Lever | Applies to | Effect |
| --- | --- | --- |
| App bundle instead of a universal APK | Android via Play | store delivers per-device code; a universal APK ships every ABI to everyone |
| `--split-per-abi` | Android, direct APK distribution | one APK per architecture |
| `--obfuscate --split-debug-info=<dir>` | Android, iOS | symbols out of the binary; keep the directory to symbolize crashes |
| Asset audit | all | unused images, oversized source resolutions, duplicate densities, fonts with unneeded weights |
| Icon-font tree shaking | all | on by default in release; a non-`const` `IconData` disables it |
| Deferred loading (`deferred as` + `loadLibrary()`) | web | the initial bundle carries only the entry path |

Measure with the size analysis from `flutter build --analyze-size` and read it in the DevTools app-size tool. Guessing which asset is heavy is usually wrong. Android deferred components are a separate Play delivery mechanism, not the web pattern above.

### Leaks

```dart
// Bad - controller and subscription outlive the State
late final _c = AnimationController(vsync: this, duration: d);
late final _sub = stream.listen(_onData);

// Good
@override
void dispose() { _c.dispose(); _sub.cancel(); super.dispose(); }
```

| Created | Released by |
| --- | --- |
| `AnimationController`, `TabController`, `PageController`, `ScrollController`, `TextEditingController`, `FocusNode` | `dispose()` |
| `StreamSubscription` | `cancel()` |
| `Ticker` created directly | `dispose()` on the ticker |
| `addListener` on any `Listenable` / `ValueNotifier` | matching `removeListener` |
| `Timer` / `Timer.periodic` | `cancel()` |
| Platform-channel or native listener registration | explicit unregister |

The leak is rarely unfreed memory - it is a live object graph rooted in a still-registered callback, which also retains the dead widget's captured state and its data. Symptom: memory steps up with each navigation and never returns. Confirm by repeating one navigation cycle several times in the DevTools memory view and watching the retained count for that class, not by reading the code alone. Provider-held subscriptions belong to `flutter-riverpod-patterns`.

## Output Format

When invoked from a review workflow or directly for a jank/memory investigation, emit one block per finding:

```
### [Severity] file:line

- Category: {Rebuild | Raster | ListBuild | ImageDecode | IsolateBlock | Startup | AppSize | Leak}
- Thread: {UI | Raster | Both | N/A}
- Code: {one-line citation}
- Cost: {with units - "48 MB decode per tile", "sorts 5k items per frame", "~2 MB parse on the UI isolate"}
- Evidence: {measured (profile mode, device) | estimated (no profile)}
- Fix: {concrete Dart or Flutter change}
- Verify: {what to re-measure - frame chart, --analyze-size delta, retained count}
```

`Severity: {Critical | High | Medium | Low}` - Critical = sustained dropped frames or unbounded growth on a primary path. High = a measurable regression on a common path. Medium = cost on a rare path or only at unlikely data sizes. Low = a cheap win with no observed symptom.

**No profile data:** emit `Evidence: estimated (no profile)` and cap severity at High. Never report a number you did not measure.

If there are no findings, emit exactly `No performance findings.` so the workflow knows the check ran.

When invoked from an implementation workflow, emit a budget table:

```
| Surface | Budget | Risk | Mitigation |
|---------|--------|------|------------|
| Feed list | 16ms UI / 60Hz | 5k items, network images | builder + itemExtent, cacheWidth 240 |
| Detail hero | 8ms raster / 120Hz | blur backdrop | drop BackdropFilter, precomposed asset |
```

Budget defaults to 16ms per thread (60Hz); use 8ms for surfaces the product ships to high-refresh devices (ProMotion, 120Hz Android) - take the target from the project's stated device tier, not aspiration.

## Avoid

- Timing anything in debug mode, or on an emulator, and calling it a measurement
- "Optimizing" without naming which thread was over budget
- Work in `build()`: sorting, filtering, parsing, regex construction, network calls
- `MediaQuery.of(context)` where `sizeOf` / `paddingOf` / `viewInsetsOf` would scope the subscription
- `RepaintBoundary` sprinkled without a before/after repaint measurement
- `ListView(children: ...)` for a list that can exceed a screen; `shrinkWrap: true` to escape an unbounded-height error
- Full-resolution decodes into thumbnails, or raising the image-cache ceiling instead of decoding smaller
- Blocking work before `runApp`, or gating the first route behind a future that could be a loading state
- `await` treated as a way to get work off the UI isolate
- Copying a large payload into an isolate to do trivial work on it
- A `State` that creates a controller, subscription, timer, or listener with no `dispose`
- Shipping a universal APK where an app bundle is available
- Asserting shader-jank or engine behaviour without confirming which renderer the project uses
