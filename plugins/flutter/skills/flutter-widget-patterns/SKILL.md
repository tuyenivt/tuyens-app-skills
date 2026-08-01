---
name: flutter-widget-patterns
description: "Apply Flutter widget patterns: Stateless/Stateful, const constructors, Keys, BuildContext safety, lifecycle, constraints, rebuild scoping."
metadata:
  category: mobile
  tags: [flutter, dart, widgets, const, keys, buildcontext, lifecycle, layout, rebuilds]
user-invocable: false
---

# Flutter Widget Patterns

> Language-level Dart shape decisions live in `dart-language-patterns`. This skill owns the widget tree: what a widget is, when it rebuilds, and how it is laid out. Frame-budget measurement and profiling belong to the performance lens.

## When to Use

- Building or reviewing screens, widgets, and layout code
- Deciding between `StatelessWidget` and `StatefulWidget`, whether a `Key` is required, or where a rebuild boundary belongs
- Diagnosing state that jumps to the wrong list item, disappears on rebuild, or survives when it should not
- Diagnosing layout exceptions (unbounded constraints, overflow) and "works on my device only" sizing

## Rules

- `StatelessWidget` unless the widget owns mutable state or a lifecycle-bound resource (controller, subscription, ticker). State handed down from above is a constructor parameter, not a `State`
- Every constructor that can be `const` is declared `const`, and call sites actually write `const` - an omitted `const` at the call site discards the whole benefit
- Extract subtrees into widget classes, never `Widget _buildX()` methods: a method's output has no element of its own, can never be `const`, and always rebuilds with its parent
- Children whose *identity* can change position (reorder, insert, remove) need a `Key` when anything below them holds state - their own `State`, a descendant's, or element-held state like scroll offset or text selection. Place it on the root of the subtree whose state must survive, not on an inner node
- `GlobalKey` only for cross-subtree access or moving a subtree while preserving its state; hold it in a field, never construct one in `build`
- Never touch a `BuildContext` after an `await` without re-checking `mounted` first, and never store a `BuildContext` in a field or hand it to an object that outlives the frame
- Inherited lookups (`Theme.of`, `MediaQuery.of`, and equivalents) belong in `build` or `didChangeDependencies` - never `initState` or `dispose`
- Everything `initState` creates, `dispose` releases; everything derived from `widget.*` is re-derived in `didUpdateWidget` when its source field changed
- Compose by wrapping. Never subclass a concrete framework widget to change its behavior
- Size from the incoming constraints, not from the screen: a widget must render correctly under any constraints a parent may impose
- Push rebuilds down - put the changing part behind a builder and pass the unchanged subtree through the builder's `child` parameter

## Patterns

### Stateless vs Stateful

```dart
// Bad - State holds nothing mutable; all ceremony, no benefit
class Badge extends StatefulWidget { ... }
class _BadgeState extends State<Badge> {
  @override Widget build(BuildContext context) => Text('${widget.count}');
}

// Good
class Badge extends StatelessWidget {
  const Badge({required this.count, super.key});
  final int count;
  @override Widget build(BuildContext context) => Text('$count');
}
```

The test is ownership, not mutation: if the value changes but is *owned* by a parent or a provider, the widget stays stateless and simply receives the new value. `StatefulWidget` earns its place when the widget itself owns a controller, subscription, ticker, or focus node.

### `const` and rebuild cost

```dart
// Bad - a new instance every build; the framework must diff this subtree each time
child: Padding(padding: EdgeInsets.all(16), child: Text('Total')),

// Good - canonicalized: the identical instance arrives next build
child: const Padding(padding: EdgeInsets.all(16), child: Text('Total')),
```

Identical `const` expressions are canonicalized to one instance. When a rebuild hands the element the *same* widget instance it already has, the framework short-circuits and skips that subtree entirely. That is the mechanism - `const` does not stop the parent from rebuilding, it stops the const subtree from being recreated and re-diffed.

`const` is viral: a widget is const-constructible only if every argument is a constant, so a single non-const argument deep in a tree blocks it all the way up. Turn on `prefer_const_constructors` in `analysis_options.yaml` rather than auditing by eye.

### Keys

| Key | Use when |
|-----|----------|
| none | children are positionally stable, or carry no state |
| `ValueKey(id)` | identity is a value you already hold (an item id) |
| `ObjectKey(item)` | identity is the object instance itself, with no stable id field |
| `UniqueKey()` | you deliberately want a fresh element and discarded state |
| `GlobalKey` | you need the `State`/context from outside the subtree, or must move the subtree elsewhere in the tree with its state intact |

```dart
// Bad - reorder the list and state stays with the slot, not the item
ListView(children: [for (final t in todos) TodoTile(todo: t)]);

// Good - identity travels with the item
ListView(children: [for (final t in todos) TodoTile(key: ValueKey(t.id), todo: t)]);
```

Without a key, the framework matches new widgets to existing elements by position and runtime type. That is correct for stable lists and wrong the moment items are reordered or removed: a checkbox, scroll offset, or text field keeps the old element's state and appears to jump to a neighboring row.

