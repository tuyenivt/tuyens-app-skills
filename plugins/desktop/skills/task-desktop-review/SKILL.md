---
name: task-desktop-review
description: Rust desktop code review - GUI-free core discipline, destructive-op safety, path handling, Iced state traps; spawns perf and security subagents.
agent: desktop-tech-lead
metadata:
  category: desktop
  tags: [rust, iced, desktop, code-review, working-tree, staff-review, multi-scope, workflow]
  type: workflow
user-invocable: true
---

> **Behavioral directive:** Load `Use skill: behavioral-principles` before executing this workflow.

# Rust Desktop Code Review

Staff-level Rust desktop review umbrella. Covers correctness, architecture, AI quality, maintainability. Coordinates perf / security subagents in parallel.

## When to Use

- Review of the current working-tree change set in a Rust desktop project
- Post-AI-generation quality gate
- Architecture drift detection - specifically, erosion of the GUI-free core boundary
- Pre-merge risk assessment

**Not for:** pre-implementation design (`task-desktop-implement`), single-error triage (`desktop-engineer`), single-scope reviews (delegate to perf/security).

## Depth

| Depth | When | Runs |
|-------|------|------|
| `standard` | Default | Phases A-E |
| `deep` | Architecture changes, post-incident, Principal sign-off | A-E + historical patterns + cross-change context |

**Auto-promote to `deep`:** After Phase A, if Risk is High/Critical, set depth to `deep` and surface `Depth auto-promoted: standard -> deep (Risk: <level>)`.

## Scope

| Scope | What runs |
|-------|-----------|
| Core | Phases A-E |
| + Perf | Core + `task-desktop-review-perf` subagent |
| + Sec | Core + `task-desktop-review-security` subagent |
| Full | Core + both in parallel |

Default: **Core with auto-escalation**. Pass `core-only` to suppress.

This client consumes API contracts rather than designing them: a finding that the app mishandles a contract belongs to Core, and a finding that the contract itself is wrong routes to the owning service's team.

**Auto-escalation signals:**

- **+Sec:** a new destructive operation (delete, overwrite, rename, move) reaching the filesystem; a user-supplied or config-supplied path joined without normalization; symlink traversal; a new `unsafe` block; a new FFI binding or `-sys` crate; a shell-out with interpolated arguments; a credential, token, or key read or written; an archive extraction path; a new network fetch or update-check endpoint; a dependency added with a known-advisory history
- **+Perf:** filesystem traversal added to a path that already walks; hashing or image decode on a per-item path; a blocking call inside `update` or a view function; an unbounded collection built from a scan; a widget column built per-item without virtualization; a new texture, thumbnail, or image cache with no eviction; `clone()` on a large owned value inside a loop; a new `rayon` pool or thread spawn
- **2+ categories -> Full**

Signals are matched syntactically, then checked for direction: a `clone()` removed, a scan moved off the UI thread, a cache gaining eviction, or a path normalization added is the fix, not the defect. Log it as `signal: <category> -> <path> (improving - not escalated)` and do not escalate on it alone.

There is no `+Ux` scope. Accessibility, adaptivity, and localization are reviewed at baseline depth in Phase E, and designed in `task-desktop-implement`; there is no dedicated UX lens to escalate to.

## Reviewable Surface

| Surface | Treatment |
| --- | --- |
| `.rs` under `src/`, `tests/`, `benches/`, `examples/` | full review surface |
| `Cargo.toml`, `build.rs`, `.cargo/config.toml`, `rust-toolchain.toml` | **review surface** - a dependency addition, a feature-flag change, a profile change, or a build script edit is a legitimate finding |
| `Cargo.lock` | reviewed for **what changed and why**, not line by line. A new transitive dependency, a yanked version, or an advisory-bearing bump is a finding; routine version churn is not |
| `.wgsl`, `.ftl`, `.sql`, packaging manifests, CI workflow files | review surface - shader logic, localization catalogs, migrations, and release configuration carry real defects |
| `target/`, `dist/`, generated bindings, `*.lock` files other than `Cargo.lock` | excluded - build output, never reviewed |
| Vendored third-party sources | excluded from style and structure findings; still in scope for security data-flow and version concerns |

