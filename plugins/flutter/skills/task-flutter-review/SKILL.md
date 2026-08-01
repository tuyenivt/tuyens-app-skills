---
name: task-flutter-review
description: Flutter / Dart code review - rebuild cost, state discipline, disposal leaks, error mapping; spawns perf and security subagents.
agent: flutter-tech-lead
metadata:
  category: mobile
  tags: [flutter, dart, riverpod, code-review, working-tree, staff-review, multi-scope, workflow]
  type: workflow
user-invocable: true
---

> **Behavioral directive:** Load `Use skill: behavioral-principles` before executing this workflow.

# Flutter Code Review

Staff-level Flutter/Dart review umbrella. Covers correctness, architecture, AI quality, maintainability. Coordinates perf / security subagents in parallel.

## When to Use

- Review of uncommitted Flutter work in the working tree
- Post-AI-generation quality gate
- Architecture drift detection
- Pre-commit risk assessment

**Not for:** pre-implementation design (`task-flutter-implement`), single-error triage (`flutter-engineer`), single-scope reviews (delegate to perf/security).

## Depth

| Depth | When | Runs |
|-------|------|------|
| `standard` | Default | Phases A-E |
| `deep` | Architecture changes, post-incident, Principal sign-off | A-E + historical patterns + repo-wide context |

**Auto-promote to `deep`:** After Phase A, if Risk is High/Critical, set depth to `deep` and record it in the Summary's Depth field as `auto-promoted from standard; Risk: <level>`.

## Scope

| Scope | What runs |
|-------|-----------|
| Core | Phases A-E |
| + Perf | Core + `task-flutter-review-perf` subagent |
| + Sec | Core + `task-flutter-review-security` subagent |
| Full | Core + both in parallel |

Default: **Core with auto-escalation**. Pass `core-only` to suppress.

This client consumes API contracts rather than designing them: a finding that the client mishandles a contract belongs to Core, and a finding that the contract itself is wrong routes to the owning service or the architecture plugin.

**Auto-escalation signals:**

- **+Sec:** a token or credential written to `shared_preferences` rather than secure storage, an `http://` URL, a new deep-link or app-link handler, a new platform-channel handler, a WebView, a certificate-pinning change, a secret in source or in a committed `--dart-define` file, biometric or `local_auth` usage, a new runtime permission request
- **+Perf:** work added anywhere in the widget layer that belongs below it (I/O, sorting, parsing, allocation) - `build` is the common case, not the only one; a non-builder list constructor over a dynamic collection, a widget that lost or should have gained `const`, new image loading or decoding, a new isolate or `compute` call, a new `AnimationController`, a large in-memory collection held in state
- **2+ categories -> Full**

There is no `+Ux` scope. Adaptivity, accessibility, and localization are reviewed at baseline depth in Phase E, and designed in `task-flutter-implement`; there is no dedicated UX lens to escalate to.

## Generated Code

Generated files are build output, not review surface. Exclude from findings: `*.g.dart`, `*.freezed.dart`, `*.gr.dart`, `*.config.dart`, `*.mocks.dart`, and generated localization output. When a generated file changed, review the source that produces it - the annotated model, the route declaration, the ARB file - and cite that source's `file:line`. A change set containing only generated files is a no-op for review purposes; say so rather than manufacturing findings.

## Invocation

| Form | Meaning |
|------|---------|
| `/task-flutter-review` | Review the working tree (unstaged + staged + untracked) |
| `/task-flutter-review --staged` | Review staged changes only |

When the working tree is clean, `review-precondition-check` falls back to `last-commit` mode on its own.

**Never modify the working tree.** Read with `git diff` only; uncommitted work is the review subject.

## Workflow

### Step 1 - Behavioral Principles

Use skill: `behavioral-principles`. Accept parent's confirmation if invoked as subagent.

### Step 2 - Project Shape

Read `pubspec.yaml`. If it is absent or declares no `flutter` dependency, stop - this workflow reviews Flutter projects only.

Record: state management (Riverpod / Bloc / Provider / GetX / none), navigation (go_router / Navigator / auto_route), networking client, persistence store, and the platform target directories present.

If state management is not Riverpod, record it and note in the Summary: `Detected <X>; Riverpod-specific guidance does not apply.` Review against that library's own conventions rather than flagging it for not being Riverpod.

### Step 3 - Resolve the Change Set

Use skill: `review-precondition-check`. Forward `--staged` if passed. If it fails fast, surface verbatim and stop.

The handle supplies `mode`, `base`, `current_branch`, `reviewable`, `counts`, and `notes`. Review only the paths in `reviewable`; `counts.binary` and `counts.generated` are excluded already.

