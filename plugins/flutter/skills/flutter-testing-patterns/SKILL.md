---
name: flutter-testing-patterns
description: "Plan Flutter tests by layer: unit, widget, golden, integration_test; pump vs pumpAndSettle, golden stability, mocktail, Riverpod overrides."
metadata:
  category: mobile
  tags: [flutter, dart, testing, widget-test, golden-test, integration-test, mocktail, riverpod]
user-invocable: false
---

# Flutter Testing Patterns

> This skill owns test layering, doubles, and golden stability. Provider shapes belong to `flutter-riverpod-patterns`; what to assert about accessibility belongs to `flutter-accessibility`.

## When to Use

- Choosing which layer a behavior should be tested at
- Authoring or reviewing widget, golden, or `integration_test` suites
- Diagnosing a flaky test, or a golden that passes locally and fails in CI
- Wiring fakes into Riverpod and stubbing the network

## Rules

- Test at the cheapest layer that can observe the behavior: pure logic as unit, anything needing `BuildContext`/layout/gestures as widget, pixel appearance as golden, real engine and platform plugins as `integration_test`.
- Widget tests are the default layer for UI. Goldens supplement behavioral assertions; they never replace them.
- No live backend in any layer. Stub at the HTTP client boundary, or inject a fake repository.
- Doubles enter through the seam production uses - a Riverpod override or a constructor argument. Never a mutable global swapped in `setUp`.
- `pumpAndSettle` only when pending work is finite. Continuous animation or a repeating timer requires `pump(Duration)`.
- Goldens are generated and verified on one pinned platform and Flutter version. A golden written on a developer machine and checked in a different CI image is a broken test, not a real diff.
- Fonts are loaded explicitly before any golden renders. The default test font draws every glyph as a box.
- Pin every source of nondeterminism a golden can see: surface size, device pixel ratio, clock, randomness, animation state, image sources.
- `--update-goldens` output is reviewed as an image diff. Regenerating to make CI green discards the signal the test exists for.
- One behavior per test, named for the behavior. Never `Future.delayed` to let things settle.

## Patterns

### Choosing a layer

| Behavior | Layer | Why |
|----------|-------|-----|
| Failure mapping, formatters, validators, notifier state transitions | Unit | no widget tree needed |
| Loading / empty / error states render correctly | Widget | needs a tree and finders |
| "tapping retry re-issues the request" | Widget | needs gesture plus rebuild |
| A themed component's appearance across breakpoints or themes | Golden | the contract is pixels |
| Login through to home with real plugins, permissions, deep links | Integration | needs a real engine and device |

Push down whenever possible. A widget test that only checks a formatted string should be a unit test on the formatter.

### `pump` vs `pumpAndSettle`

```dart
// Bad - the spinner animates forever, so this never settles and the test times out
await tester.tap(find.byType(SubmitButton));
await tester.pumpAndSettle();

// Good - advance exactly as far as each assertion needs
await tester.tap(find.byType(SubmitButton));
await tester.pump(); // one frame: the in-flight state is now on screen
expect(find.byType(CircularProgressIndicator), findsOneWidget);
await tester.pump(const Duration(seconds: 1)); // let timers fire, then rebuild
expect(find.text('Saved'), findsOneWidget);
```

`pump()` builds exactly one frame. `pumpAndSettle()` pumps repeatedly until no frame is scheduled, and fails on timeout if something animates indefinitely - a repeating indicator, a looping `AnimationController`, or a periodic timer.

`testWidgets` runs in a fake-async zone, so `pump(Duration)` advances timers rather than waiting in real time. Work that depends on real I/O will not complete this way; wrap that in `tester.runAsync`.

### Finders

```dart
// Bad - breaks when copy is edited or the locale changes
expect(find.text('Sign in'), findsOneWidget);

// Good - stable across copy and localization
expect(find.byKey(const Key('signInButton')), findsOneWidget);
```

Use `find.text` when the copy itself is the assertion, `find.bySemanticsLabel` when what a screen reader announces is the assertion, and `find.byKey`/`find.byType` for structure. `findsOneWidget` over `findsWidgets` - an unexpected duplicate is a real bug worth failing on.

### Golden stability

This is where Flutter test suites break most often. Address all four causes up front.

**1. Load fonts once for the whole suite.** A file named `flutter_test_config.dart` at the root of `test/` wraps every test in that directory tree:

```dart
// test/flutter_test_config.dart
Future<void> testExecutable(FutureOr<void> Function() testMain) async {
  TestWidgetsFlutterBinding.ensureInitialized();
  final loader = FontLoader('Inter')
    ..addFont(rootBundle.load('assets/fonts/Inter-Regular.ttf'))
    ..addFont(rootBundle.load('assets/fonts/Inter-Bold.ttf'));
  await loader.load();
  await testMain();
}
```