A change set touching only build output is a no-op; say so rather than manufacturing findings. A `Cargo.lock`-only change set is **not** automatically a no-op - a new transitive dependency or an advisory-bearing bump is reviewable. Pure version churn with no new dependency and no advisory is the no-op case. A bump of `iced`, `wgpu`, or `winit` is never pure churn - Step 2 treats it as a framework migration.

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

Read `Cargo.toml`. If it is absent, stop - this workflow reviews Rust projects only.

Record: the workspace layout, whether a GUI-free core crate exists, the resolved Iced version, the async runtime, the persistence crate, and the packaging tool.

**Iced version.** Read the **resolved** version from `Cargo.lock` and record it. This project tracks latest rather than pinning a minor, so `Cargo.toml` holds a range and only the lockfile identifies what actually builds. Iced is pre-1.0 and its API moves between minor releases, so **a finding resting on Iced API surface names the version it assumes**. Where the resolved version differs from this plugin's guidance, note in Summary: `Detected iced <version>; this plugin's guidance targets 0.14.x - version-specific findings are annotated.` and review rather than stopping. Reduced confidence is concrete, not an adjective: no finding is downgraded on version grounds alone.

**A `Cargo.lock` bump of `iced`, `wgpu`, or `winit` in the change set is itself review surface.** Under a track-latest policy that bump is a framework migration arriving inside an ordinary diff - check the API usage the bump affects against the new version's surface rather than the old one - in a lock-only change set that usage lives in unchanged code, so search the workspace for the APIs the release notes changed instead of confining the check to the diff - a read check, no build is run - and that any behaviour the release notes changed is accounted for. When the release notes are unreachable (offline), enumerate the workspace's usage of the bumped crate instead and state in Summary that the check is enumeration-limited - never report it clean. A defect found this way is filed as a Phase B finding, so it survives the low-risk short-circuit.

**No GUI-free core.** If the workspace has no core crate, or the existing one depends on `iced`, note it once in Summary and treat it as the standing architectural condition Phase C reports against. Do not re-raise it per file.

### Step 3 - Resolve the Change Set

Use skill: `review-precondition-check`. Forward `--staged` if passed. If it fails fast, surface verbatim and stop.

The handle supplies `mode`, `base`, `current_branch`, `reviewable`, `counts`, and `notes`. Carry all of them forward - the report and the subagent prompts are built from them.

Read the diff content once and reuse:

- `git diff <base>` for the change body
- `git diff --name-status <base>` for the file list
- Untracked files in the handle's `reviewable` list have no content in `git diff` output - read each one directly; a new file is full review surface, not an empty diff

Restrict analysis to the handle's `reviewable` paths; binary and generated paths are excluded and never diffed.

The handle's generated-path conventions are stack-neutral. Apply this workflow's Reviewable Surface table on top of it: `target/` and `dist/` are excluded for analysis even where the handle lists them. Where the two numbers differ, write the handle's count and state the reviewed count once in Summary: `<N> reviewable; <M> reviewed - <what was excluded>`.

**Skip entirely** when invoked as subagent and parent passed handle + pre-read diff.

### Step 4 - Scope Auto-Escalation

Scan file list / diff for signals listed under **Scope**, ignoring excluded surfaces. Log each as `signal: <category> -> <file:line>`. Then:

- Zero signals or `core-only` -> Core
- One category -> add matching scope
- 2+ categories -> Full
- Explicit scope -> respect; still log signals

**Scope precedence:** user flag > firing signals.

Surface decision in Summary; if escalated, append `auto-escalated from Core; signals: <list>`. When signals fired without escalating (improving direction, or suppressed by `core-only`), append `signals logged, not escalated: <list>` - a reader must be able to tell a signal-free diff from one whose escalation was declined.

### Phase A - Risk Snapshot

Assess cross-cutting risk for the change set as a whole: what breaks if this is wrong, how far the failure reaches, and how much user data it touches. Weigh the surfaces the change reaches (the core crate, the apply path of a destructive operation, the migration layer, `Cargo.toml`, packaging configuration), the size reported in the handle's `counts`, and whether the change is on a critical path (a destructive filesystem operation, a schema migration, an undo path, a credential read, an update mechanism).

**A change touching the apply path of a destructive operation is never Low risk.** Data loss is unrecoverable for the user in a way a crash is not.

Output the risk level before any findings.

**Low-risk short-circuit:** if Risk is Low **and** the change does not touch architecture-relevant files (the core crate, any `Cargo.toml`, the destructive-apply path, the migration layer, `build.rs`, packaging configuration) **and** does not touch a localized string, a catalog file, or a path-handling routine, skip Phases C-E. Escalated scopes still run (a Step 4 signal fired for a reason); their merged findings join High-Impact Findings. The streamlined report contains Summary, High-Impact Findings (Phase B + any subagent findings), and Next Steps only; Step 8 still writes the report.

### Step 5 - Re-evaluate Depth After Phase A

If Risk is High / Critical, set depth to `deep` and surface promotion in Summary **before** Phases B-E. User-passed depth wins over auto-promotion.

### Phase B - Rust Correctness and Safety

Apply atomic skills; each owns canonical patterns:

- Use skill: `rust-language-patterns` - ownership, borrowing, lifetime and `clone()` smells, iterator misuse
- Use skill: `rust-error-handling` - error types, propagation, partial-failure reporting, panic paths
- Use skill: `desktop-batch-operations` if the diff touches a preview, apply, undo, or collision path
- Use skill: `desktop-filesystem-patterns` if the diff touches traversal, path joining, or path display
- Use skill: `iced-architecture-patterns` if the diff touches state, messages, or `update`
- Use skill: `iced-async-patterns` if the diff touches `Task`, subscriptions, progress, or cancellation
- Use skill: `desktop-concurrency-patterns` if the diff touches threads, `rayon`, channels, or shared state
- Use skill: `desktop-data-persistence` if the diff changes a persisted shape. Check installed-version impact directly: an installed older build must still work after this change - it must read what this one writes wherever possible

**Named checks.** Several overlap the atomics above by design - they are the highest-recurrence Rust desktop defects and must not be lost in a long atomic report. Emit one finding per defect: when an atomic already raised it, keep that finding; never file the same defect twice.

- **Test coverage finding (named, not buried).** The change adds core logic without a matching unit test -> `[Recommend]`; escalate to `[Must]` when critical path: a destructive operation, a migration, an undo path, or collision resolution. Core logic that cannot be tested without a GUI is itself the finding - cite the coupling
- **Test files are reviewed for coverage only.** For files that are themselves tests, the only finding to raise is a coverage gap: production logic in the diff that no test exercises. Anchor that finding to the untested production `file:line` and state the case to cover, not the test file. Do not review test code for style, structure, duplication, naming, or performance
- **Destructive operation without preview or undo.** An apply path that renames, moves, deletes, or overwrites without a dry-run path and a recorded reversal is a `[Must]`
- **Panic on a user-input path.** `unwrap`, `expect`, indexing, or integer overflow reachable from a filesystem result, a parsed config, or a user-supplied path is a finding. `expect` on a genuine invariant with a message stating the invariant is not
- **Path handling.** A path built through `String`, `to_string_lossy` for anything but display, or `format!("{}/{}")` instead of `Path::join` is a finding. Non-UTF-8 paths exist on both targets
- **Blocking the UI thread.** Filesystem I/O, hashing, decode, or a long loop inside `update` or a view function freezes the window
- **`unsafe` blocks** carry a comment stating the invariant that makes them sound. An `unsafe` block with no such comment is a finding regardless of whether it is correct
- **Partial-batch reporting.** A batch that aborts on first error, or reports one aggregate success for a run with failures, hides data loss
- **Secrets and credentials.** No API keys, tokens, or signing material in source or committed config
- **Untrusted input at the edges.** Archive entry paths, config values, deep-link parameters, and downloaded content are attacker-controllable and validated before use
- **Persisted-shape changes** ship a migration, and old builds stay installed - the change must be readable by whatever version the user has
- **Build configuration.** Use skill: `desktop-build-release` if the diff touches `Cargo.toml` profiles, `build.rs`, feature flags, or packaging. A release-profile change (LTO, `panic = "abort"`, `strip`, `opt-level`) is a Core finding regardless of whether `+perf` runs: `panic = "abort"` removes unwinding, so a recovery path that relied on catching a panic silently terminates the process in release and not in debug

### Phase C - Architecture Guardrails

Check layer violations and coupling: a dependency pointing the wrong way across a boundary, a module reaching past its seam, a shared surface gaining a consumer-specific concern.

**Rust and Iced specific:**

- **The GUI-free core boundary is the primary guardrail.** A core crate that gains an `iced` dependency, or core logic written into `update` or a view function, is the highest-value architectural finding this review makes
- **Crate dependencies:** a new dependency edge is a cascading change; an edge from core outward to the UI is a violation
- **Preview and apply share one computation.** Two implementations that compute the same plan will drift, and the drift is invisible until it destroys data
- **State ownership:** view state lives in the UI crate, domain state in the core. A message carrying a re-declared parallel struct instead of the core type is duplication that will diverge
- **Injection at the seam:** a core function reaching a real clock, a real filesystem root, or a global rather than receiving it as a parameter is untestable by construction
- **Trait boundaries around volatile dependencies:** an FFI binding or a fast-moving crate used directly across many modules is a migration hazard
- **Anemic core:** logic in the UI crate while the core only holds data. Raise it as a finding only when a second file in this change set, or a prior commit read at `deep`, shows the same shape; otherwise record it in `Architecture Notes` as a watch item. `deep` widens where the evidence may be looked for; it never substitutes for it

**Multi-platform changes:** when a change affects a platform target, confirm the tier caveats were handled - Windows path length and reserved names, macOS Unicode normalization, per-platform config directories, and packaging differences.

### Phase D - AI-Generated Code Quality

- Check verbosity and over-engineering directly: overly complex functions, deep nesting, oversized modules, and over-abstraction
- Use skill: `desktop-overengineering-review` for trait-per-struct with one implementer, generic parameters with one instantiation, a builder for three fields, a channel where a return value suffices, an `Arc<Mutex<>>` around data one thread owns, premature `async`, and dead feature flags

**Additional AI smells** (not owned by the atomics above):

- Test verbosity (an integration test spinning a full workspace for an assertion a unit test covers)
- Comment cruft (restating function names, doc comments on private helpers)
- Defensive `Option` handling on a value the type system already guarantees
- `clone()` scattered to silence the borrow checker rather than to express ownership

### Phase E - Maintainability

Check that failure paths added by the change leave a log or error record behind rather than failing silently. Use skill: `desktop-accessibility` for baseline keyboard navigation, focus order, contrast, and non-colour-only signalling. Use skill: `desktop-i18n` if the diff touches a user-facing string or a catalog - and specifically for the Unicode normalization difference between Windows and macOS, which a rename tool hits directly.

**Rust-specific:**

- Naming: `snake_case` functions and locals, `PascalCase` types and traits, `SCREAMING_SNAKE_CASE` constants
- Magic numbers extracted to named constants
- Hardcoded user-facing strings routed through localization
- Logging hygiene: no `println!` for diagnostics where `tracing` is the project's mechanism, no per-item logging on a scan path
- Duplicated logic: the same computation in 3+ places becomes a shared function
- Dependency hygiene: a new dependency for what `std` already provides, or a heavy transitive tree for one function
- Accessibility baseline: keyboard reachability, focus order, and no state signalled by colour alone. **Screen-reader support does not exist in Iced** - do not raise its absence as a finding, and do not propose an AccessKit integration the framework cannot consume

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
- Pre-confirmed stack (Rust) + the resolved Iced version, workspace layout, core-crate presence, async runtime, and persistence crate
- The reviewable-surface table above
- Return only the lens's subagent sections, per its own Step 11 contract: `## Findings` plus its non-finding and deep-only sections - never the full report template