Resolve the review set in this order: take the paths listed in `reviewable`, drop any that match this workflow's wider generated-file list, and report what remains. That number is what `files` records; `counts.reviewable` is a hint, not the answer, and a disagreement with it is worth one line in the Summary.

Read the content once and reuse:

- `git diff <base>` for the change body
- `git diff --name-status <base>` for the per-file status

Untracked files in `reviewable` do not appear in `git diff <base>`; read those files directly and treat their full contents as added lines.

**Skip entirely** when invoked as subagent and parent passed handle + pre-read artifacts.

### Step 4 - Scope Auto-Escalation

Scan file list / diff for signals listed under **Scope**, ignoring generated files. Log each as `signal: <category> -> <file:line>`. Then:

- Zero signals or `core-only` -> Core
- One category -> add matching scope
- 2+ categories -> Full
- Explicit scope -> respect; still log signals

**Scope precedence:** user flag > firing signals.

Surface decision in Summary; if escalated, append `auto-escalated from Core; signals: <list>`.

### Phase A - Risk Snapshot

Assess the change set as a whole before any line-level finding. Weigh what the change can break: user-facing surface affected, data or schema touched, auth or payment paths involved, how far a failure propagates from the changed code, and whether a defect would be caught before release. Output a single risk level (Low | Medium | High | Critical) with a one-line rationale, before any findings.

**Low-risk short-circuit:** if Risk is Low **and** the change does not touch architecture-relevant files (app entry point, router configuration, auth or session state, the network client, a widget the diff shows imported by three or more features, the theme, on-device schema), skip Phases C-E. Escalated scopes still run (a Step 4 signal fired for a reason); their merged findings join High-Impact Findings. The streamlined report contains Summary, High-Impact Findings (Phase B + any subagent findings), and Next Steps only; Step 8 still writes the report.

### Step 5 - Re-evaluate Depth After Phase A

Apply the auto-promotion from the Depth table, surfacing it in the Summary **before** Phases B-E.

**Depth precedence:** user flag > this run's auto-promotion.

At `deep`, read `git log` on the touched files for the historical patterns the deep-only checks depend on; without that read, `deep` differs from `standard` only in name.

### Phase B - Flutter Correctness and Safety

Apply atomic skills; each owns canonical patterns:

- Use skill: `dart-language-patterns` - null safety discipline, exhaustive `switch` over sealed types, `late` misuse, async correctness
- Use skill: `flutter-widget-patterns` - `const`, keys, lifecycle, `BuildContext` across async gaps
- Use skill: `flutter-riverpod-patterns` if Step 2 detected Riverpod - provider scope, `ref` usage, disposal, side effects outside `build`. On any other library, review state holders against its own conventions and skip this skill rather than loading it for its generic subset
- Use skill: `flutter-error-handling` - typed failures, no silent swallows, error-to-UI-state mapping
- Use skill: `flutter-navigation-patterns` if the diff touches routes, guards, or deep links
- Use skill: `flutter-networking` if the diff touches the network layer
- Use skill: `flutter-i18n` if the diff touches ARB files, locale handling, or user-facing copy
- Use skill: `flutter-local-db-migration` if the diff changes the on-device schema. Check installed-version impact directly: whether an older app version already on a device still works after this change - on-device schema readable by the version the user has, serialized data still deserializable, and server contract expectations unchanged for that older build

**Named checks.** Several overlap the atomics above by design - they are the highest-recurrence Flutter defects and must not be lost in a long atomic report. Emit one finding per defect: when an atomic already raised it, keep that finding; never file the same defect twice.

- **Test coverage finding (named, not buried).** The change adds logic without a matching test -> `[Recommend]`; escalate to `[Must]` when critical path: auth or session handling, money or purchase flows, on-device schema migration, data sync or conflict resolution, permission gating
- **Test files are reviewed for coverage only.** For files that are themselves tests, the only finding to raise is a coverage gap: production logic in the diff that no test exercises. Anchor that finding to the untested production `file:line` and state the case to cover, not the test file. Do not review test code for style, structure, duplication, naming, or performance - a passing test with awkward setup is not a finding.
- **Disposal completeness.** Every `AnimationController`, `StreamSubscription`, `TextEditingController`, `ScrollController`, `FocusNode`, timer, and platform-channel listener created in a `State` is released in `dispose`. A missing release is a leak that survives navigation
- **`BuildContext` across async gaps.** Any `context` used after an `await` is guarded by a `mounted` check. This is the most common source of "widget disposed" crashes
- **Unawaited futures.** A future that is fired and not awaited either has an explicit reason or is a bug - errors from it bypass the caller's error handling entirely
- **Loading, error, empty, and populated states.** A screen that renders data renders all four, not just the happy path. Inferring emptiness from a null check alone is a finding; a screen with no reachable empty state says so rather than growing one
- **Secrets and endpoints.** No API keys, tokens, or credentials in source or in committed environment files. Client-side secrets are extractable from a shipped binary regardless of obfuscation
- **Untrusted input at the edges.** Deep-link parameters, platform-channel arguments, WebView messages, and notification payloads are attacker-controllable and validated before use
- **On-device schema changes** ship a migration, and old app versions stay installed - the change must be readable by whatever version the user has

