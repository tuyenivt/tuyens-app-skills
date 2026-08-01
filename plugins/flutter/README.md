# Tuyen's Agent Skills - Flutter / Dart

Claude Code plugin for Flutter/Dart client development.

This is the marketplace's first client plugin. Every other stack plugin is server-side, so these skills are authored against the mobile-client domain rather than adapted from a backend plugin - database transactions, connection pools, and server middleware do not map to a device.

## Stack

- Dart 3.x (sound null safety, records, patterns, sealed classes, class modifiers)
- Flutter (stable channel)
- State management: **Riverpod** (Bloc / Provider / GetX are detected, and guidance degrades gracefully)
- Navigation: go_router
- Networking: Dio
- Local persistence: Drift (SQL), plus sqflite / Isar / Hive / `shared_preferences` / secure storage
- Models: freezed + json_serializable
- Testing: `flutter_test`, `integration_test`, golden tests, mocktail
- Codegen: `build_runner`

## Platform Tiers

Guidance degrades predictably rather than exploding into a full platform matrix. `flutter-adaptive-responsive` owns this convention; platform-sensitive skills reference it.

| Tier | Platforms | Support level |
| ---- | --------- | ------------- |
| **Primary** | Android, iOS | Full depth. Assumed unless stated otherwise. |
| **Secondary** | Windows, macOS, Linux | Covered as caveats: window sizing, pointer and keyboard input, focus traversal, native file dialogs, packaging. |
| **Tertiary** | Web | Covered as constraints: no `dart:io`, no platform channels, URL strategy, deferred loading, bundle size, no secure-storage guarantee. |

## Agents

| Agent                             | Description                                                                                                                                              |
| --------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `flutter-engineer`                | Dart 3.x engineer - builds features end-to-end: models, repository, Riverpod state, widgets, routes, tests. Also triages widget exceptions, codegen failures, and platform-channel errors. |
| `flutter-tech-lead`               | Flutter tech lead for code review, refactoring guidance, and widget-layer standards. Reviews for idiomatic Dart, state discipline, and composition.       |
| `flutter-performance-engineer`    | Frame budget and jank, rebuild scoping, list virtualization, image cache, isolate offloading, startup time, app binary size, memory leaks.                |
| `flutter-security-engineer`       | MASVS lens: secure storage, certificate pinning, obfuscation, deep-link and platform-channel input validation, secrets, WebView, biometric.               |
| `flutter-test-engineer`           | Unit, widget, golden, and `integration_test` strategy; mocktail; Riverpod `ProviderContainer` overrides; golden stability in CI.                          |

## Workflow Skills

| Skill                                 | Agent                            | Description                                                                                                                                       |
| ------------------------------------- | -------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| `task-flutter-implement`              | `flutter-engineer`               | End-to-end feature: freezed models, repository over Dio and a local store, Riverpod state, adaptive screens, go_router wiring, and tests across four layers. |
| `task-flutter-review`                 | `flutter-tech-lead`              | Staff-level code review umbrella - Phases A-E with Dart and widget idioms; spawns perf and security subagents in parallel. |
| `task-flutter-review-perf`            | `flutter-performance-engineer`   | Performance review for jank and frame budget, rebuild scoping, `const`, list virtualization, image cache, isolates, startup, app size, memory leaks. |
| `task-flutter-review-security`        | `flutter-security-engineer`      | Security review for secure storage, certificate pinning, obfuscation, deep-link and platform-channel input, secrets, WebView, biometric, MASVS lens. |
| `task-flutter-test`                   | `flutter-test-engineer`          | Test strategy, scaffolding, and CI-failure diagnosis across unit, widget, golden, and `integration_test` layers, following the project's own mocking library and test seam. |

> Flutter does not review API contract design - a client consumes API contracts rather than designing them. The concern that does reach clients - an installed old app version must survive a server contract change - is checked inside `task-flutter-review` and `task-flutter-implement`.

> Adaptivity, accessibility, and localization have no dedicated review lens. They are designed in `task-flutter-implement` and checked at baseline depth in `task-flutter-review` Phase E.

## Atomic Skills

