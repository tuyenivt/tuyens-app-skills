---
name: task-desktop-implement
description: End-to-end Rust desktop feature implementation - GUI-free core, Iced wiring, batch preview and undo, persistence, platform integration, tests.
agent: desktop-engineer
metadata:
  category: desktop
  tags: [rust, iced, desktop, file-utility, batch-operations, undo, feature, implementation, workflow]
  type: workflow
user-invocable: true
---

> **Behavioral directive:** Load `Use skill: behavioral-principles` before executing this workflow.

# Implement Rust Desktop Feature

## When to Use

End-to-end Rust desktop feature work: core logic + Iced wiring + persistence + platform integration + tests in one pass. STEPS 3-7 write the files; the report lists what was written. STEP 2 is the gate that decides what gets written, and nothing lands before it is approved.

Not for: a widget restyle, a string change, a dependency bump, or a `Cargo.toml` edit. A change that *only* restyles an existing view is a UI edit, not this workflow.

It does apply when presentation work also adds behaviour, locales, or persistence - the steps that produce nothing are written as skipped rather than being a reason to decline. A feature built entirely on existing core operations is an anticipated shape, not an out-of-scope one.

## Rules

- **Core logic is plain Rust with no `iced` dependency**, in a crate whose `Cargo.toml` cannot reference the GUI. Every other decision in this workflow follows from that one
- **A destructive operation ships its preview and its undo in the same change.** Rename, move, delete, and overwrite are destructive; a feature that applies one without a dry-run path is incomplete, not shippable-with-a-follow-up
- Long work runs off the UI thread and reports progress through a `Task`/subscription, never a blocking call inside `update`
- Filesystem paths are `Path`/`PathBuf` and `OsStr`, never `String` - a path is not required to be UTF-8 on either target
- Every fallible operation in a batch reports its own outcome; one failure does not abandon the remaining items or silently succeed
- User-facing strings resolve through a localization key. Where the project has no localization system, keep every string in one module so extraction is mechanical, and report that at STEP 5 as a stated deviation rather than installing a localization system this feature did not ask for
- A shape change to a versioned store ships a schema version bump and a migration, because installed users have existing databases. An additive settings-file field absorbed by `#[serde(default)]` needs neither - record it as absorbed in Persistence Impact
- A capability listed as a Gap in `desktop-ecosystem-boundaries` is designed around at STEP 2, not discovered at STEP 5
- Each step completes before the next; design approved before code

## Workflow

### STEP 1 - DETECT AND GATHER

**Project gate.** Read `Cargo.toml` and confirm an `iced` dependency, then read the workspace layout. When `Cargo.toml` is absent or has no `iced` dependency, state what was found and **STOP** - this is a detect-and-report boundary, not a degradation path. Do not emit Iced guidance for a project that does not use it. When the file is unreadable, say so and ask rather than assuming.

Read the **resolved** Iced version from `Cargo.lock` and state it. This project tracks latest rather than pinning a minor, so `Cargo.toml` holds a range and only the lockfile identifies what actually builds. **Every Iced API claim in this workflow is checked against that resolved version, not against model memory** - Iced is pre-1.0 and its API moves between minor releases.

When `Cargo.lock` is absent or unreadable, say so and ask rather than falling back to the manifest range - a range cannot answer whether an API exists.

When the resolved version is known but its API surface cannot be consulted (offline, no vendored sources or docs), prefer APIs the project's existing code already exercises; every API introduced beyond that set is named in Deviations as unverified against the resolved version, never silently trusted from memory.

Then confirm:

