---
name: rust-error-handling
description: Rust error design - thiserror in core vs anyhow in the app, per-item batch failures, io::ErrorKind matching, justified panics, actionable messages.
metadata:
  category: desktop
  tags: [rust, errors, thiserror, anyhow, result, panic, unwrap, io-errorkind, partial-failure]
user-invocable: false
---

# Rust Error Handling

> This skill owns **how failure is typed, propagated, and told to the user**. Which crate an error type lives in belongs to `desktop-core-architecture`; per-item journals and undo semantics to `desktop-batch-operations`; retry and path-existence races to `desktop-filesystem-patterns`; joining threads and channel-disconnect handling to `desktop-concurrency-patterns`; where the message is rendered to `iced-widget-patterns`.

## When to Use

- Choosing an error type for a new core module or app path
- Reviewing a diff containing `unwrap`, `expect`, `?`, or a new error enum
- A batch operation must report which items failed and why
- An error crosses a thread or task boundary before reaching the UI

## Rules

- **The core crate uses `thiserror`; the app crate uses `anyhow`.** Core errors are matchable variants the UI branches on; app errors are contextual chains the user reads. Never `anyhow::Error` in a core public signature
- **Every filesystem, parse, decode, or user-input failure is a `Result`.** `unwrap`/`expect` on any of these is a defect regardless of how unlikely the failure looks
- `panic!`, `unwrap`, and `expect` are legitimate only for a genuine program invariant, and the message states the invariant, not the symptom
- An error crossing a thread boundary is `Send + 'static`. `io::Error` and `thiserror` enums are; borrowed data and `Rc` are not
- **A batch error names the item it belongs to.** An error type used per-item carries the path or index, or the caller cannot report or retry it
- Match `io::ErrorKind`, never the message string. `NotFound`, `PermissionDenied`, and `AlreadyExists` drive different UI; the rest collapse to one branch
- Every error that reaches the user states what failed, which item, and what they can do. An error the user cannot act on is a bug report, not a message
- `ErrorKind` variants stabilize over releases - check the toolchain before matching a variant outside the stable set

## Patterns

### `thiserror` in core, `anyhow` in the app

```rust
// core/src/error.rs - variants the UI can branch on
#[derive(Debug, thiserror::Error)]
pub enum ScanError {
    // PathBuf has no Display impl - interpolate via `.display()`, not `{path}`
    #[error("cannot read {}: {source}", path.display())]
    Read { path: PathBuf, #[source] source: std::io::Error },
    #[error("{} is outside the selected root", path.display())]
    OutsideRoot { path: PathBuf },
}
```

```rust
// app/src/main.rs - context chains, rendered as text
use anyhow::Context;
let cfg = load_config(&path)
    .with_context(|| format!("loading settings from {}", path.display()))?;
```

The distinction is who consumes the error. Core returns a type the UI matches on to pick a dialog, an inline warning, or a retry button. The app wraps whatever it gets in context for display. A core function returning `anyhow::Error` erases the variants and forces the UI to string-match, which is the failure this rule exists to prevent.

Keep `#[source]` on the wrapped cause rather than flattening it into the message - the chain is what makes a nested failure diagnosable.

### Partial failure: the error carries the item

```rust
// Bad - one error for the whole batch; the caller cannot say which file failed
pub fn hash_all(paths: &[PathBuf]) -> Result<Vec<ContentHash>, io::Error>

// Good - every item has an outcome, and a failure names its path
pub struct ItemError { pub path: PathBuf, pub source: ScanError }
pub fn hash_all(paths: &[PathBuf]) -> Vec<Result<ContentHash, ItemError>>
```

The bad signature stops at the first unreadable file and discards the work already done. For a 100k-file scan that is a full restart over one locked file. The good one lets the UI render "98,412 hashed, 3 unreadable" with the three paths listed and a retry that touches only those three.

Report the mix in a summary type rather than a bare count, so the UI does not recompute it:

```rust
pub struct BatchReport { pub succeeded: usize, pub failed: Vec<ItemError>, pub skipped: Vec<Skip> }
```

### When panic is legitimate

```rust
// Bad - a filesystem result; the file may be locked, missing, or on a dead drive
let meta = fs::metadata(&path).unwrap();

// Bad - the message restates the symptom and names no invariant
let idx = self.selected.expect("selected was None");

// Good - a real invariant, stated, upheld by the constructor
let group = self.groups.get(idx)
    .expect("selected_index is only set from a valid groups index; see Model::select");
```

The test: could this fail on a user's machine with valid input? Filesystem, parse, decode, network, and user input all can - those are `Result`. A slice index the constructor guarantees, a `HashMap` key inserted three lines above, and an unreachable match arm cannot - those are `expect` with the invariant written out.

Two more legitimate panics: a `Mutex` `PoisonError` in an app that has no meaningful recovery from a poisoned lock, and a startup-time configuration invariant that makes the program meaningless if violated. Both still state the invariant.

