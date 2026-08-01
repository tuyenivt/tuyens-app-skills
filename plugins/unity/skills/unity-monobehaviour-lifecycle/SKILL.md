---
name: unity-monobehaviour-lifecycle
description: Order Unity callbacks correctly - Awake/OnEnable/Start traps, static state surviving disabled domain reload, coroutine lifetime, DontDestroyOnLoad.
metadata:
  category: mobile
  tags: [unity, monobehaviour, lifecycle, domain-reload, coroutines, execution-order, singleton]
user-invocable: false
---

# Unity MonoBehaviour Lifecycle

> This skill owns **engine callback timing and object lifetime**. Where code lives and what it may depend on belongs to `unity-architecture-patterns`; what the serializer persists belongs to `unity-serialization-prefabs`; C# language mechanics including `async` cancellation belong to `csharp-unity-patterns`; per-frame cost belongs to `unity-performance`.

## When to Use

- A reference is null in `Awake` but populated by the time `Start` runs
- Behaviour differs between the first Play session and the second
- A coroutine stops without being told to
- Wiring a persistent manager, scene load, or app pause/resume handling

## Rules

- **`Awake` initializes self; `Start` reads others.** Any cross-object reference resolved in `Awake` depends on undefined ordering between objects
- Never depend on `Awake` ordering across GameObjects. If order genuinely matters, make it explicit with Script Execution Order or an initializer that calls its dependents
- **Assume domain reload is disabled.** Statics and static event subscriptions carry over between Play sessions in the editor. Reset every static explicitly via `[RuntimeInitializeOnLoadMethod]`
- Unsubscribe in `OnDisable` from anything subscribed in `OnEnable`, and in `OnDestroy` from anything subscribed in `Awake`. Symmetric pairs, always
- Coroutines are owned by the MonoBehaviour that started them: disabling the component stops them, destroying the object kills them. Neither raises an error
- `DontDestroyOnLoad` objects survive scene loads, so a scene containing one that is loaded twice produces duplicates. The instance must guard against that itself
- Do not put per-frame work in `Update` when the state only changes on an event

## Patterns

### Callback order

Within a single object, for one frame:

| Phase | Callbacks | Runs |
| --- | --- | --- |
| Initialization | `Awake` -> `OnEnable` -> `Start` | `Awake`/`OnEnable` on instantiation, `Start` before the first `Update` after enablement |
| Physics | `FixedUpdate` -> physics step -> `OnCollision*` / `OnTrigger*` | zero or more times per frame, on the fixed timestep |
| Logic | `Update` -> coroutine resumption -> `LateUpdate` | once per frame |
| Teardown | `OnDisable` -> `OnDestroy` | on disable, destroy, and application quit |

`Awake` and `OnEnable` run on instantiation even if the object is later disabled before `Start`; `Start` is deferred until the object is actually enabled, and never runs if it never is. `LateUpdate` is where you read a transform another script moved this frame - camera follow belongs there, not in `Update`.

### Awake versus Start

```csharp
// Bad - other's Awake may not have run yet; _board is null or half-built
void Awake() { _board = FindAnyObjectByType<BoardController>().Board; }

// Good - self-init in Awake, cross-object reads in Start
void Awake() { _cells = new Cell[Width * Height]; }
void Start() { _board = _boardController.Board; }
```

The safest structure removes the question: inject the reference through a serialized field and have a single composition root call an explicit `Initialize(...)`, so ordering is code rather than engine policy (`unity-architecture-patterns`).

### Script Execution Order

Project Settings -> Script Execution Order assigns a numeric order; lower runs first, and it applies to `Awake`, `Start`, `Update`, and the rest. Use it only for one or two genuine infrastructure scripts (a bootstrap or service registry). It is a project-wide setting invisible from the source file, so a class whose correctness depends on it needs a comment saying so. More than a handful of entries means the initialization design is wrong.

### Domain reload disabled: the session-two trap

With Enter Play Mode Options enabled and domain reload disabled (the default configuration for fast iteration), entering Play mode does **not** reset the C# domain. Statics keep the values the previous session left, and static event subscriptions from destroyed objects remain subscribed.

```csharp
// Bad - session 2 starts with session 1's score, and the stale handler still fires
public static int HighScore;
static event Action OnGameOver;

// Good - explicit reset before any scene loads
[RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.SubsystemRegistration)]
static void ResetStatics() { HighScore = 0; OnGameOver = null; }
```

`SubsystemRegistration` is the earliest load type and runs before scene objects exist, which is what you want for a reset. This is editor-only behaviour - a fresh build always resets - so the symptom is "works in a build, wrong in the editor on the second Play", which is easy to misdiagnose as a save bug. Every static field and static event needs a line in a reset method; a static holding a `UnityEngine.Object` reference from a prior session is worse than stale, it is destroyed (`csharp-unity-patterns`).

### Singletons and DontDestroyOnLoad

```csharp
// Bad - returning to the menu scene a second time creates a second AudioManager
void Awake() { Instance = this; DontDestroyOnLoad(gameObject); }

// Good - the duplicate destroys itself before it can register
void Awake() {
    if (Instance != null && Instance != this) { Destroy(gameObject); return; }
    Instance = this; DontDestroyOnLoad(gameObject);
}
```

`DontDestroyOnLoad` moves the object to a separate scene, so it does not appear in the loaded scene hierarchy and is not unloaded with it. Combined with disabled domain reload, `Instance` also survives into the next Play session pointing at a destroyed object - so the static needs the reset from the previous pattern too. Prefer a single bootstrap scene that creates persistent services once over one self-registering singleton per service.

