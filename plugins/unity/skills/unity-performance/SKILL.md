---
name: unity-performance
description: "Diagnose Unity 2D mobile frame cost: profiler-first discipline, GC spikes, Update cost, pooling, batching, overdraw, texture memory, load time."
metadata:
  category: mobile
  tags: [unity, performance, profiler, gc-allocation, frame-budget, batching, overdraw, object-pool, texture-memory, mobile]
user-invocable: false
---

# Unity Performance

> Confirm the project's target platforms first - they decide which of the texture, fill-rate, and load-time guidance applies. C# allocation mechanics belong to `csharp-unity-patterns`; sprite atlases, sorting, and tilemap setup to `unity-2d-rendering`; UI Toolkit structure to `unity-ui-patterns`; where code lives to `unity-architecture-patterns`. This skill owns **cost**: what a frame, a byte of texture memory, and a scene load cost the player.

## When to Use

- Investigating stutter, hitches, dropped frames, or battery/thermal complaints
- Reviewing a diff for per-frame allocation, `Update` growth, instantiation, or fill rate
- Cutting scene load time or runtime memory on a low-end device
- Deciding whether ECS/Jobs/Burst is warranted

## Rules

- **Measure with the Profiler attached to a development *build* on a real target device.** Editor Play-mode timings are not the game: the editor runs extra tooling, uses a different scripting backend than the shipped IL2CPP build, and hides device thermal and GPU limits. An editor number is not evidence
- Development builds carry profiler instrumentation and deep-call overhead. Use them to find *where* the cost is; confirm the shipped number in a non-development build (`unity-build-release`)
- Frame budget is **16ms at 60fps, 33ms at 30fps**. Read the project's `Application.targetFrameRate` before assuming 60 - casual 2D games commonly ship 30fps deliberately for battery and thermal headroom, and 30 stable beats 60 unstable. When the target is stated by a person rather than read from the project, say so and use the stated figure
- `Owner: Memory` and `Owner: Load` need their own budgets, since a frame figure says nothing about either. Memory: a 3GB Android device gives a game roughly 1GB before the OS starts killing it, so state total against that ceiling and split managed from native - they have different causes and different fixes. Load: cold start to first playable, measured from process start on a mid-tier device, not from the editor's first frame
- **Per-frame managed allocation is the dominant mobile Unity performance failure.** Steady garbage forces collections, and a collection during play is a visible hitch. Target zero allocation in the steady-state frame loop
- Never `Instantiate`/`Destroy` in a spawn loop or per-frame path. Pool
- Attribute an overrun to CPU (main thread), GPU (fill rate/draw calls), or GC before changing anything. The Profiler timeline names which
- A fix without a before/after number on the same device is a guess. Report the number or report that you have none

## Patterns

### Attributing the cost

| Symptom | Likely owner | Typical cause |
| --- | --- | --- |
| Periodic hitch every few seconds, otherwise smooth | GC | steady per-frame allocation triggering collection |
| Steady low framerate, main thread saturated | CPU scripts | `Update` on many objects, per-frame `GetComponent`/`Find`, physics queries |
| Slower the more of the screen the effect covers | GPU fill rate | overdraw from stacked transparent sprites |
| Slower with more distinct on-screen objects, resolution-independent | GPU draw calls | broken batching, material variants |
| Hitch on spawn, wave start, or first use of an effect | CPU + alloc | `Instantiate`, shader/material warm-up, lazy asset load |
| Hitch on scene change | Load | synchronous `LoadScene`, large asset dependency graph |
| Fine on a flagship, unplayable on a budget device | GPU or memory | fill rate, texture memory, or thermal throttling |

The Profiler's CPU timeline separates scripts, rendering, physics, and GC. The Memory Profiler package attributes retained bytes by asset. Attribute first; the wrong fix costs the same effort as the right one.

### GC allocation: the dominant problem

```csharp
// Bad - allocates a string every frame; 60 collections' worth of garbage per minute
void Update() { label.text = $"Score: {score}"; }

// Good - allocation only when the displayed value actually changes
void Update() { if (score != _shown) { _shown = score; label.text = _cached[score]; } }
```

Common per-frame allocation sources: string building and interpolation, LINQ, capturing lambdas, `new` collections, allocating physics/component query overloads, and `foreach` over an interface-typed collection. Mechanics and the full list belong to `csharp-unity-patterns`; the point here is that each one shows in the Profiler's `GC.Alloc` column, and that column should read 0 B in a steady frame.

**Incremental GC is a mitigation, not a fix.** It splits collection across frames so a single hitch becomes several smaller ones. It does not reduce the garbage, and under heavy allocation it still overruns. Enabling it while continuing to allocate per frame converts a visible stall into sustained frame-time noise. Remove the allocation.

