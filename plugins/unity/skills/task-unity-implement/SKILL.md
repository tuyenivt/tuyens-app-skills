---
name: task-unity-implement
description: End-to-end Unity 2D feature implementation - engine-free rules core, ScriptableObject config, MonoBehaviour wiring, UI Toolkit screens, save, tests.
agent: unity-engineer
metadata:
  category: mobile
  tags: [unity, csharp, 2d, gameplay, ui-toolkit, scriptableobject, save, feature, implementation, workflow]
  type: workflow
user-invocable: true
---

> **Behavioral directive:** Load `Use skill: behavioral-principles` before executing this workflow.

# Implement Unity 2D Feature

## When to Use

End-to-end Unity 2D feature work: rules core + runtime wiring + presentation + persistence + tests in one pass. STEPS 3-7 write the files; the report lists what was written. STEP 2 is the gate that decides what gets written, and nothing lands before it is approved.

Not for: a sprite or palette swap, an inspector value tweak, a scene-layout nudge, or engine-version upgrade work. A change that *only* restyles an existing screen is a UI edit, not this workflow.

It does apply when presentation work also adds behaviour, locales, or persistence - the steps that produce nothing are written as skipped rather than being a reason to decline. A feature built entirely on existing rules is an anticipated shape, not an out-of-scope one.

## Rules

- **Game rules are plain C# with no `UnityEngine` dependency**, in a rules assembly whose `.asmdef` cannot reference the engine. Every other decision in this workflow follows from that one
- `MonoBehaviour` only where an engine hook is needed - lifecycle callback, collision, coroutine, inspector wiring. A class needing none is a plain class
- ScriptableObjects hold authored configuration, never mutable runtime state
- Time and randomness reach the rules layer as injected `IClock` and `IRandom` - each only where the feature actually uses it, and its absence said rather than a seam with no implementer invented. No `DateTime.Now`, `Time.deltaTime`, or `UnityEngine.Random` inside a rule
- Never `?.`, `??`, or `is null` on a `UnityEngine.Object` - use `== null` / `!= null`
- Every screen with an async or failable source renders loading, error, and empty, not just the happy path
- User-facing strings resolve through a localization key. Where the project has no localization system, keep every string in one holder type so extraction is mechanical, and report that at STEP 5 as a stated deviation rather than installing a localization system this feature did not ask for
- Any save-shape change ships a schema version bump and a migration, because old saves exist on installed devices
- Each step completes before the next; design approved before code

## Workflow

### STEP 1 - DETECT AND GATHER

**Engine floor gate.** Read `ProjectSettings/ProjectVersion.txt` and compare `m_EditorVersion` numerically against the **Unity 6.3 LTS floor (`6000.3.x`)**. Below the floor, state the detected version and the floor, and **STOP** - this is a detect-and-report boundary, not a degradation path. Do not emit guidance the project cannot compile. Above the floor (`6000.5.x` and later) proceeds normally; the gate is a minimum, not an equality check. When `ProjectVersion.txt` is unreadable or absent, say so and ask for the editor version rather than assuming one.

Then confirm from `Packages/manifest.json` and project settings:

| Concern | Confirm | If it differs |
| --- | --- | --- |
| Render pipeline | URP 2D (Renderer 2D asset assigned) | Built-in RP: state the mismatch; 2D lights guidance does not apply |
| Input | Input System package | Legacy `Input` manager: flag it, do not rewrite the project |
| UI | UI Toolkit | **uGUI: report the UI portion out of scope and stop it.** Continue the non-UI steps |
| Persistence | JSON in `Application.persistentDataPath` | PlayerPrefs as the primary store: report it at STEP 6 as a finding with its severity, and say whether this feature's data goes into it or into a new store |
| Content loading | Addressables | `Resources/`: report at STEP 6 as a standing project condition, and say whether this feature adds to it |
| Rules assembly | An `.asmdef` with no `UnityEngine` reference | **None anywhere, or the existing one references the engine: report at STEP 2 as the blocking design finding.** Do not build rules into MonoBehaviours around it |