| Concern | Confirm | If it differs |
| --- | --- | --- |
| Core split | A crate with no `iced` in its `Cargo.toml` | **None anywhere, or the existing one depends on `iced`: report at STEP 2 as the blocking design finding.** Do not build logic into the UI crate around it |
| Iced version | Resolved from `Cargo.lock`, matching this plugin's guidance | State the resolved version; where it differs, verify each API against it and say that you did |
| Async runtime | Iced's executor, or an explicit `tokio`/`smol` choice | State which; do not introduce a second runtime |
| Persistence | `rusqlite`, or a stated alternative | `sled`: report at STEP 6 as a finding - it is unmaintained, 0.34.7 dates to 2021 |
| Platform targets | Windows primary, macOS secondary | A Linux-only or macOS-primary project: state the mismatch and which guidance still applies |
| Packaging | `cargo-packager`, or a stated alternative | **`cargo-bundle`: flag it, it is self-declared alpha.** None at all: report at STEP 6, since signing gates several runtime capabilities |

Ask before writing code, grouped so each cluster surfaces its own follow-ups. Skip clusters the feature does not touch:

**Feature**
1. What the user selects, and what the operation does to it
2. Entry point: new view, existing view, dialog, or background task

**Core**
3. What the core crate owns, and what a single operation produces
4. Whether the operation is destructive, and what its preview shows
5. Undo: whether it is reversible, what the reversal needs recorded, and whether undo must survive an app restart

**Data**
6. What persists, and whether the persisted shape changes
7. Scale: expected item count, and whether results stream or arrive whole

**Presentation**
8. Views, and what each renders while scanning or on failure
9. Progress: what the user sees during a long operation, and how they cancel it

**Platform** - skip only when the feature stays inside the app window
10. Which OS capability the feature needs - dialogs, tray, notifications, watching, clipboard
11. Whether any of them appear in `desktop-ecosystem-boundaries` as a Gap or a silent-failure trap

**Reach**
12. Platform tiers shipping this feature (Windows primary; macOS secondary)
13. Locales and accessibility expectations beyond the defaults

Ask targeted questions for gaps. Do not guess.

### STEP 2 - DESIGN (APPROVAL GATE)

Use skill: `desktop-ecosystem-boundaries` **first** - before any other design work - so a Gap capability is caught here rather than after implementation starts. Use skill: `desktop-core-architecture` for the crate plan and the boundary. Use skill: `desktop-overengineering-review` to check the design against the feature's actual size before it is built.

Present the plan:

- **Crate plan**: which types go in the GUI-free core, which in the UI crate, and each crate's dependencies
- **Core model**: operation input type, plan type, outcome type, and the injected seams
- **Destructive-operation plan**: what the preview shows, what undo records, and what makes a partial application recoverable - or an explicit statement that the feature is non-destructive
- **View plan**: views, their place in the app's navigation, and the scanning/error/empty presentation per async surface
- **Concurrency plan**: what runs on the UI thread, what runs on a worker, how progress reaches the view, and how cancellation propagates
- **Persistence impact**: fields added or reshaped, the schema version bump, and the migration step
- **Platform capabilities**: each OS capability the feature needs, its verdict from `desktop-ecosystem-boundaries`, and the escape hatch chosen for any Gap
- **Platform tiers and locales** in scope, with each tier's constraints on this feature's paths, threading, and packaging stated here rather than discovered later

Where the design deviates from this skill's defaults (logic in the UI crate, `unsafe`, a blocking call in `update`, a second async runtime), call out the deviation with its reason so the approver sees the choice rather than discovering it in review.

Wait for approval. When the invocation itself granted approval up front ("proceed without asking"), present the plan, treat that grant as the approval, and record every question it left unanswered in Open Assumptions.

### STEP 3 - CORE

Use skill: `desktop-core-architecture` for the boundary. Use skill: `rust-language-patterns` for ownership, borrowing, and the mechanics. Use skill: `rust-error-handling` for the error type and partial-failure reporting. Use skill: `desktop-batch-operations` for preview, undo, collision handling, and atomic apply. Use skill: `desktop-filesystem-patterns` for traversal and path handling. Use skill: `desktop-concurrency-patterns` when the operation parallelizes.

