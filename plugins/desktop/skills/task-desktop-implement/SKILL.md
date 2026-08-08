---
name: task-desktop-implement
description: End-to-end C# desktop feature work - UI-free core, Avalonia MVVM wiring, batch preview and undo, persistence, platform integration, tests.
agent: desktop-engineer
metadata:
  category: desktop
  tags: [csharp, avalonia, desktop, file-utility, batch-operations, undo, feature, implementation, workflow]
  type: workflow
user-invocable: true
---

> **Behavioral directive:** Load `Use skill: behavioral-principles` before executing this workflow.

# Implement C# Desktop Feature

## When to Use

End-to-end C# desktop feature work: core logic + Avalonia wiring + persistence + platform integration + tests in one pass. STEPS 3-7 write the files; the report lists what was written. STEP 2 is the gate that decides what gets written, and nothing lands before it is approved.

Not for: a control restyle, a string change, a dependency bump, or a `.csproj` edit. A change that *only* restyles an existing view is a UI edit, not this workflow.

It does apply when presentation work also adds behaviour, locales, or persistence - the steps that produce nothing are written as skipped rather than being a reason to decline. A feature built entirely on existing core operations is an anticipated shape, not an out-of-scope one.

## Rules

- **Core logic is plain C# with no Avalonia dependency**, in a class library whose `.csproj` cannot reference the GUI. Every other decision in this workflow follows from that one
- **A destructive operation ships its preview and its undo in the same change.** Rename, move, delete, and overwrite are destructive; a feature that applies one without a dry-run path is incomplete, not shippable-with-a-follow-up
- Long work runs off the UI thread through an async command and reports progress as coalesced messages - never a blocking call, `.Result`, or a synchronous query inside a command body or property getter
- Paths are built with `Path.Combine`/`Path.Join`, never string concatenation, and filename comparison honours the filesystem's case rules and platform Unicode normalization
- Every fallible operation in a batch reports its own outcome; one failure does not abandon the remaining items or silently succeed
- User-facing strings resolve through a resource key (`.resx`). Where the project has no localization system, keep every string in one class so extraction is mechanical, and report that at STEP 5 as a stated deviation rather than installing a localization system this feature did not ask for
- A shape change to a versioned store ships a `user_version` bump and a migration, because installed users have existing databases. An additive settings-file field absorbed by the serializer's default needs neither - record it as absorbed in Persistence Impact. A new persisted format (a document type, a settings file, a journal) is the version-1 case: it carries a format version from first release, refuses a higher version number with a clear error, and still ignores unknown fields within a version it accepts - the version integer gates, the field set does not. Where no settings store exists yet, creating one is this case, not the absorbed case
- **The app is AOT-safe by design**: source-generated serialization, compiled bindings, explicit registration. A reflection-based shortcut passes under `dotnet run` and fails in the published build
- A capability listed as a Gap in `desktop-ecosystem-boundaries` is designed around at STEP 2, not discovered at STEP 5
- Each step completes before the next; design approved before code

## Workflow

### STEP 1 - DETECT AND GATHER

**Project gate.** Read the `.csproj` files (through the `.sln` where one exists) and confirm an Avalonia PackageReference, then read the solution layout. When no project file is found or none references Avalonia, state what was found and **STOP** - this is a detect-and-report boundary, not a degradation path. Do not emit Avalonia guidance for a project that does not use it. When the files are unreadable, say so and ask rather than assuming.

State the Avalonia version from the PackageReference (or `Directory.Packages.props` under central package management). Avalonia 12.x is stable semver on .NET 10 LTS - the manifest version is the version, and no lockfile resolution step is needed.

Then confirm:

| Concern | Confirm | If it differs |
| --- | --- | --- |
| Core split | A class library with no Avalonia PackageReference in its dependency tree | **None anywhere, or the existing one references Avalonia: report at STEP 2 as the blocking design finding.** Do not build logic into the UI project around it |
| MVVM toolkit | CommunityToolkit.Mvvm, or the project's established choice | ReactiveUI already in place: keep it and build within its idioms; never migrate mid-feature |
| Persistence | `Microsoft.Data.Sqlite`, or a stated alternative | EF Core present: build on what exists and note it as the heavier path. A reflection-based serializer in an AOT-published app: report at STEP 6 as a finding |
| Publish mode | NativeAOT, self-contained, or framework-dependent, from the publish properties | `PublishAot` set: compiled bindings and source generators are mandatory - reflection-based designs are ruled out at STEP 2, not discovered at STEP 8 |
| Platform targets | Windows primary, macOS secondary - from the request or the Reach answers below; project files rarely state tiers | A Linux-only or macOS-primary answer: state the mismatch and which guidance still applies |
| Packaging | Velopack, or a stated alternative | None at all: report at STEP 6, since packaging identity gates notifications and credential storage |

Ask before writing code, grouped so each cluster surfaces its own follow-ups. Skip clusters the feature does not touch:

**Feature**
1. What the user selects, and what the operation does to it
2. Entry point: new view, existing view, dialog, or background task

**Core**
3. What the core project owns, and what a single operation produces
4. Whether the operation is destructive, and what its preview shows
5. Undo: whether it is reversible, what the reversal needs recorded, and whether undo must survive an app restart

**Data**
6. What persists, and whether the persisted shape changes
7. Scale: expected item count, and whether results stream or arrive whole

**Presentation**
8. Views, and what each renders while scanning or on failure
9. Progress: what the user sees during a long operation, and how they cancel it

**Platform** - skip only when the feature stays inside the app window
10. Which OS capability the feature needs - dialogs, tray, notifications, watching, clipboard - and, for a feature that opens or owns a file type, how file-open activation arrives (argv on Windows, Apple Events on macOS, second-instance forwarding)
11. Whether any of them appear in `desktop-ecosystem-boundaries` as a Gap or a silent-failure trap

**Reach**
12. Platform tiers shipping this feature (Windows primary; macOS secondary)
13. Locales and accessibility expectations beyond the defaults

Ask targeted questions for gaps. Do not guess.

### STEP 2 - DESIGN (APPROVAL GATE)

Use skill: `desktop-ecosystem-boundaries` **first** - before any other design work - so a Gap capability is caught here rather than after implementation starts. Use skill: `desktop-core-architecture` for the project plan and the boundary. Use skill: `desktop-overengineering-review` to check the design against the feature's actual size before it is built.

Present the plan:

- **Project plan**: which types go in the UI-free core, which in the UI project, and each project's references
- **Core model**: operation input type, plan type, outcome type, and the injected seams
- **Destructive-operation plan**: what the preview shows, what undo records, and what makes a partial application recoverable - or an explicit statement that the feature is non-destructive
- **View plan**: views, their place in the app's navigation, the scanning/error/empty presentation per async surface, and the accessibility surface: automation names for interactive controls, a peer for any custom-drawn control, and the keyboard path through the flow
- **Concurrency plan**: what runs on the UI thread, what runs on a worker, how progress reaches the view, and how cancellation propagates
- **Persistence impact**: fields added or reshaped, the `user_version` bump, and the migration step
- **Platform capabilities**: each OS capability the feature needs, its verdict from `desktop-ecosystem-boundaries`, and the escape hatch chosen for any Gap
- **Platform tiers and locales** in scope, with each tier's constraints on this feature's paths, threading, and packaging stated here rather than discovered later

Where the design deviates from this skill's defaults (logic in the UI project, `unsafe` or P/Invoke, a blocking call in a command body, reflection in an AOT-published app), call out the deviation with its reason so the approver sees the choice rather than discovering it in review.

Wait for approval. When the invocation itself granted approval up front ("proceed without asking"), present the plan, treat that grant as the approval, and record every question it left unanswered in Open Assumptions.

### STEP 3 - CORE