### Update cost and the update manager

`Update` on a MonoBehaviour is invoked through native-to-managed interop per object per frame. That per-call overhead is small individually and material at hundreds or thousands of objects, before the body does any work.

```csharp
// Bad - 500 tiles each with an empty-most-frames Update
public class Tile : MonoBehaviour { void Update() { if (_dirty) Refresh(); } }

// Good - one Update; tiles implement a plain interface and are ticked from a list
public class TileTicker : MonoBehaviour {
    private readonly List<ITickable> _tiles = new();
    void Update() { for (int i = 0; i < _tiles.Count; i++) _tiles[i].Tick(); }
}
```

The update-manager pattern also gives you ordering control, cheap enable/disable by list membership, and a single place to batch or stagger work. Staggering (tick a slice of the list per frame) is the standard fix when the work itself is the cost rather than the dispatch.

An empty `Update`, `FixedUpdate`, or `LateUpdate` method still costs the interop call - delete it rather than leaving it as a placeholder.

### Object pooling

```csharp
// Bad - per-shot allocation plus Destroy garbage; hitches on wave start
var b = Instantiate(bulletPrefab, pos, rot);
Destroy(b, 2f);

// Good - fixed-size reuse, no allocation after warm-up
_pool = new ObjectPool<Bullet>(
    createFunc: () => Instantiate(bulletPrefab),
    actionOnGet: b => b.gameObject.SetActive(true),
    actionOnRelease: b => b.gameObject.SetActive(false),
    actionOnDestroy: b => Destroy(b.gameObject),
    maxSize: 64);
```

`UnityEngine.Pool.ObjectPool<T>` ships with the engine, so a hand-rolled pool needs a reason. It is a stack-backed collection, is not thread-safe, and destroys returned instances beyond `maxSize`. `Get(out PooledObject<T>)` releases on dispose for scoped use.

Two failure modes that make pooling a bug source rather than a win: **state leaking across reuse** (a pooled object must be fully reset on get or release, not partially), and **pooling a single instance** or a rarely-spawned object, which adds indirection for no measured gain (`unity-overengineering-review`). Pre-warm the pool at load rather than paying the first `Instantiate` mid-gameplay.

### Draw calls, batching, and what breaks it

Each draw call is a CPU-side submission. 2D casual games rarely saturate a modern GPU on geometry, but they can easily saturate the main thread with submissions if batching breaks.

| Mechanism | Batches when | Broken by |
| --- | --- | --- |
| SRP Batcher (URP) | consecutive draws share the same **shader variant**, per-object data going to a constant buffer | a different shader variant between draws; a `MaterialPropertyBlock` set on the renderer, which removes SRP Batcher compatibility outright |
| Sprite/dynamic batching | small sprites share one material and one texture | a different material or texture between draws |
| Sprite atlas | sprites drawn together live in the same atlas page | sprites split across atlases, or an atlas that overflowed to a second page |

The recurring 2D cause is **material variants**: duplicating a material to tweak one colour or tint gives every copy a separate binding and splits an otherwise single batch into many. `MaterialPropertyBlock` is not the escape hatch it is often assumed to be - `Renderer.SetPropertyBlock` is the documented way to *remove* SRP Batcher compatibility for a renderer, and the dynamic batcher likewise refuses to merge renderers carrying different blocks. Prefer a built-in per-renderer channel (`SpriteRenderer.color`) or a shader parameter the batcher keeps in the per-object constant buffer. Sorting also matters: interleaving objects from different atlases in sorting order forces a state change between each, so a batch that "should" merge does not - sorting layer and atlas assignment are `unity-2d-rendering`.

Confirm the reason in the Frame Debugger, which names why a batch broke, rather than inferring it from the code.

### Overdraw: the 2D mobile fill-rate killer

Transparent sprites do not depth-reject. Every stacked transparent layer shades every covered pixel again, so cost scales with layers times screen coverage. This is the most common reason a casual 2D game runs fine on a flagship and badly on a budget phone, and it is invisible in a draw-call count.

Typical sources: a full-screen transparent background stack, large mostly-empty sprite quads with big alpha borders, particle effects covering the screen, several full-screen UI panels left active behind the top one, and tilemaps layered with transparent decoration passes.

Fixes, in order of usual payoff: draw large opaque backgrounds as opaque rather than transparent; deactivate or hide fully covered UI screens instead of leaving them rendering behind a popup (`unity-ui-patterns`); tighten sprite meshes so the quad is not mostly transparent border; reduce particle overdraw by cutting count and size, not just alpha. Confirm with the Frame Debugger and the platform's GPU profiler; the editor's overdraw view is directional only.

