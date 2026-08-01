---
name: dart-language-patterns
description: "Apply Dart 3 idioms: null safety, records, pattern matching, sealed classes, class modifiers, extension types, async/Stream, isolates."
metadata:
  category: mobile
  tags: [flutter, dart, null-safety, patterns, sealed-classes, records, async, isolates]
user-invocable: false
---

# Dart Language Patterns

> Confirm the project's target platforms first - they change the isolate guidance below. Widget-tree, rebuild, and lifecycle concerns live in `flutter-widget-patterns`. This skill owns language-level shape decisions.

## When to Use

- Writing or reviewing Dart that is not widget-tree code: models, repositories, mappers, controllers, pure logic
- Choosing between a record and a class, a sealed hierarchy and an enum, `late` and `?`, an extension type and a typedef
- Reviewing whether code reads as Dart 3 vs null-safety-retrofitted Dart 2 or Kotlin/TypeScript translated into Dart

## Rules

- Model absence with `?`, never a sentinel (`-1`, `''`, epoch). `!` only where a non-null invariant is locally proven; every other `!` is a scheduled crash
- `late` only for a value guaranteed assigned in an earlier callback (`initState`) before any read; prefer `late final`. Never `late` to silence the compiler on a genuinely optional value - that is `?`
- Sealed hierarchy + exhaustive `switch` for closed variant sets. No `default` or `_` arm on a sealed switch: it makes the switch trivially exhaustive and adding a variant stops being a compile error
- `switch` expressions and object patterns over if-else chains on type or shape; destructure and bind in one step
- Records for multi-value returns with no behavior; a class once the group has invariants, methods, or is part of a public API
- Class modifiers state the boundary contract explicitly (see the table below) - an unmodified `class` means "anyone may extend or implement this", which is rarely the intent
- `extension type` for zero-cost domain wrappers over primitives; it is compile-time only, so never depend on it for runtime type tests or polymorphism
- Every `StreamSubscription`, `StreamController`, and `Timer` has a matching `cancel`/`close` on the owner's teardown path
- CPU-bound work (large JSON decode, parsing, crypto, image processing) goes to `Isolate.run`/`compute`; `async` alone does not free the UI thread
- `rethrow`, never `throw e` - `throw e` discards the original stack trace. Catch with `on <Type>`; bare `catch` only at a boundary that reports the error
- Build collections declaratively with collection-`if`/`for` and spreads, not `add` sequences

## Patterns

### Null safety and promotion

```dart
// Bad - a public field does not promote; `user.email` is still String? inside the if
if (user.email != null) send(user.email!);

// Good - a local promotes, and the `!` disappears
final email = user.email;
if (email != null) send(email);
```

Only **private final** fields promote (Dart 3.2+). Public or mutable fields never do, because a getter could return a different value on the second read - so the `!` a reviewer sees is often a promotion failure, not a real invariant.

Prefer the null-aware operators over branching: `user.address?.city ?? 'unknown'`, `_cache ??= compute()`.

### `late` discipline

```dart
// Bad - `late` used to dodge "field must be initialized"; read order is now a runtime bet
late String _token;

// Good - genuinely optional
String? _token;

// Good - guaranteed assigned in initState, never reassigned
late final AnimationController _controller;
```

`late` converts a compile-time error into a `LateInitializationError` at runtime. That trade is only worth it when the assignment is guaranteed by a framework-ordered callback.

### Records vs classes

```dart
// Bad - a Map for a two-value return: no types, no checking
Map<String, Object?> parseRange(String s) => {'start': a, 'end': b};

// Good - a record: structural, immutable, value-equal
({DateTime start, DateTime end}) parseRange(String s) => (start: a, end: b);

final (:start, :end) = parseRange(input);
```

Records have value equality and no declaration cost, which makes them right for local plumbing and multi-value returns. Promote to a class (typically freezed) once the group needs a name in an API, an invariant, or a method.

Names on **positional** record fields are documentation only - `(int x, int y)` is still accessed as `$1` and `$2`. Use named fields when you want to read them by name.

### Sealed classes and exhaustive `switch`

```dart
sealed class LoadState {}
final class Loading extends LoadState { const Loading(); }
final class Loaded extends LoadState { const Loaded(this.items); final List<Item> items; }
final class Failed extends LoadState { const Failed(this.error); final Object error; }

final label = switch (state) {
  Loading()            => 'Loading...',
  Loaded(:final items) => '${items.length} items',
  Failed(:final error) => 'Failed: $error',
};
```

`sealed` is implicitly abstract and confines subtypes to the same library, which is what lets the compiler check exhaustiveness. Adding a `Refreshing` variant then breaks every `switch` that must handle it - the whole point. Guards refine an arm without breaking that: `Loaded(:final items) when items.isEmpty => 'Nothing here'`.

Use an `enum` (with fields and methods if needed) when variants carry no distinct data; use `sealed` when they do.

### Class modifiers

| Modifier | Outside the declaring library | Use for |
|----------|-------------------------------|---------|
| `final` | neither extend nor implement | leaf and value types; closes the hierarchy |
| `base` | extend, not implement | invariants that live in the base and must be inherited, not re-declared |
| `interface` | implement, not extend | pure contracts: repository ports, service boundaries |
| `sealed` | neither; implicitly abstract | closed variant sets driving exhaustive `switch` |

Test doubles are the common tension: `interface`/unmodified classes can be implemented by a fake, `final` cannot. Choose `final` for types you own end-to-end, `interface` for seams you expect callers to stub.

### Extension types