Plain Rust, no `iced`. An operation resolves to a **plan** before anything is applied, so preview and apply read the same computation rather than two implementations that drift. The plan is a value the caller can inspect, count, and render. Apply consumes the plan and returns a per-item outcome list. Time, randomness, and the filesystem root arrive as parameters or trait objects, so the core is testable without touching a real disk.

### STEP 4 - UI WIRING

Use skill: `iced-architecture-patterns` for the Model-Message-Update-View shape and where state belongs. Use skill: `iced-async-patterns` for `Task`, subscriptions, streamed progress, and cancellation.

The UI crate holds view state only; it calls into the core and renders what comes back. A long operation dispatches a `Task` and receives progress as messages, so `update` never blocks. Cancellation sets a flag the worker observes, and the view reflects the cancelled state rather than freezing. Messages carry core types, not re-declared parallel structs.

### STEP 5 - PRESENTATION

Use skill: `iced-widget-patterns` for composition, virtualized lists, and large result sets. Use skill: `desktop-image-processing` when the feature renders thumbnails or compares images. Use skill: `desktop-gpu-compute` when pixel work is heavy enough to move to the GPU. Use skill: `desktop-media-processing` when the feature touches audio or video. Use skill: `desktop-accessibility` for keyboard navigation, focus order, contrast, and text scaling. Use skill: `desktop-i18n` for every user-facing string and for the Unicode normalization difference between Windows and macOS.

A result list of unknown size is virtualized, not built as one column of every row. Thumbnails decode off the UI thread and are cached with a bounded eviction policy. Every view with an async or failable source renders scanning, error, and empty states, not just the populated one. A destructive action is never a single unconfirmed click - the preview is the confirmation surface.

### STEP 6 - PERSISTENCE AND PLATFORM

Run when the feature persists anything or touches an OS capability; skip and say so when it does neither.

Use skill: `desktop-data-persistence` when the feature persists state: the schema version bump, the migration from the previous version, and where the file lives per platform. Use skill: `desktop-platform-integration` for dialogs, drag-and-drop, tray, notifications, watching, and startup. Use skill: `desktop-security-patterns` when the feature resolves a user-supplied path, follows a symlink, deletes, or overwrites - path traversal, symlink escape, and TOCTOU on a destructive operation are its subject, and the accepted exposure it names is what the report's Accepted Exposure slot records.

Installed users have existing databases, so a shape change loads the old one in this build. Every async surface the feature adds gets a timeout, a cancellation path, and a defined resume behaviour.

**Silent-failure traps apply here.** Notifications no-op without a Windows AUMID shortcut or a macOS `.app` bundle; Keychain fails for unsigned binaries. When the feature depends on one of these, the report says whether the packaging prerequisite is in place or is an open gap.

### STEP 7 - TESTS

The core crate is GUI-free, which is what makes this cheap. Write **unit tests** against it: plan generation, collision and auto-suffix naming, undo round-trips, partial-failure outcomes, migration from each shipped schema version, and path handling for each target's edge cases. Use a temporary directory or an injected filesystem seam - no test touches the user's real files.

Write **integration tests** only where crate composition is genuinely under test. Iced view logic is not unit-tested; keep view code thin enough that this costs nothing.

For a full strategy, coverage assessment, or CI wiring, hand off to `task-desktop-test`.

When STEP 3 produced nothing, unit tests still cover whatever GUI-free logic the feature added - a display mapping, a format under each shipped locale, a settings round-trip. `Unit: 0` is correct only when the feature genuinely added no core code; say which it is rather than leaving the count unexplained.

### STEP 8 - VALIDATE

Run in order, fixing failures before reporting done:

1. `cargo fmt --check`
2. `cargo clippy --all-targets -- -D warnings`
3. `cargo test --workspace`
4. `cargo build --release` for the primary target - Use skill: `desktop-build-release` for packaging, signing, and the release-only hazards

Read the actual output for the result. If a command is unavailable in this environment (no toolchain, no cross-target, no signing identity), name which one and why rather than reporting a clean run. macOS artifacts built on Windows are a known gap: state that the macOS path is unverified rather than implying it ran.

