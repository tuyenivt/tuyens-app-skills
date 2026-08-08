---
name: task-desktop-review
description: C# desktop code review - UI-free core discipline, destructive-op safety, XAML bindings, async traps; spawns perf and security subagents.
agent: desktop-tech-lead
metadata:
  category: desktop
  tags: [csharp, avalonia, desktop, code-review, working-tree, staff-review, multi-scope, workflow]
  type: workflow
user-invocable: true
---

> **Behavioral directive:** Load `Use skill: behavioral-principles` before executing this workflow.

# C# Desktop Code Review

Staff-level C# desktop review umbrella. Covers correctness, architecture, AI quality, maintainability. Coordinates perf / security subagents in parallel.

## When to Use

- Review of the current working-tree change set in a C#/Avalonia desktop project
- Post-AI-generation quality gate
- Architecture drift detection - specifically, erosion of the UI-free core boundary
- Pre-merge risk assessment

**Not for:** pre-implementation design (`task-desktop-implement`), single-error triage (`desktop-engineer`), single-scope reviews (delegate to perf/security).

## Depth

| Depth | When | Runs |
|-------|------|------|
| `standard` | Default | Phases A-E |
| `deep` | Architecture changes, post-incident, Principal sign-off | A-E + historical patterns + cross-change context |

**Auto-promote to `deep`:** After Phase A, if Risk is High/Critical, set depth to `deep` and surface `Depth auto-promoted: standard -> deep (Risk: <level>)`.

`deep` evidence (prior commits, historical patterns, cross-change context) lands inside findings' System Risk lines and `Architecture Notes` - it adds evidence, never new report sections.

## Scope

| Scope | What runs |
|-------|-----------|
| Core | Phases A-E |
| + Perf | Core + `task-desktop-review-perf` subagent |
| + Sec | Core + `task-desktop-review-security` subagent |
| Full | Core + both in parallel |

Default: **Core with auto-escalation**. Pass `core-only` to suppress.

**Auto-escalation signals:**

- **+Sec:** a new destructive operation (delete, overwrite, rename, move) reaching the filesystem; a user-supplied or config-supplied path joined without normalization or a containment proof; symlink or junction traversal; archive extraction (`System.IO.Compression`); a `Process.Start` with a concatenated argument string; a new P/Invoke signature or `unsafe` block; a credential, token, or key read or written; a new network fetch or update-check endpoint; a dependency added with a known-advisory history
- **+Perf:** a blocking call, `.Result`, `.Wait()`, or a synchronous query in a ViewModel command body, property getter, or event handler; `async void` outside an event handler; filesystem traversal added to a path that already walks; hashing or image decode on a per-item path; an unbounded collection built from a scan; an unvirtualized list bound to user-scale data; a new cache with no eviction; per-item `ObservableCollection.Add` over a large result set; a new `Parallel.ForEach`, `Task.Run` fan-out, or thread spawn
- **2+ categories -> Full**

Signals are matched syntactically, then checked for direction: a `.Result` removed, a scan moved off the UI thread, a cache gaining eviction, or a containment proof added is the fix, not the defect. Log it as `signal: <category> -> <path> (improving - not escalated)` and do not escalate on it alone. A restructuring of existing code on a matched path is still the signal; direction then decides, and a semantics-preserving refactor that neither adds nor removes the defect logs `(neutral - not escalated)`.

There is no `+Ux` scope. Accessibility, adaptivity, and localization are reviewed at baseline depth in Phase E, and designed in `task-desktop-implement`; there is no dedicated UX lens to escalate to.

## Reviewable Surface

