---
name: unity-2d-gameplay-patterns
description: Model 2D board and turn games as engine-free C# - pure reversible moves, grid state, cascade termination, seeded RNG, snapshot undo.
metadata:
  category: mobile
  tags: [unity, csharp, gameplay, grid, board, determinism, undo, state-machine, rng]
user-invocable: false
---

# Unity 2D Gameplay Patterns

> This skill owns **rule algorithms and how game state is modelled and advanced**. Where that code lives and the engine-free assembly boundary belong to `unity-architecture-patterns`; drawing the board belongs to `unity-2d-rendering`; reading the player's gesture belongs to `unity-2d-physics-input`; currency and progression math belong to `unity-game-economy-progression`; writing state to disk belongs to `unity-save-persistence`.

## When to Use

- Modelling a board, grid, or turn structure for a puzzle or casual 2D game
- Adding undo/redo, replay, or a daily-seeded puzzle
- Implementing move legality, match detection, or cascade resolution
- Reviewing gameplay code for determinism, mutation, or non-terminating loops

## Rules

- **Every rule in this skill is plain C# with no `UnityEngine` dependency.** Board state, legality, resolution, and scoring compile and run outside the engine. This is the plugin's central constraint, owned by `unity-architecture-patterns`
- A move returns a **new state**; it never mutates the state passed in. Reversibility comes from keeping the previous value, not from writing an inverse operation
- A move returns whether it **changed anything**. "No tile moved" is a distinct outcome from "the move was illegal", and both differ from a successful move
- Randomness arrives as an injected seeded source. `UnityEngine.Random` and an unseeded `System.Random` both make a bug unreproducible
- Every resolution loop has a **stated termination bound** checked in code. An unbounded cascade is a hang, not a slow frame
- Grid indexing uses one documented convention throughout - row-major or column-major, origin corner named, applied identically in rules, input, and rendering
- Validation and application are separate: `IsLegal(state, move)` answers a question, `Apply(state, move)` produces a state. A method that both checks and mutates cannot be used to build a legal-move list

## Patterns

### State as a value, moves as functions

```csharp
// Bad - mutates in place; undo needs a hand-written inverse for every move type
public void SlideLeft() { for (var r = 0; r < 4; r++) CollapseRow(_cells, r); }

// Good - previous state is still intact, so undo is a pop
public readonly record struct MoveResult(BoardState Board, int ScoreGained, bool Changed);
public static MoveResult SlideLeft(in BoardState board) { /* builds a new BoardState */ }
```

Undo/redo becomes two stacks of `BoardState`. Push before applying, pop to undo. For a 4x4 or 9x9 board a snapshot is tens of bytes, so snapshot-per-move costs less than maintaining inverse operations and cannot drift out of sync with the forward move.

For large boards, store snapshots as the flat backing array only, and reconstruct derived data (score totals, match caches) on restore rather than snapshotting it.

### Board representation

| Board | Representation | Why |
| --- | --- | --- |
| 2048, Sudoku, Match-3 | flat array `T[width * height]`, index `y * width + x` | one allocation, cache-friendly, trivial to copy and hash |
| Chess-like, 64 squares | `ulong` bitboards per piece type plus a mailbox array | set operations and attack masks become single instructions |
| Irregular or sparse board | dictionary keyed by a coordinate struct | only where the rectangle is mostly empty |

`T[,]` (true 2D) is slower to index than a flat array in Unity's IL2CPP builds and cannot be sliced or copied as cheaply. Prefer the flat array with one indexing helper.

**The row-major trap.** `y * width + x` and `x * height + y` are both valid; mixing them silently transposes the board and only shows up on non-square grids. Write one accessor and never index the backing array directly:

```csharp
// Bad - the convention is re-derived at each call site, and one of them is wrong
var cell = _cells[x * _height + y];

// Good - one definition, one place to be wrong
public int At(int x, int y) => y * Width + x;
```

