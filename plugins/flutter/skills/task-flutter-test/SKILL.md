---
name: task-flutter-test
description: Flutter test plan and scaffolding - unit, widget, golden, and integration_test layering, mocktail, provider overrides, golden stability in CI.
agent: flutter-test-engineer
metadata:
  category: mobile
  tags: [flutter, dart, testing, widget-test, golden, integration-test, mocktail, riverpod, workflow]
  type: workflow
user-invocable: true
---

> **Behavioral directive:** Load `Use skill: behavioral-principles` before executing this workflow.

# Flutter Test

Flutter-aware test strategy and scaffolding across unit, widget, golden, and `integration_test` layers, using mocktail and provider overrides, with golden stability treated as a first-class concern.

## When to Use

- Test strategy for a new Flutter app or feature
- Test-coverage gap assessment across layers
- Scaffolding tests for under-covered screens, state holders, or repositories
- Test pyramid review
- Adding failure-path tests to happy-path-only tests
- Diagnosing goldens that pass locally and fail in CI

**Not for:** debugging a failing test whose production code is wrong (`flutter-engineer`), general review (`task-flutter-review`).

## Workflow

### Step 1 - Stack and Project Shape

Accept the project shape from a parent workflow when invoked as a subagent. Otherwise read `pubspec.yaml`. Record state management, navigation, networking client, persistence store, mocking library (mocktail / mockito / hand-written), and whether the project uses code generation.

If state management is not Riverpod, say so and use that library's own test seams rather than provider overrides.

### Step 2 - Read Code Under Test + Existing Tests

Ground output in project conventions, not generic templates.

- Read each target top-to-bottom: the screen's widget tree, its state holder, the repository it depends on, and the failure types it can surface
- Glob `test/**/*_test.dart` and read one existing widget test, one unit test, one golden test, and any shared setup file - learn the project's finders, pump conventions, fake construction, and override style
- Read `test/flutter_test_config.dart` if present; it is where global golden and font setup usually lives
- Read CI configuration for how tests run, whether goldens run in a separate job, and how failures surface
- Read `integration_test/` for existing driver setup

If no existing tests: say so and propose conventions explicitly rather than inventing them silently.

### Step 3 - Flutter Test Pyramid

| Layer | Tooling | What belongs |
|-------|---------|--------------|
| Unit | `test` / `flutter_test` + mocktail | State-holder logic, failure mapping, validators, formatters, pure functions |
| Widget | `testWidgets` + finders | A screen or component in isolation: renders, responds to input, shows each state |
| Golden | `matchesGoldenFile` | Pixel-level appearance of visually complex or regression-prone UI only |
| Integration | `integration_test` on a device or emulator | Critical end-to-end journeys only |

**Many** unit, **some** widget, **few** golden and integration. Goldens are a deliberate minority: they are the most expensive layer to keep stable and the least specific about what broke.

### Step 4 - Apply Flutter Test Patterns

Use skill: `flutter-testing-patterns` for canonical finders, pump semantics, golden stability, mocktail usage, and provider overrides. Notes below cover layer-specific items.

**Unit tests:**

- Test the state holder directly, without building a widget tree. If a unit test needs a widget, it is misclassified
- One group per public method; cover success, each failure type, and the edges between them
- Failure mapping deserves its own tests: assert that a transport error becomes the intended domain failure, not just that "an error happened"
- Inject fakes through the same seam production uses, so the test proves the wiring too

**Widget tests:**

- Build the widget under a test harness that supplies its dependencies via overrides, not a real network client or database
- Assert on every state the screen can render: loading, error, empty, and populated. A screen tested only in its populated state is half-tested
- Prefer pumping a bounded number of frames over settling when the widget has an indefinite animation or a pending timer - settling on those hangs the test rather than failing it usefully
- Find by semantics or key where possible; finding by literal text couples the test to copy and breaks under localization
- Test user interaction, not just initial render: tap, scroll, enter text, then assert the resulting state

**Golden tests:**

Golden instability is the dominant failure mode, and it is almost always environmental rather than a real regression.