### Coroutine lifetime

```csharp
// Bad - disabling the component silently stops this mid-sequence, leaving the flag set
IEnumerator Resolve() { _busy = true; yield return _delay; _busy = false; }

// Good - state that must survive lives outside the coroutine, and cleanup is in OnDisable
void OnDisable() { _busy = false; }
```

Disabling the component or its GameObject stops its coroutines; re-enabling does not resume them. Destroying the object kills them. Neither logs anything, so a half-completed sequence leaves whatever invariant it was holding broken. `StopCoroutine` needs the same handle `StartCoroutine` returned - passing the method or a fresh iterator does not match. A coroutine that must outlive its object belongs on a persistent host, or should be an `Awaitable` with an explicit token (`csharp-unity-patterns`).

Note `WaitForSeconds` uses scaled time, so it never completes while `Time.timeScale` is 0 - use `WaitForSecondsRealtime` for pause menus.

### Pause, focus, and quit on mobile

```csharp
// Bad - OnApplicationQuit is not reliably called when a mobile OS kills a backgrounded app
void OnApplicationQuit() { SaveGame(); }

// Good - persist at the last guaranteed point
void OnApplicationPause(bool paused) { if (paused) SaveGame(); }
```

On mobile, backgrounding raises `OnApplicationPause(true)`; a subsequent kill may deliver no further callback at all. Treat pause as the save point. `OnApplicationFocus` also fires around backgrounding, and the exact pairing and ordering of pause and focus differs by platform, OS version, and interruption type (call, notification shade, app switcher). Do not encode a specific sequence - make the handler idempotent and verify on device.

### Scene load callbacks

`SceneManager.sceneLoaded` fires after the new scene's objects have run `Awake` and `OnEnable`, but before their first `Start`. `activeSceneChanged` and `sceneUnloaded` cover the other transitions. Subscribe from a persistent object and unsubscribe symmetrically - a subscription from an object in the outgoing scene, with domain reload disabled, becomes a stale handler firing into destroyed objects. An additive load leaves both scenes' objects live simultaneously, so anything assuming "one board controller exists" needs to hold under additive loading.

## Output Format

When reviewing, emit one block per defect, highest severity first. One line carrying two distinct failures is two blocks; the same defect reachable from several lines is one.

```
### [Severity] {file:line | symptom, when no source was supplied}

- Category: {InitOrder | CallbackPlacement | StaticNotReset | SubscriptionLeak | CoroutineLifetime | ScaledTimeWait | SingletonDuplication | PauseSaveGap | SceneCallback | ExecutionOrderDependency}
- Evidence: {source | inferred (state what was not seen)}
- Code: {one-line citation, or `not supplied` when the finding is inferred}
- Impact: {observable failure - "null on first frame in a build", "session 2 starts with session 1 state"}
- Fix: {concrete change}
```

`CallbackPlacement` covers work in the wrong callback (a transform read in `Update` that belongs in `LateUpdate`, per-frame polling of event-driven state). `ScaledTimeWait` covers a wait that stalls at `Time.timeScale == 0`.

`Severity: {Critical | High | Medium | Low}` - Critical = unreset static or static event under disabled domain reload, or progress lost because saving depends on `OnApplicationQuit`. High = cross-object reference resolved in `Awake`, subscription without its symmetric unsubscribe, or a coroutine holding an invariant that disablement breaks. Medium = singleton without a duplicate guard, callback placement that another script can observe mid-frame, or undocumented Script Execution Order dependence. Low = a lifecycle nit with no reachable failure.

Severity that does not fit a listed band: assign the nearest lower band and state why in `Impact`. `Category` takes exactly one value - where a defect fits two, pick the one the `Fix` addresses and name the other in `Impact`; where it fits none, pick the closest and name the real concern in `Impact`.

`Evidence: inferred` is required whenever the source was not read, and caps the block at High - write the capped severity in the header, not the uncapped one, and name the uncapped band in `Impact`. `Fix` opens with the one check that confirms the diagnosis (for the session-two signature: Project Settings -> Editor -> Enter Play Mode Options).

A defect owned by a sibling named in the ownership blockquote is not emitted as a finding. Write those after the findings, one per line, as `Deferred: {defect} -> {owning skill}`, so the workflow routes rather than drops them. Omit entirely when there are none.

Close with exactly one status line, after any `Deferred:` lines:

| Condition | Line |
| --- | --- |
| One or more findings emitted | none - the findings are the output |
| No findings, and a symptom or report was available to reason from | `No lifecycle findings.` |
| No source, symptom, or report of any kind was supplied | `Lifecycle check not run: no source supplied.` |

A symptom-only report (a QA repro, a verbal description) is checkable input: emit `Evidence: inferred` findings from it rather than the not-run line.

## Avoid

- Cross-object reference lookups in `Awake`
- Static fields or static events with no `[RuntimeInitializeOnLoadMethod]` reset
- Subscribing in `OnEnable`/`Awake` without the matching unsubscribe
- Saving in `OnApplicationQuit` on a mobile target
- `DontDestroyOnLoad` without a duplicate-instance guard
- Coroutines holding state whose cleanup only exists in the coroutine body
- `WaitForSeconds` in a path that must run while `Time.timeScale` is 0
- Silent reliance on Script Execution Order, uncommented at the call site
- Assuming a fixed `OnApplicationPause`/`OnApplicationFocus` sequence across platforms