Use skill: `desktop-core-architecture` for the boundary. Use skill: `csharp-language-patterns` for types, nullability, and the mechanics. Use skill: `csharp-error-handling` for the outcome types and partial-failure reporting. Use skill: `desktop-batch-operations` when the operation is destructive - preview, undo, collision handling, and atomic apply. Use skill: `desktop-filesystem-patterns` for traversal and path handling. Use skill: `desktop-concurrency-patterns` when the operation parallelizes.

Plain C#, no Avalonia. An operation resolves to a **plan** before anything is applied, so preview and apply read the same computation rather than two implementations that drift. The plan is a value the caller can inspect, count, and render. Apply consumes the plan and returns a per-item outcome list. Time, randomness, and the filesystem root arrive as parameters or interfaces (`TimeProvider` for the clock), so the core is testable without touching a real disk.

### STEP 4 - UI WIRING

Use skill: `avalonia-mvvm-patterns` for the ViewModel shape, commands, and where state belongs. Use skill: `csharp-async-patterns` for async commands, coalesced progress, cancellation, and dispatcher marshalling.

The UI project holds view state only; it calls into the core and renders what comes back. A long operation runs through an async command and receives progress coalesced through `IProgress<T>`, so the UI thread never blocks and the dispatcher is never starved. Cancellation flows through a `CancellationToken` the work loop observes, and the view reflects the cancelled state rather than freezing. ViewModels wrap core types, not re-declared parallel classes. After an undo, the result list is stale by construction - state how it reconciles (a rescan prompt, or in-place reversal of the affected rows).

### STEP 5 - PRESENTATION

Use skill: `avalonia-control-patterns` for composition, compiled bindings, virtualized lists, and large result sets. Use skill: `desktop-image-processing` when the feature renders thumbnails or compares images. Use skill: `desktop-media-processing` when the feature touches audio or video. Use skill: `desktop-accessibility` for automation names, peers, keyboard navigation, focus order, contrast, and text scaling. Use skill: `desktop-i18n` for every user-facing string and for the Unicode normalization difference between Windows and macOS.

A result list of unknown size is virtualized, not realized one control per row. Thumbnails decode off the UI thread at thumbnail resolution and are cached with a bounded eviction policy. Every view with an async or failable source renders scanning, error, and empty states, not just the populated one. A destructive action is never a single unconfirmed click - the preview is the confirmation surface.