| Skill                          | Description                                                                                                                                        |
| ------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| `dart-language-patterns`       | Dart 3.x idioms: sound null safety, records, pattern matching and exhaustive `switch`, sealed classes, class modifiers, extension types, async, isolates. |
| `flutter-widget-patterns`      | Stateless vs Stateful, `const` and rebuild cost, key semantics, `BuildContext` across async gaps, lifecycle, constraint-based layout, builder scoping. |
| `flutter-riverpod-patterns`    | Providers, `Notifier`/`AsyncNotifier`, `AsyncValue`, `ref` lifecycle, `family`, `autoDispose`, side effects, DI and test overrides. Riverpod-only, degrades gracefully. |
| `flutter-navigation-patterns`  | go_router routes, nested and shell routes, redirects and guards, typed routes, deep links as untrusted input, web URL strategy.                    |
| `flutter-networking`           | Dio interceptors, timeouts on every call, `CancelToken`, error mapping to typed failures, auth-token refresh and the concurrent-401 stampede, caching. |
| `flutter-data-persistence`     | Choosing the right on-device store (Drift, sqflite, Isar, Hive, `shared_preferences`, secure storage), repository pattern, local transactions.     |
| `flutter-local-db-migration`   | On-device schema migration: versioning, composing steps across skipped app versions, data preservation, testing against real old-version databases. |
| `flutter-adaptive-responsive`  | Platform tier convention, Material vs Cupertino, responsive breakpoints, safe areas, input modalities, foldables, desktop windowing, web layout.   |
| `flutter-accessibility`        | `Semantics` and labelling, screen readers, contrast, touch-target size, text scaling, focus traversal, dynamic state announcements.                |
| `flutter-i18n`                 | `gen_l10n` and ARB, no hardcoded user-facing strings, pluralization, locale formatting, RTL, and text expansion breaking fixed-width layouts.       |
| `flutter-performance`          | Frame budget, `const` and rebuild scoping, `RepaintBoundary`, list virtualization, image cache and decode, isolates, startup, app size, leak detection. |
| `flutter-security-patterns`    | Secure storage, certificate pinning, obfuscation and its limits, deep-link and platform-channel arguments as untrusted input, secrets, WebView, biometric. |
| `flutter-error-handling`       | Sealed failure hierarchies, Result vs throwing, mapping data-layer errors to domain failures and domain failures to UI states, no silent swallows.  |
| `flutter-testing-patterns`     | Unit, widget, golden, and integration layers; pump vs settle; finders; golden stability (fonts, tolerance, per-platform); mocktail; provider overrides. |
| `flutter-platform-channels`    | MethodChannel and EventChannel, Pigeon, FFI, plugin development, permissions, platform-tier gaps, and channel arguments as untrusted in both directions. |
| `flutter-build-release`        | Flavors, `--dart-define` and why it is not a secret mechanism, Android and iOS signing, app size reduction, obfuscation with retained symbols, CI, desktop and web packaging. |
| `flutter-overengineering-review` | Necessity review: `StatefulWidget` without state, single-field notifiers, single-impl repository interfaces, blanket freezed, gratuitous providers, dead null checks, `dynamic` escapes. Composed into `task-flutter-review` Phase D. |

## Usage Examples

### Implement a feature end-to-end

```
> task-flutter-implement

Feature: Add an order history screen with offline support
- freezed models + sealed failure types
- Repository over Dio (remote) and Drift (local cache)
- AsyncNotifier exposing loading / error / empty / data
- Adaptive list-to-detail layout for mobile and tablet
- go_router route + deep link entry
- Unit, widget, golden, and integration tests

→ Validates with build_runner, flutter analyze, dart format, flutter test, flutter build
```

### Review a PR with automatic scope escalation

```
> task-flutter-review

- Phases A-E: risk, correctness, architecture, AI-quality, maintainability
- Detects a network call with no timeout -> auto-escalates +Rel
- Detects a token written to shared_preferences -> auto-escalates +Sec
- Spawns the two lens subagents in parallel, merges findings at strongest intent
- Excludes *.g.dart / *.freezed.dart from findings and cites the source instead
```
