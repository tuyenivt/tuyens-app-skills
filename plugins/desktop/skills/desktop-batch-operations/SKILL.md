---
name: desktop-batch-operations
description: Destructive batch operations in Rust - dry-run preview, undo journals, collision and auto-suffix naming, atomic apply, per-item partial failure.
metadata:
  category: desktop
  tags: [rust, batch, rename, undo, preview, collision, auto-suffix, partial-failure, destructive]
user-invocable: false
---

# Desktop Batch Operations

> Confirm whether the operation is destructive before applying this skill - a read-only scan needs none of it, and inventing an undo for a filter is overhead.
>
> This skill owns **the semantics of applying a change to many files safely**. Where the plan type lives belongs to `desktop-core-architecture`; traversal and path mechanics to `desktop-filesystem-patterns`; TOCTOU and symlink escape to `desktop-security-patterns`; parallel execution to `desktop-concurrency-patterns`.

## When to Use

- Designing or reviewing rename, move, delete, copy, or overwrite across a set of files
- Any operation whose failure loses user data
- Collision handling and automatic name generation

## Rules

- **A destructive operation ships its preview and its undo in the same change.** Not as a follow-up
- Preview and apply share one computation. Two implementations of the same naming rules will drift, and the drift is invisible until it destroys data
- **The plan is validated again at apply time.** The tree may have changed since the preview
- Every item reports its own outcome. A batch never returns one aggregate `Ok` for a run containing failures
- A partially-applied batch is recoverable: the journal records what actually happened, not what was planned
- Collision detection accounts for names the batch itself will create, not only names already on disk
- **Nothing is applied that the user has not seen**, unless they explicitly chose to skip the preview

## Patterns

### Plan, preview, apply

```rust
pub struct Plan { pub steps: Vec<Step>, pub conflicts: Vec<Conflict> }
pub struct Step { pub from: PathBuf, pub to: PathBuf }

pub fn plan(inputs: &[PathBuf], rules: &Rules) -> Plan;   // pure, no I/O writes
pub fn apply(plan: &Plan) -> Journal;                      // executes, records
```

`plan` is pure enough to unit test exhaustively. `apply` is the only function that touches the disk destructively, which keeps the audit surface small.

`conflicts` carries what the plan cannot resolve on its own - an unwritable source, a target the policy forbids overwriting - and the preview shows them as blockers. A collision the suffix policy resolves is not a conflict; it is a step with an adjusted target.

### Collision resolution, including intra-batch collisions

The defect that separates a working rename tool from a broken one: two inputs whose new names collide with each other, where neither collides with anything on disk.

```rust
// Bad - only checks the disk; a.txt and A.txt both planning to become "photo.txt"
// produce two steps, and the second silently overwrites the first
fn resolve(target: &Path) -> PathBuf {
    if target.exists() { suffixed(target) } else { target.to_path_buf() }
}

// Good - the set of names this batch will create participates in the check
fn resolve(target: &Path, claimed: &mut HashSet<PathBuf>) -> PathBuf {
    let mut candidate = target.to_path_buf();
    let mut n = 1;
    while candidate.exists() || claimed.contains(&candidate) {
        candidate = suffixed(target, n);
        n += 1;
    }
    claimed.insert(candidate.clone());
    candidate
}
```

Three further cases the naive version misses:

- **Case-insensitive filesystems.** On Windows and default macOS, `Photo.txt` and `photo.txt` are the same name. `claimed` must compare case-insensitively where the target filesystem does, or the batch collides at apply time after a clean preview.
- **A case-only rename is not a collision.** `photo.jpg -> Photo.jpg` on a case-insensitive filesystem makes `candidate.exists()` true because the candidate *is* the source. Exempt a step's own source from the exists/claimed check, or every case-only rename gains a spurious suffix.
- **A later step's source is an earlier step's target.** Renaming `1.txt -> 2.txt` and `2.txt -> 3.txt` in that order destroys the original `2.txt`. Detect the dependency and apply the later step first (`2 -> 3`, then `1 -> 2`). A true cycle (`a -> b`, `b -> a`) has no safe order and must route through a temporary name.

### Auto-suffix naming

`(1)`, `(2)` is a convention with more edges than it appears:

```rust
// Insert before the extension, not at the end of the whole name
// photo.jpg -> photo (1).jpg    NOT   photo.jpg (1)

fn suffixed(path: &Path, n: u32) -> PathBuf {
    let stem = path.file_stem().unwrap_or_default();
    let mut name = stem.to_os_string();
    name.push(format!(" ({n})"));
    if let Some(ext) = path.extension() {
        name.push(".");
        name.push(ext);
    }
    path.with_file_name(name)
}
```

Decide and document three behaviours, because users notice all of them:

