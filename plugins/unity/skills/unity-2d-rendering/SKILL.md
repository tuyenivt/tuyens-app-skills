---
name: unity-2d-rendering
description: Set up Unity 2D rendering for mobile - sprite atlases and batching, sorting layers, URP 2D lights, tilemaps, pixel-perfect camera, overdraw control.
metadata:
  category: mobile
  tags: [unity, 2d, sprites, atlas, sorting, urp, tilemap, camera, overdraw, batching]
user-invocable: false
---

# Unity 2D Rendering

> This skill owns **how 2D visuals are composed and submitted to the GPU**. Frame budget measurement and the profiler-first workflow belong to `unity-performance`; UI Toolkit panels and screen layout belong to `unity-ui-patterns`; when a renderer is created and destroyed belongs to `unity-monobehaviour-lifecycle`; what the board looks like as data belongs to `unity-2d-gameplay-patterns`.

## When to Use

- Setting up sprites, atlases, sorting, or the camera for a 2D game
- Draw calls, batches, or fill rate appear in a profile
- Sprites render in the wrong order or flicker between frames
- Adding 2D lights, tilemaps, or a pixel-art presentation

## Rules

- **Sprites batch only when they share a material and texture.** In practice that means atlas membership: two sprites from different atlases cannot batch, however identical their material
- Sorting is decided by sorting layer, then order-in-layer, then distance to camera. Fix draw order with the first two; leave Z at a constant for a 2D game unless the camera is perspective
- A multi-sprite entity that must sort as a unit carries a `SortingGroup`. Per-child order-in-layer does not survive interleaving with other entities
- Ties in the full sort key are resolved by the renderer's internal order, which is not stable across frames or scene loads. Never leave two overlapping sprites on an identical layer and order
- Orthographic size is half the visible world height. Design against a fixed world height and let width vary with aspect ratio
- Every transparent sprite costs fill rate whether or not it is visible behind another. On mobile, overdraw is the fill-rate ceiling, not triangle count
- Material property changes made per-instance break batching. `MaterialPropertyBlock` is not the fix: on a Scriptable Render Pipeline, calling `SetPropertyBlock` removes SRP Batcher compatibility for that renderer, and the dynamic batcher will not merge renderers with differing blocks either. Prefer a per-vertex or built-in channel the batch already carries (`SpriteRenderer.color`), or accept a shared variant, and measure

## Patterns

### Atlases and what actually batches

Sprite Atlas (the `com.unity.2d.sprite` package, in the engine by default) packs loose sprites into one texture at build time. The payoff is batching and fewer texture binds. On 6000.3 the Editor's Sprite Atlas Mode defaults to **Sprite Atlas V2 - Enabled**; V1 is deprecated and the migration to V2 is one-way, so treat a project still pinned to V1 in Project Settings > Editor as a finding.

```
// Bad - each UI icon and each tile a separate texture; every distinct sprite is its own batch
Assets/Art/Icons/*.png  (no atlas)

// Good - one atlas per screen or per logical group; the group draws in one batch
Assets/Art/Atlases/Gameplay.spriteatlas   <- board tiles, effects
Assets/Art/Atlases/MetaUI.spriteatlas     <- menu and HUD art
```

Group by **what appears on screen together**, not by folder. An atlas containing gameplay tiles plus rarely-shown menu art loads the whole page into memory for either use.

Constraints worth knowing before committing to a layout:

- A sprite in two atlases is packed twice and wastes that memory; keep membership exclusive
- Atlas size is capped by the platform's max texture size; oversize atlases split into multiple pages and lose the single-batch benefit
- Tight packing saves space but the generated mesh has more vertices than a quad. For small icons, full-rect is often cheaper overall
- Compression format is per-platform (ASTC on modern Android/iOS). Set it per-atlas rather than accepting the default, and verify in the build report

A sprite drawn between two atlased sprites from a *different* atlas breaks the batch in three. Draw-order interleaving is as much a batching concern as material assignment.

### Sorting: layers, order, and Z

| Mechanism | Use for | Note |
| --- | --- | --- |
| Sorting Layer | coarse bands: Background, Board, Pieces, Effects, Overlay | project-wide, ordered list in Tags and Layers |
| Order in Layer | ordering within a band | signed int; leave gaps (10, 20, 30) so insertions do not renumber |
| Z position | avoid in 2D | affects orthographic sort but not layout; a stray Z is invisible until it reorders something |