| Surface | Treatment |
| --- | --- |
| `.cs` under source and test directories | full review surface |
| `.axaml` and `.axaml.cs` | **review surface** - bindings, resource references, templates, and code-behind carry real defects; a binding to a missing property breaks the build under compiled bindings and fails silently under reflection bindings |
| `.csproj`, `.sln`, `Directory.Build.props`, `Directory.Packages.props`, `NuGet.config` | **review surface** - a PackageReference addition, a TFM change, a publish-property change (`PublishAot`, trimming), or an analyzer change is a legitimate finding |
| `.resx`, `.sql`, packaging manifests, CI workflow files | review surface - localization catalogs, migrations, and release configuration carry real defects |
| `bin/`, `obj/`, `publish/`, `*.g.cs`, `*.Designer.cs` | excluded - build output and generated code, never reviewed |
| `review-*.md` at the repo root | excluded - this workflow's own reports. On an otherwise-clean tree a previous report leaves the next run in `working-tree` mode, so gitignore it where runs repeat |
| Vendored third-party sources | excluded from style and structure findings; still in scope for security data-flow and version concerns |

A change set touching only build output is a no-op; say so rather than manufacturing findings.

## Invocation

| Form | Meaning |
|------|---------|
| `/task-desktop-review` | Review the working-tree change set (unstaged + staged + untracked) |
| `/task-desktop-review --staged` | Review the staged change set only |
| `/task-desktop-review core-only` | Suppress auto-escalation; run Core alone. Accepted bare or as `--core-only`, and combinable with `--staged` |
| `/task-desktop-review deep` | Force `deep` depth. Accepted bare or as `--deep` |

When the tree is clean, `review-precondition-check` falls back to the last commit.

**Never modify the working tree.** Read via `git diff` only; uncommitted changes are the review subject, not an obstacle.

## Workflow

### Step 1 - Behavioral Principles

Use skill: `behavioral-principles`. Accept parent's confirmation if invoked as subagent.

### Step 2 - Stack and Project Shape

Locate the `.csproj` files (through the `.sln` where one exists). If none is found, stop - this workflow reviews .NET projects only.

Record: the solution layout, whether a UI-free core project exists, the Avalonia version (from its PackageReference, or `Directory.Packages.props` under central package management), the MVVM toolkit, the persistence package, the packaging tool, and the publish mode (`PublishAot`, `PublishTrimmed`, self-contained). Avalonia 12.x is stable semver on .NET 10 LTS - the manifest version is the version; no lockfile resolution is needed.

**Publish mode shapes findings.** In an AOT-published or trimmed app, reflection-based serialization, `Type.GetType` resolution, and reflection `{Binding}` XAML bindings fail at runtime on a user's machine while passing under `dotnet run`. A diff introducing one into an AOT project is a Phase B finding, not a style note.

**No UI-free core.** If the solution has no core project, or the existing one references Avalonia, note it once in Summary and treat it as the standing architectural condition Phase C reports against. Do not re-raise it per file.

### Step 3 - Resolve the Change Set

Use skill: `review-precondition-check`. Forward `--staged` if passed. If it fails fast, surface verbatim and stop.

The handle supplies `mode`, `base`, `current_branch`, `reviewable`, `counts`, and `notes`. Carry all of them forward - the report and the subagent prompts are built from them.

Read the diff content once and reuse:

- `git diff <base>` for the change body
- `git diff --name-status <base>` for the file list
- Untracked files in the handle's `reviewable` list have no content in `git diff` output - read each one directly; a new file is full review surface, not an empty diff

Restrict analysis to the handle's `reviewable` paths; binary and generated paths are excluded and never diffed.

The handle's generated-path conventions are stack-neutral. Apply this workflow's Reviewable Surface table on top of it: `bin/`, `obj/`, and `publish/` are excluded for analysis even where the handle lists them. Where the two numbers differ, write the handle's count and state the reviewed count once in Summary: `<N> reviewable; <M> reviewed - <what was excluded>`.

**Skip entirely** when invoked as subagent and parent passed handle + pre-read diff.

### Step 4 - Scope Auto-Escalation

Scan file list / diff for signals listed under **Scope**, ignoring excluded surfaces. Log each as `signal: <category> -> <file:line>`. Then:

- Zero signals or `core-only` -> Core
- One category -> add matching scope
- 2+ categories -> Full
- Explicit scope -> respect; still log signals

**Scope precedence:** user flag > firing signals.