**Failure isolation:** if a subagent fails or times out, continue with the rest. Note the missing scope in Summary.

### Step 7 - Synthesize (only if Step 6 ran)

Merge subagent findings into single Output Format. Do not append raw reports.

- Deduplicate cross-cutting findings (one entry citing all scopes)
- **Strongest intent wins** when labels differ across subagent reports for the same finding: `Must` > `Recommend`
- Preserve `file:line` citations
- Map lens fields into the umbrella finding shape: a lens's Cost or Consequence line becomes Impact, its Budget or Adversary line becomes System Risk; Control type / Evidence is carried verbatim
- Order by intent, not scope
- Note missing scopes as `Scope incomplete: <scope>`
- Build Next Steps from the per-finding intent the subagents return - subagents return no Next Steps of their own; tag `[Implement]` / `[Delegate]`, re-sort by intent
- Preserve deep-only sections returned by subagents (+Perf's `## Measurement Plan`, +Sec's `## Dependency Graph Audit`) verbatim under `## Lens Detail` - they are not findings; the merge must not drop them
- Lens non-finding returns have fixed homes: +Perf's `## Unattributed` becomes the Summary's `**Unattributed:**` line; +Sec's `## Dependency Triage` and `## Reviewed, Not Filed` are preserved verbatim under `## Lens Detail` at any depth

