---
name: unity-2d-physics-input
description: Apply Unity Physics 2D and the Input System correctly - body types, layer masks, fixed timestep, touch gestures, and keeping grid games off physics.
metadata:
  category: mobile
  tags: [unity, physics2d, rigidbody2d, collider, input-system, touch, gestures, raycast]
user-invocable: false
---

# Unity 2D Physics and Input

> This skill owns **simulated 2D motion and how player intent enters the game**. What a move means once received belongs to `unity-2d-gameplay-patterns`; `FixedUpdate` ordering against other callbacks belongs to `unity-monobehaviour-lifecycle`; physics cost against the frame budget belongs to `unity-performance`; on-screen buttons and UI navigation belong to `unity-ui-patterns`.

## When to Use

- A genre needs simulated motion: tower-defense projectiles, falling or tossed objects, casual physics toys
- Wiring touch, drag, or swipe input for a board or menu
- Collisions fire at the wrong time, not at all, or against the wrong objects
- Reviewing whether physics is the right tool at all

## Rules

- **Grid and board games do not use physics for board logic.** 2048, Sudoku, Chess, and Match-3 resolve on array indices in the rules layer. Colliders as cell detectors, rigidbodies as falling tiles, and trigger overlaps as match detection are correctness bugs, not just cost
- Physics runs on the fixed timestep. Read and write rigidbody state in `FixedUpdate`; read input and set visuals in `Update`
- Body type is a decision, not a default. Static never moves, Kinematic moves under code control, Dynamic moves under forces. A moved Static collider forces a broadphase rebuild
- Never write `transform.position` on a body with a collider. Use `Rigidbody2D.MovePosition` for Kinematic and forces or `linearVelocity` for Dynamic. `Rigidbody2D.velocity` is the pre-Unity-6 spelling and is superseded by `linearVelocity` - flag it on sight in a 6.3 project
- A body fast enough to cross a collider within one step needs `CollisionDetectionMode2D.Continuous` on that body
- Every query passes an explicit `LayerMask`. An unmasked `Raycast2D` tests every collider in the scene and hits things the caller never meant to
- The Physics 2D collision matrix is part of the design. Turn off every layer pair that should never interact before optimising anything else
- Input is read inside the action callback or from the action's current value in the same frame; input state is not cached across frames or read from a background thread

## Patterns

### Physics or indices

| Situation | Answer |
| --- | --- |
| Tile slides into a cell, merges by rule | indices; the rules layer decides, the presenter animates |
| Piece drops into a column, lands on the stack | indices; a fall animation is a tween, not a simulation |
| Match-3 refill "gravity" | indices; column compaction is a loop, not falling bodies |
| TD projectile travelling to a target | physics or a tween; physics if it needs collision against varied shapes |
| Casual toss/stack/ragdoll toy where the simulation *is* the game | physics |

```csharp
// Bad - detects a match by trigger overlap; result depends on collider bounds and step timing
void OnTriggerEnter2D(Collider2D other) { if (other.CompareTag("Red")) _redCount++; }

// Good - the board knows its own contents; detection is deterministic and unit-testable
var matches = MatchDetector.Find(board);
```

The failure is determinism, not framerate. Physics results depend on step timing, contact ordering, and floating-point accumulation, so the same player action can produce different boards on different devices, and no undo or replay can be trusted. Cost is a secondary reason: hundreds of colliders and rigidbodies to model something an array already represents exactly.

### Colliders and their cost

| Collider2D | Cost | Use for |
| --- | --- | --- |
| Circle | cheapest | projectiles, pickups, most casual entities |
| Box / Capsule | cheap | platforms, walls, characters |
| Polygon | moderate; vertex count dominates | authored irregular shapes, kept low-poly |
| Composite (with Composite Collider 2D) | merges many child colliders into fewer shapes | tile-based level geometry |
| Edge | thin boundaries, no volume | world bounds, one-way surfaces |

