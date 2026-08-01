---
name: task-flutter-implement
description: End-to-end Flutter feature implementation - freezed models, repository, Riverpod state, adaptive screens, routes, and tests.
agent: flutter-engineer
metadata:
  category: mobile
  tags: [flutter, dart, riverpod, go-router, dio, drift, feature, implementation, workflow]
  type: workflow
user-invocable: true
---

> **Behavioral directive:** Load `Use skill: behavioral-principles` before executing this workflow.

# Implement Flutter Feature

## When to Use

End-to-end Flutter feature work: models + data layer + state + screens + navigation + tests in one pass.

Not for: single-widget tweaks, bugfixes, pure styling changes, or backend work. A feature that only changes how an existing screen looks is a UI edit, not this workflow.

## Rules

- Widgets render state; state holders own side effects. No I/O, no navigation decisions, and no business logic in `build`
- Every network call carries a timeout and a cancellation path tied to the caller's lifetime
- Loading and error are modelled explicitly on every screen; empty is modelled on every screen whose data can be a collection or an absent record. Where no empty is reachable (a screen over fixed fields with defaults), state `empty: not reachable` with the reason instead of inventing one. None of the three is inferred from null or an empty list alone
- Data-layer errors are mapped to typed domain failures at the repository boundary; raw transport or storage exceptions never reach a widget
- Tokens and credentials go to secure storage, never `shared_preferences`
- No hardcoded user-facing strings - all display text goes through localization
- Generated files are never hand-edited; regenerate instead
- On-device schema changes ship with a migration, because old app versions stay installed
- Each step completes before the next; design approved before code

## Workflow

### STEP 1 - DETECT AND GATHER

Read `pubspec.yaml` to confirm Flutter and record state management, navigation, networking, and persistence packages, plus the platform target directories present. If `pubspec.yaml` is absent or declares no `flutter` dependency, stop and say so - this workflow implements Flutter features and has nothing to fall back on.

If the detected state management is not Riverpod, say so explicitly (`Detected Bloc; Riverpod-specific guidance does not apply`) and fall back to state-management-agnostic design. Load `flutter-riverpod-patterns` in fallback mode rather than skipping it. Do not convert the project.

The fallback is a vocabulary substitution, not a step skip. Every later mention of these reads as the detected library's equivalent:

| Riverpod term | Reads as |
|---------------|----------|
| provider, state holder | that library's state holder (`Bloc` / `Cubit` / `ChangeNotifier`) |
| `ref` | its own read handle (`context.read`, an injected instance) |
| provider override, test seam | its own injection point (`BlocProvider.value`, a constructor argument) |
| `autoDispose` / `keepAlive` | its own lifetime vocabulary |
| `AsyncValue` | its own single loading/error/data value |

Keep each delegated skill's output columns; substitute only the values.

Ask before writing code, grouped so each cluster surfaces its own follow-ups. Skip clusters the feature does not touch:

**Feature**
1. Screen(s) and the primary user flow through them
2. Entry points: tab, push from an existing screen, deep link, notification

**Data**
3. Remote API contract (endpoints, request/response shapes, error shapes)
4. Entities, fields, and validation rules
5. What must persist on device, and whether the local schema changes

**Behaviour**
6. Offline expectations: must it work offline, read-only offline, or is online required
7. State ownership and side effects (what triggers loads, what invalidates them)
8. Auth: which parts require a signed-in user

**Reach**
9. Which platform tiers ship this feature (mobile only, plus desktop, plus web)
10. Accessibility and localization expectations beyond the defaults

Ask targeted questions for gaps rather than guessing. When answers do not come back, carry each open question into STEP 2 as a stated assumption with the alternative named, so the approval gate is where they get settled - never leave one silently resolved in code.

Record the oldest app version still installed when the feature touches the on-device schema; STEP 4's migration contract needs it.

### STEP 2 - DESIGN (APPROVAL GATE)

Use skill: `flutter-riverpod-patterns` for the provider graph. Use skill: `flutter-data-persistence` for store selection. Use skill: `flutter-navigation-patterns` for the route and deep-link table. Use skill: `flutter-error-handling` for the failure-to-UI-state mapping. Use skill: `flutter-adaptive-responsive` when more than one platform tier ships.