- Load fonts deterministically in test setup; without it, text renders as boxes locally and differently in CI
- Pin the surface size and device pixel ratio rather than inheriting the host's
- Expect platform-dependent rendering differences: a golden generated on one OS is not reliably byte-identical on another. Either generate per platform or run goldens on one designated platform in CI
- Configure a tolerance rather than demanding byte equality when antialiasing differences are the only delta
- Tag goldens so the main test job can exclude them and a dedicated job can run them
- Regenerating a golden must be a deliberate act with the diff reviewed, never a reflex when CI goes red

**Integration tests:**

- Reserve for journeys that cross screens, persistence, and the network together: sign-in through to first meaningful screen, purchase, offline-then-reconnect
- Stub the network at the client boundary rather than pointing at a live backend, so the test is deterministic
- Avoid for anything a widget test could cover; they are slow and fail for environmental reasons

### Step 5 - Test Boundaries

**Unit:** state holders, failure mapping, validators, formatters, domain rules, calculations, and any pure transformation

**Widget:** every screen - each of loading, error, empty, populated; user interaction paths; navigation triggers; form validation feedback; accessibility labels present on interactive elements

**Golden:** visually complex components, custom painters, theming across light and dark, and layout at representative breakpoints. Not simple layouts whose correctness a widget test already asserts

**Integration:** critical journeys only; on-device persistence surviving a restart; permission-gated flows

**Does NOT need a test:** framework-provided behavior, generated code (`*.g.dart`, `*.freezed.dart`, and siblings), and trivial pass-through delegation with no logic

### Step 6 - Test Data and Fakes

- Factory functions with sensible defaults and named overrides, rather than hand-rolled literals repeated per test
- Fakes over mocks where the collaborator has real behavior worth simulating; mocks where you only need to assert an interaction happened
- Keep test data minimal - a 100-item fixture for a test asserting one row signals the wrong layer
- Register fallback values for any custom type used in a mocktail argument matcher, in shared setup rather than per test

### Step 7 - Prioritization (when coverage is low)

If coverage is low, run this **before** scaffolding - it determines which tests come first.

Measure, do not guess: run the project's coverage command when the suite runs locally; when it cannot run, estimate from test-file density and label the number an estimate.

| Priority | Targets |
|----------|---------|
| P1 - Auth and session | Sign-in, token refresh, expiry and forced sign-out, permission-gated screens |
| P2 - Data integrity | Failure mapping, on-device persistence and its migrations, sync and conflict resolution, optimistic-update rollback |
| P3 - Business-critical | Purchase and billing flows, anything irreversible from the user's side, state machines |
| P4 - High-churn | Files with frequent recent commits (`git log --since="3 months ago"`) or bug-fix history |
| P5 - Presentational | Simple stateless components with no logic - lower risk, can wait |

**Multi-band rule.** When a target qualifies for multiple bands, file it under the highest (lowest number) and note the secondary so the plan covers both axes.

### Step 8 - Test Infrastructure Hygiene

- [ ] Fonts loaded deterministically before golden tests
- [ ] Goldens tagged and run in a job that can be excluded from the fast feedback loop
- [ ] Golden platform expectations documented, so a cross-platform diff is not mistaken for a regression
- [ ] Network stubbed at the client boundary; no real network in CI
- [ ] Test setup overrides only what differs from production - never silently disables auth
- [ ] Fallback values registered once in shared setup for custom matcher types
- [ ] Generated code regenerated and committed state verified when models change
- [ ] Coverage command documented; thresholds, if any, stated per area rather than as one global number
- [ ] Integration tests segregated so they do not gate every pull request

## Output Format

**Which output to produce:**

- "What tests are missing?" -> Coverage Assessment
- "Write tests for X" / "scaffold" -> Test Scaffolds
- "Why does this golden/test fail in CI?" -> the diagnosis finding block from `flutter-testing-patterns` (cause + fix); no strategy doc
- "Test strategy" / "test plan", OR low coverage with no scaffolds requested -> Strategy Doc (optionally with Coverage Assessment)
- 2+ deliverables -> in this order separated by `---`: Coverage Assessment -> Strategy Doc -> Test Scaffolds
- Unclear -> default to Strategy Doc

**Coverage Assessment:**

