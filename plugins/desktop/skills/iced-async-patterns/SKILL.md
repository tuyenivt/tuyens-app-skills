---
name: iced-async-patterns
description: Iced 0.14 async - Task vs Subscription, streaming progress from long scans, cancellation, backpressure, bridging rayon, one executor only.
metadata:
  category: desktop
  tags: [rust, iced, async, task, subscription, progress, cancellation, backpressure, rayon, executor]
user-invocable: false
---

# Iced Async Patterns

> **Iced is pre-1.0, and this plugin's projects track latest rather than pinning a minor.** `Task`, `Subscription`, and the stream helpers have been renamed and re-signatured across minor versions (`Command` became `Task` at 0.13), so **read the resolved version from `Cargo.lock`** - `Cargo.toml` holds a range and cannot answer whether a helper exists - and verify every signature against that version's docs before writing or accepting one.
>
> This skill owns **work that outlives one `update` call and how its results get back**. The `Message` enum and the non-blocking `update` rule belong to `iced-architecture-patterns`; rendering progress, cancel buttons, and error states to `iced-widget-patterns`; thread pools, channels, and shared-state safety to `desktop-concurrency-patterns`; per-item failure types to `rust-error-handling`.

## When to Use

- A scan, hash, or decode job runs longer than a frame
- Progress must appear while a job is still running
- Adding a Cancel button, or a rescan that supersedes a running one
- Integrating `rayon`, a blocking library, or a file watcher with Iced

## Rules

- **Do not start a second async runtime.** Iced owns an executor; spawning a `tokio::Runtime` alongside it means two thread pools, two schedulers, and `block_on` deadlocks. Use `Task`, or a plain `std::thread` bridged by a channel
- **Never `block_on` in the UI crate.** Not in `update`, not in `view`, not in a helper either calls
- CPU-bound work does not run on the executor's async tasks. Hash and decode go to `rayon` or a dedicated thread, with results streamed back
- **Every long job is cancellable**, and cancellation is checked inside the work loop. A job that only stops between items is not cancellable for a large item
- A stale job's messages are discarded by generation id. Cancelling and starting a rescan must not let the old job's results overwrite the new one's
- Progress messages are coalesced. One message per file over 100k files floods the message queue and starves rendering - the thing progress exists to prevent
- A `Task` produced from `update` is the app's own work; a `Subscription` is an external stream the app subscribes to. A job that the user started is not a subscription because it happens to be long
- Version-sensitive claims (whether a `Task` helper exists, its exact name) are checked against the pinned version, never asserted

## Patterns

### Task for a job with one result

```rust
// Bad - blocks the UI thread for the whole scan
Message::Scan => { self.results = core::scan(&self.root); Task::none() }

// Good - handed off; update returns immediately, the result arrives as a message
Message::Scan => Task::perform(core::scan_async(self.root.clone()), Message::Scanned)
```

For work that is CPU-bound rather than await-bound, do not wrap the blocking call in an async fn and hand it to the executor - that occupies an executor thread for the duration. Move it to a thread or `rayon` and stream the outcome back, as below.

### Streaming progress from a long job

The shape that matters for this app class: a scan over 100k files reports as it goes, without blocking `update` and without one message per file.

```rust
// Worker side - std::thread plus rayon, one bounded channel, one message per chunk
let (tx, rx) = std::sync::mpsc::sync_channel(8);   // bounded: producer blocks if the UI falls behind
std::thread::spawn(move || {
    let done = AtomicUsize::new(0);
    paths.par_chunks(256).for_each(|chunk| {
        if cancel.load(Ordering::Relaxed) { return; }   // per-item granularity: see Cancellation below
        let items: Vec<_> = chunk.iter().map(hash).collect();
        let n = done.fetch_add(items.len(), Ordering::Relaxed) + items.len();
        let _ = tx.send(Event::Batch { done: n, items });   // progress and results, coalesced together
    });
    let _ = tx.send(Event::Finished);
});
```

The UI side turns that receiver into a stream of messages via the pinned version's stream-backed `Task` constructor - this scan is user-started, so per the rule above it is a `Task`, not a `Subscription`. The constructor's name and signature differ per minor version; look up the pinned version's helper rather than assuming one. The model carries the job's generation id and `update` ignores messages from any other generation.

Coalescing is not optional at this scale, and it covers results as well as progress. One message per file makes the queue the bottleneck and the progress bar update slower than if it were not there at all, and 100k individual `Item` messages cost 100k `update` calls. Send one batch per chunk - every 256 items or every 100 ms are both defensible - and extend the model's vector per batch.

### Cancellation that actually stops work

```rust
// Bad - the flag is checked once before the loop; a running job ignores Cancel entirely
if !cancel.load(Ordering::Relaxed) { for p in paths { hash(p); } }

// Bad - checked per file, but a single 4 GB file still runs to completion
for p in paths { if cancel.load(Ordering::Relaxed) { break; } hash(p); }

// Good - checked per item and inside the per-item read loop
for p in paths {
    if cancel.load(Ordering::Relaxed) { break; }
    hash_with_cancel(p, &cancel)?;   // checks between buffer reads
}
```

