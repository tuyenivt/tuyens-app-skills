---
name: unity-overengineering-review
description: Unity necessity review - ECS for a puzzle board, DI container for a small game, MonoBehaviour without engine hooks, pooling one instance, dead flags.
metadata:
  category: mobile
  tags: [unity, csharp, code-review, overengineering, necessity, ecs, scriptableobject, pooling]
user-invocable: false
---

# Unity Overengineering Review

> Confirm any DI container the project already uses from `Packages/manifest.json` first. A container in place is context, not a finding - review what the diff adds against it, and never propose migrating off it.
>
> This skill owns **whether a layer earns its keep**. Where code should live and what the composition root looks like belongs to `unity-architecture-patterns`; callback timing and static lifetime to `unity-monobehaviour-lifecycle`; allocation and language mechanics to `csharp-unity-patterns`; measured cost to `unity-performance`.

## When to Use

- Reviewing a Unity diff that adds MonoBehaviours, ScriptableObjects, interfaces, pools, events, or generic types
- Catching code that compiles, runs, and passes tests but does not need to exist

## Rules

- Every finding names what makes the abstraction unnecessary: no engine callback, no second reader, no second implementer, no measured cost, the value never varies. When several stack, comma-separate them in `Unnecessary because:`
- Intent:
  - **`[Recommend]`** (default). Name the constraint, recommend the edit. Escalate to **`[Must]`** when measurable cost is present; cite it in `Cost:`. Triggers: an abstraction that forces Play mode to test what was previously a plain-C# unit test; a pooling or caching layer whose bookkeeping exceeds the allocation it avoids; a branch presented as handling a case it can never reach
  - **`[Recommend]`** when justification is plausible but not visible in the diff - state the assumption and ask the author to confirm
- An abstraction with **visible** justification - a second implementer, a test seam, a profiler measurement in the PR - is not a finding
- **Scale is the discriminator, and scale is not genre.** Price an abstraction against the variation it absorbs: team size, shipped platforms, storefronts, locales, and runtime-selected variants. A casual 2D game on four platforms with a runtime consent split has a large distribution matrix and earns the layers that track it; a 2-scene puzzle game on one store does not. Cite the project's actual numbers, not a general principle
- **This skill has a floor as well as a ceiling.** Where structure is absent rather than excessive - a God object, global mutable statics as the only channel between systems, no test seam on a live title - say so plainly and route it to `unity-architecture-patterns`. Never read "small project" as licence for no structure: a codebase where one feature costs edits in nine places has already paid more than the abstraction would have cost
- Never propose deleting a layer the diff's own tests bind to
- Performance abstractions need a measurement, not an argument. A pool, a cache, or a Job introduced without a profile is speculative regardless of how reasonable it sounds

## Patterns

### Category 1: Engine Ceremony

#### `MonoBehaviour` with no engine hook

```csharp
// Bad - no lifecycle callback, no inspector field, no coroutine; needs a GameObject to exist
public class ScoreCalculator : MonoBehaviour {
    public int Score(Board b) => b.Tiles.Sum(t => t.Value);
}

// Good - plain class; newable in an EditMode test, no scene required
public sealed class ScoreCalculator { public int Score(Board b) => ... }
```

`Cost:` the rule now requires Play mode and a `GameObject` to test. Justified when the class uses a lifecycle callback, a serialized inspector field, a coroutine, or a collision/trigger hook.

#### ScriptableObject per constant

```csharp
// Bad - an asset, a .meta file, and an inspector reference to hold one number that never varies
[CreateAssetMenu] public class GridSizeConfig : ScriptableObject { public int size = 4; }

// Good
public const int GridSize = 4;
```

Justified when a designer genuinely tunes the value without a rebuild, it varies per level or per difficulty, or it is one field in a config asset that already exists. Do not flag a populated balance table - that is `unity-game-economy-progression` working as intended.

#### Singleton for a service with one call site

```csharp
// Bad - static instance, DontDestroyOnLoad, domain-reload hazard, for one caller
public class AudioManager : MonoBehaviour { public static AudioManager Instance; }
```

Justified when several unrelated scenes need it and its lifetime genuinely spans them. Otherwise a serialized reference from the composition root is cheaper and testable (`unity-architecture-patterns`). Static state also survives Play sessions when domain reload is disabled - see `unity-monobehaviour-lifecycle`.

### Category 2: Premature Architecture

#### ECS/DOTS for a casual 2D game

```csharp
// Bad - an entity/component/system stack to move 16 tiles on a 4x4 board
public partial struct TileMoveSystem : ISystem { ... }

// Good - a plain array and a loop; the whole board fits in a cache line
```

`Cost:` a second programming model, a separate debugging story, and rules that are no longer plain C#. ECS earns its complexity at thousands of entities. A 2048 board, a Sudoku grid, a chess position, and a typical Match-3 field are all far below that line. Justified only with a profile showing the entity count and the frame cost that motivates it.

#### DI container for a small game

```csharp
// Bad - a container, installers, and binding config for six services in three scenes
```

Justified when the project already uses one (then it is the convention - follow it), or when the object graph is genuinely large. For a new casual game, a composition root wiring a handful of services is less code and less indirection. Never propose removing a container the project already standardized on.

#### Event bus for two known callers