Bounds checks belong in the accessor too. A silent wrap from `x = -1` reading the previous row's last cell is the classic off-by-one in grid games.

### Turn and phase as an explicit state machine

Game phase is an enum plus a transition table, not a scatter of booleans:

```csharp
// Bad - illegal combinations are representable; input arrives mid-cascade
bool _isAnimating, _isPlayerTurn, _isGameOver;

// Good - exactly one phase is current; input is rejected outside AwaitingInput
enum Phase { AwaitingInput, Resolving, Animating, GameOver }
```

The presentation layer reads the phase to decide whether to accept input. The rules layer advances it. A cascade running while `AwaitingInput` is true is the bug this shape prevents.

Animation duration must not gate rule progression: resolve the full move in the rules layer first, then play the resulting animation sequence. Otherwise a skipped or interrupted animation desynchronises the board from the display.

### Genre micro-examples

Each is the rule kernel only - legality, detection, or generation. Everything around it is the same pure-function shape above.

**2048 slide-merge.** Compact non-zero cells toward the wall, then merge equal neighbours once, then compact again. The merge flag is what stops `4 4 4 4` becoming a single `16`:

```csharp
// One row, left: [2,2,4,0] -> compact [2,2,4] -> merge [4,4] -> pad [4,4,0,0]
static int[] SlideRow(int[] row) { /* compact, single-pass merge with a merged flag, compact */ }
```

A tile that merged this move cannot merge again this move. `2 2 4` slides to `4 4`, not `8`.

**Sudoku constraint check.** Legality is three set-membership tests, not a solver:

```csharp
static bool IsLegal(byte[] grid, int index, byte value) =>
    !RowHas(grid, index / 9, value) && !ColHas(grid, index % 9, value)
    && !BoxHas(grid, (index / 27) * 3 + (index % 9) / 3, value);
```

Track per-row/column/box occupancy as nine `ushort` bitmasks to make this O(1) and to make generation and hint-solving affordable.

**Chess move generation and legality.** Generation and legality are two stages, and conflating them is the common bug:

```csharp
// Pseudo-legal: piece movement rules only, ignoring self-check
IEnumerable<Move> Pseudo(Position p);
// Legal: apply, then reject if the mover's own king is attacked
bool IsLegal(Position p, Move m) => !IsKingAttacked(Apply(p, m), p.SideToMove);
```

Because `Apply` is pure, the legality filter is a one-liner and needs no make/unmake pair. Castling, en passant, and promotion are position state (rights, target square), not board contents, so they must live in `Position` alongside the squares or they are lost on snapshot.

**Match-3 detection.** Scan runs, do not compare fixed offsets:

```csharp
// Per row and per column: extend a run while colour matches; emit indices when run >= 3
// Intersecting runs merge into one group, which is how an L or T shape scores as one match
```

Detection returns the matched cell set for the caller to clear. It does not clear, score, or refill - keeping those separate is what makes cascade resolution testable.

### Cascade resolution with a termination guarantee

```csharp
// Bad - a refill rule that can regenerate a match makes this a hang, not a slow frame
while (TryFindMatches(board, out var m)) board = Refill(Clear(board, m));

// Good - bounded, and the bound is an assertion about the rules
const int MaxCascades = 32;
for (var i = 0; i < MaxCascades && TryFindMatches(board, out var m); i++)
    board = Refill(Clear(board, m));
```

The bound is not defensive decoration: hitting it means the refill can produce matches indefinitely, which is a rules bug worth surfacing in a test rather than shipping as a freeze. Assert on the bound in tests; log and break in release.

Resolve the whole cascade in the rules layer, producing an ordered list of steps. The presenter then animates that list. Interleaving resolution with animation frames is what makes cascade code both untestable and prone to input arriving mid-chain.

### Seeded randomness and replay

```csharp
// Bad - board differs per device; a reported bug cannot be reproduced
var spawn = UnityEngine.Random.Range(0, free.Count);

// Good - the seed is part of the game state and is saved with it
public sealed class SeededRandom(int seed) : IRandom { /* System.Random */ }
```