## Edge Cases

- No `Cargo.toml`, or no `iced` dependency: STEP 1 stops the workflow; no code is written
- `Cargo.toml` unreadable: ask; do not assume the project shape
- Iced version differs from this plugin's guidance: verify each API against the project's version and say that you did
- No core crate exists, or the existing one depends on `iced`: report at STEP 2 as the blocking design finding and scope the fix to **this feature only** - create the core crate for the types this feature adds, leave existing modules where they are, and say which files move. A whole-workspace re-layering is not this workflow's scope, and building logic into the UI crate because no core crate exists is not an option
- Feature is non-destructive: the preview and undo requirements do not apply; say so rather than inventing an undo for a read-only operation
- Destructive operation triggered by a watcher or rule rather than a user click: the confirmation surface is the rule's dry-run preview at configuration time, and undo records every applied event - per-event confirmation is not required. The Destructive Operations row names the configuration-time preview and the undo journal
- Feature is pure presentation over existing core operations: STEP 3 produces nothing; say so rather than inventing a core layer
- Feature needs a Gap capability (printing, file association, shell extension, drag-out): STEP 2 names the escape hatch or scopes the capability out; it is never implemented as if it works
- No persistence and no OS capability: STEP 6 is skipped and said to be skipped
- `sled` detected: report it at STEP 6 - unmaintained since 2021
- Vague input: ask in STEP 1; never guess

## Output Format

```markdown
## Project Gate
Detected: {iced <version> in <path>} | Result: {Confirmed | No iced dependency - stopped | Unknown - asked}
API claims verified against: {version}

## Project Surfaces
| Surface | Detected | Applied |
|---|---|---|
| Core split | {crate name \| none \| depends on iced} | {yes \| created for this feature \| blocking finding} |
| Async runtime | {iced executor \| tokio \| smol \| unknown} | {yes \| caveated \| unused - present but this feature does not touch it} |
| Persistence | {rusqlite \| sled \| files \| none \| unknown} | {yes \| caveated \| unused - present but this feature does not touch it \| absent} |
| Packaging | {cargo-packager \| cargo-bundle \| none \| unknown} | {yes \| caveated \| absent} |
| Platform targets | {Windows \| macOS \| both \| other} | {yes \| caveated} |

## Crates
| Crate | Dependencies | Contains |

## Files Generated
[grouped by crate: core / ui / tests. Mark an existing file the feature changed as `(modified)` rather than listing it as new]

## Open Assumptions
[each STEP 1 gap the approver did not close, and the answer built on. `none - every question answered` when there are none]

## Core Model
- Input: {type, or `unchanged - feature adds no core input`}
- Plan: {type} -> {outcome type}, or `none - feature adds no operation`
- Injected seams: {clock, filesystem root, others}, or `none added`

## Destructive Operations
| Operation | Preview shows | Undo records | Partial-failure recovery |
[`n/a - feature is non-destructive` when it is]

## Views
| View | Scanning | Error | Empty |
[one row per view the feature adds or modifies. A view with no async or failable source writes `n/a - no async source` across the three state columns; a modified view whose pre-existing states this feature does not touch writes `unchanged`]

## Concurrency
- Worker: {what runs off the UI thread, or `none - no long operation`}
- Progress: {how it reaches the view}
- Cancellation: {how it propagates, or `none - operation is not cancellable, with the reason`}

## Persistence Impact
- Schema version: {old, or `none - no versioned store`} -> {new, or `unchanged - no shape change`}
- Migration: {step | `absorbed - additive settings field with serde default` | `none - no shape change`}
- Location: {per-platform path, or `n/a - nothing persisted`}

## Platform Capabilities
| Capability | Verdict | Approach |
[each OS capability the feature needs, its `desktop-ecosystem-boundaries` verdict, and the escape hatch for any Gap. `n/a - feature uses no OS capability` when it does not]
- Packaging prerequisite: {in place \| open gap - which capability silently fails without it \| n/a}

## Deviations
[each place the build departs from this skill's rules, with the reason - logic in the UI crate, `unsafe`, a blocking call in `update`, strings not behind a localization key. Written as `none` when there are none, never omitted]

## Accepted Exposure
[for a feature that resolves user-supplied paths, follows symlinks, deletes, or overwrites: the trust level chosen and the exposure it accepts in one sentence. `n/a - feature does none of these` otherwise]

## Platform Tiers
| Tier | Shipped | Caveats applied |

## Tests
- Unit: {count} - {surfaces covered, or the STEP 7 statement of why 0 is correct}
- Integration: {count} - {composition under test, or `none - no crate composition under test`}

## Validation
**Status:** {verified - all four commands ran and passed | unverified - <which of the four could not run, and why>}

[command -> result for each STEP 8 command; unavailable commands named with the reason]
```