**Lens seams.** One defect can legitimately surface in two lenses: a directory re-walked on every keystroke is both security (a TOCTOU window widens) and perf (the I/O cost). Keep the integrity finding under +Sec and the throughput finding under +Perf, deduped to one line at the strongest intent. A hardcoded user-facing string is a Phase E maintainability finding, not +Sec, unless the string is itself a secret.

**Seam fallback.** Two lenses claiming one defect with no rule above: keep it under the lens whose fix removes the defect, and name the other lens in the finding. Never drop it because ownership was unclear.

**Cross-phase same root cause.** When one defect spans multiple phases (core logic in a view function that is also untestable), file the finding once under the phase where the root cause sits and reference its `file:line` from `Architecture Notes` or `Maintainability Notes`. Do not double-count.

### Step 8 - Write Report

Write the assembled report to `review-<branch>.md` in the current working directory, overwriting the file if it already exists. Step 8 runs in every invocation mode - the umbrella owns the report file even when it is itself invoked as a subagent; only the lens reviews return findings instead of writing.

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

Field sources: `branch` = the handle's `current_branch` (unsanitized), `scope_mode` = the handle's `mode`, `files` = the handle's `counts.reviewable`, `scope` = Step 4's resolution mapped to the enum (`Core` -> `core-only`, `+Perf` -> `+perf`, `+Sec` -> `+sec`, `Full` -> `full`), `depth` = the resolved/auto-promoted depth, `generated_at` = the current UTC time in ISO 8601.

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