```csharp
// Bad - fights the layer system with transform Z, which nothing else in the project reads
transform.position = new Vector3(x, y, -0.01f * row);

// Good - the intent is explicit and visible in the inspector
renderer.sortingLayerName = "Pieces";
renderer.sortingOrder = row * 10;
```

For a top-down board where nearer rows must overlap farther ones, derive order-in-layer from the grid row using the same coordinate convention the rules layer uses (`unity-2d-gameplay-patterns`). Deriving it from world Y instead invites float ties.

### SortingGroup for composite entities

A piece made of body plus outline plus badge has three renderers. Without grouping, another entity's sprite can render between them.

```csharp
// Bad - child order-in-layer is global; a neighbouring piece interleaves
// Piece/Body (order 10), Piece/Badge (order 12), Neighbour/Body (order 11) -> badge behind neighbour

// Good - the group sorts as one unit; children sort only against each other
[RequireComponent(typeof(SortingGroup))]
```

`SortingGroup` is a real cost: it forces its subtree into a separate sorting decision and can break batching with surrounding sprites. Add it to composite entities that visibly need it, not to every prefab.

### 2D lights under URP

2D lights require the URP 2D Renderer (Renderer 2D asset) to be the assigned renderer; with the Universal Renderer they do nothing. Verify the pipeline asset before debugging a light that is not appearing.

Cost model on mobile, in rough order:

| Feature | Mobile cost |
| --- | --- |
| Global light 2D | cheap; usually one per blend-style layer |
| Point/spot light 2D | per-light extra draw over affected sprites; watch the count |
| Shadow casters 2D | expensive; each caster adds geometry and a pass |
| Multiple blend styles | each active style is a separate render target |

Blend styles are configured on the Renderer 2D asset, which holds at most **four**, the first being `Default`; every 2D light picks one of them. Each style that is actually in use costs a render target, so the lever is using fewer of the four, not how many are defined - two is enough for most effects. For a casual 2D puzzle game, a single global light plus baked highlights in the art is usually the correct answer, and lit sprites need the Sprite-Lit material; unlit sprites ignore 2D lights entirely.

### Tilemaps

Tilemap renders in chunks. The relevant knobs:

- **Chunk Culling Bounds**: tiles whose visual extends past the chunk (tall props, glow) get culled while still on screen. Extend the bounds rather than disabling culling
- **Mode: Chunk vs Individual**: Chunk batches per chunk and is the default choice; Individual is required only when per-tile sort order matters and costs a draw call per tile
- Tiles from one Tile Palette should share one atlas, otherwise a chunk splits into several batches
- Rebuilding a large tilemap at runtime (`SetTile` in a loop) is a main-thread cost; use `SetTiles` with a bulk array, or `SetTilesBlock`, for bulk edits

For a fixed-size puzzle board, a tilemap is often more machinery than a pooled grid of `SpriteRenderer`s. Choose tilemap for authored levels and large scrolling worlds; choose sprites for a board the rules layer already indexes.

### Camera and resolution independence

```csharp
// Bad - assumes a 16:9 device; tall phones crop the board, tablets show empty margin
camera.orthographicSize = 5f;

// Good - guarantee a world width by deriving size from the current aspect
var worldHeightForWidth = targetWorldWidth / camera.aspect;
camera.orthographicSize = Mathf.Max(minSize, worldHeightForWidth * 0.5f);
```

Design a **safe world rectangle** the board must fit, then fit it to the narrower axis. Recompute on resolution change, not only at `Start` - device rotation, split-screen, and desktop window resize all change `camera.aspect`.

For pixel art, add the Pixel Perfect Camera component (Reference Resolution plus Assets Pixels Per Unit) and let it own orthographic size; a manual size assignment fights it and produces shimmer. Its Upscale Render Texture and Crop Frame options trade sharpness against letterboxing - pick one deliberately, because the default varies with the package version.

World-space versus screen-space: gameplay elements that must align to the board live in world space and move with the camera. HUD, popups, and menus belong to UI Toolkit (`unity-ui-patterns`), not to world-space sprites positioned by unprojecting screen coordinates.

### Overdraw

Overdraw is the mobile fill-rate killer in 2D, because transparent sprites cannot be depth-rejected and every layer is shaded for every covered pixel.

```
// Bad - 5 full-screen transparent layers over the board = 5x fill on every pixel
Background + Vignette + Gradient + ParticleSheet + DimPanel

// Good - flatten static layers into one authored background; dim only when a modal is open
```

Practical reductions, cheapest first:

