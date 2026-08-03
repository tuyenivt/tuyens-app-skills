# Tuyen's Agent Skills - Rust Desktop

Claude Code plugin for native-performance desktop utility development in Rust.

This is the marketplace's third client plugin, alongside `flutter` and `unity`. Like both, it is authored against the client domain rather than adapted from a backend plugin - transactions, connection pools, and server middleware do not map to a tool running on the user's own machine. Unlike both, its subject is **local-first utilities**: the app has no backend, and its risk lives in the filesystem operations it performs rather than in anything it sends over a network.

## Target Applications

Native-performance local utilities: bulk file rename, image deduplication with thumbnail preview, batch conversion, tree comparison, and comparable tools whose cost is filesystem traversal, hashing, and image decode rather than network I/O.

The workload is CPU-bound and I/O-bound, and it ends in **destructive filesystem operations**. That single fact shapes the plugin: dry-run preview, undo, and collision handling are core concerns rather than nice extras.

## The Central Rule

**File-utility logic is plain Rust with no `iced` dependency**, enforced by the core crate's `Cargo.toml` rather than by discipline.

Traversal, hashing, dedup grouping, rename planning, collision resolution, and undo all run without a window. This makes them unit-testable in milliseconds instead of requiring a GUI harness, and it is what makes the destructive paths - the ones that lose user data when wrong - cheap to cover exhaustively. `desktop-core-architecture` owns the rule; most other skills reference it.

```
crates/
  core/    walk, hash, dedup, rename planning, collision resolution, undo
           no iced, no wgpu, no winit - unit-testable without a window
  app/     Iced UI, wiring core operations to messages
```

`cargo tree -p <core> | grep iced` returning nothing is a CI-checkable invariant.

## Stack

- **Rust** 2024 edition
- **Iced 0.14.x** (MIT) - retained-mode, GPU-rendered, no attribution obligation
- `walkdir` / `jwalk` traversal, `rayon` parallel scan, `blake3` / `xxhash` hashing
- `image` 0.25 for decode; `wgpu` 29 compute via Iced's re-export for heavy pixel work
- `rusqlite` 0.40 (`bundled`) for persistence - **`sled` is unmaintained** and not used
- `rfd` dialogs, `notify` file watching, `arboard` clipboard, `tracing` logging
- `cargo-packager` for installers; `rcodesign` for macOS signing and notarization
- Built-in test harness, `tempfile` fixtures, `proptest` for invariants

### Platform tiers

| Tier | Platforms | Support level |
| ---- | --------- | ------------- |
| **Primary** | Windows | Full depth. Assumed unless stated otherwise. |
| **Secondary** | macOS | Caveats: Unicode NFD filename normalization, signing and notarization, per-platform paths. |
| Not targeted | Linux | Out of scope by design. |

### Two facts worth knowing before you start

