---
name: task-unity-review
description: Unity 2D code review - engine-free core discipline, lifecycle traps, prefab and scene hygiene; spawns perf and security subagents.
agent: unity-tech-lead
metadata:
  category: mobile
  tags: [unity, csharp, code-review, working-tree, staff-review, multi-scope, workflow]
  type: workflow
user-invocable: true
---

> **Behavioral directive:** Load `Use skill: behavioral-principles` before executing this workflow.

# Unity Code Review

Staff-level Unity 2D review umbrella. Covers correctness, architecture, AI quality, maintainability. Coordinates perf / security subagents in parallel.

## When to Use

- Review of the current working-tree change set in a Unity project
- Post-AI-generation quality gate
- Architecture drift detection - specifically, erosion of the engine-free rules boundary
- Pre-merge risk assessment

**Not for:** pre-implementation design (`task-unity-implement`), single-error triage (`unity-engineer`), new-system architecture (the architecture plugin, when installed), single-scope reviews (delegate to perf/security).

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
| + Perf | Core + `task-unity-review-perf` subagent |
| + Sec | Core + `task-unity-review-security` subagent |
| Full | Core + both in parallel |

Default: **Core with auto-escalation**. Pass `core-only` to suppress.

This client consumes API contracts rather than designing them: a finding that the game mishandles a contract belongs to Core, and a finding that the contract itself is wrong routes to the owning service's team.

**Auto-escalation signals:**

- **+Sec:** a reward, currency, or entitlement granted in client code; an IAP purchase handler; a rewarded-ad completion callback; a save write with no integrity check; an API key, signing key, or SDK secret in source or a committed config; a new deep-link handler; a remote-config value driving economy or unlocks; a new third-party SDK; a consent or ATT/GDPR/COPPA-related change
- **+Perf:** allocation inside `Update`, `FixedUpdate`, `LateUpdate`, or a per-frame callback; a new `Instantiate`/`Destroy` on a repeating path; LINQ or string concatenation in a hot path; a new sprite, texture import setting, or material variant; a new transparent or overlapping 2D layer; a `Find`/`GetComponent` call in a per-frame path; a new scene load; a large collection held in memory
- **2+ categories -> Full**

There is no `+Ux` scope. Accessibility, aspect-ratio adaptivity, and localization are reviewed at baseline depth in Phase E, and designed in `task-unity-implement`; there is no dedicated UX lens to escalate to.

## Reviewable Surface

Unity projects mix source, generated output, and authored assets. The three are not reviewed the same way.

| Surface | Treatment |
| --- | --- |
| `.cs` under `Assets/` (excluding `Assets/Plugins/`) | full review surface |
| `.unity`, `.prefab`, `.asset`, `.inputactions`, `.uxml`, `.uss` | **review surface** - a scene wiring change, a prefab override, or a config asset edit is a legitimate finding, cited at the asset path |
| `.meta` | not a finding surface on its own; a deleted or regenerated `.meta` that breaks references **is** a finding, cited at the affected asset |
| `Library/`, `Temp/`, `obj/`, `Build/`, `Logs/`, generated `*.csproj` / `*.sln` | excluded - build output, never reviewed |
| `Assets/Plugins/` and imported third-party SDK sources | excluded from style and structure findings; still in scope for security data-flow and version concerns |

Unlike a code-generation stack, Unity's assets are hand-authored and carry real defects. A diff touching only `.meta` files or build output is a no-op for review purposes; say so rather than manufacturing findings.

Asset diffs are YAML and often large and low-signal. Review what the change *means* (a reference rewired, a component removed, an override added), not the serialized line noise. When an asset diff is unreadable, say so and ask for the intent rather than guessing.

## Invocation

| Form | Meaning |
|------|---------|
| `/task-unity-review` | Review the working-tree change set (unstaged + staged + untracked) |
| `/task-unity-review --staged` | Review the staged change set only |

When the tree is clean, `review-precondition-check` falls back to the last commit.

**Never modify the working tree.** Read via `git diff` only; uncommitted changes are the review subject, not an obstacle.

## Workflow

### Step 1 - Behavioral Principles

Use skill: `behavioral-principles`. Accept parent's confirmation if invoked as subagent.