### Texture memory and compression

Texture memory is usually the largest runtime memory line in a 2D game and is a common cause of low-end crashes on load.

- **Import settings decide memory, not the source file size.** A compressed PNG on disk still uploads uncompressed if the import format says so
- **ASTC is the mobile default** on current Android and iOS hardware; block size trades quality against bytes. ETC2 remains the fallback for older Android tiers. Desktop uses BC/DXT. Confirm what the project's platform overrides actually set rather than assuming the default
- **Max size is the highest-leverage single setting.** A 2048 source imported at 2048 for a sprite drawn at 200px wastes memory proportional to the area ratio
- **Mipmaps: off for 2D UI and screen-aligned sprites**, which are drawn at a fixed scale and pay a ~33% memory increase for nothing. On for anything minified by camera zoom, where they cut sampling cost and shimmer
- Sprite atlases reduce draw calls and page waste but do not reduce pixel count - an atlas of oversized sprites is still oversized

### Physics 2D

Grid and board games should not use physics for board logic at all - legality and adjacency are rules-layer array math (`unity-architecture-patterns`). Where physics is genuinely used (TD projectiles, casual physics toys):

- Fixed timestep drives `FixedUpdate` frequency. A smaller timestep multiplies simulation cost per rendered frame; on mobile, a 30fps target with a default-ish timestep already runs multiple physics steps per frame
- Static colliders that move are the classic cost trap. Anything that moves needs a `Rigidbody2D` (kinematic is fine); moving a collider with no rigidbody forces static-geometry rebuilds
- Restrict the collision matrix so layers that can never interact are not tested
- Prefer non-allocating query overloads with a preallocated buffer; per-frame `*All` queries allocate an array each call

### UI Toolkit repaint cost

A UI Toolkit panel repaints when something in it changes; the cost scales with the number of visual elements affected, not with how small the change looked in code.

- Changing a style property that affects layout dirties layout for the subtree, which is more expensive than a property that only affects paint
- Per-frame text assignment allocates and re-tessellates - gate on value change
- Elements queried per frame with `Q<T>()` re-walk the tree; cache references at initialization
- Screens that are off-screen but still attached keep participating; detach or hide the panel rather than leaving a stack of live screens

Element lifetime, UXML structure, and panel scaling belong to `unity-ui-patterns`.

### Scene load time

- `SceneManager.LoadSceneAsync` rather than the synchronous form for anything the player waits on. Synchronous load blocks the main thread for the whole dependency graph, which reads as a freeze, not a load
- Load time is dominated by the **asset dependency graph**, not by scene file size. A prefab referencing a large atlas pulls it in whole
- `Resources` folder contents load into the build's index and are loaded eagerly in ways Addressables content is not - Addressables gives explicit control over what loads when (`unity-build-release` owns the build and catalog pipeline)
- Show a first frame quickly with a small entry scene, then load content additively behind a loading screen. `allowSceneActivation` lets you hold the swap until content is ready
- Measure cold start on a real device from process start, not from the editor's first frame

### ECS, Jobs, and Burst: when it is warranted

| Situation | Warranted? |
| --- | --- |
| 2048, Sudoku, Chess, quiz, Match-3 board of a few hundred cells | No. The board is array math costing microseconds |
| Idle game with offline progress math | No. The math is O(1) closed-form or a bounded loop |
| Tower defense with tens to a few hundred entities | No. Pooling and an update manager cover it |
| Thousands of independently simulated units in the steady frame loop, profiled as the bottleneck | Consider Jobs + Burst for that one system |
| Full ECS rewrite of a casual 2D game | No |

**For casual 2D this is almost never warranted.** Adopting ECS/DOTS restructures the whole codebase, splits the ecosystem of packages and tooling you can use, and adds a hard learning cost - to solve a bottleneck these genres do not have. The measured bottleneck in these games is GC, overdraw, and draw calls, none of which ECS fixes. Burst-compiling one profiled hot job is a bounded, reversible step; rearchitecting is not. Proposing ECS without a device profile naming it as the bottleneck is an overengineering finding (`unity-overengineering-review`).

## Output Format

Two modes, chosen by whether the request supplies something to diagnose or asks for code to be produced.

**Authoring mode** - the request is to write or design something. Emit the code or design, then any `Deferred:` lines. No finding blocks, no severity, no status line: nothing was measured, so a not-run line would misdescribe the work.

**Review mode** - a review workflow, or a direct performance investigation with a profile, source, or symptom report. Emit one block per finding:

```
### [Severity] {file:line | symbol or type.member, when source was supplied without paths | asset path | symptom, when no source was supplied}

- Category: {GCAllocation | UpdateCost | Instantiation | DrawCalls | Overdraw | TextureMemory | Physics2D | UIRepaint | LoadTime | BuildSize}
- Evidence: {measured (device, build) | estimated (no profile) | inferred (no source read)}
- Owner: {CPU | GPU | GC | Memory | Load}
- Code: {one-line citation, or `not supplied` when the finding is inferred}
- Cost: {with units - "1.4 KB/frame steady garbage", "500 Update calls/frame", "16 MB texture for a 200px sprite", "8 full-screen transparent layers"}
- Fix: {concrete change}
- Verify: {what to re-measure - GC.Alloc column, Frame Debugger batch count, Memory Profiler texture total, cold-start seconds}
```

`Severity: {Critical | High | Medium | Low}` - Critical = sustained missed frame budget, unbounded memory growth, or an OOM-risk load on the target device tier. High = a measurable regression on a primary gameplay path. Medium = cost on a rare path or only at unlikely content sizes. Low = a cheap win with no observed symptom.

Severity that does not fit a listed band: assign the nearest lower band and state why in `Cost`. `Category` takes exactly one value - where a defect fits two, pick the one the `Fix` addresses and name the other in `Cost`; where it fits none, pick the closest and name the real concern in `Cost`.

**Evidence gating.** `Evidence: measured (device, build)` requires a Profiler capture from a development build on a physical target device - name the device and build type. Anything else is `estimated (no profile)`. Use `inferred (no source read)` when the finding comes from a bug report, a CI log, or a stated fact rather than from source you read - state what was not seen. Never report a number you did not measure. When no profile and no source were supplied, `inferred (no source read)` wins - it is the stronger claim about what was not seen.

Both `estimated` and `inferred` bound the header at High: a Critical-band defect is written High, and `Cost` names the uncapped band. Neither ever raises a block - a Medium defect stays Medium. Among blocks sharing a band, order by what the reader must fix first: root cause before the symptoms it produces.

A defect owned by a sibling named in the ownership blockquote is not emitted as a finding. Write those after the findings, one per line, as `Deferred: {defect} -> {owning skill}`, so the workflow routes rather than drops them. In authoring mode the same line routes a design decision the sibling owns (`Deferred: atlas grouping for the batching fix -> unity-2d-rendering`). Omit entirely when there are none.

In review mode, close with exactly one status line, after any `Deferred:` lines:

| Condition | Line |
| --- | --- |
| One or more findings emitted | none - the findings are the output |
| No findings, and a symptom or report was available to reason from | `No performance findings.` |
| No source, diff, symptom, or report of any kind was supplied | `Performance check not run: no source supplied.` |

A symptom-only report (a QA ticket, a verbal description) is checkable input: emit `Evidence: inferred (no source read)` findings from it rather than the not-run line.

When invoked from an implementation workflow rather than a review, emit a budget table instead:

```
| Surface | Budget | Risk | Mitigation |
|---------|--------|------|------------|
| Match-3 cascade resolve | 33ms CPU / 30fps | 8x8 board, chain reactions, per-step alloc | resolve in rules layer, reuse buffers, cap steps per frame |
| Wave spawn (TD) | 33ms CPU / 30fps | 60 enemies instantiated at wave start | ObjectPool pre-warmed to 64, staggered activation |
| Board background + FX stack | GPU fill | 5 stacked transparent full-screen layers | opaque background, deactivate covered UI |
```

Budget defaults to 33ms at 30fps for mobile casual 2D; use 16ms where the project sets `Application.targetFrameRate = 60`. Take the target from the project's setting, not from aspiration.

## Avoid

- Timing in the editor, or on a non-target device, and calling it a measurement
- Reporting a number from a development build as the shipped number
- "Optimizing" without naming which of CPU, GPU, GC, or load was over budget
- Enabling incremental GC and leaving the per-frame allocation in place
- Empty `Update`/`FixedUpdate`/`LateUpdate` bodies left on many objects
- `Instantiate`/`Destroy` in a spawn loop, or a pool whose objects are not fully reset on reuse
- Duplicating a material to change one colour, then wondering why batches split
- Stacked full-screen transparent layers, and UI screens left rendering behind a popup
- Import settings left at defaults for sprites, or mipmaps enabled on screen-aligned 2D
- Physics used for grid or board legality
- Synchronous scene loads on a path the player waits through
- Proposing ECS/DOTS, Jobs, or Burst without a device profile naming that system as the bottleneck
- Asserting a batching or compression behaviour without confirming it in the Frame Debugger or import settings