Present file tree and decisions:

- Widget/screen tree, and which subtrees are invariant enough to be `const` once written
- Provider graph: each state holder, its kind, its dependencies, and its disposal scope
- Data layer: models, repository interface, remote source, local source
- Failure types and how each maps to a UI state
- Routes and deep-link entries
- Loading, error, and empty presentation per screen
- Adaptive layout plan per target tier
- Accessibility and localization intent

When the design deviates from this skill's defaults (a different store, a state holder that outlives its screen, a route outside the existing shell), call out the deviation with its reason so the approver sees the choice rather than discovering it in review.

Wait for approval.

### STEP 3 - MODELS

Use skill: `dart-language-patterns`. When the project uses code generation, freezed + json_serializable for data models; when it does not, hand-write the equivalent (`fromJson`/`toJson`, `copyWith`, `==`/`hashCode`) rather than adding a codegen toolchain the project does not have. Sealed failure types for the error model either way - Dart 3 `sealed` needs no generation. Keep serialization concerns on data-layer models; domain entities stay free of transport shapes.

### STEP 4 - DATA LAYER

Use skill: `flutter-networking` for the remote source. Use skill: `flutter-data-persistence` for the local source. Use skill: `flutter-error-handling` for the mapping.

Repository interface in the domain layer, implementation in data. Remote source configures timeouts and accepts a `CancelToken`. Local source owns the on-device store. The repository is the only place transport and storage errors become domain failures.

When the on-device schema changes: Use skill: `flutter-local-db-migration`. An installed older version of the app will read data written by this one, and vice versa - keep new fields optional and tolerate unknown ones on read.

### STEP 5 - STATE

Use skill: `flutter-riverpod-patterns`. State holders expose loading, error, and data as one value rather than parallel booleans. Side effects live in the holder's methods, not in `build`. Dependencies arrive by provider reference so tests can override them. Anything scoped to a single screen disposes with it.

### STEP 6 - UI

Use skill: `flutter-widget-patterns`. Use skill: `flutter-adaptive-responsive` when more than one platform tier ships (same condition as STEP 2; mobile-only features still respect safe areas and text scaling via `flutter-widget-patterns` and `flutter-accessibility`). Use skill: `flutter-accessibility` for labels, touch targets, focus order, and text scaling. Use skill: `flutter-i18n` for all user-facing strings.

Screens and widgets composed small and `const` where possible. Keys on list items whose identity matters. Routes wired per `flutter-navigation-patterns`. Every screen renders loading, error, and its reachable empty state, not just the happy path.

When the feature adds a platform capability - biometrics, permissions, background work, a native SDK - Use skill: `flutter-platform-channels` for the channel boundary and Use skill: `flutter-security-patterns` when the capability guards or stores anything sensitive. Native-side configuration (`AndroidManifest.xml`, `Info.plist`, entitlements, activity base class) ships with the Dart code in the same pass; a feature whose native side is unconfigured does not run.

### STEP 7 - TESTS

Use skill: `flutter-testing-patterns`. Unit tests for domain logic and failure mapping. Widget tests for each screen including its loading, error, and empty states. Golden tests for the key UI, with fonts and tolerance pinned so they survive CI. One `integration_test` for the critical flow. Fakes injected through the project's own dependency seam; the network stubbed at the client boundary, and any platform channel faked at the binary messenger.

### STEP 8 - VALIDATE

Run in order, fixing failures before reporting done:

1. `dart run build_runner build --delete-conflicting-outputs` (only if the project uses code generation)
2. `flutter analyze`
3. `dart format --set-exit-if-changed .`
4. `flutter test`
5. `flutter build <target>` once per shipped platform tier - a tier that ships unbuilt is unvalidated, and web and desktop fail for reasons mobile never surfaces

If a command is unavailable in the environment, say which one and why rather than reporting a clean run.

## Edge Cases