Ask before writing code, grouped so each cluster surfaces its own follow-ups. Skip clusters the feature does not touch:

**Feature**
1. Genre and core loop (board/turn, cascade, wave, idle accrual, quiz round)
2. Entry point: new scene, existing screen, popup over gameplay

**Rules**
3. State the rules layer owns, and what a single move or step produces
4. Legality, scoring, and termination conditions
5. Whether undo, replay, or a daily seed is required

**Data**
6. What persists, and whether the save shape changes
7. Static content banks involved (question set, level pack, balance table)

**Presentation**
8. Screens, HUD, and what each renders while loading or on failure
9. Animation and juice, and what carries the information without it

**Server** - skip only when the feature never leaves the device
10. Which endpoint the feature calls, and whether the server or the client is authoritative for each value it exchanges
11. What happens when the two disagree, and what the client does while offline

**Reach**
12. Platform tiers shipping this feature (mobile primary; desktop secondary; WebGL tertiary)
13. Locales and accessibility expectations beyond the defaults

Ask targeted questions for gaps. Do not guess.

### STEP 2 - DESIGN (APPROVAL GATE)

Use skill: `unity-architecture-patterns` for the assembly plan and the injection seams. Use skill: `unity-overengineering-review` to check the design against the feature's actual size before it is built.

Present the plan:

- **Assembly plan**: which types go in the engine-free rules assembly, which in runtime, which in the EditMode test assembly, and each `.asmdef`'s references
- **Rules model**: state type, move type, result type, phase enum, and the `IClock`/`IRandom` seams
- **Scene and prefab plan**: new scenes, prefabs and variants, the composition root, and what each MonoBehaviour exists for
- **Config ScriptableObjects**: which authored values become assets, and why each is configuration rather than runtime state
- **UI plan**: screens, the navigation stack position of each, and the loading/error/empty presentation per async surface
- **Save impact**: fields added or reshaped, the schema version bump, and the migration step
- **Content impact**: banks touched, identifier stability, and loading strategy
- **Server exchange** (when the feature calls one): which side is authoritative for each value, the conflict rule when they disagree, the offline behaviour, and how an installed older build survives this contract - the client consumes the contract rather than designing it, so a contract that must change is a finding routed to the owning service, not a thing to build around
- **Platform tiers and locales** in scope, with each tier's constraints on this feature's persistence, threading, and durability stated here rather than discovered later

Where the design deviates from this skill's defaults (a MonoBehaviour holding rule state, a ScriptableObject mutated at runtime, a DI container introduced), call out the deviation with its reason so the approver sees the choice rather than discovering it in review.

Wait for approval.

### STEP 3 - RULES CORE

Use skill: `unity-architecture-patterns` for the boundary. Use skill: `unity-2d-gameplay-patterns` for state modelling, move application, cascade termination, and seeded randomness. Use skill: `csharp-unity-patterns` for allocation and language mechanics. Use skill: `unity-game-economy-progression` when the feature accrues currency, progression, or offline time.

Plain C#, no `UnityEngine`. A move returns a new state plus whether it changed anything, so undo is a snapshot pop rather than a hand-written inverse. Resolution loops carry a stated bound. Grid indexing goes through one accessor. `IClock` and `IRandom` arrive as constructor parameters.

### STEP 4 - RUNTIME WIRING

Use skill: `unity-monobehaviour-lifecycle` for callback placement and static reset. Use skill: `unity-serialization-prefabs` for what the inspector actually persists.

Self-initialization in `Awake`, cross-object reads in `Start`. One composition root constructs the rules objects and hands them to presenters through serialized fields or an explicit `Initialize`, rather than `GameObject.Find` or a singleton per service. Subscriptions pair symmetrically with their unsubscribes. Every static and static event gets a `[RuntimeInitializeOnLoadMethod]` reset, because domain reload is disabled. Config ScriptableObjects are read, never written.

### STEP 5 - PRESENTATION

