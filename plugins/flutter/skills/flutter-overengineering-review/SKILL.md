---
name: flutter-overengineering-review
description: Flutter necessity review - StatefulWidget without state, single-field notifiers, single-impl interfaces, blanket freezed, dead null checks.
metadata:
  category: mobile
  tags: [flutter, dart, riverpod, code-review, overengineering, necessity, redundancy]
user-invocable: false
---

# Flutter Overengineering Review

> Confirm the project's state-management library from `pubspec.yaml` first. The State Abstraction category is written against Riverpod; under Bloc / Provider / GetX apply the same discriminator (does anything outside this widget read it?) using that library's own primitives.

## When to Use

- Reviewing a Flutter diff that adds widgets, notifiers, providers, interfaces, `freezed` models, or mapping layers
- Catching code that compiles, renders, and passes tests but does not need to exist

## Rules

- Every finding names what makes the abstraction unnecessary: no mutable state or lifecycle, no second reader, no second implementer, no transformation, the type is already non-nullable. When several stack, comma-separate them in `Unnecessary because:`.
- Intent:
  - **`[Recommend]`** (default). Name the constraint, recommend the edit. Escalate to **`[Must]`** when measurable cost is present; cite it in `Cost:`. Triggers: `dynamic` propagating past a decode boundary into business logic or a widget (runtime failure replaces a compile error); a mapping hop that silently drops a field; a branch presented as handling a case it can never reach, leaving the real failure mode unhandled
  - **`[Recommend]`** when justification is plausible but not visible in the diff - state the justification being assumed and ask the author to confirm
- An abstraction with **visible** justification - a second implementer, a test override, a union case - is not a finding
- `Cost:` on a pure-abstraction `[Must]` is maintenance cost (parallel definitions to keep in sync, `build_runner` time, lost jump-to-implementation), not runtime cost - still measurable and worth citing
- Never propose deleting a layer the diff's own tests bind to
- Depth is not the metric. Flutter composition is intentionally nested; the finding is indirection nothing reads, not tree height

## Patterns

### Category 1: Widget Structure

#### `StatefulWidget` with no mutable state

```dart
// Bad - no setState, no controller, no lifecycle override; State holds nothing
class _StatusChipState extends State<StatusChip> {
  @override
  Widget build(BuildContext context) => Text(widget.label);
}

// Good - StatelessWidget; one class instead of two, no widget. indirection
class StatusChip extends StatelessWidget { ... }
```

Justified when the `State` owns a controller, subscription, timer, or `FocusNode`, overrides `initState` / `didUpdateWidget` / `didChangeDependencies` / `dispose`, or holds ephemeral UI state driven by `setState` (a password-visibility toggle). An `AnimationController` alone justifies it.

#### Wrapper widget that adds nothing

```dart
// Bad - a class and a rebuild boundary that change nothing
class ContentBox extends StatelessWidget {
  const ContentBox({super.key, required this.child});
  final Widget child;
  @override
  Widget build(BuildContext context) => child;
}
```

Justified when the wrapper adds semantics, padding, theming, a key, or a deliberate `const` rebuild boundary - or when it is a named design-system seam (`AppCard`) that today forwards to one widget and is the agreed place to change it.

#### Nesting where an extracted widget reads better

```dart
// Bad - a build method returning a subtree; always rebuilds with the parent
Widget _buildHeader() => Row(children: [...]);

// Good - a widget class: its own element and rebuild boundary, const when its inputs allow
_OrderHeader(order: order),
```

Extract into a widget class, not a `_buildX()` helper. Justified nesting: structural layout the framework requires (`Scaffold` > `Column` > `Expanded` > `ListView`) is not a finding.

### Category 2: State Abstraction

#### Notifier that wraps one field with a setter

```dart
// Bad - a class, a provider, and a setter to hold one bool nothing else reads
class SortAscending extends Notifier<bool> {
  @override
  bool build() => true;
  void set(bool v) => state = v;
}

// Good - lives and dies with the widget
bool _sortAscending = true;   // setState in the State
```

Justified when a second widget reads it, it must survive the widget's disposal (route change, tab switch), or the same PR gives the notifier real transition logic. A one-field notifier is correct in Riverpod for genuinely shared state - `StateProvider` is legacy, so a `Notifier` with a setter is the supported form.

#### Provider with no dependents and no variation

```dart
// Bad - never overridden, never changes, one call site
final appNameProvider = Provider<String>((ref) => 'Acme');

// Good
const appName = 'Acme';
```

Justified when the provider exists as a DI seam - a clock, config, base URL, or feature source overridden in tests, flavors, or previews. A constant-looking provider that exists to be overridden is correct; check for override sites before flagging.

#### Custom `InheritedWidget` beside an existing state solution

```dart
// Bad - hand-rolled InheritedWidget + of() + updateShouldNotify duplicating the provider layer
class ThemeScope extends InheritedWidget { static ThemeScope of(BuildContext c) ... }
```

Justified when the value is genuinely positional - scoped to one route, one list item, or one subtree, where "which instance" depends on where you are in the tree - or when the code is a reusable package that must not depend on a state library.

