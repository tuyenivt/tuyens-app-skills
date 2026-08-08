# Tuyen's Agent Skills - C# Desktop

Claude Code plugin for native desktop utility development in C# and Avalonia.

This is the marketplace's third client plugin, alongside `flutter` and `unity`. Like both, it is authored against the client domain rather than adapted from a backend plugin - transactions, connection pools, and server middleware do not map to a tool running on the user's own machine. Unlike both, its subject is **local-first utilities**: the app has no backend, and its risk lives in the filesystem operations it performs rather than in anything it sends over a network.

## Target Applications

Local desktop utilities: bulk file rename, image deduplication with thumbnail preview, batch conversion, tree comparison, and comparable tools whose cost is filesystem traversal, hashing, and image decode rather than network I/O.

The workload is I/O-bound first and CPU-bound second, and it ends in **destructive filesystem operations**. That single fact shapes the plugin: dry-run preview, undo, and collision handling are core concerns rather than nice extras.

## The Central Rule

**File-utility logic lives in a core class library with no Avalonia `PackageReference`**, enforced by the project file rather than by discipline.

Traversal, hashing, dedup grouping, rename planning, collision resolution, and undo all run without a window. This makes them unit-testable in milliseconds instead of requiring a UI harness, and it is what makes the destructive paths - the ones that lose user data when wrong - cheap to cover exhaustively. `desktop-core-architecture` owns the rule; most other skills reference it.

```
src/
  MyApp.Core/    walk, hash, dedup, rename planning, collision resolution, undo
                 no Avalonia reference - unit-testable without a window
  MyApp/         Avalonia UI, ViewModels, views
```

A CI check that fails when Avalonia appears in the core's dependency tree makes the rule mechanical.

## Stack

- **.NET 10** (LTS, supported to November 2028)
- **Avalonia 12.1.x** (MIT) - own compositor on Skia, real UI Automation support
- **CommunityToolkit.Mvvm** for MVVM; ReactiveUI respected where a project already uses it
- **Microsoft.Data.Sqlite** for persistence and scan caches
- **SkiaSharp** for image decode (ships with Avalonia; native codecs)
- **System.IO.Hashing** (XxHash3) - SIMD-accelerated content hashing
- `Parallel.ForEach`, `System.Threading.Channels`, `IProgress<T>`, `CancellationToken`
- **NativeAOT-compatible by design**: compiled bindings and source generators, no reflection-dependent code
- xUnit with temp-directory fixtures; Velopack or WiX for packaging

### Platform tiers

| Tier | Platforms | Support level |
| ---- | --------- | ------------- |
| **Primary** | Windows | Full depth. Assumed unless stated otherwise. |
| **Secondary** | macOS | Caveats: Unicode normalization, signing and notarization, per-platform paths. |
| Not targeted | Linux | Out of scope by design. |

### Two facts worth knowing before you start

**Some desktop capabilities are hard blocks, and not all of them are .NET's fault.** File associations are impossible by design on Windows 10+ for every stack (`UserChoice` is hash-protected and guarded by `UCPD.sys`); cross-platform printing has no good story; Finder Sync and Quick Look need Xcode. `desktop-ecosystem-boundaries` is the register of what is a Gap, what silently no-ops, and which escape hatch to use instead. `/task-desktop-implement` loads it before any design work so an impossible requirement is caught at design time.

**Avalonia's free tier is complete for a commercial closed-source app**, at $0, with the full framework and its built-in controls. What is paid is tooling and premium components - the current DevTools and IDE extensions, and a charts/rich-text control suite. The Community tier is restricted to non-commercial use, so a commercial project uses Free or a paid tier. Worth knowing: free developer experience has moved behind paywalls twice in 18 months.

## Decisions

Why this stack, in short. These are settled - the skills encode them as constraints, not choices to re-open.

| Decision | Why |
| --- | --- |
| **C# on .NET 10** | Nearest mainstream language to a Java background, which matters because development is AI-assisted and the maintainer reviews the output. Shared-memory parallelism without an FFI boundary. |
| **Avalonia** over Iced (Rust) | Iced has no screen-reader support and cannot get it downstream; Avalonia ships UI Automation on Windows and NSAccessibility on macOS. Iced also has a bus factor near one, with foundational issues open five-plus years. |
| **UI-free core** | The destructive paths must be exhaustively testable without a window. Also makes any future UI change a UI-layer change. |
| **NativeAOT-compatible by design** | Startup is the most visible latency on a frequently launched utility. Compiled bindings and source generators cost nothing and keep the option open. |
| **OS media APIs before FFmpeg** | Media Foundation and AVFoundation are free, hardware-accelerated, and already patent-licensed by Microsoft and Apple - sidestepping both the FFmpeg LGPL/GPL trap and the H.264 patent question. FFmpeg is the fallback for format coverage, LGPL and dynamically linked. |
| **Closed-source, perpetual licence + free tier** | 12-month update window; the bought version runs forever. Private repo, so macOS CI bills at 10x and signing is mandatory. Licence checks are offline - there is no backend. |

Rejected: **Rust + Iced** (no accessibility, single maintainer, and the performance edge does not materialize on an I/O-bound workload), **Compose Multiplatform** (Windows accessibility runs through Java Access Bridge, disabled by default on the primary platform), **JavaFX** (direct UI Automation but shrinking ecosystem and known screen-reader bugs), **Tauri** (IPC on the hot path, second language to review), **Electron** (fails the performance bar outright), **MAUI** (Mac Catalyst is an iPad shim and needs a Mac to build).