Use skill: `unity-ui-patterns` for UXML/USS structure, query caching, panel scaling, and the screen stack. Use skill: `unity-2d-rendering` for sprites, atlases, sorting, camera fit, and overdraw. Use skill: `unity-2d-physics-input` for the input actions and gesture thresholds that feed moves into the rules layer. Use skill: `unity-accessibility` for redundant non-colour signalling, touch targets, contrast, text scaling, and a reduced-motion path. Use skill: `unity-i18n` for every user-facing string.

The rules layer resolves a move completely; the presenter animates the resulting step list afterwards. **Rule progression is never gated on animation completion** - no move, resolution step, or state transition waits on a tween. Gating a *UI control* on a presentation finishing is a different thing and is allowed: a button disabled until a reveal completes needs a skip that fast-forwards to the end state, a reduced-motion path that reaches the enabled state immediately, and an `OnDisable` that clears the gate so a disabled component cannot strand it. Queries resolve once in `OnEnable` and are cached. Every screen with an async or failable source renders loading, error, and empty states, not just the populated one.

Skip the UI Toolkit portion when STEP 1 detected uGUI, and say that it was skipped.

### STEP 6 - PERSISTENCE

Run when the feature persists anything or touches a content bank; skip and say so when it does neither.

Use skill: `unity-security-patterns` when the feature grants currency, an entitlement, or a reward, or accepts a deep link, remote-config value, or downloaded content - a client-granted reward and an editable save are its subject, and the accepted exposure it names is what the report's Accepted Exposure slot records. Use skill: `unity-save-persistence` when the feature persists state: atomic write, corruption recovery, `schemaVersion` bump, and a migration step from the previous version - old saves exist on installed devices and must load in this build. Old builds stay installed, so a save this build writes must still load in the previous one wherever possible. Use skill: `unity-content-data` for content banks: stable identifiers, import-time validation that fails the build, and a loading strategy sized to the bank.

Write the save on pause, not on quit - a mobile process is killed without a quit callback. Give every async surface the feature adds a timeout, a cancellation path, and a defined resume behaviour.

**When the feature syncs with a server**, the exchange is built here to the STEP 2 decisions: the authoritative side per value, the merge applied when they disagree, a queue that survives a kill so an offline completion is not lost, and a retry that backs off rather than spinning. Merge logic is a rule - it belongs in the rules assembly with a test per conflict case, not in the networking class. The client never blocks a local grant on the round-trip landing; that is what makes the offline path work.

### STEP 7 - TESTS

The rules assembly is engine-free, which is what makes this cheap. Write **EditMode** tests against it: move legality, state transitions, scoring, cascade termination at its bound, undo/redo round-trips, migration from each shipped save version, and economy math driven by a fake `IClock`. No scene, no Play mode, no mocking framework where dependencies are already interfaces.

Write **PlayMode** tests only where engine behaviour is genuinely under test: lifecycle ordering, scene load, prefab wiring, and a screen's state rendering. Keep the count low - PlayMode is slow, and a rule tested there is a rule in the wrong layer.

Set up a test `.asmdef` referencing the rules assembly so the EditMode suite compiles and runs without the runtime assembly. An engine-free assembly the design added beyond the rules core - a networking seam, a merge layer - gets the same treatment: its own EditMode assembly referencing only what it needs. For a full strategy, coverage assessment, determinism policy, or CI wiring, hand off to `task-unity-test`.

When STEP 3 produced nothing, EditMode still covers whatever engine-free logic the feature added - a display-model mapping, a format under each shipped locale, a table's key coverage. `EditMode: 0` is correct only when the feature genuinely added no engine-free code; say which it is rather than leaving the count unexplained.

### STEP 8 - VALIDATE

Unity validation is batch-mode based. Run in order, fixing failures before reporting done:

1. Compile check - open the project or run the editor in batch mode and confirm zero compile errors, including in the test assemblies
2. `Unity -batchmode -nographics -runTests -testPlatform EditMode -testResults <path> -projectPath <project> -logFile -`
3. `Unity -batchmode -nographics -runTests -testPlatform PlayMode -testResults <path> -projectPath <project> -logFile -`
4. A build for the primary target tier - Use skill: `unity-build-release` for the invocation and the stripping hazards a release build exposes