- Delete or bake full-screen transparent overlays that never change
- Disable, do not just fade to alpha 0 - a fully transparent sprite still costs fill
- Trim sprite alpha borders (tight mesh in the sprite editor) so the quad is not mostly empty
- Cap particle overdraw with fewer, larger, shorter-lived particles rather than many stacked ones
- Use the Rendering Debugger's overdraw visualisation to find the layers, rather than guessing

`unity-performance` owns whether overdraw is actually your bottleneck; this skill owns what to do once it is.

### Material and shader variants

```csharp
// Bad - allocates a material instance per sprite; batching drops to one draw call each
spriteRenderer.material.color = tint;

// Also bad under URP - SetPropertyBlock removes SRP Batcher compatibility for this renderer
var mpb = new MaterialPropertyBlock();
mpb.SetColor(ColorId, tint);
spriteRenderer.SetPropertyBlock(mpb);

// Good - a built-in per-renderer channel the sprite batch already carries
spriteRenderer.color = tint;
```

`renderer.material` instantiates on access; `renderer.sharedMaterial` does not. Cache shader property IDs (`Shader.PropertyToID`) rather than passing strings per call. For a simple tint, `SpriteRenderer.color` is the batch-safe path and needs neither a material copy nor a property block. When a per-instance value genuinely has no built-in channel, confirm the cost in the Frame Debugger rather than assuming the block is free.

Keyword variants multiply shader compilation and can stall on first use. For casual 2D, one sprite shader with a small fixed feature set beats a variant per effect.

## Output Format

Two modes, chosen by whether the request supplies code to judge or asks for code to be produced.

**Authoring mode** - the request is to write or design something. Emit the code or design, then any `Deferred:` lines. No finding blocks, no severity, no status line: nothing was reviewed, so a not-run line would misdescribe the work.

**Review mode** - source, a diff, or a symptom report was supplied. Emit one block per finding.

```
### [Severity] {file:line | symbol or type.member, when source was supplied without paths | asset path | symptom, when no source was supplied}

- Category: {Batching | AtlasLayout | SortingOrder | SortingGroup | Lighting2D | Tilemap | CameraFit | Overdraw | MaterialVariant}
- Evidence: {source | inferred (state what was not seen)}
- Code: {one-line citation - code, component setting, or asset configuration; or `not supplied` when the finding is inferred}
- Impact: {what it costs - "board cropped on 20:9 devices", "5x full-screen overdraw"}
- Fix: {concrete change}
```

`Severity: {Critical | High | Medium | Low}` - Critical = content unreachable or illegible on a supported device (board cropped, sprites invisible), or a frame-rate collapse traced to this cause. High = a measurable fill-rate or draw-call cost on the primary mobile tier, or a nondeterministic sort tie in gameplay-critical art. Medium = an inefficiency with headroom to absorb it. Low = a convention or maintainability nit.

Severity that does not fit a listed band: assign the nearest lower band and state why in `Impact`. `Category` takes exactly one value - where a defect fits two, pick the one the `Fix` addresses and name the other in `Impact`; where it fits none, pick the closest and name the real concern in `Impact`.

`Evidence: inferred` is required whenever the source was not read. It bounds the header at High: a Critical-band defect is written High, and `Impact` names the uncapped band. It never raises a block - a Medium defect stays Medium. Among blocks sharing a band, order by what the reader must fix first: root cause before the symptoms it produces.

A defect owned by a sibling named in the ownership blockquote is not emitted as a finding. Write those after the findings, one per line, as `Deferred: {defect} -> {owning skill}`, so the workflow routes rather than drops them. Omit entirely when there are none.

In review mode, close with exactly one status line, after any `Deferred:` lines:

| Condition | Line |
| --- | --- |
| One or more findings emitted | none - the findings are the output |
| No findings, and a symptom or report was available to reason from | `No rendering findings.` |
| No source, diff, symptom, or report of any kind was supplied | `Rendering check not run: no source supplied.` |

A symptom-only report (a QA ticket, a verbal description) is checkable input: emit `Evidence: inferred` findings from it rather than the not-run line.

## Avoid

- Loose sprites where an atlas would batch them
- One atlas mixing gameplay and menu art
- Transform Z used to order 2D sprites
- Two overlapping sprites sharing a sorting layer and order
- `SortingGroup` added to every prefab by default
- 2D lights or shadow casters added without checking the Renderer 2D asset is assigned
- Tilemap chunk culling left at default when tiles overflow their cell
- `orthographicSize` set to a constant
- Manual orthographic size assignment alongside Pixel Perfect Camera
- Full-screen transparent overlays stacked and left enabled at alpha 0
- `renderer.material` touched at runtime for a tint
- Shader property names passed as strings per frame
