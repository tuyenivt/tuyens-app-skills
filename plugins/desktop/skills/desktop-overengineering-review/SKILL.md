---
name: desktop-overengineering-review
description: Rust/Iced necessity review - trait with one implementer, Arc<Mutex<>> on single-owner data, premature async, a second runtime, dead feature flags.
metadata:
  category: desktop
  tags: [rust, iced, code-review, overengineering, necessity, traits, async, generics]
user-invocable: false
---

# Desktop Overengineering Review

> Confirm the crate layout and the async runtime already in the workspace from `Cargo.toml` and `Cargo.lock` first. An established convention is context, not a finding - review what the diff adds against it.
>
> This skill owns **whether a layer earns its keep**. Where code lives and which way dependencies point belongs to `desktop-core-architecture`; Iced's Model-Message-Update-View shape to `iced-architecture-patterns`; ownership, borrowing, and idiom to `rust-language-patterns`; error type design to `rust-error-handling`; measured cost to `desktop-performance`; GPU transfer-cost analysis to `desktop-gpu-compute`.

## When to Use

- Reviewing a Rust desktop diff that adds traits, generics, channels, `async`, feature flags, custom widgets, or a framework
- Catching code that compiles, passes `clippy`, and passes tests but does not need to exist

## Rules

- Every finding names what makes the abstraction unnecessary: one implementer and no test double, one instantiation, one thread, one call site, no measurement, the branch is unreachable. When several stack, comma-separate them in `Unnecessary because:`
- Intent:
  - **`[Recommend]`** (default). Name the constraint, recommend the edit. Escalate to **`[Must]`** when measurable or structural cost is present; cite it in `Cost:`. In a design-only review with nothing to measure, the structural triggers below still escalate. Triggers: an abstraction that forces a test to spin an executor where a plain function call sufficed; a lock or channel whose contention or latency is measurable; a second async runtime in the process; a branch presented as handling a case it can never reach
  - **`[Recommend]`** when justification is plausible but not visible in the diff - state the assumption and ask the author to confirm
- An abstraction with **visible** justification - a second implementer, a test double, a benchmark in the PR - is not a finding
- **Scale is the discriminator, and scale is not domain.** Price an abstraction against the variation it absorbs: maintainer count, shipped platforms, supported file formats, locales, and runtime-selected backends. A solo-maintained two-platform file utility absorbs almost no variation and earns almost no layers; cite the project's actual numbers, not a general principle
- **This skill has a floor as well as a ceiling.** Where structure is absent rather than excessive - a single `main.rs` holding traversal, hashing, and view code; no core crate; `iced` reachable from domain logic; no seam to test a destructive operation - say so plainly and route it to `desktop-core-architecture`. Never read "solo project" or "small utility" as licence for no structure: a codebase where one rename rule costs edits in nine places has already paid more than the abstraction would have cost
- Never propose deleting a layer the diff's own tests bind to
- **Performance abstractions need a measurement, not an argument.** A cache, a channel, a thread pool, or a GPU path introduced without a benchmark is speculative regardless of how reasonable it sounds

## Patterns

### Category 1: Type-System Ceremony

#### Trait with one implementer and no test double

```rust
// Bad - one impl, never substituted, never faked
pub trait FileScanner { fn scan(&self, root: &Path) -> Vec<PathBuf>; }
pub struct WalkdirScanner;

// Good - the function
pub fn scan(root: &Path) -> Vec<PathBuf> { ... }
```

Justified when a fake or a second implementation exists, or arrives in the same PR. A `Clock` or `Rng` trait substituted in tests is a genuine seam (`desktop-core-architecture`) - check for the substitution before flagging.

#### Generic parameter with one instantiation

```rust
// Bad - Store<T> only ever Store<Settings>; monomorphized once, generic forever
pub struct Store<T: Serialize + DeserializeOwned> { ... }
```

Justified at 2+ instantiations, or when the type is a published crate's public API.

#### Newtype per primitive with no invariant

```rust
// Bad - a wrapper that enforces nothing and is unwrapped at every call site
pub struct FileCount(usize);

// Good - a newtype that upholds something
pub struct NonEmptyPattern(String);   // constructor rejects ""
```

Justified when the constructor validates, when two same-typed parameters would otherwise be swappable, or when it carries a unit.

#### Builder for three fields

```rust
// Bad - four methods and a `build()` returning Result for a struct with three fields
RenameOptions::builder().pattern(p).recursive(true).dry_run(false).build()?

// Good
RenameOptions { pattern: p, recursive: true, dry_run: false }
```

Justified above roughly five fields, with genuine optionality, or when the type is public API needing backwards-compatible extension. `..Default::default()` covers most of the remaining cases.

### Category 2: Premature Concurrency

#### `Arc<Mutex<T>>` around data one thread owns

```rust
// Bad - the lock is never contended; one thread touches this
let state = Arc::new(Mutex::new(ScanState::default()));

// Good - plain ownership; the borrow checker already proved single access
let mut state = ScanState::default();
```

`Cost:` every read becomes a fallible lock acquisition and a poisoning path that must be handled or unwrapped. Justified when a second thread or an Iced task genuinely holds a handle.

#### `async` in a CPU-bound path

```rust
// Bad - async signature over pure computation; every caller must be async or block
pub async fn hash_file(path: &Path) -> Result<Hash> { ... }

// Good - sync core; the UI moves it off-thread at the boundary
pub fn hash_file(path: &Path) -> Result<Hash> { ... }
```

`Cost:` the async colour propagates through every caller, and unit tests now need an executor for a function that does no I/O. Hashing, comparison, and image decode are CPU-bound. Offloading belongs at the Iced boundary (`iced-async-patterns`), not in the core signature.