```csharp
// Bad - publish/subscribe indirection where the caller and callee are both known and adjacent
EventBus.Publish(new TileMergedEvent(value));

// Good - a C# event on the class that owns the state, or a direct call
```

`Unnecessary because:` the indirection hides the call graph without decoupling anything that varies. Justified when publishers and subscribers genuinely do not know each other, or subscribers are added at runtime from unrelated systems.

#### Interface with one implementer and no test seam

```csharp
// Bad - IBoardRenderer with exactly one implementation, never substituted
public interface IBoardRenderer { void Render(Board b); }
```

Justified when a fake or a second implementation exists, or arrives in the same PR. Note the contrast with a genuine seam: `IClock` and `IRandom` are justified by their test substitution even with one production implementation (`unity-architecture-patterns`) - check for the substitution before flagging.

### Category 3: Speculative Performance

#### Pooling one instance, or an object that is never destroyed

```csharp
// Bad - a pool wrapping a single persistent object
_pool = new ObjectPool<GameOverPanel>(...);   // exactly one, alive for the session
```

`Cost:` pool bookkeeping and lifecycle bugs (a returned object that keeps running) in exchange for an allocation that happens once. Justified for objects spawned and released repeatedly - TD projectiles, Match-3 particles, damage numbers - where a profile shows the allocation.

#### Caching or micro-optimizing without a profile

```csharp
// Bad - a hand-rolled cache added "for performance" with no measurement
```

`unity-performance` owns the measurement discipline. The finding here is the abstraction added on a guess; if a profile is cited, this is not a finding regardless of the outcome.

#### Struct/generic gymnastics for a cold path

Allocation avoidance in code that runs once per level load buys nothing and costs readability. The finding is optimization applied where frequency does not justify it - state the actual call frequency.

### Category 4: Type-System Waste

#### Null check against a Unity object that cannot be null there

```csharp
// Bad - a serialized field guaranteed assigned in the inspector, checked on every frame
void Update() { if (target == null) return; ... }
```

Justified when the reference can genuinely be destroyed at runtime, or the field is optional. Note the real hazard in the opposite direction: `?.` and `??` on a destroyed `UnityEngine.Object` bypass the lifetime check and misbehave - that is `csharp-unity-patterns`, and is a correctness bug, not overengineering.

#### Generic type with one instantiation

```csharp
// Bad - Repository<T> only ever Repository<SaveData>
```

Justified at 2+ instantiations, or when the type is a package's public API.

#### Dead feature flag

```csharp
// Bad - a constant flag; the other branch is never compiled into any test
if (Features.NewBoardRenderer) { ... } else { ... }
```

Justified when remote config, a staged rollout, or a kill switch backs it. Otherwise delete the dead branch - an untested branch is a liability.

## Output Format

One block per finding; the consuming workflow merges them:

```
### [Must | Recommend] {file:line | asset path | symptom, when no source was supplied}

- Category: {Engine Ceremony | Premature Architecture | Speculative Performance | Type-System Waste | Absent Structure}
- Code: {one-line citation, or `not supplied` when the finding is inferred}
- Unnecessary because: {what makes it dead or unread; comma-separate when stacked} -- OR, for Absent Structure -- Missing because: {what the absence costs}
- Cost: {required for [Must]; omit otherwise}
- Recommendation: {concrete C# or asset edit; for Absent Structure, the extraction and its owning skill}
- Justified when: {one-line note if a legitimate reason might apply; otherwise omit}
```

`Absent Structure` is the floor rule's category. Its `Cost:` is the edit-site count or the regression count already being paid, and that count is what escalates the block to `[Must]`.

An abstraction examined and found justified is written before the per-category lines, one per line, so the reader can tell a defended layer from an unexamined one:

```
Justified as-is: {abstraction} - {the visible justification: implementer count, the test double, the measurement}
```

This is the required form whenever the request questions an existing layer, since `No <category> findings.` alone reads as "nothing was checked" rather than "this was checked and it holds".

For each category with zero findings, emit exactly: `No <category> findings.` (using the category name from the enum) so the workflow knows the check ran. Omit this line for categories that have at least one finding. Emit `Necessity check not run: no source supplied.` instead of the per-category lines only when nothing at all was supplied - a prose description of the architecture is checkable input, and yields findings, `Justified as-is:` lines, or `Deferred:` lines like any other source.

A defect owned by a sibling named in the ownership blockquote is not emitted as a finding. List those under a final `Deferred:` line naming the defect and the owning skill, so the workflow routes rather than drops them.

## Avoid

- Flagging a `MonoBehaviour` that uses a lifecycle callback, serialized field, coroutine, or collision hook
- Flagging `IClock`, `IRandom`, or another interface that exists as a visible test seam
- Flagging a ScriptableObject that a designer tunes, or one holding a real balance table
- Flagging a DI container the project already standardized on, or proposing migration off it
- Flagging a pool, cache, or Job whose PR cites a profile
- Flagging ECS in a project that already uses it throughout
- Treating file count, class count, or folder depth as a complexity metric
- Removing a layer the diff's own tests bind to
- Raising findings against generated files, `.meta` files, or imported third-party SDK sources
- Confusing the `?.`-on-destroyed-object correctness bug with an unnecessary null check