### Step 2 - Stack and Project Shape

Read `ProjectSettings/ProjectVersion.txt`. If it is absent, stop - this workflow reviews Unity projects only.

Record: engine version from `ProjectVersion.txt`, render pipeline, input system, UI system, persistence, and whether Addressables is present.

**Engine floor.** The plugin targets Unity 6.3 LTS (`6000.3.x`) and newer. Compare numerically by component, not by string prefix. Below the floor, note in Summary: `Detected <version>; this plugin targets 6000.3.x and newer - version-specific guidance may not apply.` and review at reduced confidence rather than stopping; a review that refuses to run helps nobody mid-change. Above the floor (`6000.5.x`), proceed normally.

**UI system.** If the diff's UI is uGUI, note once in Summary: `uGUI detected; UI Toolkit guidance does not apply.` Review uGUI changes for correctness and layering only - do not flag uGUI for not being UI Toolkit, and do not propose a migration.

### Step 3 - Resolve the Change Set

Use skill: `review-precondition-check`. Forward `--staged` if passed. If it fails fast, surface verbatim and stop.

The handle supplies `mode`, `base`, `current_branch`, `reviewable`, `counts`, and `notes`. Carry all of them forward - the report and the subagent prompts are built from them.

Read the diff content once and reuse:

- `git diff <base>` for the change body
- `git diff --name-status <base>` for the file list

Restrict analysis to the handle's `reviewable` paths; binary and generated paths are excluded and never diffed.

**Skip entirely** when invoked as subagent and parent passed handle + pre-read diff.

### Step 4 - Scope Auto-Escalation

Scan file list / diff for signals listed under **Scope**, ignoring excluded surfaces. Log each as `signal: <category> -> <file:line>`. Then:

- Zero signals or `core-only` -> Core
- One category -> add matching scope
- 2+ categories -> Full
- Explicit scope -> respect; still log signals

**Scope precedence:** user flag > firing signals.

Surface decision in Summary; if escalated, append `auto-escalated from Core; signals: <list>`.

### Phase A - Risk Snapshot

Assess cross-cutting risk for the change set as a whole: what breaks if this is wrong, how far the failure reaches, and how many players or systems it touches. Weigh the surfaces the change reaches (the rules assembly, the bootstrap scene, shared prefabs, the save layer, `ProjectSettings/`), the size reported in the handle's `counts`, and whether the change is on a critical path (save migration, purchase or reward grant, economy math, progression unlock, scoring).

Output the risk level before any findings.

**Low-risk short-circuit:** if Risk is Low **and** the change does not touch architecture-relevant files (the rules assembly, any `.asmdef`, the bootstrap scene, a shared prefab, the save layer, the composition root, `ProjectSettings/`), skip Phases C-E. Escalated scopes still run (a Step 4 signal fired for a reason); their merged findings join High-Impact Findings. The streamlined report contains Summary, High-Impact Findings (Phase B + any subagent findings), and Next Steps only; Step 8 still writes the report.

### Step 5 - Re-evaluate Depth After Phase A

If Risk is High / Critical, set depth to `deep` and surface promotion in Summary **before** Phases B-E. User-passed depth wins over auto-promotion.

### Phase B - Unity Correctness and Safety

Apply atomic skills; each owns canonical patterns:

- Use skill: `csharp-unity-patterns` - the `UnityEngine.Object` lifetime `==` overload, allocation in hot paths, async correctness
- Use skill: `unity-monobehaviour-lifecycle` - callback order, initialization traps, static state across Play sessions, coroutine lifetime
- Use skill: `unity-architecture-patterns` - the engine-free rules boundary, injectability, global lookups
- Use skill: `unity-serialization-prefabs` if the diff touches serialized fields, prefabs, scenes, or `.meta` files
- Use skill: `unity-2d-gameplay-patterns` if the diff touches board, turn, or rule logic
- Use skill: `unity-save-persistence` if the diff changes the save schema. Check installed-version impact directly: an installed older build must still work after this change - it must read what this one writes and vice versa, serialized data must stay deserializable, and its server contract expectations must still hold