Without this, goldens encode boxes instead of glyphs, so every later typography change passes unnoticed. The path passed to `rootBundle.load` is the asset path as declared in `pubspec.yaml`; fonts shipped by a package need a `packages/<package_name>/` prefix.

**2. Fix the surface.** Physical size and device pixel ratio change every rendered pixel:

```dart
tester.view.physicalSize = const Size(1080, 1920);
tester.view.devicePixelRatio = 3.0;
addTearDown(tester.view.reset);
```

Without the tear-down the override leaks into later tests in the same file, so failures depend on test order.

**3. Set a tolerance deliberately.** The default comparator is an exact byte match, so a single antialiased pixel fails the test. `goldenFileComparator` is a global that can be replaced in `flutter_test_config.dart`; the standard approach subclasses `LocalFileComparator` and overrides its comparison to permit a small fraction of differing pixels. Keep the threshold as low as CI tolerates - a generous tolerance silently accepts real regressions, which is worse than a brittle test.

**4. Generate and verify on one platform.** Text rasterization and shader output differ across host operating systems and across Flutter versions, so the same widget yields different bytes on macOS and in a Linux CI container. Two workable setups:

| Approach | Cost | When |
|----------|------|------|
| Run goldens only in the pinned CI job; tag them and skip that tag elsewhere | Low | default choice |
| Commit per-platform golden directories and select by host platform | High | a team that genuinely reviews goldens on multiple hosts |

Tag with `@Tags(['golden'])` above `library;` at the top of the file, then select that tag with `flutter test --tags golden` in the pinned job and exclude it from the general job. Pin the Flutter version in CI: an engine upgrade legitimately rewrites every golden, and that should land as one reviewed commit rather than as scattered flakes.

Diagnosing a drift:

| Symptom | Cause |
|---------|-------|
| Text renders as boxes or blank | fonts never loaded |
| Everything shifted or scaled uniformly | different surface size or device pixel ratio |
| Faint edge and antialiasing differences only | golden generated on a different host OS |
| Every golden in the repo changes at once | Flutter or engine version bump |
| Passes locally, fails only in CI | golden was regenerated on a developer machine |

### mocktail

```dart
class MockUserRepo extends Mock implements UserRepo {}
class FakeUser extends Fake implements User {}

setUpAll(() => registerFallbackValue(FakeUser()));

test('save posts the edited user', () async {
  final repo = MockUserRepo();
  when(() => repo.save(any())).thenAnswer((_) async {});

  await EditUser(repo).call(user);

  verify(() => repo.save(user)).called(1);
});
```

mocktail needs no code generation: stubs are closures, which is why every `when`/`verify` is wrapped in `() =>`. `any()` on a non-primitive argument type requires a `registerFallbackValue` for that type first, or matching throws at call time.

```dart
// Bad - a mock with five stubbed methods reimplementing a repository
when(() => repo.get(any())).thenAnswer(...);
when(() => repo.put(any(), any())).thenAnswer(...);
// ...

// Good - a hand-written fake holds the behavior once and is reusable
class InMemoryUserRepo implements UserRepo {
  final _users = <String, User>{};
  @override Future<User> fetch(String id) async => _users[id] ?? (throw const NotFoundFailure());
  @override Future<void> save(User u) async => _users[u.id] = u;
}
```

Mocks are for verifying interactions. When the double needs behavior, a fake is shorter and does not rot every time the interface gains a method.

### Riverpod overrides

```dart
// Notifier / provider tests
final container = ProviderContainer(
  overrides: [userRepoProvider.overrideWithValue(InMemoryUserRepo())],
);
addTearDown(container.dispose);

final user = await container.read(userProvider('u1').future);
expect(user.name, 'Ada');

// Widget tests - the same seam, one layer up
await tester.pumpWidget(
  ProviderScope(
    overrides: [userRepoProvider.overrideWithValue(InMemoryUserRepo())],
    child: const App(),
  ),
);
```

`overrideWithValue` fits a plain `Provider` holding a dependency; `overrideWith((ref) => fake)` covers the other provider types. Always `addTearDown(container.dispose)` - an undisposed container keeps listeners and timers alive into the next test.

Override the dependency, not the thing under test. Overriding `userProvider` itself in a test about `userProvider` asserts only that the override works.

### Stubbing the network at the client boundary

```dart
// Bad - hits the real API: slow, credentialed, fails when someone else deploys
final repo = UserRepo(Dio());

// Good - swap the transport, keep the repository under test
final dio = Dio()..httpClientAdapter = stubAdapter; // e.g. http_mock_adapter
final repo = UserRepo(dio);
```