### Phase C - Architecture Guardrails

Check the change set for layer violations and coupling drift: a dependency pointing the wrong way across a boundary, a module reaching past its public surface, or a new import that couples two previously independent areas.

**Flutter-specific:**

- **Layering:** presentation -> domain -> data. Widgets render and dispatch; state holders coordinate; repositories own I/O. A widget importing the HTTP client or the database directly is a violation
- **Repository interface in domain, implementation in data.** The UI depends on the abstraction so tests can substitute it
- **Dependency injection through providers, not singletons.** A global mutable instance reached from anywhere defeats override-based testing
- **Feature-module boundaries.** Cross-feature imports go through a shared layer, not sideways into another feature's internals
- **Navigation ownership.** Route decisions belong to the router configuration and guards, not scattered imperative pushes inside widgets
- **Theme and design tokens centralized** rather than per-widget colors and sizes
- **Platform-conditional code isolated** behind an abstraction rather than sprinkled `Platform.isX` branches through the widget tree
- **Anemic state holders (deep depth only):** logic in widgets while state holders only hold fields - flag for extraction. Raise it when the change set itself shows the split, or when `git log` on the touched files shows logic accumulating in the widget layer across commits. One file whose holder merely looks thin is not evidence on its own

**Multi-target changes:** when a change adds or affects a platform target, confirm the platform tier caveats were handled; use skill: `flutter-adaptive-responsive`.

### Phase D - AI-Generated Code Quality

- Check verbosity and over-engineering directly: overly complex methods, deep nesting, oversized files, and over-abstraction
- Use skill: `flutter-overengineering-review` for `StatefulWidget` where `Stateless` suffices, over-abstracted notifiers, single-implementation repository interfaces, freezed applied to everything, gratuitous providers, deep widget nesting, defensive null checks after non-nullable types, and custom `InheritedWidget` where a provider suffices

**Additional AI smells** (not owned by the atomics above):

- Test verbosity (a 30-line pump-and-settle setup for one assertion; golden tests for widgets with no visual complexity)
- Comment cruft (restating widget names, doc comments on private helpers)

### Phase E - Maintainability

Check naming against the language's own conventions (below), and check that failure paths added by the change leave a log or error record behind rather than failing silently. Use skill: `flutter-accessibility` for baseline label and target-size presence.

**Flutter-specific:**

- Naming: `lowerCamelCase` members, `UpperCamelCase` types, `snake_case` file names; no `Util` / `Manager` / `Helper` grab-bag classes
- Magic numbers and strings extracted to named constants or theme tokens
- Hardcoded user-facing strings routed through localization
- Widget `build` length: extract past roughly 50 lines or 3 levels of nesting; a deeply nested tree is a composition failure, not a formatting one
- Duplicated widget subtrees: the same tree in 3+ places becomes a shared widget
- Logging hygiene: no `print` in production paths; no PII or tokens in log output
- `flutter analyze` clean; `dart format` applied; `analysis_options.yaml` lints not suppressed inline without a reason

### Step 6 - Delegate Extra Scopes in Parallel

Skip if scope is **Core only**. For each selected scope, spawn one independent subagent **in parallel** with the main thread. Use the **declared subagent for that scope** (`subagent_type` below) - do not infer the agent from the scope name; a security review is not a `flutter-tech-lead` spawn:

| Scope | Skill                          | Subagent (`subagent_type`)     |
| ----- | ------------------------------ | ------------------------------ |
| +Perf | `task-flutter-review-perf`     | `flutter-performance-engineer` |
| +Sec  | `task-flutter-review-security` | `flutter-security-engineer`    |

`Full` = 2 subagents.

**Subagent prompt contract:**