**Assessment:** Approve | Request Changes | Discuss
**Risk Level:** Low | Medium | High | Critical
**Stack Detected:** Rust <edition> / iced <version>
**Core Crate:** <name> | none - logic in UI crate | depends on iced
**Async Runtime:** iced executor | tokio | smol | none
**Persistence:** <crate> | none
**Packaging:** <tool> | none
**Scope:** Core | +Sec | +Perf | Full _(if auto-escalated: `auto-escalated from Core; signals: <list>`; if signals fired without escalating: `signals logged, not escalated: <list>`)_
**Depth:** standard | deep _(if auto-promoted: `auto-promoted from standard; Risk: <level>`)_
**Files:** `<N> reviewable` _(add `; <M> reviewed - <what was excluded>` only when the two differ)_
**Unattributed:** <carried from the +Perf lens's `## Unattributed`> _(omit when no lens returned one)_

## High-Impact Findings

### [Must] file:line

- Issue: [name the Rust or Iced pattern]
- Impact: [user-visible or operational]
- System Risk: [why this is system-level]
- Fix: [concrete Rust change]
- Control type / Evidence: [carried verbatim from a lens subagent where it supplied one - `prevented | cost-raising only | accepted exposure`, `measured | estimated | inferred`; omit the line for Core findings]

### [Recommend] file:line
- Issue, Impact, Fix

## Architecture Notes

_Cross-cutting commentary. Reference findings by file:line._
- GUI-free boundary:
- Crate and coupling change:
- Drift detected:

## Maintainability Notes

- Over-engineering detected:
- Simplification opportunities:

## Key Takeaways

2-4 bullets on systemic impact.

## Next Steps

Each item tagged `[Implement]` or `[Delegate]`. Order: Must > Recommend; within a band, order by fix dependency - the item another item's fix depends on goes first.