Read the `-testResults` XML and the build report for the result. A zero process exit is not proof of a pass in batch mode. If a command is unavailable in this environment (no editor installed, no licence, no device), name which one and why rather than reporting a clean run.

## Edge Cases

- Unity below `6000.3.x`: STEP 1 stops the workflow; no code is written
- `ProjectVersion.txt` missing or unparseable: ask for the version; do not assume the floor is met
- uGUI project: STEP 5's UI Toolkit portion is skipped and reported; rules, wiring, persistence, and tests still run
- Built-in Render Pipeline: 2D lights guidance in STEP 5 does not apply; state that and continue
- Legacy `Input` manager: flag it at STEP 1 and use the project's existing input path; do not convert the project
- No persistence and no content bank: STEP 6 is skipped and said to be skipped
- Feature is pure presentation over existing rules: STEP 3 produces nothing; say so rather than inventing a rules layer
- No rules assembly exists, or the existing one references `UnityEngine`: report at STEP 2 as the blocking design finding and scope the fix to **this feature only** - create the rules assembly and its EditMode test assembly for the types this feature adds, leave the project's existing scripts where they are, and say which files move. A whole-project re-layering is not this workflow's scope, and building rules into MonoBehaviours because no assembly exists is not an option
- WebGL in scope: flag the persistence and threading constraints at STEP 2, not after the save code is written
- Vague input: ask in STEP 1; never guess

## Output Format

```markdown
## Engine Gate
Detected: {m_EditorVersion} | Floor: 6000.3.x | Result: {Above floor | Below floor - stopped | Unknown - asked}

## Project Surfaces
| Surface | Detected | Applied |
|---|---|---|
| Render pipeline | {URP 2D \| Built-in \| unknown} | {yes \| caveated \| unused - present but this feature does not touch it \| absent} |
| Input | {Input System \| legacy \| unknown} | {yes \| caveated \| unused - present but this feature does not touch it \| absent} |
| UI | {UI Toolkit \| uGUI \| unknown} | {yes \| out of scope \| n/a} |
| Persistence | {JSON file \| PlayerPrefs \| none \| unknown} | {yes \| caveated \| unused - present but this feature does not touch it \| absent} |
| Content loading | {Addressables \| Resources \| none \| unknown} | {yes \| caveated \| unused - present but this feature does not touch it \| absent} |

## Assemblies
| Assembly | References | Contains |

## Files Generated
[grouped by layer: rules / runtime / config assets / scenes and prefabs / ui / tests. Mark an existing file the feature changed as `(modified)` rather than listing it as new]

## Open Assumptions
[each STEP 1 gap the approver did not close, and the answer built on - so the approver sees what was assumed rather than discovering it in review. `none - every question answered` when there are none]

## Rules Model
- State: {type, or `unchanged - feature adds no rules state`}
- Move: {type} -> {result type}, or `none - feature adds no move`
- Phases: {enum values}, or `none - no phase enum added`
- Injected seams: {IClock, IRandom, others}, or `none added`

## Scenes and Prefabs
| Asset | Kind | Composition root | Purpose |

## Screens
| Screen | Stack position | Loading | Error | Empty |

## Save Impact
- Schema version: {old, or `none - unversioned legacy store`} -> {new}
- Migration: {step, or "none - no save-shape change"}
- Offline queue: {where it lives, its drain trigger, and the key that makes a replayed entry idempotent | `n/a - nothing queued`}
- Store: {the store this feature's data goes into - the existing one, or a new one with the reason; where STEP 1 detected PlayerPrefs as the primary store, the standing condition and its severity | `n/a - nothing persisted`}

## Content Impact
- Banks: {banks touched, with identifier stability and import-time validation | `none - no content bank touched`}
- Loading: {strategy sized to the bank; where content ships via `Resources/`, the standing condition and whether this feature adds to it | `n/a`}

## Deviations
[each place the build departs from this skill's rules, with the reason - a MonoBehaviour holding rule state, a mutated ScriptableObject, strings not behind a localization key, a DI container. Written as `none` when there are none, never omitted]

## Accepted Exposure
[for a feature that accrues currency, progression, or offline time: the clock-trust level chosen and the exploit it accepts in one sentence. `n/a - feature accrues nothing` otherwise]

## Server Exchange
| Value | Authoritative side | Conflict rule | Offline behaviour |
[`n/a - feature makes no server call` when it does not]
- Old-build compatibility: {how an installed older version behaves against this contract}

## Platform Tiers
| Tier | Shipped | Caveats applied |

## Tests
- EditMode: {count}
- PlayMode: {count}

## Validation
**Status:** {verified - all four commands ran and passed | unverified - <which of the four could not run, and why>}

[command -> result for each STEP 8 command; unavailable commands named with the reason]
```

