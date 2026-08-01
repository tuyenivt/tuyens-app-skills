# Tuyen's Agent Skills - Unity 2D

Claude Code plugin for Unity 2D game development.

This is the marketplace's second client plugin, alongside `flutter`. Like it, it is authored against the client domain rather than adapted from a backend plugin - transactions, connection pools, and server middleware do not map to a game running on a device. Unlike it, a Unity project is engine-and-asset-centric: scenes, prefabs, ScriptableObjects, and the serialization layer carry as much correctness risk as the C# does, and several skills exist specifically to cover that surface.

## Target Games

Casual and puzzle 2D titles: 2048, Sudoku, Chess, Match-3, simple Tower Defense, Idle games, and quiz apps (driver-license, flag quiz, word and trivia quiz), plus comparable simple casual 2D games.

There are **no genre-specific skills**. A per-genre skill would be a template rather than a pattern and would not generalize. Genre knowledge lives as short worked examples inside the gameplay and economy skills.

## The Central Rule

**Game rules are plain C# with no `UnityEngine` dependency**, enforced by an assembly definition rather than by discipline.

Every target genre has rules - board state, move legality, scoring, economy, progression - that can and should run without the engine. This makes them testable in EditMode in milliseconds instead of requiring Play mode and a scene, and it is what makes a wide genre range cheap to support. `unity-architecture-patterns` owns the rule; most other skills reference it.

## Stack

- Unity **6.3 LTS** (`6000.3.x`) - hard floor, no back-support for 6.0/6.1 LTS or 2022.3
- C# as shipped by the 6.3 toolchain
- URP 2D (Renderer 2D); HDRP is out of scope for 2D
- **UI Toolkit only** - uGUI is out of scope (new games, no legacy maintenance)
- Input System package
- Plain C# + ScriptableObject composition; Zenject / VContainer respected where already present
- JSON at `Application.persistentDataPath`; PlayerPrefs treated as non-durable and non-secure
- Addressables for content delivery
- Unity Test Framework (EditMode + PlayMode), NSubstitute

### Version scheme

Unity ships two names for the same release; skills do not mix them.

| Form | Example | Where it appears |
| ---- | ------- | ---------------- |
| Marketing name | `Unity 6.3 LTS` | Docs, prose |
| Internal version | `6000.3.21f1` | `ProjectSettings/ProjectVersion.txt`, installers, CI |

`6000` is the major-version token for the whole Unity 6 line, so `6000.3.x` **is** Unity 6.3. Detection compares the internal form numerically by component; a string prefix match would rank `6000.5` below `6000.3`. The floor is a minimum, not an equality check - a project on a newer Supported Update release proceeds normally.

## Platform Tiers

Guidance degrades predictably rather than exploding into a full platform matrix.

| Tier | Platforms | Support level |
| ---- | --------- | ------------- |
| **Primary** | Android, iOS | Full depth. Assumed unless stated otherwise. |
| **Secondary** | Windows, macOS, Linux | Caveats: input modality, window and resolution handling, packaging. |
| **Tertiary** | WebGL | Constraints: no threads by default, no durable `System.IO`, memory ceiling, IL2CPP-only, compressed-load and hosting concerns. |

## Agents

| Agent | Description |
| ----- | ----------- |
| `unity-engineer` | Builds 2D features end-to-end: engine-free rules core, MonoBehaviour and ScriptableObject wiring, scenes and prefabs, UI Toolkit, save, tests. Also triages destroyed-object exceptions, lifecycle-order bugs, and serialization failures. |
| `unity-tech-lead` | Code review, refactoring guidance, and scene/prefab architecture standards. Reviews for engine-free core discipline, composition over inheritance, and asset-layer hygiene. |
| `unity-performance-engineer` | Frame budget, GC allocation spikes, draw calls and batching, overdraw, texture memory, physics cost, load time, build size. |
| `unity-security-engineer` | Save tampering, IAP receipt validation, rewarded-ad grant integrity, secrets in builds, IL2CPP exposure limits, SDK data flow, deep links, store privacy. |
| `unity-test-engineer` | EditMode and PlayMode strategy, engine-free core coverage, determinism with seeded RNG and injected clocks, NSubstitute, test assemblies, CI batch mode. |

## Workflow Skills

| Skill | Agent | Description |
| ----- | ----- | ----------- |
| `task-unity-implement` | `unity-engineer` | End-to-end 2D feature: engine-free rules core, ScriptableObject config, MonoBehaviour wiring, scene and prefab layout, UI Toolkit, save, and tests. |
| `task-unity-review` | `unity-tech-lead` | Staff-level review umbrella - Phases A-E; spawns perf and security subagents in parallel. |
| `task-unity-review-perf` | `unity-performance-engineer` | Performance review for frame budget, GC spikes, batching, overdraw, texture memory, physics, load time, build size. |
| `task-unity-review-security` | `unity-security-engineer` | Security review for save tampering, IAP and rewarded-ad grant integrity, secrets, SDK data flow, deep links, privacy. |
| `task-unity-test` | `unity-test-engineer` | Test strategy and scaffolding across EditMode and PlayMode, with determinism, test assemblies, and CI batch mode. |