Surface decision in Summary; if escalated, append `auto-escalated from Core; signals: <list>`. When signals fired without escalating, append `signals logged, not escalated: <list>`, prefixed with `core-only passed; ` when the flag did the suppressing, and annotate each listed signal with its direction (`improving`, `neutral`) where that is why it did not escalate - a reader must be able to tell a signal-free diff from one whose escalation was declined.

### Phase A - Risk Snapshot

Assess cross-cutting risk for the change set as a whole: what breaks if this is wrong, how far the failure reaches, and how much user data it touches. Weigh the surfaces the change reaches (the core project, the apply path of a destructive operation, the migration layer, a `.csproj` or `Directory.Build.props`, packaging configuration), the size reported in the handle's `counts`, and whether the change is on a critical path (a destructive filesystem operation, a schema migration, an undo path, a credential read, an update mechanism).

**A change touching the apply path of a destructive operation is never Low risk.** Data loss is unrecoverable for the user in a way a crash is not.

Output the risk level before any findings. Rate by worst plausible consequence: **Critical** - irreversible loss or corruption of user data; **High** - wrong results or a frozen app on a main path; **Medium** - degraded behaviour with a workaround; **Low** - none of these reachable.

**Low-risk short-circuit:** if Risk is Low **and** the change does not touch architecture-relevant files (any `.csproj`, `.sln`, `Directory.Build.props`, or `Directory.Packages.props`, publish or packaging configuration, the destructive-apply path, the migration layer) **and** does not touch a localized string, a `.resx` catalog, or a path-handling routine (one that joins, normalizes, compares, or displays paths - not every routine that merely receives one), skip Phases C-E. Escalated scopes still run (a Step 4 signal fired for a reason); their merged findings join High-Impact Findings. The streamlined report contains Summary, High-Impact Findings (Phase B + any subagent findings), and Next Steps only - Architecture Notes, Maintainability Notes, Key Takeaways, and Lens Detail are omitted; Step 8 still writes the report. With zero findings, High-Impact Findings is written as `none - <one line naming the checks that ran clean>` and Next Steps is omitted.

### Step 5 - Re-evaluate Depth After Phase A

If Risk is High / Critical, set depth to `deep` and surface promotion in Summary **before** Phases B-E. User-passed depth wins over auto-promotion.

### Phase B - C# Correctness and Safety

Apply atomic skills; each owns canonical patterns:

- Use skill: `csharp-language-patterns` - nullability suppressions, mutable structs, LINQ hot-path cost, disposal, AOT-hostile reflection
- Use skill: `csharp-error-handling` - outcome types, propagation, partial-failure reporting, catch discipline
- Use skill: `desktop-batch-operations` if the diff touches a preview, apply, undo, or collision path
- Use skill: `desktop-filesystem-patterns` if the diff touches traversal, path joining, or path display
- Use skill: `avalonia-mvvm-patterns` if the diff touches a ViewModel, command, state placement, or DI registration
- Use skill: `avalonia-control-patterns` if the diff touches an `.axaml` view, template, binding, or list rendering
- Use skill: `csharp-async-patterns` if the diff touches `Task`, async commands, progress, or cancellation
- Use skill: `desktop-concurrency-patterns` if the diff touches threads, `Parallel.ForEach`, channels, or shared state
- Use skill: `desktop-data-persistence` if the diff changes a persisted shape. Check installed-version impact directly: an installed older build must still work after this change - it must read what this one persists wherever possible

Atomic skills contribute checks; their own output contracts do not survive here. Their closers (`No async findings.`), `Catch:`/`Suppression:`/`Justified as-is:` lines, and severity vocabulary are dropped - every defect they surface is re-expressed as a finding in this report's shape and labels.

**Named checks.** Several overlap the atomics above by design - they are the highest-recurrence C# desktop defects and must not be lost in a long atomic report. Emit one finding per defect: when an atomic already raised it, keep that finding; never file the same defect twice.

