---
name: unity-architecture-patterns
description: Structure Unity 2D games around an engine-free rules core - assembly definitions, ScriptableObject config, composition over MonoBehaviour inheritance.
metadata:
  category: mobile
  tags: [unity, csharp, architecture, assembly-definition, scriptableobject, testability, composition]
user-invocable: false
---

# Unity Architecture Patterns

> This skill owns **where code lives and what it may depend on**. Engine callback timing belongs to `unity-monobehaviour-lifecycle`; what the serializer persists belongs to `unity-serialization-prefabs`; rule algorithms themselves belong to `unity-2d-gameplay-patterns`; allocation and C# language mechanics belong to `csharp-unity-patterns`; measured frame cost belongs to `unity-performance`.

## When to Use

- Starting a new 2D game or a new feature within one
- Deciding what belongs in a `MonoBehaviour` versus plain C#
- Reviewing a diff for testability, coupling, or scene-dependence
- A rule cannot be tested without entering Play mode

## Rules

- **Game rules are plain C# with no `UnityEngine` dependency.** Board state, move legality, scoring, economy, and progression compile and run without the engine. This is the plugin's central constraint - every other architectural choice follows from it
- Enforce the boundary with an assembly definition, not discipline. A rules `.asmdef` that does not reference `UnityEngine` makes the violation a compile error rather than a review comment
- `MonoBehaviour` is for engine hooks only: lifecycle callbacks, inspector wiring, coroutines, collisions. A class needing none of these is a plain class
- Prefer composition over inheritance. Deep `MonoBehaviour` hierarchies couple unrelated concerns to a shared base and are untestable in isolation
- ScriptableObjects hold **configuration and shared assets**, not mutable runtime state. A SO edited at runtime persists that edit in the editor and silently diverges from a build
- Scene and prefab references are injected at the seam (serialized field, initializer), never fetched from deep inside logic via `GameObject.Find` or singletons
- Every dependency a rule needs - time, randomness, persistence - arrives as an interface the test can substitute

## Patterns

### The three-assembly layout

The minimum structure that makes the engine-free rule enforceable:

| Assembly | References | Contains |
| --- | --- | --- |
| `Game.Rules` | nothing (no `UnityEngine`) | board state, move legality, scoring, economy, progression |
| `Game.Runtime` | `Game.Rules`, `UnityEngine` | MonoBehaviours, presenters, ScriptableObjects, scene wiring |
| `Game.Tests.EditMode` | `Game.Rules` | fast rule tests, no scene, no Play mode |

`Game.Rules` referencing `UnityEngine` is the single failure that collapses the whole design - it is what makes rules untestable and Play-mode-bound. Keep it out of the `.asmdef`.

Assembly definitions also cut compile time: a rules change no longer recompiles the runtime assembly.

### Engine-free rules

```csharp
// Bad - rules trapped in a MonoBehaviour; needs a scene, a GameObject, and Play mode to test
public class Board2048 : MonoBehaviour {
    void Update() { if (Input.GetKeyDown(KeyCode.LeftArrow)) SlideLeft(); }
    void SlideLeft() { /* merge logic reading Time.deltaTime and Random.Range */ }
}

// Good - pure, deterministic, instant to test
public readonly record struct MoveResult(BoardState Board, int ScoreGained, bool Changed);

public static class Board2048 {
    public static MoveResult SlideLeft(BoardState board) { /* ... */ }
}
```

The good version has no engine dependency, no hidden global state, and returns the outcome rather than mutating the world. A `MonoBehaviour` calls it and renders the result.

### Substituting time and randomness

`Time`, `Random`, and `DateTime.Now` read ambient state, which makes any rule touching them non-deterministic and untestable.

```csharp
// Bad - unrepeatable; a failing test cannot be reproduced
int roll = Random.Range(0, 6);

// Good - seeded and injected; the same seed replays the same game
public interface IRandom { int Next(int minInclusive, int maxExclusive); }
public sealed class SeededRandom(int seed) : IRandom { /* System.Random */ }
```

Same move for time: rules take an `IClock` rather than reading `DateTime.Now`. Offline-progress math depends on this being substitutable - see `unity-game-economy-progression` for the wall-clock-versus-monotonic correctness boundary.

### MonoBehaviour or plain class

| Needs | Type |
| --- | --- |
| Lifecycle callback, collision, coroutine, inspector wiring | `MonoBehaviour` |
| Configuration authored in the editor, shared across scenes | `ScriptableObject` |
| Rules, algorithms, state machines, pure data | plain C# class / `record` / `struct` |
| A single shared service (audio, save) | plain C# behind an interface, with one `MonoBehaviour` owning its lifetime |