**Named checks.** Several overlap the atomics above by design - they are the highest-recurrence Unity defects and must not be lost in a long atomic report. Emit one finding per defect: when an atomic already raised it, keep that finding; never file the same defect twice.

- **Test coverage finding (named, not buried).** The change adds rule logic without a matching EditMode test -> `[Recommend]`; escalate to `[Must]` when critical path: save migration, purchase or reward grant, economy math, progression unlock, or scoring. Rules that cannot be tested without Play mode are themselves the finding - cite the engine coupling
- **Test files are reviewed for coverage only.** For files that are themselves tests, the only finding to raise is a coverage gap: production logic in the diff that no test exercises. Anchor that finding to the untested production `file:line` and state the case to cover, not the test file. Do not review test code for style, structure, duplication, naming, or performance
- **Destroyed-object access.** `?.`, `??`, and `is null` bypass Unity's lifetime `==` overload, so a destroyed object passes those checks and then throws. Any of them applied to a `UnityEngine.Object` is a finding
- **Await on a destroyed object.** Any `await` inside a `MonoBehaviour` that resumes and touches `this`, a component, or a `GameObject` needs a cancellation path tied to the object's lifetime
- **Engine-free boundary.** Rule logic added to a `MonoBehaviour` when a rules assembly exists is architectural drift, not a style preference
- **Loading, error, and empty states.** Any screen driven by async work renders all of them, not just the happy path
- **Secrets and grants.** No API keys, signing keys, or SDK secrets in source or committed config. No currency, reward, or entitlement granted by client code alone
- **Untrusted input at the edges.** Deep-link parameters, remote-config values, and downloaded content are attacker-controllable and validated before use
- **Save schema changes** ship a migration, and old builds stay installed - the change must be readable by whatever version the user has

### Phase C - Architecture Guardrails

Check layer violations and coupling: a dependency pointing the wrong way across a boundary, a module reaching past its seam, a shared surface gaining a consumer-specific concern.

**Unity-specific:**

- **The engine-free rules boundary is the primary guardrail.** A rules assembly that gains a `UnityEngine` reference, or rule logic written into a `MonoBehaviour`, is the highest-value architectural finding this review makes
- **Assembly definitions:** a new `.asmdef` reference is a cascading change; a reference from rules outward is a violation
- **`MonoBehaviour` only for engine hooks.** A class with no lifecycle callback, serialized field, coroutine, or collision hook does not need one
- **ScriptableObjects hold configuration, not runtime state.** Runtime mutation of a SO persists in the editor and diverges from a build
- **Dependency injection at the seam,** not `GameObject.Find` / `FindObjectOfType` / static singletons reached from inside logic
- **Scene and prefab ownership:** shared prefabs and the bootstrap scene are global surfaces; changes to them affect every consumer
- **Anemic rules layer (deep depth only):** logic in presenters while the rules layer only holds data - flag for extraction. Do not raise on a single change alone

**Multi-platform changes:** when a change affects a platform target, confirm the tier caveats were handled - WebGL has no threads by default and no durable `System.IO`; desktop changes input modality and window handling.

### Phase D - AI-Generated Code Quality

- Check verbosity and over-engineering directly: overly complex methods, deep nesting, oversized files, and over-abstraction
- Use skill: `unity-overengineering-review` for `MonoBehaviour` without engine hooks, ECS or a DI container for a small casual game, ScriptableObject-per-constant, event buses for two callers, pooling one instance, single-implementer interfaces, and dead feature flags

**Additional AI smells** (not owned by the atomics above):

- Test verbosity (a PlayMode test with full scene setup for an assertion an EditMode test covers)
- Comment cruft (restating method names, doc comments on private helpers)
- Defensive null checks on serialized fields the inspector guarantees

### Phase E - Maintainability

Check that failure paths added by the change leave a log or error record behind rather than failing silently. Use skill: `unity-accessibility` for baseline touch-target, contrast, and non-colour-only signalling presence.

**Unity-specific:**