Cancellation granularity is set by the largest single item, not the average. Dropping a `Task` handle stops the future from being polled, but it does not stop a `std::thread` or a `rayon` job already running - those need the flag. `Task` in recent Iced versions offers an abortable handle; confirm whether the pinned version has it and use it in addition to, not instead of, the worker-side flag.

State cleanup is part of cancellation: on Cancel, bump the generation, clear in-flight progress, and return the model to a state the user can act from. A cancel that leaves `scanning: true` forever is a hang with extra steps.

### Superseded jobs and generation ids

```rust
// Bad - the user rescans; the first scan's late results overwrite the second's
Message::Scanned(results) => { self.results = results; Task::none() }

// Good - late messages from a superseded job are dropped
Message::Scanned(gen, results) if gen == self.generation => { self.results = results; Task::none() }
Message::Scanned(..) => Task::none(),
```

This is the defect behind "the results are from the folder I scanned before". Any message carrying the output of a job the user can restart carries the generation it belongs to.

### Backpressure

An unbounded channel from a fast producer to a UI that consumes one batch per frame grows without limit - a scan producing faster than the UI drains is a memory climb over a large tree. Use a bounded channel so the producer blocks when the UI falls behind, or aggregate on the producer side so what is sent is already bounded (counts and batched chunks rather than every event).

The producer blocking is the correct outcome: the scan is I/O bound anyway, and a stalled sender costs nothing the user perceives, whereas an unbounded queue costs memory and latency they do.

### Rayon alongside Iced's executor

`rayon` and Iced's executor are different pools with different purposes and coexist fine - `rayon` for data-parallel CPU work, the executor for driving the UI and awaiting. The bridge between them is a channel, never `block_on`.

```rust
// Bad - blocks an executor thread waiting on rayon, and can deadlock
Task::perform(async move { rayon::scope(|s| { /* ... */ }); results }, Message::Done)

// Good - rayon runs on its own pool; the channel carries results to the UI
std::thread::spawn(move || { paths.par_iter().for_each(|p| { let _ = tx.send(hash(p)); }); });
```

The same rule covers a blocking third-party library: run it on `std::thread`, report through a channel. Adding `tokio` with its own runtime to get `spawn_blocking` is the mistake this rule prevents - it doubles the thread pools for a job a channel already solves.

### Subscriptions for external streams

A file watcher, a global keyboard hook, or a timer is a `Subscription`: it produces events the app did not request, for as long as the model keeps declaring it. Stop one by no longer returning it from `subscription()`. Keep the subscription identity stable across re-evaluations, or Iced tears it down and restarts it on every frame - which for a file watcher means re-registering the watch continuously.

## Output Format

When this skill produces a finding:

```
[Must|Recommend] {file:line | symbol, when source was supplied without paths}
Category: <blocking-executor | second-runtime | missing-cancellation | cancel-granularity | stale-job | progress-flooding | backpressure | rayon-bridge | subscription-identity>
Issue: <the defect, named>
Consequence: <what the user experiences - "Cancel does nothing until the current 4 GB file finishes", "results from a superseded scan replace the current ones">
Fix: <the concrete change>
```

`[Must]` for blocking-executor, second-runtime, stale-job, and missing-cancellation findings - freezes, wrong results, and a dead Cancel are user-visible breakage. `[Recommend]` otherwise.

When designing an async path rather than reviewing:

```
Iced version: <the resolved version from Cargo.lock | UNRESOLVED - read before proceeding>
Job: <what runs, and whether it is CPU-bound or I/O-bound>
Mechanism: <Task | Subscription | std::thread + channel | rayon + channel>
Progress: <what is emitted, and the coalescing interval | none - job is sub-frame>
Cancellation: <the flag, where it is checked, and the worst-case latency>
Stale results: <generation id and where it is compared | job cannot be restarted>
Backpressure: <bounded channel with capacity | producer-side aggregation>
Executor: <Iced's only - no second runtime>
```

Any signature stated for an Iced async API carries `verified against <version>` or `UNVERIFIED - confirm against the pinned version`. No Iced signature is asserted without one of the two.

When a symptom is reported without source ("the app freezes when I scan"), never emit `file:line` findings against code that was not shown. Name the candidate causes in likelihood order - blocking work in `update`, `block_on` on the UI path, CPU-bound work occupying the executor, one message per item flooding the queue - and request the `update` arm that starts the job, the worker code, and `Cargo.lock` before filing findings.

## Avoid

- A second async runtime (`tokio::Runtime`, `async_std`) alongside Iced's executor
- `block_on` anywhere in the UI crate
- CPU-bound hashing or decoding awaited on an executor thread
- A long job with no cancellation flag
- A cancellation flag checked only between items when a single item can run for seconds
- Assuming dropping a `Task` stops a running `std::thread` or `rayon` job
- Leaving `scanning: true` after a cancel
- One progress message per file over a large set
- One message per result item instead of batched chunks
- An unbounded channel from a fast producer to the UI
- Applying a job's result without checking its generation against the current one
- A `Subscription` whose identity changes each frame, restarting the underlying stream
- Using a `Subscription` for a job the user explicitly started
- Reproducing an Iced `Task` or `Subscription` signature from memory instead of the pinned version's docs
