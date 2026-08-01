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
- Loading, error, and empty are modelled explicitly - never inferred from null or an empty list alone
- Data-layer errors are mapped to typed domain failures at the repository boundary; raw transport or storage exceptions never reach a widget
- Tokens and credentials go to secure storage, never `shared_preferences`
- No hardcoded user-facing strings - all display text goes through localization
- Generated files are never hand-edited; regenerate instead
- On-device schema changes ship with a migration, because old app versions stay installed
- Each step completes before the next; design approved before code

## Workflow

### STEP 1 - DETECT AND GATHER

Read `pubspec.yaml` to confirm Flutter and record state management, navigation, networking, and persistence packages, plus the platform target directories present.

If the detected state management is not Riverpod, say so explicitly (`Detected Bloc; Riverpod-specific guidance does not apply`) and fall back to state-management-agnostic design for STEP 5. Do not convert the project.

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

Ask targeted questions for gaps. Do not guess.

### STEP 2 - DESIGN (APPROVAL GATE)

Use skill: `flutter-riverpod-patterns` for the provider graph. Use skill: `flutter-data-persistence` for store selection. Use skill: `flutter-adaptive-responsive` when more than one platform tier ships.

Present file tree and decisions:

- Widget/screen tree and which widgets are `const`
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

Use skill: `dart-language-patterns`. freezed + json_serializable for data models. Sealed failure types for the error model. Keep serialization concerns on data-layer models; domain entities stay free of transport shapes.

### STEP 4 - DATA LAYER

Use skill: `flutter-networking` for the remote source. Use skill: `flutter-data-persistence` for the local source. Use skill: `flutter-error-handling` for the mapping.

Repository interface in the domain layer, implementation in data. Remote source configures timeouts and accepts a `CancelToken`. Local source owns the on-device store. The repository is the only place transport and storage errors become domain failures.

When the on-device schema changes: Use skill: `flutter-local-db-migration`. An installed older version of the app will read data written by this one, and vice versa - keep new fields optional and tolerate unknown ones on read.

### STEP 5 - STATE

Use skill: `flutter-riverpod-patterns`. State holders expose loading, error, and data as one value rather than parallel booleans. Side effects live in the holder's methods, not in `build`. Dependencies arrive by provider reference so tests can override them. Anything scoped to a single screen disposes with it.

### STEP 6 - UI

Use skill: `flutter-widget-patterns`. Use skill: `flutter-adaptive-responsive` when more than one platform tier ships (same condition as STEP 2; mobile-only features still respect safe areas and text scaling via `flutter-widget-patterns` and `flutter-accessibility`). Use skill: `flutter-accessibility` for labels, touch targets, focus order, and text scaling. Use skill: `flutter-i18n` for all user-facing strings.

Screens and widgets composed small and `const` where possible. Keys on list items whose identity matters. Routes wired per `flutter-navigation-patterns`. Every screen renders its loading, error, and empty states, not just the happy path.

### STEP 7 - TESTS

Use skill: `flutter-testing-patterns`. Unit tests for domain logic and failure mapping. Widget tests for each screen including its loading, error, and empty states. Golden tests for the key UI, with fonts and tolerance pinned so they survive CI. One `integration_test` for the critical flow. Fakes injected through provider overrides; the network stubbed at the client boundary.

### STEP 8 - VALIDATE

Run in order, fixing failures before reporting done:

1. `dart run build_runner build --delete-conflicting-outputs` (only if the project uses code generation)
2. `flutter analyze`
3. `dart format --set-exit-if-changed .`
4. `flutter test`
5. `flutter build <target>` for at least the primary platform tier

If a command is unavailable in the environment, say which one and why rather than reporting a clean run.

## Edge Cases

- Vague input: ask in STEP 1; never guess
- No persistence: skip the local source in STEP 4
- No remote API (fully local feature): skip the remote source; the repository still maps storage errors to failures
- Existing screen being extended: read it first and match its existing state and layout conventions
- Non-Riverpod project: STEP 5 falls back to the project's own state management; the rest of the workflow is unchanged
- No code generation in the project: skip the `build_runner` step and write the models by hand
- Web tier included: flag `dart:io` and platform-channel unavailability at STEP 2, not after the code is written
- Deep-link entry point: treat link parameters as untrusted input and validate before use

## Output Format

```markdown
## Files Generated
[grouped by layer: models / data / state / ui / navigation / tests]

## Screens and Routes
| Screen | Route | Entry point | Auth |

## State Holders
| Provider | Kind | Depends on | Disposal |

## Platform Tiers
| Tier | Shipped | Caveats applied |

## Tests
- Unit: {count}
- Widget: {count}
- Golden: {count}
- Integration: {count}

## Validation
[command -> result for each STEP 8 command]
```

## Self-Check

- [ ] `behavioral-principles` loaded before the workflow ran
- [ ] Stack detected; non-Riverpod state management surfaced rather than converted
- [ ] Requirements gathered; design approved before code
- [ ] Deviations from the skill's defaults called out at the approval gate
- [ ] Models typed; failures sealed and exhaustively handled
- [ ] Repository maps every transport and storage error to a domain failure
- [ ] Network calls carry timeouts and a cancellation path
- [ ] Local schema change ships a migration and stays backward compatible
- [ ] State holders own side effects; nothing I/O-bound in `build`
- [ ] Every screen renders loading, error, and empty states
- [ ] Adaptive layout applied for each shipped platform tier
- [ ] Accessibility labels present; no hardcoded user-facing strings
- [ ] Tests at all four layers; fakes injected via provider overrides
- [ ] `build_runner`, `analyze`, `format`, `test`, and a build for the primary tier all pass

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