- Naming: `PascalCase` types and methods, `camelCase` locals and private fields, `PascalCase` for serialized fields shown in the inspector (the inspector prettifies names, so field naming is user-visible)
- Magic numbers extracted to named constants or config assets
- Hardcoded user-facing strings routed through localization
- `Debug.Log` on per-frame paths, or verbose logging left in release without `[Conditional]`
- Duplicated prefab or UXML subtrees: the same structure in 3+ places becomes a shared prefab or a reusable UI component
- Asset hygiene: no orphaned `.meta`, no missing script references, no unintended prefab overrides
- Accessibility baseline: touch targets meet minimum size, no state signalled by colour alone

### Step 6 - Delegate Extra Scopes in Parallel

Skip if scope is **Core only**. For each selected scope, spawn one independent subagent **in parallel** with the main thread. Use the **declared subagent for that scope** (`subagent_type` below) - do not infer the agent from the scope name; a security review is not a `unity-tech-lead` spawn:

| Scope | Skill | Subagent (`subagent_type`) |
| ----- | ----- | -------------------------- |
| +Perf | `task-unity-review-perf` | `unity-performance-engineer` |
| +Sec | `task-unity-review-security` | `unity-security-engineer` |

`Full` = 2 subagents.

**Subagent prompt contract:**

- The `review-precondition-check` handle (`mode`, `base`, `current_branch`, `reviewable`, `counts`, `notes`) + the pre-read diff (no re-running git)
- Depth level
- Pre-confirmed stack (Unity) + engine version, render pipeline, input system, UI system, persistence
- The reviewable-surface table above
- Return findings in own Output Format

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

**Lens seams.** One defect can legitimately surface in two lenses: a save re-serialized every frame is both security (an unverified write is a tamper surface) and perf (the allocation and I/O cost). Keep the integrity finding under +Sec and the frame-cost finding under +Perf, deduped to one line at the strongest intent. A hardcoded user-facing string is a Phase E maintainability finding, not +Sec, unless the string is itself a secret.

**Seam fallback.** Two lenses claiming one defect with no rule above: keep it under the lens whose fix removes the defect, and name the other lens in the finding. Never drop it because ownership was unclear.

**Cross-phase same root cause.** When one defect spans multiple phases (rule logic in a `MonoBehaviour` that is also untestable), file the finding once under the phase where the root cause sits and reference its `file:line` from `Architecture Notes` or `Maintainability Notes`. Do not double-count.

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

No `[Question]`, `[Suggestion]`, `[Consider]`, `[Nit]`, `[Nitpick]`, or `[Praise]` - if it isn't `[Must]` or `[Recommend]`, don't write it down.

## Output Format

The fence below delimits the template for display only - it is not part of the report. Emit the report body as raw Markdown so headings, tables, and lists render; never wrap the whole report in a code fence.

```markdown
## Summary

**Assessment:** Approve | Request Changes | Discuss
**Risk Level:** Low | Medium | High | Critical
**Stack Detected:** Unity <internal version> (Unity <marketing version>)
**Render Pipeline:** URP | HDRP | Built-in | unknown
**Input:** Input System | legacy | both
**UI:** UI Toolkit | uGUI | both | none
**Persistence:** <store> | none
**Scope:** Core | +Sec | +Perf | Full _(if auto-escalated: `auto-escalated from Core; signals: <list>`)_
**Depth:** standard | deep _(if auto-promoted: `auto-promoted from standard; Risk: <level>`)_

## High-Impact Findings

### [Must] file:line

- Issue: [name the Unity or C# pattern]
- Impact: [player-visible or operational]
- System Risk: [why this is system-level]
- Fix: [concrete C# or asset change]

### [Recommend] file:line
- Issue, Impact, Fix

## Architecture Notes

_Cross-cutting commentary. Reference findings by file:line._
- Engine-free boundary:
- Assembly and coupling change:
- Drift detected:

## Maintainability Notes

- Over-engineering detected:
- Simplification opportunities:

## Key Takeaways

2-4 bullets on systemic impact.

## Next Steps

Each item tagged `[Implement]` or `[Delegate]`. Order: Must > Recommend.

1. **[Implement]** [Must] file:line - [one-line action]
2. **[Implement]** [Recommend] BoardView.cs:88 - subscription never released
3. **[Delegate]** [Recommend] [scope: server contract] - [one-line action]

_Omit if no actionable findings._

## Lens Detail _(deep depth only; omit at standard, and omit when no subagent returned one)_

_Deep-only sections returned by lens subagents, preserved verbatim under their lens name. These are analysis, not findings - do not merge them into High-Impact Findings and do not drop them._

### <Lens> - <section title as returned>
[the subagent's section, unmodified]
```