Every slot above is written. A step that did not run is written as `skipped - {reason}` rather than omitted. A run STEP 1 stopped writes the Project Gate section alone - the sections below it describe work that never ran and are omitted, not back-filled with `skipped`.

## Self-Check

- [ ] `behavioral-principles` loaded before the workflow ran
- [ ] `Cargo.toml` read and the `iced` dependency confirmed; absence stopped rather than degraded
- [ ] Iced version read and every API claim checked against it, not model memory
- [ ] Core split, async runtime, persistence, packaging, and platform targets confirmed
- [ ] `desktop-ecosystem-boundaries` consulted at STEP 2 before any other design work
- [ ] Requirements gathered by cluster; design approved before code
- [ ] Deviations from the skill's defaults called out at the approval gate
- [ ] Core crate has no `iced` dependency; where none existed, one was created for this feature's types and the files that move were named
- [ ] Operations resolve to an inspectable plan before apply; preview and apply share one computation
- [ ] Every destructive operation ships its preview and its undo in this change
- [ ] Batch failures reported per item; one failure abandons neither the batch nor the report
- [ ] Paths handled as `Path`/`OsStr`, never assumed UTF-8
- [ ] Long work runs off the UI thread; `update` never blocks; cancellation propagates
- [ ] Result lists virtualized; thumbnails decoded off-thread with bounded caching
- [ ] Every view with an async or failable source renders scanning, error, and empty states
- [ ] Keyboard navigation, focus order, contrast, and text scaling present
- [ ] User-facing strings behind localization keys, or held in one module with the deviation stated
- [ ] Deviations and accepted exposure written in the report, `none` / `n/a` where there are none
- [ ] Versioned-store shape change ships a version bump and a migration; a default-absorbed settings field is recorded as absorbed
- [ ] Gap capabilities designed around at STEP 2, never implemented as if they work
- [ ] Packaging prerequisite stated where a silent-failure capability depends on it
- [ ] Unit tests cover the core; no test touches the user's real files
- [ ] `fmt`, `clippy`, `test`, and a release build all executed, with unavailable commands named

## Avoid

- Core logic written inside `update` or a view function
- `iced` referenced from the core crate
- A destructive operation shipped without its preview or its undo
- Preview and apply implemented as two separate computations
- Blocking I/O or a long loop inside `update`
- `std::fs` paths converted through `String` or `to_string_lossy` for anything but display
- `unwrap` or `expect` on a filesystem result in a batch path
- A batch that aborts on first error, or reports one aggregate success for a partial run
- Building an unbounded widget column for a result set of unknown size
- Decoding thumbnails on the UI thread, or caching them without eviction
- Colour as the only carrier of a result-relevant distinction
- Hardcoded user-facing strings
- A versioned-store shape change with no version bump or migration
- Introducing a second async runtime alongside Iced's executor
- Planning a feature on a capability listed as a Gap
- Claiming an Iced API from memory rather than the project's resolved version
- Writing code before design approval
- Reporting a clean run without executing the STEP 8 commands