```markdown
## Flutter Test Coverage Assessment

**Stack:** Flutter <version> / Dart <version>
**State Management:** Riverpod | Bloc | Provider | GetX | none
**Mocking:** mocktail | mockito | hand-written
**Coverage gaps:**

- **Unit:** [state holders / failure mapping / validators without coverage]
- **Widget:** [screens without tests; screens missing loading, error, or empty coverage]
- **Golden:** [visually complex components without goldens; unstable goldens]
- **Integration:** [critical journeys without coverage]
- **Auth and session:** [flows without expiry, refresh, or forced-sign-out coverage]

**Recommended pyramid balance:** Unit [target] / Widget [target] / Golden + integration [target - keep small]

**Prioritization** _(include when coverage is low or gaps exceed 5)_:

1. **P1 - Auth and session:** [specific flows]
2. **P2 - Data integrity:** [failure mapping, migrations, sync, rollback]
3. **P3 - Business-critical:** [purchase, billing, irreversible actions]
4. **P4 - High-churn:** [files with frequent recent commits or bug-fix history]
5. **P5 - Presentational:** [simple stateless components]
```

**Test Scaffolds:** ready-to-run files using project conventions:

- Right test layer for the behaviour
- Grouped cases with descriptive names, not copy-pasted bodies
- Factories over raw literals
- Widget tests covering loading, error, empty, and populated
- Fakes injected through the project's own dependency seam
- Goldens only where a widget test cannot express the concern, with font and size setup included
- **Verified before delivery:** run `flutter analyze` and the generated tests. If the environment cannot run them (no Flutter toolchain, no device for integration tests), say which subset was not run rather than implying a clean pass. Never deliver a scaffold that does not analyze cleanly.

**Strategy Doc:**

```markdown
## Flutter Test Strategy

**Objective:** [what this strategy achieves]
**Pyramid balance:** Unit {x}% / Widget {y}% / Golden + integration {z}%
**Tooling:** flutter_test, mocktail, provider overrides, integration_test
**Golden policy:** [which platform generates them, tolerance, how they are regenerated]
**Network isolation:** [where the client boundary is stubbed]
**Gaps to close (prioritized):**

1. [Highest risk - typically auth/session or failure mapping]
2. ...
```

## Self-Check

**Always:**

- [ ] `behavioral-principles` loaded before the workflow ran
- [ ] Stack confirmed; state management, mocking library, and codegen usage recorded
- [ ] Code under test + existing tests + shared setup read directly
- [ ] `flutter-testing-patterns` consulted
- [ ] Non-Riverpod projects use their own test seam rather than provider overrides

**Strategy / Coverage:**

- [ ] Pyramid mapped to Flutter layers (unit -> state holders; widget -> screens; golden -> visual regression; integration -> journeys)
- [ ] Boundaries defined: each layer covers what it does best; no duplicated assertions
- [ ] Risk-based prioritization when coverage is low (P1 auth, P2 integrity, P3 business, P4 churn, P5 presentational)
- [ ] Golden policy stated: platform, fonts, tolerance, regeneration discipline
- [ ] Screens missing loading, error, or empty coverage flagged explicitly

**Scaffolds:**

- [ ] Grouped and descriptive, not copy-pasted
- [ ] Factories over raw literals
- [ ] Widget tests cover loading, error, empty, and populated
- [ ] Finders prefer key or semantics over literal copy
- [ ] Fakes injected through the production dependency seam
- [ ] Golden scaffolds include font and surface-size setup
- [ ] Fallback values registered for custom matcher types
- [ ] Scaffolds analyzed and run before delivery; any subset not run is named

## Avoid

- Scaffolding without reading existing tests and shared setup
- Chasing a coverage number instead of prioritizing by risk
- Testing only the populated state of a screen that also has loading, error, and empty
- Golden tests as a substitute for asserting the logic that produced the pixels
- Regenerating a failing golden without reading the diff
- Byte-exact golden comparison across platforms
- Running goldens in the same job as fast unit tests
- Finding widgets by literal user-facing text in a localized app
- Settling a widget test that contains an indefinite animation or pending timer
- Pointing integration tests at a live backend
- Real network calls in CI
- Full integration tests for what a widget test could cover
- Testing framework internals or generated code
- Disabling auth in test setup to make unrelated tests pass
- Delivering scaffolds that were never analyzed or run