```dart
// Bad - every id is a String; swapping the arguments compiles fine
void link(String userId, String orderId) { ... }

// Good - distinct types at compile time, no wrapper object at runtime
extension type UserId(String value) {}
extension type OrderId(String value) {}
void link(UserId userId, OrderId orderId) { ... }
```

An extension type is erased to its representation type at runtime, so runtime type tests and dynamic dispatch cannot tell it apart from the underlying value - it is a static-checking tool, not a runtime one. Adding `implements String` makes the wrapper substitutable for the raw value, which reopens exactly the hole it was created to close; add it only deliberately.

### Async: start concurrently, await once

```dart
// Bad - independent calls serialized; latencies add
final user = await fetchUser();
final prefs = await fetchPrefs();

// Good - both in flight, then collected
final userFuture = fetchUser();
final prefsFuture = fetchPrefs();
final user = await userFuture;
final prefs = await prefsFuture;
```

A `Future` starts when it is created, not when it is awaited. Use `Future.wait([...])` instead when the results are homogeneous and one combined error surface is acceptable.

Fire-and-forget needs to be explicit - wrap it in `unawaited(...)` from `dart:async` so an intentional gap is distinguishable from a forgotten `await`, and attach error handling, because an unobserved failed future is silent.

### Streams

```dart
// Bad - subscription outlives its owner: leaks, and fires after teardown
repo.watch(id).listen(_onData);

// Good - own it, then cancel it
_sub = repo.watch(id).listen(_onData);   // created in initState
await _sub.cancel();                     // in dispose
```

A plain stream is single-subscription: a second `listen` throws. Use a broadcast controller or `asBroadcastStream()` only when there really are multiple listeners - and note broadcast streams drop events emitted while nobody is listening.

`async*` with `yield` is the readable way to produce a stream; it stops when the listener cancels, so no manual controller bookkeeping is needed.

### Isolates

```dart
// Bad - async does not yield the UI thread; a 40ms decode still drops frames
final data = await Future(() => jsonDecode(hugeJson));

// Good - real parallelism, off the UI isolate
final data = await Isolate.run(() => jsonDecode(hugeJson) as Map<String, Object?>);
```

The closure and everything it captures are copied to the new isolate, so captured state must be sendable - no `BuildContext`, no open subscriptions, no platform-channel handles. Startup and copying cost real time, so small payloads finish faster inline; move work off-isolate because it is measurably long, not on principle.

`compute` is Flutter's wrapper over the same mechanism; pass it a top-level or static function. On web there are no isolates, so this work stays on the main thread and the jank remains - if web is a target tier, reduce the work instead of relocating it.

### Declarative collections

```dart
// Bad - imperative assembly, nothing can be const
final actions = <Widget>[];
actions.add(const SaveAction());
if (canDelete) actions.add(const DeleteAction());

// Good
final actions = <Widget>[
  const SaveAction(),
  if (canDelete) const DeleteAction(),
  for (final tag in tags) TagChip(tag),
  ...?extraActions,
];
```

`...?` spreads a possibly-null list without a branch. The same syntax works in map and set literals.

## Output Format

When invoked from an implementation workflow, emit decisions per concern:

```
| Concern | Decision | Rationale |
|---------|----------|-----------|
| Load state | sealed class + exhaustive switch | 4 variants carrying distinct data |
| Ids | extension type UserId(String) | compile-time swap detection, no runtime cost |
| Parse result | record ({DateTime start, DateTime end}) | two values, no behavior |
| Feed decode | Isolate.run | 30-50ms payloads on the UI isolate |
```

One row per shape decision the task actually forced; do not pad with concerns the task never raised.

When invoked from a review workflow, emit one block per finding:

```
### [Must | Recommend] file:line

- Category: {Null Safety | Exhaustiveness | Type Modeling | Async | Resource Lifecycle | Collection Style}
- Code: {one-line citation}
- Issue: {the Dart 3 rule broken and the failure it produces}
- Recommendation: {concrete edit}
```

`[Must]` when the violation produces a runtime failure or leak (a scheduled-crash `!` or `late`, disabled exhaustiveness, missing `cancel`/`close`, swallowed or unobserved errors, UI-thread jank). `[Recommend]` for shape and readability (type modeling, collection style, serialized independent awaits).

For each category with zero findings, emit exactly `No <category> findings.` using the category name from the enum, so the workflow knows the check ran. Omit that line for categories that have findings.

When the Dart SDK version is below 3.0, report `Dart 3 patterns unavailable: SDK <version>` once and restrict findings to null safety, async, and resource lifecycle. On 3.0-3.2, skip `extension type` recommendations (3.3+); below 3.2, treat private-final field promotion as unavailable.

## Avoid

- `!` to clear an analyzer error - find why promotion failed; usually the field needs to be private and final, or copied to a local
- `late` on anything genuinely optional - use `?`
- `default` or `_` on a switch over a sealed type - it silently disables exhaustiveness checking
- `dynamic` to escape a type error; `Object?` plus a pattern match keeps the check
- Records where a named domain type belongs, or a class where a two-value return would do
- Plain `class` for public types with no modifier considered - state `final`, `base`, `interface`, or `sealed`
- `implements <representation>` on an extension type unless substitutability is the goal
- `throw e` inside `catch` - use `rethrow`
- Bare `catch (e)` outside a reporting boundary, and `catch` blocks that swallow without logging
- Un-awaited futures with no `unawaited()` and no error handler - failures vanish
- `listen` without a stored subscription, or a `StreamController` with no `close`
- `Isolate.run` for work measured in microseconds - the copy costs more than the work
- `Future.delayed` as synchronization between async steps