Auto-generated polygon colliders from sprite outlines can produce dozens of vertices per object. Simplify them or replace with a primitive - a circle over a round sprite is exact and free by comparison.

`isTrigger` colliders still participate in the broadphase; they are not free, they only skip contact resolution.

### Rigidbody2D body types

```csharp
// Bad - a moving obstacle with a collider and no rigidbody; Unity treats it as a moved Static
// collider and rebuilds the static broadphase every frame
transform.Translate(Vector2.left * speed * Time.deltaTime);

// Good - Kinematic body, moved on the physics step, swept correctly against Dynamic bodies
void FixedUpdate() => _rb.MovePosition(_rb.position + _velocity * Time.fixedDeltaTime);
```

| Body type | Moves by | Right for |
| --- | --- | --- |
| Static | not at all | walls, world bounds, permanent geometry |
| Kinematic | `MovePosition` / `MoveRotation` from code | scripted paths, TD enemies on a lane, player-dragged objects |
| Dynamic | forces, gravity, collisions | objects whose motion the simulation should decide |

Kinematic bodies do not collide with other Kinematic or Static bodies by default and will not generate contacts unless `useFullKinematicContacts` is enabled. If two Kinematic objects must report collisions, enable it or make one Dynamic with gravity scale 0.

Set `Rigidbody2D.gravityScale = 0` for top-down games rather than zeroing project gravity, which also affects UI-adjacent and future systems.

### Fixed timestep and interpolation

Fixed Timestep (Project Settings > Time) defaults to 0.02s (50Hz). For mobile casual 2D, a slower step (0.0333s, 30Hz) halves physics cost where precision allows.

**Tunnelling is a per-body setting, not a timestep problem.** A fast body passing through a collider is fixed with `Rigidbody2D.collisionDetectionMode = CollisionDetectionMode2D.Continuous`, which sweeps that one body between steps. Raising the global step rate to fix it multiplies physics cost for every body in the scene, and on a device already dropping frames it makes the tunnelling worse, not better - the frame that stretches is the frame the body crosses the wall in. Set Continuous on the handful of bodies that need it (the ball, the bullet) and leave the rest Discrete.

The related containment lever is `Time.maximumDeltaTime` (default 0.333s), which bounds how many catch-up steps one long frame may run. Lowering it toward ~0.1s makes a catastrophic frame degrade into visible slow motion rather than a large unsimulated jump.

```csharp
// Bad - reads a tap in FixedUpdate, so an input between steps is missed entirely
void FixedUpdate() { if (_fire.WasPressedThisFrame()) Shoot(); }

// Good - latch intent in Update, consume it on the next physics step
void Update() { if (_fire.WasPressedThisFrame()) _fireQueued = true; }
void FixedUpdate() { if (_fireQueued) { Shoot(); _fireQueued = false; } }
```

Set Rigidbody2D Interpolation to Interpolate for anything the camera follows or the player watches closely; without it, motion visibly stutters when the render rate and the physics rate differ. Interpolate costs a small per-body overhead, so leave it off for off-screen or fast-moving projectiles.

`Time.fixedDeltaTime` is the correct delta inside `FixedUpdate`; `Time.deltaTime` returns the same value there, but writing `fixedDeltaTime` states the intent.

### Queries and layer masks

```csharp
// Bad - hits the shooter's own collider, background decor, and triggers
var hit = Physics2D.Raycast(origin, dir, range);

// Good - masked, distance-bounded, allocation-free
var count = Physics2D.Raycast(origin, dir, _filter, _hits, range);
```

Points to get right:

- Build masks with `LayerMask.GetMask("Enemy")` once and cache them; a per-frame string lookup is waste, and a hand-written integer literal breaks silently when layers are reordered
- Use the `ContactFilter2D` + results-buffer overloads in anything running per frame; the array-returning overloads allocate every call. The older `*NonAlloc` family is deprecated in the Unity 6 line in favour of these - do not reintroduce it. Build the filter once (`ContactFilter2D.CreateLegacyFilter` converts a layer mask) and reuse it
- `Physics2D.queriesHitTriggers` and `queriesStartInColliders` are project-wide defaults that change query results; set them deliberately or pass an explicit `ContactFilter2D`
- `OverlapCircle` / `OverlapBox` are the right tools for "what is in this area", not a raycast fan