```dart
// Bad - a new GlobalKey each build; the subtree is remounted and state is lost
Widget build(BuildContext context) => Form(key: GlobalKey<FormState>(), child: ...);

// Good - created once, held by the State
final _formKey = GlobalKey<FormState>();
```

A `GlobalKey` must be unique across the whole tree and is more expensive than a local key. Reach for it only when nothing else can express the need.

### `BuildContext` across async gaps

```dart
// Bad - the widget may already be gone; this context is dead
Future<void> _save() async {
  await repo.save(draft);
  Navigator.of(context).pop();
}

// Good - re-check after every await that precedes a context use
Future<void> _save() async {
  await repo.save(draft);
  if (!mounted) return;
  Navigator.of(context).pop();
}
```

Better still, resolve what you need *before* the gap, so no context survives it:

```dart
final messenger = ScaffoldMessenger.of(context);
await repo.save(draft);
messenger.showSnackBar(const SnackBar(content: Text('Saved')));
```

Inside a `State`, check `mounted`; elsewhere, `context.mounted`. The `use_build_context_synchronously` lint flags these, so keep it enabled instead of relying on review to catch them.

A `BuildContext` is a handle on a position in the element tree, valid for as long as that element is mounted. Storing one in a field, a controller, or a singleton outlives that guarantee and leaks the element.

### Lifecycle

| Hook | Runs | Responsibility |
|------|------|----------------|
| `initState` | once, before first build | create controllers, subscriptions, tickers; no inherited lookups |
| `didChangeDependencies` | after `initState`, then whenever an inherited dependency changes | first inherited reads; react to theme/locale/media changes. An inherited lookup in `initState` usually returns a value rather than throwing - it just never registers the dependency, so the cached value goes stale silently |
| `didUpdateWidget(old)` | parent rebuilt with a new config of the same type and key | diff `old.x` against `widget.x` and re-wire whatever depended on it |
| `dispose` | once, on removal | cancel, close, and dispose everything `initState` created |

```dart
// Bad - the subscription stays bound to the first id forever
@override
void initState() {
  super.initState();
  _sub = repo.watch(widget.id).listen(_onData);
}

// Good - follow the config when it changes
@override
void didUpdateWidget(covariant ItemView oldWidget) {
  super.didUpdateWidget(oldWidget);
  if (oldWidget.id != widget.id) {
    _sub.cancel();
    _sub = repo.watch(widget.id).listen(_onData);
  }
}
```

A `StatefulWidget`'s `State` outlives the widget object. The parent rebuilding gives the same `State` a new `widget`, so anything captured from `widget.*` in `initState` silently goes stale unless `didUpdateWidget` refreshes it. Call `super` in every hook, and `super.dispose()` last.

`setState` after `dispose` throws; if a callback can arrive late, cancel its source in `dispose` rather than guarding on `mounted` at every call site.

### Composition over inheritance

```dart
// Bad - subclassing a concrete widget to bolt on behavior
class LoggingButton extends ElevatedButton { ... }

// Good - wrap it
class LoggingButton extends StatelessWidget {
  const LoggingButton({required this.onPressed, required this.child, super.key});
  final VoidCallback onPressed;
  final Widget child;

  @override
  Widget build(BuildContext context) => ElevatedButton(
        onPressed: () { log('tap'); onPressed(); },
        child: child,
      );
}
```

Framework widgets are configuration objects with private state contracts; subclassing binds you to internals that change between releases. Wrapping composes the same behavior with no coupling.

```dart
// Bad - not a widget: no element, no const, rebuilds with the entire parent
Widget _buildHeader() => Row(children: [...]);

// Good - its own element, its own rebuild boundary, const-able
class _Header extends StatelessWidget {
  const _Header();
  @override Widget build(BuildContext context) => const Row(children: [...]);
}
```

### Constraints and sizing

Constraints go down, sizes go up, the parent sets the position. A widget cannot know its size before layout; it chooses one within the constraints it was handed.

```dart
// Bad - sizes off the screen; breaks in split-screen, on desktop, and on rotation
width: MediaQuery.of(context).size.width * 0.8,

// Good - relative to whatever the parent actually granted
FractionallySizedBox(widthFactor: 0.8, child: ...)

// Good - when the number is needed explicitly
LayoutBuilder(
  builder: (context, constraints) => SizedBox(width: constraints.maxWidth * 0.8, child: ...),
)
```

```dart
// Bad - Column hands its child unbounded height; the ListView cannot lay out and throws
Column(children: [Header(), ListView(...)])

// Good - Expanded bounds it to the leftover space
Column(children: [Header(), Expanded(child: ListView(...))])
```

Unbounded-constraint errors nearly always mean a scrollable or flexible child sits inside a parent that imposes no limit on that axis. `Expanded`/`Flexible` fix it and must be direct children of a `Row`, `Column`, or `Flex`. `shrinkWrap: true` and `IntrinsicHeight`/`IntrinsicWidth` also silence the error but force extra layout work over the children - use them only when the bounded alternative genuinely does not fit.