Avalonia has real screen-reader support - design for it. Every interactive control gets an accessible name (`AutomationProperties.Name`, `LabeledBy`, or text content the peer surfaces); a custom-drawn control overrides `OnCreateAutomationPeer` or it does not exist to assistive technology. A screen built on a known-gap surface - TextBox caret announcement (#9770), DataGrid keyboard access (#10175) - is smoke-tested with a screen reader rather than assumed to work.

### STEP 6 - PERSISTENCE AND PLATFORM

Run when the feature persists anything or touches an OS capability; skip and say so when it does neither.

Use skill: `desktop-data-persistence` when the feature persists state: the `user_version` bump, the migration from the previous version, and where the file lives per platform. Use skill: `desktop-platform-integration` for dialogs, drag-and-drop, tray, notifications, watching, and startup. Use skill: `desktop-security-patterns` when the feature resolves a user-supplied path, follows a link, deletes, or overwrites - path traversal, symlink escape, and TOCTOU on a destructive operation are its subject, and the accepted exposure it names is what the report's Accepted Exposure slot records.

Installed users have existing databases, so a shape change loads the old one in this build. Every async surface the feature adds gets a cancellation path and a defined resume behaviour, and a timeout wherever a call can hang instead of progressing - a long local batch is bounded by cancellation, not by a wall clock.

**Silent-failure traps apply here.** Notifications no-op without a Windows AUMID shortcut or a signed macOS `.app` bundle; macOS Keychain fails for unsigned binaries, and every rebuild changes an ad-hoc identity; `FileSystemWatcher` drops events when its buffer overflows, so it is paired with a rescan. When the feature depends on one of these, the report says whether the packaging prerequisite is in place or is an open gap.

### STEP 7 - TESTS

The core project is UI-free, which is what makes this cheap. Write **unit tests** against it (xUnit, or the project's existing framework): plan generation, collision and auto-suffix naming, undo round-trips, partial-failure outcomes, migration from each shipped schema version, and path handling for each target's edge cases. Use a temporary directory or an injected filesystem seam - no test touches the user's real files.

Write **integration tests** only where project composition is genuinely under test. Avalonia view logic is not unit-tested; keep view code thin enough that this costs nothing.

For a full strategy, coverage assessment, or CI wiring, hand off to `task-desktop-test`.

When STEP 3 produced nothing, unit tests still cover whatever UI-free logic the feature added - a display mapping, a format under each shipped locale, a settings round-trip. `Unit: 0` is correct only when the feature genuinely added no core code; say which it is rather than leaving the count unexplained.

### STEP 8 - VALIDATE

Run in order, fixing failures before reporting done:

1. `dotnet format --verify-no-changes`
2. `dotnet build -warnaserror`
3. `dotnet test`
4. `dotnet publish -c Release` for the primary target - Use skill: `desktop-build-release` for packaging, signing, and the publish-only hazards

Read the actual output for the result. Fix failures this change introduced; a pre-existing failure (an inherited warning, a dependency advisory promoted by `-warnaserror`, an unresolvable pinned package) is reported alongside the validation status, not fixed - surgical scope holds. Where that failure blocks compilation of the code this change adds, verify in a throwaway copy with the blocker resolved, leave the working tree untouched, and report the side-check separately from the four commands rather than claiming they covered it. If a command is unavailable in this environment (no SDK, no native toolchain, no signing identity), name which one and why rather than reporting a clean run. macOS artefacts built on Windows are a known gap - NativeAOT does not cross-compile across operating systems - so state that the macOS path is unverified rather than implying it ran.

## Edge Cases

- No `.csproj`, or no Avalonia PackageReference: STEP 1 stops the workflow; no code is written
- Project files unreadable: ask; do not assume the project shape
- No core project exists, or the existing one references Avalonia: report at STEP 2 as the blocking design finding and scope the fix to **this feature only** - create the core class library for the types this feature adds (or, in a single-project app, the namespace boundary `desktop-core-architecture` describes), leave existing modules where they are, and say which files move. A whole-solution re-layering is not this workflow's scope, and building logic into the UI project because no core project exists is not an option
- Feature is non-destructive: the preview and undo requirements do not apply; say so rather than inventing an undo for a read-only operation
- Document Save-As overwriting an existing file through the OS save dialog: the dialog's replace prompt is the confirmation surface and an atomic write (temp + rename) is the required safety; the batch preview-and-undo rule does not apply to a single user-chosen document write. Record the row as `save-as - dialog-confirmed atomic write`
- Destructive operation triggered by a watcher or rule rather than a user click: the confirmation surface is the rule's dry-run preview at configuration time, and undo records every applied event - per-event confirmation is not required. The Destructive Operations row names the configuration-time preview and the undo journal
- Feature is pure presentation over existing core operations: STEP 3 adds no operation - but a display predicate or mapping the feature introduces (a filter, a format) still lands in the core library as a non-operation type. "Produces nothing" means no plan/apply pair, not no core file; say which it is rather than inventing a core operation or burying testable logic in a ViewModel
- Feature needs a Gap capability (file-association default, cross-platform printing, Finder or Explorer shell extensions): STEP 2 names the escape hatch or scopes the capability out; it is never implemented as if it works
- No persistence and no OS capability: STEP 6 is skipped and said to be skipped
- Project on ReactiveUI: build within its idioms; never migrate it to the Toolkit mid-feature
- Vague input: ask in STEP 1; never guess

## Output Format

```markdown
## Project Gate
Detected: {Avalonia <version> in <csproj path>} | Result: {Confirmed | No Avalonia PackageReference - stopped | Unknown - asked}

## Project Surfaces
| Surface | Detected | Applied |
|---|---|---|
| Core split | {project name \| none \| references Avalonia} | {yes \| created for this feature \| blocking finding} |
| MVVM toolkit | {CommunityToolkit.Mvvm \| ReactiveUI \| none \| unknown} | {yes \| caveated \| unused - present but this feature does not touch it} |
| Persistence | {Microsoft.Data.Sqlite \| EF Core \| files \| none \| unknown} | {yes \| caveated \| unused - present but this feature does not touch it \| supplemented - new store(s) added beside it \| absent} |
| Publish mode | {NativeAOT \| self-contained \| framework-dependent \| unknown} | {yes \| caveated} |
| Packaging | {Velopack \| MSIX \| none \| unknown} | {yes \| caveated \| absent} |
| Platform targets | {Windows \| macOS \| both \| other} | {yes \| caveated} |

## Projects
| Project | References | Contains |

## Files Generated
[grouped by project: core / ui / tests. Mark an existing file the feature changed as `(modified)` rather than listing it as new]

## Open Assumptions
[each STEP 1 gap the approver did not close, and the answer built on. `none - every question answered` when there are none]

## Core Model
- Input: {type, or `unchanged - feature adds no core input`}
- Plan: {type} -> {outcome type}, or `none - feature adds no operation`
- Injected seams: {clock, filesystem root, others}, or `none added`
- Non-operation types: {core types that are not an operation - a filter, a mapping}, or `none`

## Destructive Operations
| Operation | Preview shows | Undo records | Partial-failure recovery | Interrupted-run surfacing |
[the last column states what the next launch offers when the journal shows an unfinished run. `n/a - feature is non-destructive` when it is]

## Views
| View | Scanning | Error | Empty |
[one row per view the feature adds or modifies. A view with no async or failable source writes `n/a - no async source` across the three state columns; a modified view whose pre-existing states this feature does not touch writes `unchanged`]

## Concurrency
- Worker: {what runs off the UI thread, or `none - this feature adds no long operation`, naming any pre-existing worker it leaves untouched}
- Progress: {how it reaches the view}
- Cancellation: {how it propagates - one line per long operation where the answers differ (apply and undo may not match), or `none - operation is not cancellable, with the reason`}

## Persistence Impact
[one block per store the feature changes or creates - untouched stores are not listed; `n/a - nothing persisted` when there is no block]
- Store: {name} - {changed | created this feature}
- Schema version: {old -> new | v1 - created this feature | unchanged - no shape change}
- Migration: {step | `absorbed - additive settings field with a serializer default` | `refuse-newer guard - created store` | `none - no shape change`}
- Location: {per-platform path}

## Platform Capabilities
| Capability | Verdict | Approach |
[each OS capability the feature needs, its `desktop-ecosystem-boundaries` verdict, and the escape hatch for any Gap; a capability considered and rejected gets a row with verdict `rejected - <reason>` when the rejection shaped the design. `n/a - feature uses no OS capability` when it does not]
- Packaging prerequisite: {in place \| open gap - which capability silently fails without it \| n/a}

## Deviations
[each place the build departs from this skill's rules or from a skill it delegates to, with the reason - logic in the UI project, `unsafe` or P/Invoke, a blocking call in a command body, reflection in an AOT-published app, strings not behind a resource key. Written as `none` when there are none, never omitted]

## Observations
[pre-existing defects noticed en route that this feature does not fix - surfaced rather than silently absorbed. `none` when there are none]

## Accepted Exposure
[for a feature that resolves user-supplied paths, follows links, deletes, or overwrites: the trust level chosen and the exposure it accepts in one sentence. `n/a - feature does none of these` otherwise]

## Platform Tiers
| Tier | Shipped | Caveats applied |

## Tests
- Unit: {count} - {surfaces covered, or the STEP 7 statement of why 0 is correct}
- Integration: {count} - {composition under test, or `none - no project composition under test`}

## Validation
**Status:** {verified - all four commands ran and passed | unverified - <which of the four could not run, and why>}
**Compiled:** {every project this feature touched | <the projects the commands could not cover, and what of this change went uncompiled>}
**Side-check:** {throwaway-copy result where a pre-existing blocker was resolved to compile this change | `none - not needed`}

[command -> result for each STEP 8 command; unavailable commands named with the reason]
```

The report is delivered inline as the final reply; it is not written to a file unless the request names a destination. Every slot above is written. A step that did not run is written as `skipped - {reason}` rather than omitted. A run STEP 1 stopped writes the Project Gate section alone - the sections below it describe work that never ran and are omitted, not back-filled with `skipped`.

## Self-Check

- [ ] `behavioral-principles` loaded before the workflow ran
- [ ] `.csproj` read and the Avalonia PackageReference confirmed; absence stopped rather than degraded
- [ ] Core split, MVVM toolkit, persistence, publish mode, packaging, and platform targets confirmed
- [ ] `desktop-ecosystem-boundaries` consulted at STEP 2 before any other design work
- [ ] Requirements gathered by cluster; design approved before code
- [ ] Deviations from the skill's defaults called out at the approval gate
- [ ] Core project has no Avalonia reference; where none existed, one was created for this feature's types and the files that move were named
- [ ] Operations resolve to an inspectable plan before apply; preview and apply share one computation
- [ ] Every destructive operation ships its preview and its undo in this change, and states what the next launch offers for an interrupted run
- [ ] Batch failures reported per item; one failure abandons neither the batch nor the report
- [ ] Paths built with `Path.Combine`/`Path.Join`; comparison honours case rules and normalization
- [ ] Long work runs off the UI thread through async commands; no `.Result`/`.Wait()`; no `async void` outside event handlers; cancellation propagates
- [ ] Result lists virtualized; thumbnails decoded off-thread with bounded caching
- [ ] Every view with an async or failable source renders scanning, error, and empty states
- [ ] Automation names on interactive controls, peers on custom controls, keyboard navigation, focus order, contrast, and text scaling present; known-gap surfaces smoke-tested
- [ ] User-facing strings behind resource keys, or held in one class with the deviation stated
- [ ] Deviations, observations, and accepted exposure written in the report, `none` / `n/a` where there are none
- [ ] Versioned-store shape change ships a `user_version` bump and a migration; a default-absorbed settings field is recorded as absorbed; a created store carries v1 and a refuse-newer guard
- [ ] Gap capabilities designed around at STEP 2, never implemented as if they work
- [ ] Packaging prerequisite stated where a silent-failure capability depends on it
- [ ] Unit tests cover the core; no test touches the user's real files
- [ ] `format`, `build -warnaserror`, `test`, and a Release publish all executed, with unavailable commands named and what they left uncompiled stated

## Avoid

- Core logic written inside a ViewModel command body or view code-behind
- Avalonia referenced from the core project
- A destructive operation shipped without its preview or its undo
- Preview and apply implemented as two separate computations
- Blocking I/O, `.Result`, `.Wait()`, or a synchronous query inside a command body or property getter
- `async void` anywhere but an event handler
- Paths built by string concatenation or interpolation instead of `Path.Combine`/`Path.Join`
- A batch that aborts on first error, or reports one aggregate success for a partial run
- Building an unvirtualized list for a result set of unknown size
- Decoding thumbnails on the UI thread, or caching them without eviction
- An icon-only control with no `AutomationProperties.Name`
- Colour as the only carrier of a result-relevant distinction
- Hardcoded user-facing strings
- A versioned-store shape change with no version bump or migration
- Reflection-based serialization or `Type.GetType` resolution in an AOT-published app
- Planning a feature on a capability listed as a Gap
- Writing code before design approval
- Reporting a clean run without executing the STEP 8 commands