1. **[Implement]** [Must] scanner.rs:112 - apply path has no dry-run
2. **[Implement]** [Recommend] view.rs:88 - subscription never released
3. **[Delegate]** [Recommend] [scope: server contract] - [one-line action]

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
- Apply Rust and Iced conventions
- Actionable feedback with concrete Rust changes
- Build output is excluded from findings; `Cargo.toml` and `Cargo.lock` are review surface
- Default Core; auto-escalate; honor `core-only`
- Delegate perf / security depth to subagents

## Self-Check

- [ ] `behavioral-principles` loaded (or accepted from parent)
- [ ] Stack confirmed; workspace layout, core-crate presence, Iced version, async runtime, persistence, and packaging recorded
- [ ] Iced version recorded and version-specific findings annotated inline
- [ ] Missing or `iced`-dependent core crate surfaced once in Summary, not re-raised per file
- [ ] Step 3 - `review-precondition-check` ran (or handle received); `git diff <base>` read once and reused; analysis restricted to the handle's `reviewable` paths
- [ ] Build output excluded from findings and signal scanning; `Cargo.toml`/`Cargo.lock` treated as review surface; reviewed count stated in Summary where it differs from the handle's `files`
- [ ] Step 4 - scope auto-escalation evaluated; promotion (or `core-only`) recorded; fired-but-not-escalated signals surfaced in Summary
- [ ] Step 5 - depth auto-promoted to `deep` when Risk is High/Critical
- [ ] Risk stated before any finding; a destructive-apply-path change never rated Low
- [ ] Phase B: atomic skills applied; test coverage, destructive preview/undo, panic paths, path handling, UI-thread blocking, `unsafe` justification, partial-batch reporting, secrets, untrusted input, migration checked
- [ ] Phase C: GUI-free boundary, crate edges, preview/apply sharing, state ownership, injection seams, trait boundaries around volatile dependencies
- [ ] Phase D: complexity and over-engineering checked; `desktop-overengineering-review` applied
- [ ] Phase E: naming, magic numbers, logging hygiene, dependency hygiene, accessibility baseline; `desktop-i18n` applied where a string or catalog changed
- [ ] Screen-reader absence not raised as a finding
- [ ] Missing tests raised as named finding (not buried)
- [ ] Every Must cites system risk
- [ ] Every finding has label + `file:line` + concrete fix
- [ ] Step 6 - extra scopes ran in parallel with the pre-resolved handle, pre-read diff, and detected project shape
- [ ] Step 7 - subagent findings merged into one intent-ordered list; no raw reports appended
- [ ] Lens seams deduped to one line at strongest intent; unlisted seams resolved by the fallback rather than dropped
- [ ] Lens non-finding returns routed to their fixed homes: deep-only sections, `Dependency Triage`, and `Reviewed, Not Filed` under `## Lens Detail`; `Unattributed` folded into Summary - none merged into findings, none dropped
- [ ] Failed / missing subagent scope noted as `Scope incomplete: <scope>`
- [ ] Next Steps produced with `[Implement]` / `[Delegate]` tags, ordered by intent
- [ ] Step 8 - report written to `review-<branch>.md` with the sanitized branch name and complete frontmatter (`branch`, `scope_mode`, `files`, `scope`, `depth`, `generated_at`); confirmation line printed

## Avoid

- State-changing git from this workflow (checkout/merge/pull/rebase/fetch/stash) - uncommitted work is the review subject and must not be disturbed.
- Raising findings against `target/`, `dist/`, or generated bindings.
- Treating `Cargo.toml` or `Cargo.lock` as generated output and skipping them.
- Reviewing `Cargo.lock` line by line instead of stating what dependency actually changed.
- Reviewing without reading the full diff first
- Flagging a project for a runtime, error crate, or architecture it already standardized on
- Reviewing the server's API contract here - it belongs to the owning service's team
- Generic backend conventions where a Rust desktop idiom exists ("move it out of `update`", not "optimize the query")
- Raising the absence of screen-reader support, which Iced cannot provide
- Proposing a GUI framework migration as a finding
- Vague feedback ("this could be better")
- Blocking on personal preference
- Running extra scopes when `core-only` was passed
- Duplicating perf / security depth here
- Sequential extra scopes that could parallelize
- Appending raw subagent reports