A `MonoBehaviour` whose body has no engine callback is a plain class wearing a costume: it forces a `GameObject` to exist, blocks constructor injection, and cannot be `new`ed in a test.

### ScriptableObject configuration

```csharp
// Bad - mutable runtime state on a SO; the editor persists changes between plays
[CreateAssetMenu] public class PlayerData : ScriptableObject { public int currentScore; }

// Good - immutable config; runtime state lives in the rules layer
[CreateAssetMenu] public class LevelConfig : ScriptableObject {
    [SerializeField] private int targetScore;
    public int TargetScore => targetScore;
}
```

The distinction is authored-and-fixed versus changes-during-play. Balance tables, level definitions, and enemy stats are configuration. Current score, board state, and wave number are runtime state and belong in the rules layer where they can be saved and tested.

### Wiring without global lookups

```csharp
// Bad - couples logic to scene shape; breaks on rename, silently returns null
var mgr = GameObject.Find("GameManager").GetComponent<GameManager>();

// Good - the dependency arrives at the seam
[SerializeField] private BoardPresenter presenter;
```

`GameObject.Find` and the `FindAnyObjectByType` / `FindObjectsByType` family (which supersede the obsolete `FindObjectOfType`) are also linear scene scans; in `Awake` across many objects they add measurable startup cost. For genuinely global services, one composition root - a single scene-entry `MonoBehaviour` that constructs and hands out dependencies - beats a static singleton per service, which is untestable and carries domain-reload hazards (`unity-monobehaviour-lifecycle`).

A DI container (Zenject, VContainer) is respected where a project already uses one, but is not required. For a casual 2D game a composition root is usually sufficient - see `unity-overengineering-review`.

## Output Format

When reviewing, emit one block per defect, highest severity first. One defect spanning several lines is one block; cite the clearest line and name the others in `Impact`.

```
### [Severity] {file:line | asset path | symptom, when no source was supplied}

- Category: {EngineCoupling | AssemblyBoundary | Inheritance | SOMisuse | GlobalLookup | Injectability | CompositionRoot}
- Evidence: {source | inferred (state what was not seen)}
- Code: {one-line citation, or `not supplied` when the finding is inferred}
- Impact: {what it prevents - "rule untestable without Play mode", "SO mutation persists into the editor"}
- Fix: {concrete change}
```

`Severity: {Critical | High | Medium | Low}` - Critical = game rules carry a `UnityEngine` dependency (whether inside a rules assembly or because no rules layer exists yet), or runtime state mutates a ScriptableObject. High = a rule is untestable without a scene. Medium = global lookup or avoidable inheritance in a contained spot. Low = a structural nit with no current cost.

Severity that does not fit a listed band: assign the nearest lower band and state why in `Impact`. `Category` takes exactly one value - where a defect fits two, pick the one the `Fix` addresses and name the other in `Impact`; where it fits none, pick the closest and name the real concern in `Impact`.

`Evidence: inferred` is required whenever the source was not read, and caps the block at High - write the capped severity in the header, not the uncapped one, and name the uncapped band in `Impact`.

A defect owned by a sibling named in the ownership blockquote is not emitted as a finding. Write those after the findings, one per line, as `Deferred: {defect} -> {owning skill}`, so the workflow routes rather than drops them. Omit entirely when there are none.

Close with exactly one status line, after any `Deferred:` lines:

| Condition | Line |
| --- | --- |
| One or more findings emitted | none - the findings are the output |
| No findings, and a symptom or report was available to reason from | `No architecture findings.` |
| No source, diff, symptom, or report of any kind was supplied | `Architecture check not run: no source supplied.` |

A symptom-only report (a QA ticket, a verbal description) is checkable input: emit `Evidence: inferred` findings from it rather than the not-run line.

## Avoid

- `UnityEngine` referenced from the rules assembly
- Game rules written inside `Update` or any other lifecycle callback
- `MonoBehaviour` on a class with no engine callback
- ScriptableObjects mutated at runtime as a state store
- `GameObject.Find` / `FindAnyObjectByType` / `FindObjectsByType` inside logic
- Static singletons as the default service mechanism
- Deep `MonoBehaviour` inheritance where composition would do
- `Random` or `DateTime.Now` read directly inside a rule
- A DI container introduced for a game with a handful of services