- The resolved handle (`mode`, `base`, `current_branch`, `reviewable`) + the pre-read diff, so no subagent re-resolves the change set. Reading beyond the diff is expected and permitted - perf reads the unchanged file a regression rippled into, security reads manifests, entitlements, build config, and `git log -p` for a removed control
- Depth level, for +Perf only; +Sec has no depth axis and always runs every step
- Pre-confirmed stack (Flutter) + state management, navigation, networking, persistence, and platform targets. For +Sec add the fields its own Step 2 needs: secure-storage plugin, WebView presence, platform channels, deep-link mechanism, biometric plugin
- The generated-file exclusion list
- Return the Output Format body, minus the frontmatter and confirmation line

**Failure isolation:** if a subagent fails or times out, continue with the rest. Note the missing scope in Summary.

### Step 7 - Synthesize (only if Step 6 ran)

Merge subagent findings into single Output Format. Do not append raw reports.

- Deduplicate cross-cutting findings (one entry citing all scopes)
- **Strongest intent wins** when labels differ across subagent reports for the same finding: `Must` > `Recommend`
- Preserve `file:line` citations
- Order by intent, not scope
- Note missing scopes as `Scope incomplete: <scope>`
- Merge Next Steps with `[Implement]` / `[Delegate]` tags; re-sort by intent
- Preserve deep-only sections returned by subagents as their own section after Next Steps - they are not findings; the merge must not drop them

**Lens seams.** Two lenses reporting the same `file:line` are one finding or two depending on the defect, not on the lens count. One defect seen twice - a token cached in an oversized collection, where the storage risk and the memory cost are the same cache - merges into one entry at the strongest intent, naming both consequences in its Impact line. Two defects that happen to share a line - a hardcoded string that is both untranslated and unlabelled for a screen reader - stay separate, because each has its own fix. A hardcoded user-facing string is a Phase E maintainability finding, not +Sec, unless the string is itself a secret.

**Cross-phase same root cause.** When one defect spans multiple phases (a layering violation that also degrades testability), file the finding once under the phase where the root cause sits and reference its `file:line` from `Architecture Notes` or `Maintainability Notes`. Do not double-count.

### Step 8 - Write Report

Write the assembled report to `review-<branch>.md` in the current working directory, overwriting the file if it already exists.

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

Field sources: `branch` = the handle's `current_branch` (unsanitized), `scope_mode` = the handle's `mode`, `files` = the number of paths actually reviewed, `scope` = Step 4's resolution mapped to the enum (`Core` -> `core-only`, `+Perf` -> `+perf`, `+Sec` -> `+sec`, `Full` -> `full`), `depth` = the resolved/auto-promoted depth, `generated_at` = the current UTC time in ISO 8601.

In the confirmation line, `<branch>` is the sanitized form matching the filename, `<N>` is the same value as `files`, and `<scope>` is the frontmatter enum value.

Atomic skills emit `No <category> findings.` lines so the workflow knows a check ran. Those are working notes, not report content - they confirm coverage for the Self-Check and stay out of the report.

After writing, print exactly one confirmation line:

```
Report written to review-<branch>.md (<N> files, scope: <scope>)
```

## Feedback Labels

| Label        | Meaning                                                                  |
| ------------ | ------------------------------------------------------------------------ |
| [Must]       | Do not merge until this is fixed.                                        |
| [Recommend]  | Fix, or push back with reasoning. Cannot be silently acked.              |

Atomic skills grade findings on their own severity scales. Translate on the way in: `Blocker` and `Critical` and `High` -> `[Must]`; `Medium` and `Low` -> `[Recommend]`. A perf finding labelled `unverified` is `[Recommend]` whatever its impact. Nothing else is carried through - the atomic's severity word does not appear in the report.

**Assessment** follows from the labels: any `[Must]` -> `Request Changes`; only `[Recommend]` -> `Approve`, and say so plainly; no findings -> `Approve`. `Discuss` replaces `Request Changes` only when the `[Must]` findings would all be answered by one unresolved direction decision - a refactor whose target architecture is itself in question - and the Summary names that decision.

## Output Format

The fence below delimits the template for display only - it is not part of the report. Emit the report body as raw Markdown so headings, tables, and lists render; never wrap the whole report in a code fence.