Swapping the adapter keeps interceptors, serialization, and error mapping inside the test - which is exactly the code most likely to be wrong. Substitute a fake repository only when the test is about the consumer rather than the repository.

`flutter_test` installs an `HttpClient` that fails every real request, so `Image.network` renders as an error in widget tests. Inject a fake `ImageProvider` or preloaded bytes rather than expecting network images to appear.

### `integration_test`

```dart
// integration_test/login_test.dart
void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  testWidgets('user logs in and reaches home', (tester) async {
    app.main();
    await tester.pumpAndSettle();
    await tester.enterText(find.byKey(const Key('email')), 'ada@example.com');
    await tester.tap(find.byKey(const Key('submit')));
    await tester.pumpAndSettle();
    expect(find.byType(HomeScreen), findsOneWidget);
  });
}
```

Reserve this layer for a handful of flows that are worthless if broken: launch, auth, purchase. It needs a device or emulator and runs orders of magnitude slower than widget tests. Point it at a stubbed or seeded backend selected by `--dart-define`, not at a shared environment that other people are changing.

## Output Format

When invoked from a test-strategy workflow, emit the plan as one row per target:

```
| Target | Layer | Doubles | Determinism | Rationale |
|--------|-------|---------|-------------|-----------|
| `FailureMapper.fromDio` | Unit | None | Deterministic | pure mapping, no tree |
| `LoginScreen` error state | Widget | Fake | Deterministic | needs tree + tap |
| `PrimaryButton` variants | Golden | None | Needs-Pinning | fonts + 1080x1920 @3.0, Linux CI only |
| login -> home | Integration | Stubbed-Transport | At-Risk | real plugins, device-dependent |
```

- `Layer: {Unit | Widget | Golden | Integration}`
- `Doubles: {None | Fake | Mock | Stubbed-Transport}`
- `Determinism: {Deterministic | Needs-Pinning | At-Risk}` - `Needs-Pinning` means it is stable once fonts, surface, tolerance, and platform are fixed; `At-Risk` means a residual source of flake remains and is named in the rationale.

Follow the table with gaps, omitting the section when there are none:

```
### Coverage Gaps

- {target} - Missing Layer: {Unit | Widget | Golden | Integration} - Risk: {High | Medium | Low} - {why it matters}
```

When invoked from a review workflow or directly to diagnose a flaky test, emit one finding block per defect:

```
### [Blocker | High | Medium | Low] test/path/file_test.dart:LINE

- Test: {test name}
- Defect: {Wrong-Layer | Flaky-Pump | Unstable-Golden | Live-Dependency | Leaky-Double | Weak-Assertion}
- Impact: {what breaks, or what the test fails to catch}
- Fix: {concrete edit}
```

| Defect | Meaning |
|--------|---------|
| `Wrong-Layer` | tested at a slower layer than the behavior requires |
| `Flaky-Pump` | `pumpAndSettle` on unbounded animation, or a missing pump before an assertion |
| `Unstable-Golden` | fonts, surface, tolerance, or platform not pinned |
| `Live-Dependency` | real network, real clock, real filesystem, or shared backend |
| `Leaky-Double` | global or static swapped in `setUp`, or an undisposed `ProviderContainer` |
| `Weak-Assertion` | asserts the double was configured, or only that the widget tree built |

`Blocker` = the test lies (passes while the behavior is broken: Weak-Assertion on the only coverage, tolerance raised to green, regenerated golden). `High` = flaky or environment-dependent (Flaky-Pump, Unstable-Golden, Live-Dependency). `Medium` = order-dependence or maintenance drag (Leaky-Double, Wrong-Layer). `Low` = style. If there are no findings, emit exactly `No testing findings.` so the workflow knows the check ran.

## Avoid

- `pumpAndSettle` after triggering anything that animates continuously
- `Future.delayed` or `sleep` inside a test to wait for a rebuild
- Goldens as the only coverage for a screen - they assert appearance, not behavior
- Regenerating goldens with `--update-goldens` to clear a red build without inspecting the diff
- Golden tests without loaded fonts, a fixed surface size, or a pinned platform
- Golden generation on developer machines when CI verifies on a different OS
- Tolerance raised until the suite passes - fix the nondeterminism instead
- A live backend, real clock, or real filesystem in unit, widget, or golden tests
- `any()` on a custom type without `registerFallbackValue` for it
- Overriding the provider under test rather than its dependencies
- A `ProviderContainer` created without `addTearDown(container.dispose)`
- `find.text` on user-facing copy that localization or product will change
- `integration_test` for logic a widget test can already observe
- Asserting only `findsOneWidget` on a screen while never checking its rendered state