Requirements this buys, all of which are common in the target genres:

- **Daily puzzle**: seed derived from the date, identical board for every player
- **Replay**: seed plus the move list reproduces the game exactly; store those instead of every frame
- **Reproducible bug reports**: seed plus moves is the whole repro

Two constraints. Save the **generator's position** (or the move count that reconstructs it), not just the seed, or a resumed game diverges from the original. And do not assume `System.Random`'s sequence is stable across .NET runtime versions - if cross-version reproducibility matters, implement a small explicit PRNG (xorshift, PCG) in the rules assembly so the sequence is yours.

### Advancing the loop

Turn-based genres (2048, Sudoku, Chess, quiz) advance on input, not on frames - there is no per-frame simulation to run, and `Update` should only poll for the phase change. Continuous genres (tower defense, idle) advance on a fixed accumulated step so behaviour is frame-rate independent:

```csharp
// Bad - spawn rate and projectile travel vary with device frame rate
void Update() { _sim.Advance(Time.deltaTime); }

// Good - the rules layer takes fixed ticks; the remainder carries to the next frame
_acc += dt; while (_acc >= Tick) { _sim.Step(Tick); _acc -= Tick; }
```

Cap the number of catch-up steps per frame (backgrounding produces a huge `dt`), and let `unity-game-economy-progression` own the elapsed-time math for anything longer than a frame hitch.

## Output Format

When reviewing, emit one block per finding:

```
### [Severity] {file:line | asset path | symptom, when no source was supplied}

- Category: {Mutation | Determinism | Termination | GridIndexing | PhaseModel | EngineCoupling | LegalitySeam | SnapshotIntegrity}
- Evidence: {source | inferred (state what was not seen)}
- Code: {one-line citation, or `not supplied` when the finding is inferred}
- Impact: {what breaks - "undo restores a corrupted board", "cascade can hang the main thread"}
- Fix: {concrete change}
```

`Severity: {Critical | High | Medium | Low}` - Critical = an unbounded resolution loop, or state corruption that survives into a save. High = non-deterministic rules, in-place mutation defeating undo/replay, or `UnityEngine` referenced from a rule. Medium = an indexing or phase-model flaw contained to one screen. Low = a clarity or structure nit with no current behavioural cost.

Severity that does not fit a listed band: assign the nearest lower band and state why in `Impact`. `Category` takes exactly one value - where a defect fits two, pick the one the `Fix` addresses and name the other in `Impact`; where it fits none, pick the closest and name the real concern in `Impact`.

`Evidence: inferred` is required whenever the source was not read, and caps the block at High - write the capped severity in the header, not the uncapped one, and name the uncapped band in `Impact`.

A defect owned by a sibling named in the ownership blockquote is not emitted as a finding. Write those after the findings, one per line, as `Deferred: {defect} -> {owning skill}`, so the workflow routes rather than drops them. Omit entirely when there are none.

Close with exactly one status line, after any `Deferred:` lines:

| Condition | Line |
| --- | --- |
| One or more findings emitted | none - the findings are the output |
| No findings, and a symptom or report was available to reason from | `No gameplay findings.` |
| No source, diff, symptom, or report of any kind was supplied | `Gameplay check not run: no source supplied.` |

A symptom-only report (a QA ticket, a verbal description) is checkable input: emit `Evidence: inferred` findings from it rather than the not-run line.

## Avoid

- Mutating board state in place where undo, replay, or a legal-move list is needed
- `UnityEngine.Random`, `Time.deltaTime`, or `DateTime.Now` inside a rule
- Resolution loops with no bound
- Re-deriving the flat-array index convention at each call site
- Booleans standing in for game phase
- Rule progression gated on animation completion
- Detection methods that also clear, score, or refill
- Saving a seed without the generator's position
- Depending on `System.Random`'s exact sequence for cross-version reproducibility
- Chess position state (castling rights, en passant target) stored outside the snapshotted position