**Omit empty sections.** No Must heading if there are none.

## Rules

- Review whole-change system impact, not file-by-file
- Lead with risk; line-level findings follow
- Apply C# and Unity conventions
- Actionable feedback with C# code or a concrete asset change
- Build output is excluded from findings; authored assets are review surface
- Default Core; auto-escalate; honor `core-only`
- Delegate perf / security depth to subagents

## Self-Check

- [ ] `behavioral-principles` loaded (or accepted from parent)
- [ ] Stack confirmed; engine version, render pipeline, input, UI system, and persistence recorded
- [ ] Engine version compared numerically against the `6000.3.x` floor; below-floor noted rather than silently assumed
- [ ] uGUI, where present, surfaced once rather than flagged as a defect
- [ ] Step 3 - `review-precondition-check` ran (or handle received); `git diff <base>` read once and reused; analysis restricted to the handle's `reviewable` paths
- [ ] Build output excluded from findings and signal scanning; authored assets treated as review surface
- [ ] Step 4 - scope auto-escalation evaluated; promotion (or `core-only`) recorded
- [ ] Step 5 - depth auto-promoted to `deep` when Risk is High/Critical
- [ ] Risk stated before any finding
- [ ] Phase B: atomic skills applied; test coverage, destroyed-object access, await lifetime, engine-free boundary, UI states, secrets and grants, untrusted input, save migration checked
- [ ] Phase C: rules boundary, assembly references, MonoBehaviour necessity, SO mutation, injection seams, shared-asset ownership
- [ ] Phase D: complexity and over-engineering checked; `unity-overengineering-review` applied
- [ ] Phase E: naming, magic numbers, localization, logging hygiene, asset hygiene, accessibility baseline
- [ ] Missing tests raised as named finding (not buried)
- [ ] Every Must cites system risk
- [ ] Every finding has label + `file:line` + concrete fix
- [ ] Step 6 - extra scopes ran in parallel with the pre-resolved handle, pre-read diff, and detected project shape
- [ ] Step 7 - subagent findings merged into one intent-ordered list; no raw reports appended
- [ ] Lens seams (sec/perf, perf/rendering overlap) deduped to one line at strongest intent; unlisted seams resolved by the fallback rather than dropped
- [ ] Deep-only sections returned by subagents preserved under `## Lens Detail`, not merged into findings and not dropped
- [ ] Failed / missing subagent scope noted as `Scope incomplete: <scope>`
- [ ] Next Steps produced with `[Implement]` / `[Delegate]` tags, ordered by intent
- [ ] Step 8 - report written to `review-<branch>.md` with the sanitized branch name and complete frontmatter (`branch`, `scope_mode`, `files`, `scope`, `depth`, `generated_at`); confirmation line printed

## Avoid

- State-changing git from this workflow (checkout/merge/pull/rebase/fetch/stash) - uncommitted work is the review subject and must not be disturbed.
- Raising findings against `Library/`, `Temp/`, `obj/`, `Build/`, or generated `*.csproj` / `*.sln`.
- Treating an authored asset (`.unity`, `.prefab`, `.asset`, `.uxml`) as generated output and skipping it.
- Quoting serialized YAML line noise instead of stating what the asset change means.
- Emitting `[Question]`, `[Suggestion]`, `[Consider]`, `[Nit]`, `[Nitpick]`, or `[Praise]` labels.
- Reviewing without reading the full diff first
- Flagging a project for using uGUI, a DI container, or ECS it already standardized on
- Reviewing the server's API contract here - it belongs to the owning service's team
- Generic backend conventions where a Unity idiom exists ("move it off the rules assembly", not "optimize the query")
- Vague feedback ("this could be better")
- Blocking on personal preference
- Running extra scopes when `core-only` was passed
- Duplicating perf / security depth here
- Sequential extra scopes that could parallelize
- Appending raw subagent reports