### Category 3: Premature Abstraction

#### Single-implementation interface for testability

```dart
// Bad - abstract class with one implementer; the test substitutes via provider override anyway
abstract class UserRepository { Future<User> fetch(String id); }
class HttpUserRepository implements UserRepository { ... }

// Good - one concrete class; Dart's implicit interface still allows the fake
class UserRepository { ... }
class FakeUserRepository implements UserRepository { ... }   // valid in tests
```

Every Dart class defines an implicit interface, so `implements` works against a concrete class - testability alone does not require the abstract declaration. Justified when 2+ implementations exist or arrive in the same PR (local + remote source), the type is a published package's public API, or the class carries a `final` / `base` modifier that blocks implicit-interface substitution.

#### `freezed` on models with no copy, union, or equality need

```dart
// Bad - generated copyWith / == / union machinery no caller uses
@freezed
class LogEvent with _$LogEvent { const factory LogEvent(String message) = _LogEvent; }

// Good - only serialization is needed
@JsonSerializable()
class LogEvent { LogEvent(this.message); final String message; }
```

`Cost:` `build_runner` time and a generated file per model. `freezed`'s class-declaration syntax differs across major versions - match the project's existing models rather than the snippet above. Justified for sealed unions and state machines, value equality (provider `family` keys, set/map membership, test comparisons), and `copyWith` on a wide model.

#### Mapping hops with no transformation

```dart
// Bad - identity mapping; every field copied unchanged, then again into an entity
User toModel(UserDto d) => User(id: d.id, name: d.name, email: d.email);
```

`Unnecessary because:` the hop is field-for-field - the type is renamed, not transformed. Justified when a hop actually converts (wire-nullable to domain non-nullable, epoch int to `DateTime`, flattening, unit or enum mapping), or it isolates a volatile wire format from a domain many screens read.

#### Speculative configuration and feature flags

```dart
// Bad - one call site, constant value, other branch never compiled into a test
if (FeatureFlags.newCheckout) { ... } else { ... }   // else is dead
```

Justified when a remote-config source, staged rollout, or kill switch backs the flag. Otherwise delete the dead branch - an untested branch is a liability, not a safety net.

### Category 4: Type-System Waste

#### Null check on a non-nullable value

```dart
// Bad - name is String, not String?; the branch cannot run
void greet(String name) {
  if (name == null) return;
  ...
}
```

Same smell as `user?.name ?? 'unknown'` where `user` is non-nullable. Justified where a value crosses an unsound boundary - JSON decode, a platform-channel result, a plugin returning `dynamic`, FFI - because the declared type there is a claim, not a guarantee. Validate once at that boundary, not repeatedly downstream.

#### `dynamic` to silence a type error

```dart
// Bad - erases checking; a wrong shape fails at runtime, deep in the widget tree
dynamic payload = jsonDecode(body);
final id = payload['data']['id'];

// Good - narrow at the boundary, stay typed after it
final map = jsonDecode(body) as Map<String, dynamic>;
final id = User.fromJson(map).id;
```

`Map<String, dynamic>` at the decode boundary is idiomatic. The finding is `dynamic` propagating past it into repositories, notifiers, or widgets. Prefer `Object?` plus a pattern match or cast when the shape is genuinely open.

#### Generic type parameter with one instantiation

```dart
// Bad - declared as Cache<T>, only ever Cache<User>
class Cache<T> { ... }

// Good - concrete until a second instantiation exists
class UserCache { ... }
```

Justified when 2+ instantiations exist in the codebase or the type is a package's public API.

## Output Format

One block per finding; the consuming workflow merges them:

```
### [Must | Recommend] file:line

- Category: {Widget Structure | State Abstraction | Premature Abstraction | Type-System Waste}
- Code: {one-line citation}
- Unnecessary because: {what makes it dead or unread; comma-separate when stacked}
- Cost: {required for [Must]; omit otherwise}
- Recommendation: {concrete Dart edit}
- Justified when: {one-line note if a legitimate reason might apply; otherwise omit}
```

For each category with zero findings, emit exactly: `No <category> findings.` (using the category name from the enum) so the workflow knows the check ran. Omit this line for categories that have at least one finding.

## Avoid

- Flagging a `StatefulWidget` that owns a controller, subscription, timer, or `FocusNode`
- Flagging `freezed` on a sealed union or on a type used as a provider `family` key
- Flagging an abstract class with 2+ implementers or one a shipped package exposes
- Flagging a provider that exists to be overridden in tests, flavors, or previews
- Flagging `Map<String, dynamic>` at a JSON decode boundary
- Flagging null checks on values arriving from JSON, a platform channel, a plugin, or FFI
- Flagging structural nesting the layout requires
- Flagging `const` constructors, `Key`, or `super.key` as ceremony
- Proposing a migration off the project's state-management library
- Treating widget count, file count, or tree depth as a complexity metric
- Removing a layer the diff's own tests bind to
- Raising findings against generated files (`*.g.dart`, `*.freezed.dart`, `*.gr.dart`, `*.mocks.dart`)