- **Test coverage finding (named, not buried).** The change adds core logic without a matching unit test -> `[Recommend]`; escalate to `[Must]` when critical path: a destructive operation, a migration, an undo path, or collision resolution. Core logic that cannot be tested without a GUI is itself the finding - cite the coupling
- **Test files are reviewed for coverage only.** For files that are themselves tests, the only finding to raise is a coverage gap: production logic in the diff that no test exercises. Anchor that finding to the untested production `file:line` and state the case to cover, not the test file. Do not review test code for style, structure, duplication, naming, or performance
- **Destructive operation without preview or undo.** An apply path that renames, moves, deletes, or overwrites without a dry-run path and a recorded reversal is a `[Must]`
- **Uncaught crash on a user-input path.** An unvalidated `Parse`, an index or `First()` on a possibly-empty result, or a `!` null suppression reachable from a filesystem result, parsed file content, or a user-supplied path is a finding. A guarded invariant with the invariant stated is not
- **`async void` and sync-over-async.** `async void` outside an event handler crashes the process on exception; `.Result`, `.Wait()`, or `.GetAwaiter().GetResult()` on the UI thread deadlocks. Both are findings wherever the diff adds them
- **Path handling.** A path built by string concatenation or interpolation instead of `Path.Combine`/`Path.Join`, or compared ordinally where the filesystem is case-insensitive or normalization-sensitive, is a finding
- **Blocking the UI thread.** Filesystem I/O, hashing, decode, or a synchronous SQLite query inside a command body, property getter, or event handler freezes the window
- **XAML binding integrity.** A binding to a property that does not exist is a finding either way - it breaks the build under compiled bindings, and fails silently at runtime under reflection bindings. A surface that opts back out in a project standardized on compiled bindings is a second, separate finding - a view or template without `x:DataType`, an `x:CompileBindings="False"`, or a reflection binding in an AOT-published app - because it reopens the silent-failure class
- **`unsafe` blocks and P/Invoke signatures** carry a comment stating the invariant that makes them sound. Absent that comment it is a finding regardless of whether the code is correct
- **Partial-batch reporting.** A batch that aborts on first error, or reports one aggregate success for a run with failures, hides data loss
- **Secrets and credentials.** No API keys, tokens, or signing material in source or committed config
- **Untrusted input at the edges.** Archive entry paths, image headers, parsed metadata, config values, and downloaded content are attacker-controllable and validated before use
- **Persisted-shape changes** ship a migration, and old builds stay installed - the change must be readable by whatever version the user has
- **Build configuration.** Use skill: `desktop-build-release` if the diff touches publish properties, trimming or AOT flags, or packaging. A publish-property change (`PublishAot`, `PublishTrimmed`, `InvariantGlobalization`, `ServerGarbageCollection`) is a Core finding regardless of whether `+perf` runs: trimming and AOT make reflection-dependent code fail in the published build and nowhere else, and `InvariantGlobalization` deletes collation and per-culture formatting app-wide - behaviour changes that never reproduce under `dotnet run`

### Phase C - Architecture Guardrails

Check layer violations and coupling: a dependency pointing the wrong way across a boundary, a module reaching past its seam, a shared surface gaining a consumer-specific concern.

**C# and Avalonia specific:**

- **The UI-free core boundary is the primary guardrail.** A core project that gains an Avalonia PackageReference (directly or transitively), or core logic written into a ViewModel command body or view code-behind, is the highest-value architectural finding this review makes
- **Project references:** a new dependency edge is a cascading change; an edge from core toward the UI is a violation
- **Preview and apply share one computation.** Two implementations that compute the same plan will drift, and the drift is invisible until it destroys data
- **State ownership:** view state lives in the UI project, domain state in the core. A ViewModel re-declaring a parallel copy of a core type instead of wrapping it is duplication that will diverge
- **Injection at the seam:** a core method calling `DateTime.Now`, `Random.Shared`, or reading an environment variable rather than receiving the capability as a parameter is untestable by construction
- **Interface boundaries around volatile dependencies:** a P/Invoke surface or a fast-moving package used directly across many classes is a migration hazard
- **Anemic core:** logic in the UI project while the core only holds data. Raise it as a finding only when a second file in this change set, or a prior commit read at `deep`, shows the same shape; otherwise record it in `Architecture Notes` as a watch item. `deep` widens where the evidence may be looked for; it never substitutes for it

