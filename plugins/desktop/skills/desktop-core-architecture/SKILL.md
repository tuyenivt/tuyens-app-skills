---
name: desktop-core-architecture
description: GUI-free core boundary for Rust desktop apps - what lives in core vs UI, keeping iced out of the manifest, injection seams, plan-and-apply shape.
metadata:
  category: desktop
  tags: [rust, iced, architecture, crate-boundary, testability, dependency-injection, workspace]
user-invocable: false
---

# Desktop Core Architecture

> Confirm the workspace layout from `Cargo.toml` first - a project with one crate and no core split is a different starting point from one that already has the boundary.
>
> This skill owns **where code lives and which way dependencies point**. Iced's Model-Message-Update-View shape belongs to `iced-architecture-patterns`; the semantics of preview and undo to `desktop-batch-operations`; whether a layer earns its keep to `desktop-overengineering-review`.

## When to Use

- Planning a new Rust desktop feature's crate layout
- Reviewing a diff that adds a module, a crate, or a dependency edge
- Deciding whether a type belongs in the core or the UI crate

## Rules

- **The core crate has no `iced` dependency.** This is enforced by its `Cargo.toml`, not by convention. A core that can `use iced::` will eventually do so
- Dependencies point one way: UI depends on core, never the reverse. A core module that knows a view exists is a violation regardless of how it is spelled
- An operation resolves to an **inspectable plan** before anything is applied. Preview renders the plan; apply consumes the same plan
- Ambient capabilities - the clock, the filesystem root, randomness - arrive as parameters or trait objects. A core function that reaches a global is untestable by construction
- The core owns domain types. A message carrying a re-declared parallel struct duplicates a definition that will drift
- **A trait is introduced for a seam that exists**, not for symmetry. One implementer plus one test double is a seam; one implementer alone is not

## Patterns

### The boundary, enforced by the manifest

```toml
# Bad - core/Cargo.toml; nothing stops a view type leaking in
[dependencies]
iced = "0.14"
walkdir = "2"

# Good - core/Cargo.toml; the boundary is mechanical
[dependencies]
walkdir = "2"
rayon = "1"
```

The UI crate depends on core:

```toml
# app/Cargo.toml
[dependencies]
iced = "0.14"
myapp-core = { path = "../core" }
```

A workspace makes this cheap to set up and cheap to check. `cargo tree -p myapp-core | grep iced` returning nothing is a CI-checkable invariant.

### What goes where

| Concern | Crate | Why |
| --- | --- | --- |
| Traversal, filtering, grouping | core | Pure computation over paths |
| Hashing, comparison, dedup grouping | core | No UI involvement |
| Rename planning, collision resolution | core | The rules that must be exhaustively tested |
| Apply and undo | core | Destructive; needs test coverage a GUI cannot give |
| Progress *events* | core | The operation reports; it does not know who listens |
| Progress *rendering* | UI | Widget state |
| Selection state, scroll position, dialog visibility | UI | View state, meaningless to the core |
| Message enum, `update`, view functions | UI | Iced's shape |
| Settings *schema and migration* | core | Testable, versioned |
| Settings *editing widgets* | UI | View state |

The recurring mistake is putting "progress" wholly in the UI. The core emits progress as values; the UI decides they become a progress bar.

### Plan and apply share one computation

```rust
// Bad - preview and apply are two implementations of the same rules; they will drift
pub fn preview_renames(files: &[PathBuf]) -> Vec<String> { /* compute names */ }
pub fn apply_renames(files: &[PathBuf]) -> Result<()> { /* compute names AGAIN, then rename */ }

// Good - one computation, inspected or executed
pub struct RenamePlan { pub steps: Vec<RenameStep> }
pub struct RenameStep { pub from: PathBuf, pub to: PathBuf }

pub fn plan_renames(files: &[PathBuf], rules: &Rules) -> Result<RenamePlan>;
pub fn apply(plan: &RenamePlan) -> Vec<StepOutcome>;
```

The preview is `plan_renames` rendered. The apply is `plan_renames` executed. There is no second place for the naming rules to live, so preview cannot lie about what apply will do - which for a destructive operation is the difference between a safe tool and an unsafe one.

### Injection seams

```rust
// Bad - reaches the real world; a test must create real files at real times
pub fn stale_entries(dir: &Path) -> Vec<PathBuf> {
    let cutoff = SystemTime::now() - Duration::from_secs(86400);
    // ...
}

// Good - the capability arrives; the test supplies a fixed instant
pub fn stale_entries(dir: &Path, now: SystemTime) -> Vec<PathBuf> {
    let cutoff = now - Duration::from_secs(86400);
    // ...
}
```

Prefer a plain parameter over a trait. Introduce a trait only where the seam has real variation - a filesystem abstraction is worth it when tests must simulate permission errors; it is overhead when `tempfile` already gives a real directory cheaply.

### Domain types cross the boundary; view types do not

```rust
// Bad - the UI re-declares what core already defines; two definitions to keep in sync
#[derive(Debug, Clone)]
enum Message { DuplicateFound { path: String, size: u64, hash: String } }

// Good - the message carries the core type
#[derive(Debug, Clone)]
enum Message { DuplicateFound(core::DuplicateGroup) }
```

The core type derives `Clone` and `Debug` so Iced messages can carry it. That is the core accommodating a general constraint, not a UI dependency.

### Brownfield: the boundary starts as a module

In a single-crate app with logic inline in `update()`, scope the fix to the feature at hand: extract its rules into a plain module with no `iced` import, called from `update()` rather than written in it. The module gets the plan/apply shape and injected capabilities like any core code; only the manifest enforcement is missing, so `grep -r "use iced" src/<module>/` returning nothing stands in for `cargo tree`. Promote the module to a real core crate when a second feature joins it - as its own change, not this one.

## Output Format

When this skill produces a finding, each carries:

```
[Must|Recommend] {file:line | symbol, when source was supplied without paths}
Boundary: <core -> UI | UI -> core | within core | within UI>
Issue: <the violation, named>
Why it matters: <what becomes untestable, or what will drift>
Fix: <the concrete move or signature change>
```

`core -> UI` covers any dependency from core toward the UI side, including a GUI-framework crate in the core manifest.

`[Must]` when the boundary is breached mechanically (a GUI crate in core's manifest, core importing a UI type) or when preview and apply compute the plan in two places. `[Recommend]` for seams, type placement, and structure.

A defect owned by a sibling named in the ownership blockquote is written after the findings as `Deferred: {defect} -> {owning skill}`, one per line. Omit when there are none.

When designing a new layout or assessing an existing one, rather than reviewing a diff, produce instead:

```
Workspace: <crates and their dependency edges>
Core purity: <clean | `iced` present in core | no core crate>
Seams: <injected capabilities, or `none - core reaches ambient state at <file:line>`>
Misplaced: <each type in the wrong crate, with its destination | none>
```

## Avoid

- `iced` in the core crate's `Cargo.toml`
- A core module importing a view type, a message enum, or a widget
- Preview and apply computing the same plan in two places
- A core function calling `SystemTime::now()`, `rand::random()`, or reading an environment variable directly
- Re-declaring a core type in the UI crate to avoid a dependency
- A trait with one implementer and no test double, introduced for symmetry
- Putting progress reporting entirely in the UI so the core cannot be tested for it
- A whole-workspace re-layering proposed as a fix for one feature's misplacement
- `pub` on everything in the core so the UI can reach internals the boundary should hide