Screen-to-world for tap targeting goes through the camera, and the Z matters:

```csharp
// Bad - ScreenToWorldPoint with an orthographic camera returns the near-plane Z
var world = _cam.ScreenToWorldPoint(screenPos);

// Good - flatten to the gameplay plane explicitly
var world = (Vector2)_cam.ScreenToWorldPoint(screenPos);
```

For a board game, convert the world point to a grid coordinate with the rules layer's own indexing convention rather than raycasting against per-cell colliders.

### Collision matrix

Physics 2D settings hold a layer-versus-layer matrix. Default is everything against everything, which is both a cost and a source of surprise contacts.

Start from a small layer set - `Player`, `Enemy`, `Projectile`, `Pickup`, `World` - and disable pairs that make no sense (`Projectile` vs `Projectile`, `Pickup` vs `Pickup`, `Enemy` vs `Enemy` in a lane-based TD). This is a settings change with no code cost and often the largest single physics win.

### Input System: actions and action maps

```csharp
// Bad - legacy polling, no rebinding, no device abstraction, misses taps between frames
if (Input.GetMouseButtonDown(0)) SelectCell(Input.mousePosition);

// Good - one action, works for mouse and touch, callback fires on the event
_actions.Gameplay.Select.performed += ctx => SelectCell(_point.ReadValue<Vector2>());
```

Structure:

- **Action map per input context**: `Gameplay`, `UI`, `Paused`. Enable exactly one gameplay map at a time; leaving both enabled is how a paused game still accepts board input
- **Action per intent**, not per device. `Select` binds `<Pointer>/press`, and mouse, pen, and touch all satisfy it
- Bind through `<Pointer>` for anything that is "the primary pointing device". Reach for `<Touchscreen>` only for genuinely multi-touch behaviour (pinch, two-finger pan)
- Generate the C# wrapper class from the `.inputactions` asset and use the typed API rather than string lookups
- Enable and disable maps in `OnEnable`/`OnDisable` so a disabled object stops receiving input (`unity-monobehaviour-lifecycle`)

### Touch gestures for board games

Tap, drag, and swipe are the whole input vocabulary for the target genres. Derive them from pointer position and press state rather than adding a gesture library.

```csharp
// Swipe (2048): direction from press-to-release delta, gated on distance and time
var d = endPos - startPos;
if (d.magnitude >= MinSwipePx && elapsed <= MaxSwipeSeconds)
    Dispatch(Mathf.Abs(d.x) > Mathf.Abs(d.y) ? (d.x > 0 ? Dir.Right : Dir.Left)
                                             : (d.y > 0 ? Dir.Up : Dir.Down));
```

Thresholds are in **screen-independent units, not raw pixels**. A 50-pixel swipe is a flick on a 320dpi phone and a twitch on a 600dpi one - scale by `Screen.dpi` (which can return 0 on some devices, so clamp to a fallback) or express thresholds as a fraction of the shorter screen dimension.

For Match-3 swap-by-drag, resolve the gesture to a source cell and a direction, then hand a single move to the rules layer. Do not let the drag continuously mutate the board.

Interactions (Tap, Hold, SlowTap) and processors on the action itself handle press-duration semantics without hand-rolled timers, and their exact parameter set is package-version dependent - read the action asset rather than assuming defaults.

### Reading input outside the callback

```csharp
// Bad - the button was pressed and released between two Update calls; the tap is lost
void Update() { if (Gamepad.current.buttonSouth.isPressed) Fire(); }

// Good - the callback fires for the event, not for the state at sample time
_actions.Gameplay.Fire.performed += _ => Fire();
```