```markdown
## Summary

**Assessment:** Approve | Request Changes | Discuss
**Risk Level:** Low | Medium | High | Critical
**Stack Detected:** Flutter <version> / Dart <version>
**State Management:** Riverpod | Bloc | Provider | GetX | none
**Navigation:** go_router | Navigator | auto_route | other
**Persistence:** <store> | none
**Platform Targets:** <list>
**Scope:** Core | +Sec | +Perf | Full _(if auto-escalated: `auto-escalated from Core; signals: <list>`)_
**Depth:** standard | deep _(if auto-promoted: `auto-promoted from standard; Risk: <level>`)_

## High-Impact Findings

### [Must] file:line

- Issue: [name the Flutter or Dart pattern]
- Impact: [user-visible or operational]
- System Risk: [why this is system-level]
- Fix: [concrete Dart change with code]

### [Recommend] file:line
- Issue, Impact, Fix

## Architecture Notes

_Cross-cutting commentary. Reference findings by file:line._
- Boundary impact:
- Coupling change:
- Drift detected:

## Maintainability Notes

- Over-engineering detected:
- Simplification opportunities:

## Key Takeaways

2-4 bullets on systemic impact.

## Next Steps

Each item tagged `[Implement]` or `[Delegate]`. Order: Must > Recommend.

1. **[Implement]** [Must] file:line - [one-line action]
2. **[Implement]** [Recommend] old_screen.dart:88 - controller never disposed
3. **[Delegate]** [Recommend] [scope: server contract] - [one-line action]

_Omit if no actionable findings._
```

**Omit empty sections.** No Must heading if there are none.

A review with no findings is Summary followed by the single line `No findings.`, and nothing else. A short-circuited review is Summary, whatever findings exist under High-Impact Findings, and Next Steps - Phases C-E contribute no sections.

## Rules

- Review whole-change system impact, not file-by-file
- Lead with risk; line-level findings follow
- Apply Dart and Flutter conventions (Effective Dart, Flutter style guidance)
- Actionable feedback with Dart code
- `dart format` applies; don't nitpick style
- Generated files are excluded from findings; review the source instead
- Default Core; auto-escalate; honor `core-only`
- Delegate perf / security depth to subagents

## Self-Check

- [ ] Step 1 - `behavioral-principles` loaded (or accepted from parent)
- [ ] Step 2 - `pubspec.yaml` confirms Flutter; state management, navigation, persistence, and platform targets recorded
- [ ] Non-Riverpod state management surfaced rather than flagged as a defect
- [ ] Step 3 - `review-precondition-check` ran (or handle received); diff read once against the handle's `base` and reused; untracked reviewable files read directly
- [ ] Generated files excluded from findings and from signal scanning
- [ ] Step 4 - scope auto-escalation evaluated; promotion (or `core-only`) recorded
- [ ] Step 5 - depth auto-promoted to `deep` when Risk is High/Critical
- [ ] Risk stated before any finding
- [ ] Phase B: atomic skills applied, conditional ones only where the diff fires them; test coverage, disposal, `BuildContext` across async gaps, unawaited futures, UI states, secrets, untrusted edge input checked
- [ ] Phases C-E ran, or the Phase A short-circuit fired and is recorded
- [ ] Phase C: layering, repository abstraction, DI seam, feature boundaries, navigation ownership
- [ ] Phase D: complexity and over-engineering checked; `flutter-overengineering-review` applied
- [ ] Phase E: naming, magic numbers, localization, build length, logging hygiene
- [ ] Missing tests raised as named finding (not buried)
- [ ] Atomic severities translated to `[Must]` / `[Recommend]`; Assessment follows from the labels
- [ ] Every Must cites system risk
- [ ] Every finding has label + `file:line` + Dart fix
- [ ] Step 6 - extra scopes ran in parallel with pre-resolved handle and detected project shape
- [ ] Step 7 - subagent findings merged into one intent-ordered list; no raw reports appended
- [ ] Lens seams (sec/perf overlap) deduped to one line at strongest intent
- [ ] Failed / missing subagent scope noted as `Scope incomplete: <scope>`
- [ ] Next Steps produced with `[Implement]` / `[Delegate]` tags, ordered by intent
- [ ] Step 8 - report written to `review-<branch>.md` with the sanitized branch name and complete frontmatter (`branch`, `scope_mode`, `files`, `scope`, `depth`, `generated_at`); confirmation line printed

## Avoid

- State-changing git from this workflow (fetch/checkout/merge/pull/rebase/stash) - uncommitted work is the review subject and must not be disturbed.
- Reviewing without reading the full diff first
- Flagging a project for using Bloc, Provider, or GetX instead of Riverpod
- Reviewing the server's API contract here - it belongs to the owning service or the architecture plugin
- Generic backend conventions where a Flutter idiom exists ("scope the rebuild", not "optimize the query")
- Nitpicking style where `dart format` applies
- Vague feedback ("this could be better")
- Blocking on personal preference
- Running extra scopes when `core-only` was passed
- Duplicating perf / security depth here
- Sequential extra scopes that could parallelize
- Appending raw subagent reports