Prefer `SizedBox` over `Container` when only sizing or spacing is needed: it is const-constructible and cheaper.

### Scoping rebuilds with builders

```dart
// Bad - one changing value rebuilds the whole screen
setState(() => _count++);   // in a State whose build() returns the entire page

// Good - only the Text rebuilds; the header is built once and passed through
ValueListenableBuilder<int>(
  valueListenable: _counter,
  child: const ExpensiveHeader(),
  builder: (context, value, child) => Column(children: [child!, Text('$value')]),
)
```

The `child` parameter is the point of these builders: it is built once outside the builder callback and handed back on every invocation, keeping an expensive subtree out of the rebuild. `AnimatedBuilder` takes the same `child` for the same reason, which matters most there because it rebuilds every frame.

```dart
// Bad - a new Future every rebuild; the builder restarts and the UI flickers
FutureBuilder(future: repo.load(widget.id), builder: ...)

// Good - created once, held on the State
late final Future<Item> _item = repo.load(widget.id);
FutureBuilder(future: _item, builder: (context, snapshot) => ...)
```

The rule generalizes: a builder rebuilds whenever its source changes, so the source must not be *created* during build. Handle every branch a snapshot can present - waiting, error, and data - rather than assuming data.

## Output Format

When invoked from an implementation workflow, emit the tree decisions:

```
| Widget | Type | Key | Rebuild Scope | Notes |
|--------|------|-----|---------------|-------|
| OrderScreen | Stateless | none | full | reads state from above |
| OrderList | Stateless | none | ListView.builder | items keyed by id |
| OrderTile | Stateful | ValueKey(order.id) | self | owns expansion controller |
| TotalBar | Stateless | none | ValueListenableBuilder | isolates the total |
| EmptyCartHint | Stateless (const) | none | full | no runtime inputs |
```

One row per widget where a decision was made; skip widgets with nothing to decide.

- `Type: {Stateless | Stateless (const) | Stateful}` - write `(const)` only when every call site can pass `const`
- `Rebuild Scope` names what limits this widget's own rebuild: `self` when it rebuilds alone, `full` when it rebuilds with its parent, or the name of the builder or scoped-list widget that bounds it

When invoked from a review workflow, emit one block per finding:

```
### [Must | Recommend] file:line

- Category: {Widget Type | Const Usage | Key Correctness | Context Safety | Lifecycle | Composition | Constraints | Rebuild Scope}
- Code: {one-line citation}
- Issue: {the rule broken and the user-visible symptom it causes}
- Recommendation: {concrete edit}
```

`[Must]` when the violation crashes, leaks, or loses user-visible state (dead context, missing dispose, missing key on stateful list items, unbounded constraints, stale `initState` capture). `[Recommend]` for cost and clarity (const usage, composition, rebuild scope, `Container` vs `SizedBox`).

When invoked directly to diagnose a symptom, emit the same finding block per diagnosed cause, with two changes: the heading is `### [Must | Recommend] <pattern section>` when no source file was supplied, and each block ends with `- Confidence: {Confirmed | Likely | Needs-Source}` - `Confirmed` when the cited code was read, `Likely` when the reported symptom matches one pattern uniquely, `Needs-Source` when more than one cause fits. Rank blocks by confidence, highest first.

For each category with zero findings, emit exactly `No <category> findings.` using the category name from the enum, so the workflow knows the check ran. Omit that line for categories that have findings, and omit the enumeration entirely when no source was read - emit `Diagnosed from symptom only; no category sweep performed.` instead.

Generated widget code (`*.g.dart`, `*.freezed.dart`, and other build_runner output) is out of scope - report against the source that generates it.

## Avoid

- `StatefulWidget` whose `State` has no fields - it is a `StatelessWidget`
- Omitting `const` at the call site of a const constructor
- `Widget _buildSection()` helper methods in place of widget classes
- Unkeyed stateful children in a reorderable or filterable list
- A `Key` on an inner node when the state to preserve lives above it
- `GlobalKey` constructed inside `build`, or used where a `ValueKey` would do
- `context` after `await` with no `mounted` check, and `BuildContext` stored in a field or passed to a repository, controller, or singleton
- `Theme.of(context)` / `MediaQuery.of(context)` in `initState`, and any inherited lookup in `dispose`
- `initState` reading `widget.*` into a field with no matching `didUpdateWidget`
- Controllers, subscriptions, tickers, and focus nodes created without a `dispose`
- `setState` inside `build`, inside `dispose`, or wrapping work that changes nothing observable
- Subclassing `ElevatedButton`, `Scaffold`, or any concrete framework widget to alter behavior
- `MediaQuery.of(context).size` for layout that a constraint could drive
- `shrinkWrap: true` or `IntrinsicHeight` used to silence an unbounded-constraint error that `Expanded` should fix
- `Container` where `SizedBox`, `Padding`, or `ColoredBox` says it more precisely and can be const
- A `Future` or `Stream` constructed inside `build` and fed to a builder