In the UI crate specifically, a panic inside `update` or `view` takes down the window mid-operation and loses unsaved work - the bar for panicking there is higher, not lower.

### `io::ErrorKind`, not message strings

```rust
// Bad - locale-dependent, formatting-dependent, silently wrong
if e.to_string().contains("not found") { /* ... */ }

// Good - matches the discriminant, exhaustive with a fallback
match e.kind() {
    ErrorKind::NotFound => Action::RemoveFromList,
    ErrorKind::PermissionDenied => Action::PromptElevate,
    ErrorKind::AlreadyExists => Action::OfferSuffix,
    _ => Action::ReportAndContinue,
}
```

`ErrorKind` is `#[non_exhaustive]`, so a `_` arm is required and is the honest default: unhandled kinds report and continue rather than being mapped to a guess. On Windows, kinds that map poorly (sharing violations, path-too-long) arrive as `Uncategorized` or with a raw OS error - read `e.raw_os_error()` when a platform-specific branch is genuinely needed, and confine that branch behind `cfg(windows)`.

### Errors across threads

```rust
// Bad - the worker panics; the join yields Box<dyn Any> nobody decodes, and the UI hangs
let h = thread::spawn(move || hash_all(&paths).unwrap());

// Good - the error is a value, sent on the same channel as the results
let h = thread::spawn(move || {
    for p in paths {
        let _ = tx.send(match hash(&p) {
            Ok(h) => Event::Hashed(p, h),
            Err(e) => Event::Failed(ItemError { path: p, source: e }),
        });
    }
});
```

A failure is an event, not an exception - it travels the same channel as success so ordering and progress accounting stay correct. Error types crossing the boundary need `Send + 'static`; `thiserror` enums over owned data satisfy this, `anyhow::Error` also does. When a worker can panic despite this, treat a `JoinHandle` `Err` as an unconditional failure of every item that worker owned rather than dropping it silently.

### The message that reaches the user

```rust
// Bad - the user sees the Debug form of an io::Error and can do nothing with it
"Os { code: 32, kind: Uncategorized, message: \"The process cannot access the file\" }"

// Good - item, cause, action
"Could not rename 'report.pdf': the file is open in another program. Close it and retry."
```

Build this by keeping the path in the error type and mapping `ErrorKind` to the action sentence at the UI edge. The technical chain still belongs in the log or a "details" expander - it just is not the headline. A batch summary follows the same shape: the counts, then the failing items, then one retry affordance scoped to those items.

## Output Format

When this skill produces a finding:

```
[Must|Recommend] {file:line | symbol, when source was supplied without paths}
Category: <error-type-choice | unwrap-on-fallible | panic-message | partial-failure | thread-boundary | errorkind-matching | user-message | context-loss>
Issue: <the defect, named>
Consequence: <what the user or caller loses - "the batch aborts at the first locked file", "the UI cannot distinguish missing from denied">
Fix: <the concrete type or call change>
```

A defect owned by a sibling named in the ownership blockquote is written after the findings as `Deferred: {defect} -> {owning skill}`, one per line. Omit when there are none.

When designing an error path rather than reviewing:

```
Layer: <core | app>
Error type: <the enum or anyhow, and why that layer takes it>
Per-item: <what each failed item carries | not a batch operation>
Thread crossing: <how the error travels, or `stays on one thread`>
Panics: <each remaining unwrap/expect with the invariant it asserts | none>
User message: <the sentence the user sees, and the action it offers>
```

Every `unwrap`/`expect` in reviewed code gets a line, findings or not: `Panic: <file:line> - <invariant asserted | FALLIBLE - must be a Result>`.

`[Must]` marks a defect the Rules name - an unwrap on a fallible result, a batch that aborts on its first item, a message the user cannot act on. `[Recommend]` marks a working path with a better shape - a context string worth adding, a summary type over a bare count.

A review that produces no finding closes with exactly `No error handling findings.` after the `Panic:` lines - a justified `expect` with its invariant stated is not a finding.

## Avoid

- `anyhow::Error` in a core crate's public signature
- `unwrap`/`expect` on a filesystem, parse, decode, or user-input result
- `expect` whose message restates the symptom instead of the invariant
- `Result<Vec<T>, E>` for a batch, where one failure discards all successes
- An error type that omits the path or index of the item that failed
- `?` inside a per-item loop, aborting the batch on the first failure
- Matching `e.to_string()` or a message substring instead of `e.kind()`
- Omitting the `_` arm on a `match` over `io::ErrorKind`
- Dropping a `JoinHandle` result, so a worker panic vanishes
- Flattening `#[source]` into the display string, losing the cause chain
- Showing the user a `Debug`-formatted `io::Error`
- `panic!` inside Iced's `update` or `view`
- `.ok()` or `let _ =` discarding a `Result` that carries a user-visible failure