Every slot above is written. A step that did not run is written as `skipped - {reason}` rather than omitted.

## Self-Check

- [ ] `behavioral-principles` loaded before the workflow ran
- [ ] Engine version read from `ProjectVersion.txt` and compared numerically to `6000.3.x`; below-floor stopped rather than degraded
- [ ] Render pipeline, input, UI, persistence, and content loading confirmed; uGUI reported out of scope where detected
- [ ] Requirements gathered by cluster; design approved before code
- [ ] Deviations from the skill's defaults called out at the approval gate
- [ ] Rules assembly has no `UnityEngine` reference, enforced by its `.asmdef`; where none existed, one was created for this feature's types and the files that move were named
- [ ] Moves return new state plus a changed flag; resolution loops carry a stated bound
- [ ] `IClock` and `IRandom` injected; no ambient time or randomness in a rule
- [ ] MonoBehaviours exist only for engine hooks; composition root wires them, no `GameObject.Find`
- [ ] Statics and static events reset via `[RuntimeInitializeOnLoadMethod]`; subscriptions paired
- [ ] ScriptableObjects hold configuration only, never mutated at runtime
- [ ] Queries cached at bind time; every screen renders loading, error, and empty
- [ ] Non-colour redundancy, touch targets, text scaling, and a reduced-motion path present
- [ ] User-facing strings behind localization keys, or held in one type with the deviation stated where the project has no localization system
- [ ] Deviations and accepted exposure written in the report, `none` / `n/a` where there are none
- [ ] Save-shape change ships a version bump and a migration; save written on pause
- [ ] Where the feature calls a server: authoritative side, conflict rule, offline queue, and old-build compatibility stated; merge logic lives in the rules assembly with a test per conflict case
- [ ] Content bank identifiers stable and validated at import
- [ ] EditMode tests cover the rules core; PlayMode limited to engine behaviour; test asmdef references the rules assembly
- [ ] Compile, EditMode run, PlayMode run, and a primary-target build all executed, with unavailable commands named

## Avoid

- Game rules written inside `Update` or any other lifecycle callback
- `UnityEngine` referenced from the rules assembly
- `MonoBehaviour` on a class with no engine callback
- ScriptableObjects used as mutable runtime state
- `?.`, `??`, or `is null` applied to a `UnityEngine.Object`
- `GameObject.Find` or `FindObjectOfType` inside logic
- Mutating board state in place where undo or replay is required
- Resolution loops with no bound
- Rule progression gated on animation completion (a UI control gated on a presentation is not this - see STEP 5)
- `Q`/`Query` called per frame instead of cached at bind time
- Colliders, rigidbodies, or trigger overlaps used to model a grid board
- Colour as the only carrier of a game-relevant distinction
- Hardcoded user-facing strings
- A save-shape change with no schema version bump or migration
- Saving in `OnApplicationQuit` on a mobile target
- PlayMode tests for logic that the engine-free rules assembly could test in EditMode
- Writing code before design approval
- Proceeding with guidance on a below-floor engine version
- Reporting a clean run without executing the STEP 8 commands