#### A channel where a return value suffices

```rust
// Bad - a channel to move one value between two adjacent, known points
let (tx, rx) = mpsc::channel();
compute(tx);
let result = rx.recv()?;

// Good
let result = compute();
```

Justified for streaming progress from a long operation, or when producer and consumer genuinely do not know each other. Progress reporting is a real case; a single result is not.

#### A second async runtime alongside Iced's executor

```rust
// Bad - tokio spun up inside an Iced app that already has an executor
let rt = tokio::runtime::Runtime::new()?;
```

`Cost:` two thread pools, two schedulers, and a hard-to-diagnose class of bug where a future is polled by the wrong one. Justified only when a dependency mandates a specific runtime and no alternative exists - `ureq` replaces `reqwest` precisely to avoid this (`desktop-ecosystem-boundaries`).

### Category 3: Speculative Performance

#### A cache or index with no benchmark

A memoization layer, a precomputed index, or a thread pool added "for performance" with no measurement cited. `desktop-performance` owns the measurement discipline. The finding here is the abstraction added on a guess; if a benchmark is cited, this is not a finding regardless of the outcome.

#### `clone()` scattered to silence the borrow checker

```rust
// Bad - a clone per call site because the ownership story was never decided
let plan = self.plan.clone();
```

Clones that express ownership transfer are fine. Clones that appear wherever the compiler complained are an unresolved design; the finding names the ownership decision that was skipped and routes mechanics to `rust-language-patterns`. State the data size when it is large - that is the `Cost:`.

#### A GPU path with no CPU baseline

A `wgpu` compute port with no timing against the CPU implementation it replaces is speculative. `desktop-gpu-compute` owns the transfer-cost analysis.

### Category 4: Structural Excess

#### `Box<dyn Error>` across the core's public API

A missing type, not a surplus one, and it makes every layer above it unreviewable because callers can only print. Always `Deferred: -> rust-error-handling`.

#### Custom widget where built-in composition works

```rust
// Bad - a hand-rolled Widget impl with layout, draw, and event plumbing for a labelled row
// Good - row![text(label), horizontal_space(), toggler(value)]
```

Justified for genuinely novel drawing or interaction. `iced-widget-patterns` owns the composition vocabulary; check it before accepting that a custom widget was needed.

#### ECS or an actor framework for a file utility

`Cost:` a second programming model, a separate debugging story, and domain logic that is no longer plain Rust. A bulk renamer and an image deduplicator are batch pipelines over a list. Justified only with a profile showing the entity or message volume that motivates it.

#### Dead feature flag

```rust
// Bad - a cfg branch no test or build profile ever compiles
#[cfg(feature = "new_scanner")]
```

Justified when a build profile, a platform split, or an optional dependency backs it, and that configuration is actually built. Otherwise delete the dead branch - an uncompiled branch rots.

## Output Format

One block per finding; the consuming workflow merges them:

```
### [Must | Recommend] {file:line | symbol, when source was supplied without paths | symptom, when no source was supplied}

- Category: {Type-System Ceremony | Premature Concurrency | Speculative Performance | Structural Excess | Absent Structure}
- Code: {one-line citation, or `not supplied` when the finding is inferred}
- Unnecessary because: {what makes it dead, unread, or single-valued; comma-separate when stacked} -- OR, for Absent Structure -- Missing because: {what the absence costs}
- Cost: {required for [Must]; omit otherwise}
- Recommendation: {concrete Rust or manifest edit; for Absent Structure, the extraction and its owning skill}
- Justified when: {one-line note if a legitimate reason might apply; otherwise omit}
```

`Absent Structure` is the floor rule's category. Its `Cost:` is the edit-site count or the regression count already being paid, and that count is what escalates the block to `[Must]`.

Output order: finding blocks, then `Justified as-is:` lines, then the per-category zero-finding lines, then `Deferred:` lines. An abstraction examined and found justified is written one per line, so the reader can tell a defended layer from an unexamined one:

```
Justified as-is: {abstraction} - {the visible justification: implementer count, the test double, the benchmark}
```

This is the required form whenever the request questions an existing layer, since `No <category> findings.` alone reads as "nothing was checked" rather than "this was checked and it holds".

For each category with zero findings, emit exactly: `No <category> findings.` (using the category name from the enum) so the workflow knows the check ran; append ` - not assessable from this input` when the input could not exercise that category (structure is unobservable from a design sketch). Omit this line for categories that have at least one finding. Emit `Necessity check not run: no source supplied.` instead of the per-category lines only when nothing at all was supplied - a prose description of the design is checkable input, and yields findings, `Justified as-is:` lines, or `Deferred:` lines like any other source.

A defect owned by a sibling named in the ownership blockquote is not emitted as a finding. Write those at the end, one per line, as `Deferred: {defect} -> {owning skill}`, so the workflow routes rather than drops them. Omit entirely when there are none.

## Avoid

- Flagging a trait that a test double or a second implementation substitutes
- Flagging `Arc<Mutex<>>` where an Iced task or a worker thread holds a second handle
- Flagging `async` on a function that awaits real I/O - synchronous reads inside an `async fn` do not count
- Flagging a channel that streams progress from a long-running operation
- Flagging a cache, thread pool, or GPU path whose PR cites a benchmark
- Flagging the runtime, crate layout, or error convention the workspace already standardized on, or proposing migration off it
- Treating file count, module count, or line count as a complexity metric
- Removing a layer the diff's own tests bind to
- Raising findings against generated code, vendored sources, or `target/`
- Reading "solo maintainer" as licence to skip the floor check