## Agents

| Agent | Description |
| ----- | ----------- |
| `desktop-engineer` | Builds features end-to-end: UI-free core, Avalonia MVVM wiring, preview and undo, persistence, platform integration, tests. Also triages unhandled exceptions, path bugs, frozen UI threads, and binding failures. |
| `desktop-tech-lead` | Holistic quality gate: staff-level review, UI-free core discipline, refactoring direction, idiomatic C# enforcement across changes. |
| `desktop-performance-engineer` | Throughput and responsiveness: scan, hash, and decode cost, UI-thread blocking, GC pressure, caching, startup latency. |
| `desktop-security-engineer` | Local-first threat model: path traversal, symlink and junction escape, TOCTOU on destructive operations, P/Invoke boundaries, dependency advisories, update integrity. |
| `desktop-test-engineer` | Test strategy: core-library unit tests, filesystem fixtures, destructive-operation and migration coverage, cross-platform CI matrix. |

## Workflow Skills

Invoked as slash commands.

| Skill | Purpose |
| ----- | ------- |
| `/task-desktop-implement` | End-to-end feature: UI-free core, MVVM wiring, preview and undo, persistence, platform integration, tests. |
| `/task-desktop-review` | Staff-level umbrella review of the working tree. Auto-escalates into parallel perf and security lenses. |
| `/task-desktop-review-perf` | Throughput and UI-responsiveness lens. |
| `/task-desktop-review-security` | Local-first security lens. |
| `/task-desktop-test` | Test strategy, coverage assessment, and scaffolds. |

Reviews read the **working tree** by default, so uncommitted work is the subject. Pass `--staged` for staged changes only; on a clean tree the review falls back to the last commit.

## Atomic Skills

Composed automatically by the workflows. Not invocable directly.

| Skill | Purpose |
| ----- | ------- |
| `behavioral-principles` | Behavioral guardrails loaded as Step 1 of every workflow. |
| `review-precondition-check` | Resolves the working-tree change set and gates review workflows. |
| `csharp-language-patterns` | Nullable reference types, records and structs, spans, LINQ cost, disposal, and AI-generated C# smells. |
| `csharp-error-handling` | Exceptions vs result types at the core boundary, per-item batch outcomes, cancellation as control flow, filesystem exception handling. |
| `csharp-async-patterns` | async/await, never `async void`, cancellation, `IProgress<T>`, bounded channels, dispatcher marshalling, progress coalescing. |
| `avalonia-mvvm-patterns` | CommunityToolkit source generators, where state lives, commands, DI, dialogs without Window references. |
| `avalonia-control-patterns` | XAML, virtualization for 100k-row results, templates, styles and themes, compiled bindings, focus order, empty and error states. |
| `desktop-core-architecture` | The UI-free core rule: what lives in core vs UI, injection seams, the plan-and-apply shape, the brownfield recipe. |
| `desktop-batch-operations` | Dry-run preview, undo journals, intra-batch and case-insensitive collisions, rename chains, auto-suffix naming, per-item outcomes. |
| `desktop-filesystem-patterns` | Enumeration, Windows reserved names and long paths, macOS normalization, symlinks and junctions, atomic writes. |
| `desktop-image-processing` | SkiaSharp scaled decode, bounded thumbnail cache, the dedup funnel, EXIF orientation, off-thread decode. |
| `desktop-concurrency-patterns` | Parallelism that helps vs hurts, bounded channels, cooperative cancellation, progress coalescing, determinism. |
| `desktop-data-persistence` | SQLite with `user_version` schema versioning, forward migrations with a fixture per shipped version, per-platform paths. |
| `desktop-performance` | Two-tier hashing, GC and allocation discipline, SIMD, startup cost, evidence grading. |
| `desktop-security-patterns` | Local-first threat model, path confinement, symlink and hardlink hazards, TOCTOU, untrusted file parsing, control types. |
| `desktop-platform-integration` | Storage provider dialogs, drag-and-drop, tray, notifications, file watching, clipboard, credential storage. |
| `desktop-accessibility` | Automation properties and peers, keyboard navigation, focus order, contrast, text scaling, and the gaps that need a real screen-reader test. |
| `desktop-i18n` | Resource-based localization, culture switching, filename normalization divergence, locale-aware and natural sorting. |
| `desktop-build-release` | Publish modes and NativeAOT, signing and notarization, licence enforcement, installers, auto-update, CI matrix. |
| `desktop-media-processing` | OS media APIs first, FFmpeg as the fallback with LGPL discipline, audio decode vs playback vs analysis. |
| `desktop-ecosystem-boundaries` | The gap and trap register: hard blocks, silent-failure traps, the Avalonia tier boundary, and the escape hatch for each. |
| `desktop-overengineering-review` | Necessity review for C# and Avalonia abstractions - and the floor case, where structure is absent rather than excessive. |

## Notes

- This plugin is self-contained. It references nothing from `flutter` or `unity`, and exactly one plugin is installed per project.
- `behavioral-principles` and `review-precondition-check` are duplicated verbatim from the sibling plugins by design - there is no shared plugin, so each carries its own copy.
- There is no networking, API-contract, or auth skill surface: these apps have no backend.
- Review lenses are perf and security only. Accessibility, adaptivity, and localization are implementation concerns checked at baseline depth inside the umbrella review.