**Multi-platform changes:** when a change affects a platform target, confirm the tier caveats were handled - Windows path length and reserved names, macOS Unicode normalization, per-platform config directories, and packaging differences.

### Phase D - AI-Generated Code Quality

- Check verbosity and over-engineering directly: overly complex methods, deep nesting, oversized classes, and over-abstraction
- Use skill: `desktop-overengineering-review` for interface-per-class with one implementer, generic parameters with one instantiation, MediatR for in-process calls, a builder for three properties, DI ceremony where a constructor argument suffices, premature `async`, and dead feature flags

**Additional AI smells** (not owned by the atomics above):

- Test verbosity (an integration test spinning a full solution for an assertion a unit test covers)
- Comment cruft (restating member names, doc comments on private helpers)
- Defensive null checks on values the nullable annotations already guarantee
- `ToList()` copies scattered where a single enumeration would serve

### Phase E - Maintainability

Check that failure paths added by the change leave a log or error record behind rather than failing silently. Use skill: `desktop-accessibility` for the baseline: automation names, peers, keyboard reachability, focus order, contrast, and non-colour-only signalling. Use skill: `desktop-i18n` if the diff touches a user-facing string or a `.resx` catalog - and specifically for the Unicode normalization difference between Windows and macOS, which a rename tool hits directly. Its single-locale scoping governs the hardcoded-string check below: in a project with no localization stack, report normalization, collation, and formatting defects and record hardcoded strings in `Maintainability Notes` rather than filing them.

**C#-specific:**