- Vague input: ask in STEP 1; never guess
- No `pubspec.yaml`, or no `flutter` dependency: stop at STEP 1; do not produce generic advice
- No persistence: skip the local source in STEP 4
- No remote API (fully local feature): skip the remote source; the repository still maps storage errors to failures
- Existing screen being extended: read it first and match its existing state and layout conventions
- Non-Riverpod project: state holders, dependency handles, and test seams read as the detected library's equivalents throughout; no step is skipped
- No code generation in the project: skip the `build_runner` step and hand-write the models
- Web tier included: flag `dart:io` and platform-channel unavailability at STEP 2, and confirm the on-device store has a web story (drift needs a WASM setup) before the schema is designed
- Deep-link entry point: validate link parameters before use; a custom scheme carries no ownership claim and does not resolve in a browser, so a web tier needs a verified `https://` link
- Feature triggered by app lifecycle (resume, background, app-switcher): name the `AppLifecycleState` transitions that drive it at STEP 2, and inject the clock so any time threshold is testable

## Output Format

Atomic skills invoked in STEP 2 and STEP 4 emit their own structured blocks (provider graph, store-selection table, migration contract, adaptive plan, route table). Those blocks are the content of the sections below - reproduce each under its heading rather than restating it in prose or dropping it.

```markdown
## Files Generated
[grouped by layer: models / data / state / ui / navigation / tests; mark each created or modified]

## Screens and Routes
[the route table from `flutter-navigation-patterns`, plus entry point and auth per screen]

## State Holders
[the provider graph from `flutter-riverpod-patterns`, in that skill's columns; in fallback mode, its fallback-mode terms]

## Data and Storage
[the store-selection table from `flutter-data-persistence`; and when the on-device schema changed, the migration contract block from `flutter-local-db-migration`]

## Platform Tiers
| Tier | Shipped | Caveats applied |

[the adaptive plan from `flutter-adaptive-responsive` when more than one tier ships]

## UI States
| Screen | Loading | Error | Empty |

[`empty: not reachable` with its reason where no empty state exists]

## Failure Mapping
| Failure | UI state |

[every sealed failure the repository can produce, and what the user sees]

## Tests
- Unit: {count}
- Widget: {count}
- Golden: {count}
- Integration: {count}
- Migration: {count, when the schema changed}

## Validation
[command -> result for each STEP 8 command; name any command that could not run and why]
```

## Self-Check

- [ ] `behavioral-principles` loaded before the workflow ran
- [ ] `pubspec.yaml` confirms Flutter; non-Riverpod state management surfaced rather than converted
- [ ] Requirements gathered; design approved before code
- [ ] Deviations from the skill's defaults called out at the approval gate
- [ ] Each delegated skill's block reproduced under its Output Format heading
- [ ] Models typed; failures sealed and exhaustively handled; hand-written where the project has no codegen
- [ ] Repository maps every transport and storage error to a domain failure
- [ ] Network calls carry timeouts and a cancellation path
- [ ] Local schema change ships a migration and stays backward compatible
- [ ] State holders own side effects; nothing I/O-bound in `build`
- [ ] Every screen renders loading, error, and its reachable empty state; unreachable empties recorded with a reason
- [ ] Adaptive layout applied for each shipped platform tier
- [ ] Platform capability added: channel boundary, native configuration, and security review covered
- [ ] Accessibility labels present; no hardcoded user-facing strings
- [ ] Tests at all four layers, plus migration tests when the schema changed; fakes injected through the project's own dependency seam
- [ ] `build_runner`, `analyze`, `format`, `test`, and a build for every shipped tier all pass

## Avoid

- Business logic, I/O, or navigation decisions inside `build`
- Raw transport or storage exceptions surfacing in the widget layer
- Parallel `isLoading` / `error` / `data` booleans instead of one state value
- Tokens or credentials in `shared_preferences`
- Hardcoded user-facing strings
- Hand-editing generated files
- Non-`const` widgets that could be `const`
- Unbounded list rendering where the item count is dynamic
- Changing the on-device schema without a migration
- Using a deep-link parameter without validating it
- Generating code before design approval
- Reporting done without running the STEP 8 commands