> Unity does not review API contract design - a game client consumes contracts rather than designing them. A finding that the game mishandles a contract stays in-plugin; a finding that the contract itself is wrong hands off to the team owning that service.

> Accessibility, aspect-ratio adaptivity, and localization have no dedicated review lens. They are designed in `task-unity-implement` and checked at baseline depth in `task-unity-review` Phase E.

## Atomic Skills

### Language and engine core

| Skill | Description |
| ----- | ----------- |
| `csharp-unity-patterns` | C# as Unity runs it: allocation in hot paths, `struct` and boxing, LINQ cost, async versus coroutines, and the `UnityEngine.Object` lifetime `==` overload that makes `?.` lie on a destroyed object. |
| `unity-architecture-patterns` | The engine-free rules core: assembly definitions enforcing the boundary, composition over MonoBehaviour inheritance, ScriptableObject configuration, injection seams, composition root. |
| `unity-monobehaviour-lifecycle` | Callback order, initialization-order traps, domain reload and static state across Play sessions, `DontDestroyOnLoad`, coroutine lifetime, application pause and focus. |
| `unity-serialization-prefabs` | What the serializer persists and silently drops, `[SerializeReference]`, prefab variants and overrides as a review surface, `.meta` and GUID stability, scene merge conflicts. |

### Gameplay

| Skill | Description |
| ----- | ----------- |
| `unity-2d-gameplay-patterns` | Game loop and turn resolution, deterministic grid and board modelling, pure reversible moves, cascade termination, snapshot undo, seeded RNG. |
| `unity-2d-rendering` | Sprites and atlases, sorting layers and groups, 2D lights under URP, tilemaps, pixel-perfect camera, overdraw from stacked transparency, batching breakers. |
| `unity-2d-physics-input` | Physics 2D where a genre needs it, body types and query filters, and why grid games use array indices rather than physics. Input System actions, touch and gesture handling. |
| `unity-game-economy-progression` | Offline progress and elapsed-time math, wall-clock versus monotonic time as a correctness boundary, big-number overflow, prestige loops, data-driven balance. |

### Content and presentation

| Skill | Description |
| ----- | ----------- |
| `unity-ui-patterns` | UI Toolkit: UXML and USS structure, `VisualElement` query cost, runtime binding, panel scaling, screen and popup stacks, safe area, aspect-ratio adaptivity. |
| `unity-content-data` | Large content banks as data rather than prefabs: authoring format, import-time validation, Addressables streaming, content updates without a store release. |
| `unity-accessibility` | Colour-blind-safe signalling as a correctness issue, touch targets, contrast, text scaling, reduced motion, remappable input, and the real limits of screen-reader support in Unity. |
| `unity-i18n` | Localization package and string tables, plurals, CJK font atlas memory, text expansion breaking fixed-width layouts, RTL limits, locale-aware formatting. |

### Cross-cutting quality

| Skill | Description |
| ----- | ----------- |
| `unity-performance` | Profiler-first discipline on a real device, GC allocation spikes, update cost and pooling, batching and overdraw, texture memory, load time, and when ECS is not warranted. |
| `unity-save-persistence` | Save format choice, atomic write and corruption recovery, schema versioning and migration across installed builds, cloud save conflicts, WebGL persistence limits. |
| `unity-security-patterns` | Client-hostility posture, server-side IAP and rewarded-ad validation, save integrity, secrets absent from builds, untrusted external input, store privacy requirements. |
| `unity-build-release` | Build profiles, IL2CPP stripping and its reflection hazards, app bundles and signing, WebGL compression and hosting, Addressables catalogs, CI batch mode, size analysis. |

### Repo-required

| Skill | Description |
| ----- | ----------- |
| `unity-overengineering-review` | Necessity review: ECS for a puzzle board, a DI container for a small game, MonoBehaviour without engine hooks, ScriptableObject-per-constant, pooling one instance, dead flags. Composed into `task-unity-review` Phase D. |

## Usage Examples

### Implement a feature end-to-end

```
> task-unity-implement

Feature: Add a daily-challenge mode to a Match-3 game
- Engine-free rules: seeded board generation, match detection, cascade resolution
- ScriptableObject config for challenge parameters
- UI Toolkit screen with loading / error / empty states
- Save schema change with a migration for installed builds
- EditMode tests for the rules, one PlayMode test for the scene flow

-> Validates with a compile check, EditMode and PlayMode runs, and a build for the primary target
```

### Review a PR with automatic scope escalation

```
> task-unity-review

- Phases A-E: risk, correctness, architecture, AI-quality, maintainability
- Detects a reward granted in client code -> auto-escalates +Sec
- Detects allocation inside Update -> auto-escalates +Perf
- Spawns the two lens subagents in parallel, merges findings at strongest intent
- Treats scenes and prefabs as review surface; excludes Library/ and generated .csproj
```