- Naming: PascalCase types and public members, camelCase locals and parameters, the project's field convention applied consistently
- Magic numbers extracted to named constants
- Hardcoded user-facing strings routed through the resource catalog
- Logging hygiene: no `Console.WriteLine` for diagnostics where `ILogger` is the project's mechanism, no per-item logging on a scan path
- Duplicated logic: the same computation in 3+ places becomes a shared method
- Dependency hygiene: a NuGet package for what the BCL already provides, or a heavy transitive tree for one method
- Accessibility baseline: **Avalonia has real screen-reader support - hold the change to it.** An interactive control the diff adds carries an `AutomationProperties.Name` (or a `LabeledBy` reference or text content the peer surfaces); a custom-drawn control overrides `OnCreateAutomationPeer`; every action is keyboard-reachable in visual order; no state is signalled by colour alone. Known gaps are surfaces to test around, not to excuse: a screen built on TextBox caret announcement (#9770) or DataGrid keyboard access (#10175) shipped without a screen-reader smoke test is a finding. Never file a finding claiming screen-reader support is impossible on this stack

### Step 6 - Delegate Extra Scopes in Parallel

Skip if scope is **Core only**. For each selected scope, spawn one independent subagent **in parallel** with the main thread. Use the **declared subagent for that scope** (`subagent_type` below) - do not infer the agent from the scope name:

| Scope | Skill | Subagent (`subagent_type`) |
| ----- | ----- | -------------------------- |
| +Perf | `task-desktop-review-perf` | `desktop-performance-engineer` |
| +Sec | `task-desktop-review-security` | `desktop-security-engineer` |

`Full` = 2 subagents.

**Subagent prompt contract:**

- The `review-precondition-check` handle (`mode`, `base`, `current_branch`, `reviewable`, `counts`, `notes`) + the pre-read diff (no re-running git)
- Depth level - the parent's resolved depth, which overrides the lens's own depth table. A lens invoked at `deep` returns its deep-only section even where its own trigger did not fire
- Pre-confirmed stack (C#/.NET) + the Avalonia version, solution layout, core-project presence, MVVM toolkit, and persistence package
- The reviewable-surface table above
- Return only the lens's subagent sections, per its own Step 11 contract: `## Findings` plus its non-finding and deep-only sections - never the full report template

**Failure isolation:** if a subagent fails or times out, continue with the rest. Note the missing scope in Summary.

### Step 7 - Synthesize (only if Step 6 ran)

Merge subagent findings into single Output Format. Do not append raw reports.

- Deduplicate cross-cutting findings (one entry citing all scopes)
- **Strongest intent wins** when labels differ for the same finding - across subagent reports, or between a lens finding and a Core finding: `Must` > `Recommend`. A merged finding notes its contributing scopes after the heading, e.g. `_(Core, +Sec)_`. Lens-local grouping headings (`### High Impact`) are dropped in the merge; the per-finding label stays
- Preserve `file:line` citations
- Map lens fields into the umbrella finding shape: a lens's Cost or Consequence line becomes Impact, its Budget or Adversary line becomes System Risk; Control type / Evidence is carried verbatim
- Order by intent, not scope
- Note missing scopes as `Scope incomplete: <scope>`
- Build Next Steps from the per-finding intent the subagents return - subagents return no Next Steps of their own; tag `[Implement]` / `[Delegate]`, re-sort by intent
- Preserve deep-only sections returned by subagents (+Perf's `## Measurement Plan`, +Sec's `## Dependency Graph Audit`) verbatim under `## Lens Detail` - they are not findings; the merge must not drop them
- Lens non-finding returns have fixed homes: +Perf's `## Unattributed` becomes the Summary's `**Unattributed:**` line; +Perf's `## Build Configuration` and `Not filed:` block, and +Sec's `## Dependency Triage` and `## Reviewed, Not Filed`, are preserved verbatim under `## Lens Detail` at any depth. A `routed: <file:line> -> <owner>` line inside a lens return is actioned - the parent files it with the named owner rather than only preserving it. Where the owner names an atomic's concern or the build rather than a phase, it becomes a Core finding

**Lens seams.** One defect can legitimately surface in two lenses: a directory re-walked on every keystroke is both security (a TOCTOU window widens) and perf (the I/O cost). Keep the integrity finding under +Sec and the throughput finding under +Perf, deduped to one line at the strongest intent. A hardcoded user-facing string is a Phase E maintainability finding, not +Sec, unless the string is itself a secret.

**Seam fallback.** Two lenses claiming one defect with no rule above: keep it under the lens whose fix removes the defect, and name the other lens in the finding. Never drop it because ownership was unclear.

**Cross-phase same root cause.** When one defect spans multiple phases (core logic in a command body that is also untestable), file the finding once under the phase where the root cause sits and reference its `file:line` from `Architecture Notes` or `Maintainability Notes`. Do not double-count.

### Step 8 - Write Report

Write the assembled report to `review-<branch>.md` at the reviewed repository's root, overwriting the file if it already exists - the one sanctioned write in a workflow that otherwise never touches the tree. Step 8 runs in every invocation mode - the umbrella owns the report file even when it is itself invoked as a subagent; only the lens reviews return findings instead of writing.

Derive `<branch>` from the handle's `current_branch`, sanitized for a filename: replace `/` and any character outside `[A-Za-z0-9_-]` with `-`, collapse consecutive `-` into one, strip leading and trailing `-`.

The file is this YAML frontmatter followed by the report body (raw Markdown, unfenced):

```yaml
---
branch: <branch>
scope_mode: working-tree | staged-only | last-commit
files: <N>
scope: core-only | +perf | +sec | full
depth: standard | deep
generated_at: <ISO 8601 UTC timestamp>
---
```

Field sources: `branch` = the handle's `current_branch` (unsanitized), `scope_mode` = the handle's `mode`, `files` = the handle's `counts.reviewable`, `scope` = Step 4's resolution mapped to the enum (`Core` -> `core-only`, `+Perf` -> `+perf`, `+Sec` -> `+sec`, `Full` -> `full`), `depth` = the resolved/auto-promoted depth, `generated_at` = the current UTC time in ISO 8601. The confirmation line's `<scope>` uses this frontmatter enum, not the Summary's.

After writing, print exactly one confirmation line:

```
Report written to review-<branch>.md (<N> files, scope: <scope>)
```

## Feedback Labels

| Label | Meaning |
| ----- | ------- |
| [Must] | Do not merge until this is fixed. |
| [Recommend] | Fix, or push back with reasoning. Cannot be silently acked. |

Every finding carries exactly one label: `[Must]` or `[Recommend]`. No other label is written.

## Output Format

The fence below delimits the template for display only - it is not part of the report. Emit the report body as raw Markdown so headings, tables, and lists render; never wrap the whole report in a code fence.

```markdown
## Summary

**Assessment:** Approve | Request Changes | Discuss _(Request Changes when any Must stands; Discuss when the blocking question is a tradeoff, not a defect; Approve otherwise)_
**Risk Level:** Low | Medium | High | Critical
**Risk Basis:** [one line: what breaks if this change is wrong, and how far it reaches]
**Stack Detected:** .NET <version> / Avalonia <version>
**Core Project:** <name> | none - logic in UI project | references Avalonia
**MVVM Toolkit:** CommunityToolkit.Mvvm | ReactiveUI | none
**Persistence:** <package> | none
**Packaging:** <tool> | none
**Publish Mode:** NativeAOT | trimmed | self-contained | framework-dependent | unknown
**Phases:** A-E | A-B _(low-risk short-circuit; C-E not run)_
**Scope:** Core | +Sec | +Perf | Full _(if auto-escalated: `auto-escalated from Core; signals: <list>`; if signals fired without escalating: `signals logged, not escalated: <list>`)_
**Depth:** standard | deep _(if auto-promoted: `auto-promoted from standard; Risk: <level>`)_
**Files:** `<N> reviewable` _(add `; <M> reviewed - <what was excluded>` only when the two differ)_
**Unattributed:** <carried from the +Perf lens's `## Unattributed`> _(omit when no lens returned one)_

## High-Impact Findings

### [Must] file:line

- Issue: [name the C# or Avalonia pattern]
- Impact: [user-visible or operational]
- System Risk: [why this is system-level]
- Fix: [concrete C# change]
- Control type: [`prevented | cost-raising only | accepted exposure`, carried verbatim from +Sec; omit the line when +Sec did not supply one]
- Evidence: [`measured | estimated | inferred`, carried verbatim from +Perf; omit the line when +Perf did not supply one]

### [Recommend] file:line
- Issue, Impact, Fix

## Architecture Notes

_Cross-cutting commentary. Reference findings by file:line._
- UI-free boundary:
- Project and coupling change:
- Drift detected:

## Maintainability Notes

- Over-engineering detected:
- Simplification opportunities:

## Key Takeaways

2-4 bullets on systemic impact.

## Next Steps

Each item tagged `[Implement]` or `[Delegate]`. Order: Must > Recommend; within a band, order by fix dependency - the item another item's fix depends on goes first.

1. **[Implement]** [Must] RenameService.cs:112 - apply path has no dry-run
2. **[Implement]** [Recommend] MainView.axaml:88 - results list realizes every row
3. **[Delegate]** [Recommend] [scope: task-desktop-test] - [one-line action]

_Omit if no actionable findings._

## Lens Detail _(omit when no subagent returned a non-finding section)_

_Non-finding sections returned by lens subagents - deep-only analysis, +Sec's `Dependency Triage` and `Reviewed, Not Filed` - preserved verbatim under their lens name. These are analysis, not findings - do not merge them into High-Impact Findings and do not drop them._

### <Lens> - <section title as returned>
[the subagent's section, unmodified]
```

**Omit empty sections.** No Must heading if there are none.

## Rules

- Review whole-change system impact, not file-by-file
- Lead with risk; line-level findings follow
- Apply C# and Avalonia conventions
- Actionable feedback with concrete C# changes
- Build output is excluded from findings; `.csproj` and `.axaml` are review surface
- Default Core; auto-escalate; honor `core-only`
- Delegate perf / security depth to subagents

## Self-Check

- [ ] `behavioral-principles` loaded (or accepted from parent)
- [ ] Stack confirmed; solution layout, core-project presence, Avalonia version, MVVM toolkit, persistence, packaging, and publish mode recorded in Summary
- [ ] Missing or Avalonia-referencing core project surfaced once in Summary, not re-raised per file
- [ ] Step 3 - `review-precondition-check` ran (or handle received); `git diff <base>` read once and reused; analysis restricted to the handle's `reviewable` paths
- [ ] Build output excluded from findings and signal scanning; `.csproj` and `.axaml` treated as review surface; reviewed count stated in Summary where it differs from the handle's `files`
- [ ] Step 4 - scope auto-escalation evaluated; promotion (or `core-only`) recorded; fired-but-not-escalated signals surfaced in Summary
- [ ] Step 5 - depth auto-promoted to `deep` when Risk is High/Critical
- [ ] Risk stated before any finding; a destructive-apply-path change never rated Low
- [ ] Phase B: atomic skills applied; test coverage, destructive preview/undo, crash paths, `async void` and sync-over-async, path handling, UI-thread blocking, XAML binding integrity, `unsafe`/P/Invoke justification, partial-batch reporting, secrets, untrusted input, migration, and build configuration checked
- [ ] Phase C: UI-free boundary, project references, preview/apply sharing, state ownership, injection seams, interface boundaries around volatile dependencies
- [ ] Phase D: complexity and over-engineering checked; `desktop-overengineering-review` applied
- [ ] Phase E: naming, magic numbers, logging hygiene, dependency hygiene, accessibility baseline; `desktop-i18n` applied where a string or catalog changed
- [ ] Automation names and peers checked on controls the change adds; known-gap surfaces flagged for a smoke test, never excused; no finding claims screen-reader support is impossible
- [ ] Missing tests raised as named finding (not buried)
- [ ] Every Must cites system risk
- [ ] Every finding has label + `file:line` + concrete fix
- [ ] Step 6 - extra scopes ran in parallel with the pre-resolved handle, pre-read diff, and detected project shape
- [ ] Step 7 - subagent findings merged into one intent-ordered list; no raw reports appended
- [ ] Lens seams deduped to one line at strongest intent; unlisted seams resolved by the fallback rather than dropped
- [ ] Lens non-finding returns routed to their fixed homes: deep-only sections, `Dependency Triage`, `Reviewed, Not Filed`, `Build Configuration`, and `Not filed:` blocks under `## Lens Detail`; `Unattributed` folded into Summary; `routed:` lines actioned with their named owner - none merged into findings, none dropped
- [ ] Failed / missing subagent scope noted as `Scope incomplete: <scope>`
- [ ] Next Steps produced with `[Implement]` / `[Delegate]` tags, ordered by intent
- [ ] Step 8 - report written to `review-<branch>.md` with the sanitized branch name and complete frontmatter (`branch`, `scope_mode`, `files`, `scope`, `depth`, `generated_at`); confirmation line printed

## Avoid

- State-changing git from this workflow (checkout/merge/pull/rebase/fetch/stash) - uncommitted work is the review subject and must not be disturbed.
- Raising findings against `bin/`, `obj/`, `publish/`, or generated code.
- Treating `.csproj` or `.axaml` as generated output and skipping them.
- Reviewing without reading the full diff first
- Flagging a project for a toolkit, error convention, or architecture it already standardized on
- Generic backend conventions where a C# desktop idiom exists ("move it off the UI thread", not "optimize the query")
- Claiming screen-reader support is impossible on Avalonia, or scoping assistive technology out of Phase E
- Proposing a GUI framework migration as a finding
- Vague feedback ("this could be better")
- Blocking on personal preference
- Running extra scopes when `core-only` was passed
- Duplicating perf / security depth here
- Sequential extra scopes that could parallelize
- Appending raw subagent reports