- A name that **already ends in a suffix**: does `photo (1).jpg` become `photo (2).jpg`, or `photo (1) (1).jpg`? Re-using the existing counter is usually expected
- **Multi-part extensions**: `archive.tar.gz` - `file_stem` gives `archive.tar`, so the naive version yields `archive.tar (1).gz`, which is wrong for this case and right for `photo.jpg`. Default: keep a small list of known multi-part extensions (`.tar.gz`, `.tar.bz2`, `.tar.xz`) and suffix before the whole unit - `archive (1).tar.gz`; split on the last extension for everything else. Test whichever rule ships
- **Length limits**: appending a suffix can push a name past the filesystem's limit; truncate the stem rather than failing the item

### The undo journal

```rust
// Bad - records the plan, so an interrupted run's journal claims work that never happened
let journal = Journal::from_plan(&plan);
for step in &plan.steps { fs::rename(&step.from, &step.to)?; }

// Good - records each step as it completes; an interrupted run leaves an accurate journal
let mut journal = Journal::new();
for step in &plan.steps {
    match fs::rename(&step.from, &step.to) {
        Ok(()) => journal.record(Applied { from: step.from.clone(), to: step.to.clone() }),
        Err(e) => journal.record(Failed { path: step.from.clone(), error: e.to_string() }),
    }
}
```

The journal is written incrementally and durably where the batch is large enough that a crash mid-run is plausible. A journal held only in memory does not survive the process being killed - which is exactly when undo matters most. Its home is the app's data directory, not the tree being modified; `desktop-data-persistence` owns the schema and location conventions.

Undo replays the journal in reverse. It is itself a batch operation and gets the same collision handling: the original name may have been taken since.

### Per-item outcomes

```rust
// Bad - one failure aborts, leaving the tree half-changed and the caller uninformed
pub fn apply(plan: &Plan) -> Result<(), Error> {
    for step in &plan.steps { fs::rename(&step.from, &step.to)?; }
    Ok(())
}

// Good - every item gets an outcome; the caller sees the mix
pub fn apply(plan: &Plan) -> Vec<StepOutcome> { /* ... */ }

pub enum StepOutcome {
    Applied { from: PathBuf, to: PathBuf },
    Skipped { path: PathBuf, reason: SkipReason },
    Failed { path: PathBuf, error: OpError },
}
```

The `?` version is the more common defect and the worse one: it stops at the first permission error, having already renamed some files, and returns an `Err` that says nothing about what was done.

### Delete is not rename

For delete specifically, prefer the OS recycle bin (`trash` crate) over `fs::remove_file`. It gives the user an undo path the app does not have to implement, and it is what they expect from a desktop tool. Where a permanent delete is genuinely wanted, make it an explicit choice, not the default.

## Output Format

When this skill produces a finding:

```
[Must|Recommend] {file:line | symbol, when source was supplied without paths}
Operation: <rename | move | delete | overwrite | copy>
Gap: <preview | undo | preview-apply drift | apply-time revalidation | intra-batch collision | case-insensitive collision | rename cycle | partial-failure reporting | journal accuracy | journal durability | suffix rule>
Consequence: <what the user loses, concretely>
Fix: <the concrete change>
```

`[Must]` when the consequence is lost, overwritten, or unrecoverable user data - most findings here are. `[Recommend]` when the defect costs clarity or convention (suffix wording, report formatting) without risking data.

A destructive operation reviewed with no findings closes with exactly `No batch-operation findings.` A request about a read-only operation closes with exactly `Not applicable: operation is not destructive.` and nothing else - do not invent preview or undo requirements for it.

A defect owned by a sibling named in the ownership blockquote is written after the findings as `Deferred: {defect} -> {owning skill}`, one per line. Omit when there are none.

When designing rather than reviewing:

```
Operation: <name>
Preview shows: <what the user sees before applying>
Collision policy: <disk + intra-batch + case sensitivity, and the suffix rule chosen>
Undo: <what the journal records, where it lives, and whether it survives a kill>
Partial failure: <what a mid-batch failure leaves, and how the report conveys it>
Revalidation: <what is re-checked at apply time>
```

## Avoid

- Shipping an apply path with no preview
- Computing the planned names twice, once for preview and once for apply
- Checking only the disk for collisions, ignoring names the batch will create
- Comparing names case-sensitively on a case-insensitive filesystem
- Ignoring the rename-cycle case where a step's source is an earlier step's target
- Appending `(1)` after the extension
- `?` inside the apply loop, aborting the batch on the first error
- Returning one aggregate `Ok` for a run that had failures
- Building the journal from the plan rather than from what completed
- Holding the journal only in memory for a long-running destructive batch
- `fs::remove_file` as the default delete where the recycle bin is available
- Trusting a preview computed minutes ago without revalidating at apply time
- An undo that ignores collisions on the restored names