`WasPressedThisFrame()` on an `InputAction` is safe inside `Update` because it reports edges within the frame. Reading `.isPressed` or `ReadValue` alone samples a level and drops short presses. Never read input in `FixedUpdate` (see above) or off the main thread.

Input System updates on a configurable mode (Dynamic, Fixed, Manual, in Project Settings > Input System Package). Changing it changes when callbacks fire relative to `Update` - leave it at the default unless a specific need justifies the change, and re-test input timing if it is changed.

### Rebinding

Interactive rebinding (`PerformInteractiveRebinding`) is only needed where the target platforms have keyboards or gamepads - the secondary desktop tier. Persist overrides with the action asset's binding-override JSON, saved through `unity-save-persistence`, and always ship a reset-to-default path. A touch-only build does not need rebinding UI at all.

## Output Format

Two modes, chosen by whether the request supplies code to judge or asks for code to be produced.

**Authoring mode** - the request is to write or design something. Emit the code or design, then any `Deferred:` lines. No finding blocks, no severity, no status line: nothing was reviewed, so a not-run line would misdescribe the work.

**Review mode** - source, a diff, or a symptom report was supplied. Emit one block per finding.

```
### [Severity] {file:line | symbol or type.member, when source was supplied without paths | asset path | symptom, when no source was supplied}

- Category: {PhysicsMisuse | BodyType | CollisionDetection | TimestepCoupling | QueryMask | CollisionMatrix | ColliderCost | ActionMap | GestureThreshold | InputTiming | LegacyInput}
- Evidence: {source | inferred (state what was not seen)}
- Code: {one-line citation - code, component setting, or project setting; or `not supplied` when the finding is inferred}
- Impact: {what breaks - "board state diverges across devices", "tap dropped on slow frames"}
- Fix: {concrete change}
```

`Severity: {Critical | High | Medium | Low}` - Critical = physics governs board or rule outcomes, or input is unreachable on a primary-tier device. High = dropped or duplicated input under normal play, a moved Static collider, or an unmasked per-frame query. Medium = a body-type, timestep, or collider-cost inefficiency with headroom. Low = a convention nit such as an uncached layer mask.

Severity that does not fit a listed band: assign the nearest lower band and state why in `Impact`. `Category` takes exactly one value - where a defect fits two, pick the one the `Fix` addresses and name the other in `Impact`; where it fits none, pick the closest and name the real concern in `Impact`.

`Evidence: inferred` is required whenever the source was not read. It bounds the header at High: a Critical-band defect is written High, and `Impact` names the uncapped band. It never raises a block - a Medium defect stays Medium. Among blocks sharing a band, order by what the reader must fix first: root cause before the symptoms it produces.

A defect owned by a sibling named in the ownership blockquote is not emitted as a finding. Write those after the findings, one per line, as `Deferred: {defect} -> {owning skill}`, so the workflow routes rather than drops them. Omit entirely when there are none.

In review mode, close with exactly one status line, after any `Deferred:` lines:

| Condition | Line |
| --- | --- |
| One or more findings emitted | none - the findings are the output |
| No findings, and a symptom or report was available to reason from | `No physics or input findings.` |
| No source, diff, symptom, or report of any kind was supplied | `Physics and input check not run: no source supplied.` |

A symptom-only report (a QA ticket, a verbal description) is checkable input: emit `Evidence: inferred` findings from it rather than the not-run line.

## Avoid

- Colliders, rigidbodies, or trigger overlaps used to model a grid board
- `transform.position` written on an object with a collider
- A moving collider with no Rigidbody2D
- Rigidbody state read or written in `Update`
- Input read in `FixedUpdate`
- Unmasked `Physics2D` queries, or masks built from layer-index literals
- Allocating query overloads called per frame
- The collision matrix left fully enabled
- Auto-generated polygon colliders left unsimplified
- Legacy `Input.GetMouseButton` / `Input.touches` alongside the Input System
- Gameplay and UI action maps enabled at the same time
- Swipe and drag thresholds expressed in raw pixels
- `.isPressed` polled where an edge is meant