**Iced has no accessibility support.** Issue [#552](https://github.com/iced-rs/iced/issues/552) has been open since October 2020, and Iced does not consume AccessKit (egui and Slint do). This is a stack-level exclusion that cannot be fixed downstream. The plugin covers what remains achievable - keyboard navigation, focus order, contrast, text scaling - and never implies screen-reader support exists. If a project must serve government, education, healthcare, or a11y-procurement enterprise markets, this is the wrong stack.

**Some desktop capabilities are hard blocks.** Printing is unsolved in Rust; file associations are impossible by design on Windows 10+ for every stack; shell extensions need COM or Xcode. `desktop-ecosystem-boundaries` is the register of what is a Gap, what silently no-ops, and which escape hatch to use instead. `/task-desktop-implement` loads it before any design work so an impossible requirement is caught at design time.

## Decisions

Why this stack, in short. These are settled - the skills encode them as constraints, not choices to re-open.

| Decision | Why |
| --- | --- |
| **Rust** | Fearless concurrency: the workload is parallel walk, hash, and decode, where a data race is a compile error rather than silent corruption. No GC pause on a large scan. |
| **Iced** over Slint | MIT with no attribution obligation. The Elm architecture makes dry-run preview and undo fall out of the design, which matters because the operations are destructive. |
| **Track latest**, not pinned | Accepted cost: the resolved version moves. `Cargo.lock` is the source of truth, and `cargo update` is a tested migration, not housekeeping. |
| **FFmpeg LGPL**, dynamically linked | `--enable-gpl` would make the whole app GPL. LGPL keeps the app's own licensing free. Cost: no x264/x265, so H.264/H.265 encode goes through hardware encoders. |
| **Closed-source, perpetual licence + free tier** | 12-month update window; the bought version runs forever. Private repo, so macOS CI bills at 10x and signing is mandatory. Licence checks are offline - there is no backend. |
| **Accessibility gap accepted** | Personal and general-consumer tools, not procurement markets. See above. |

Rejected: **Tauri** (JSON IPC on the hot path, plus a second language to review), **egui** (hand-rolled thumbnail caching, debug-UI look), **Flutter desktop** (isolates copy rather than share; FFI means a systems language anyway, plus Dart), **C# + Avalonia** (GC pauses on large scans; NativeAOT cannot cross-compile), **Qt** (memory-unsafe, dual-platform packaging upkeep), **Electron** (fails the performance bar outright).

## Agents

| Agent | Description |
| ----- | ----------- |
| `desktop-engineer` | Builds features end-to-end: GUI-free core, Iced wiring, preview and undo, persistence, platform integration, tests. Also triages panics, path bugs, hangs, and frozen windows. |
| `desktop-tech-lead` | Holistic quality gate: staff-level review, GUI-free core discipline, refactoring direction, idiomatic Rust enforcement across changes. |
| `desktop-performance-engineer` | Throughput and responsiveness: scan, hash, and decode cost, UI-thread blocking, allocation, caching, startup latency. |
| `desktop-security-engineer` | Local-first threat model: path traversal, symlink and junction escape, TOCTOU on destructive operations, `unsafe` and FFI, dependency advisories, update integrity. |
| `desktop-test-engineer` | Test strategy: core-crate unit tests, filesystem fixtures, destructive-operation and migration coverage, cross-platform CI matrix. |

## Workflow Skills

Invoked as slash commands.

| Skill | Purpose |
| ----- | ------- |
| `/task-desktop-implement` | End-to-end feature: GUI-free core, Iced wiring, preview and undo, persistence, platform integration, tests. |
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
| `rust-language-patterns` | Ownership, borrowing, when `clone()` signals a design problem, `unsafe` justification, AI-generated Rust smells. |
| `rust-error-handling` | `thiserror` at the core boundary vs `anyhow` in the app, propagation across threads, per-item batch failure, legitimate vs defective panics. |
| `iced-architecture-patterns` | Model-Message-Update-View, message design, where state belongs, why `update` never blocks. |
| `iced-widget-patterns` | Composition, virtualized lists for large result sets, the native 0.14 `table`, focus order, scanning/error/empty states. |
| `iced-async-patterns` | `Task` and subscriptions, streamed progress, cancellation, integrating `rayon` without a second runtime. |
| `desktop-core-architecture` | The GUI-free core rule: what lives in `core` vs `app`, injection seams, the plan-and-apply shape. |
| `desktop-batch-operations` | Dry-run preview, undo journals, intra-batch and case-insensitive collisions, auto-suffix naming, per-item outcomes. |
| `desktop-filesystem-patterns` | Traversal, non-UTF-8 paths, Windows reserved names and long paths, macOS NFD normalization, symlinks and junctions, atomic writes. |
| `desktop-image-processing` | Thumbnail-resolution decode, bounded cache eviction, exact vs perceptual comparison, EXIF orientation. |
| `desktop-concurrency-patterns` | `rayon` scan and when parallelism hurts, bounded pools, backpressure, cooperative cancellation, progress coalescing. |
| `desktop-data-persistence` | `rusqlite` with `user_version` schema versioning, forward migrations with a fixture per shipped version, per-platform directories. |
| `desktop-performance` | Two-tier hashing, hash selection, I/O ordering, allocation discipline, startup cost, evidence grading. |
| `desktop-security-patterns` | Local-first threat model, path confinement, symlink and hardlink hazards, TOCTOU, untrusted file parsing, control types. |
| `desktop-platform-integration` | Dialogs, drag-and-drop, tray, notifications, watching, clipboard, hotkeys, single-instance, credential storage. |
| `desktop-accessibility` | Keyboard reachability, focus order and indicators, contrast, text scaling, no colour-only meaning - stated against Iced's actual support. |
| `desktop-i18n` | Fluent catalogs, runtime locale switching, and the NFC/NFD filename divergence a rename tool hits directly. |
| `desktop-build-release` | Release profile hazards, `cargo-packager`, Windows signing, macOS notarization without a Mac, CI matrix, auto-update. |
| `desktop-gpu-compute` | wgpu compute for pixel work through Iced's re-export, staying GPU-side, the `tiny-skia` fallback. |
| `desktop-media-processing` | Audio via `symphonia`/`cpal`/`rodio`; video behind a trait boundary, with the FFmpeg LGPL-vs-GPL trap. |
| `desktop-ecosystem-boundaries` | The gap and trap register: hard blocks, silent-failure traps, and the escape hatch for each. |
| `desktop-overengineering-review` | Necessity review for Rust and Iced abstractions - and the floor case, where structure is absent rather than excessive. |

## Notes

- This plugin is self-contained. It references nothing from `flutter` or `unity`, and exactly one plugin is installed per project.
- `behavioral-principles` and `review-precondition-check` are duplicated verbatim from the sibling plugins by design - there is no shared plugin, so each carries its own copy.
- There is no networking, API-contract, or auth skill surface: these apps have no backend.
- Review lenses are perf and security only. Accessibility, adaptivity, and localization are implementation concerns checked at baseline depth inside the umbrella review.
