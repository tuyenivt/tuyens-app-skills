---
name: desktop-concurrency-patterns
description: Rust desktop parallelism - rayon scans, bounded pools, channel backpressure, cooperative cancellation, throttled progress, contention, determinism.
metadata:
  category: desktop
  tags: [rust, rayon, concurrency, backpressure, bounded-channel, cancellation, progress, mutex-contention, determinism, worker-pool]
user-invocable: false
---

# Desktop Concurrency Patterns

> Confirm what the workload is bound by before parallelising - CPU-bound hashing and I/O-bound traversal on one spinning disk have opposite answers, and applying the CPU answer to the I/O case makes it slower.
>
> This skill owns **running work on more than one thread safely and usefully**. Whether the parallel version is actually faster belongs to `desktop-performance`; the Iced side of delivering results to the view to `iced-async-patterns`; what the workers touch on disk to `desktop-filesystem-patterns`; error type design to `rust-error-handling`.

## When to Use

- Adding `rayon`, threads, or a worker pool to a scan, hash, or decode path
- Reviewing a channel, `Arc<Mutex<_>>`, or shared counter
- A cancel button that does not cancel, a UI that stutters during a scan, or memory that climbs during one
- Results that differ between runs on the same input

## Rules

- **Parallelism is applied to the CPU-bound stage, not the I/O-bound one.** Eight threads calling `read_dir` on one spinning disk contend on a single arm and lose to one thread
- **Every producer-to-consumer channel is bounded.** An unbounded channel from a fast producer to a slow consumer is a memory defect that presents as an OOM on the largest input
- **Cancellation is cooperative and checked inside the loop.** A flag that is only read between stages does not cancel a running stage
- **Progress is coalesced before it reaches the UI.** One message per item at 40k items/sec is a livelock of the event loop, not a progress bar
- Parallel output is sorted by a stable key before it is compared, displayed, or written. Completion order is not a specification
- Shared mutable state is partitioned per worker and merged once. A `Mutex` taken per item serializes the parallelism it sits inside
- **A panicking worker does not lose the batch.** Its item is recorded as failed and the rest continue

## Patterns

### Where parallelism helps, and where it hurts

| Stage | Bound by | Parallelise? |
| --- | --- | --- |
| Directory traversal, one local spinning disk | seek latency, one arm | No. One thread; `walkdir` |
| Directory traversal, SSD or network share | per-call latency, deep queue | Yes, modestly. `jwalk` or a small pool |
| `stat`/metadata on a known path list | syscall latency | Yes, bounded to a small multiple of cores |
| Content hashing | CPU + sequential read | Yes, but bound concurrent readers so seeks do not thrash |
| Image decode | CPU | Yes. `rayon`, one item per task |
| Applying renames | filesystem serialization | No. Serial, ordered, journalled (`desktop-batch-operations`) |

The single most common wrong turn is parallelising the walk on a hard disk and reporting the result as an optimization. Measure both (`desktop-performance`); when the disk type is unknown, keep the walk serial and parallelise the hashing.

### Backpressure

```rust
// Bad - the walker enumerates 400k paths in seconds; the hasher consumes them
// in minutes. The queue holds every unprocessed PathBuf. Memory climbs to OOM.
let (tx, rx) = std::sync::mpsc::channel();
std::thread::spawn(move || { for p in walk(root) { tx.send(p).unwrap(); } });

// Good - the walker blocks when the consumer is behind; memory is bounded by capacity
let (tx, rx) = crossbeam_channel::bounded(1024);
std::thread::spawn(move || { for p in walk(root) { if tx.send(p).is_err() { break; } } });
```

`send` returning `Err` means the receiver dropped: the consumer is gone, so the producer stops. Treating it as fatal with `unwrap` turns a normal cancellation into a panic.

Capacity is chosen so a stalled consumer costs bounded memory: 1024 `PathBuf`s is kilobytes, 1024 decoded 4K images is gigabytes. Size the bound by the payload, and prefer sending a handle over sending pixels.

### Bounded worker pools

```rust
// Bad - one thread per file; 50k threads, and the scheduler dies before the disk does
for path in paths { std::thread::spawn(move || hash(path)); }

// Good - rayon's pool is sized to the machine and reuses threads
let results: Vec<_> = paths.par_iter().map(|p| hash(p)).collect();
```

Where the concurrency limit must differ from the core count - concurrent open file handles, or network share latency - build a scoped pool rather than fighting the global one:

```rust
let pool = rayon::ThreadPoolBuilder::new().num_threads(4).build()?;
pool.install(|| paths.par_iter().map(|p| hash(p)).collect::<Vec<_>>());
```

Do not perform blocking file I/O on an async runtime's core threads if the app has one; that is what `spawn_blocking` exists for.

### Cancellation

```rust
// Bad - checked only between stages; a scan already inside the hash phase runs to completion
if !cancel.load(Ordering::Relaxed) { hash_all(&paths); }

// Good - checked inside the loop, at a granularity the user perceives as immediate
let out: Vec<_> = paths.par_iter()
    .map(|p| if cancel.load(Ordering::Relaxed) { None } else { Some(hash(p)) })
    .while_some()
    .collect();
```

A cancel must reach the worker within a fraction of a second at human scale, so the check goes on the per-item path, not the per-batch one. For a single item that is itself long (a 2 GB file hash), check between read chunks.

Cancellation is not failure. It returns `Outcome::Cancelled { completed: n }` so the UI can say what was done, rather than an error the user reads as a crash. A cancelled destructive batch still leaves an accurate journal (`desktop-batch-operations`).

### Progress without flooding

```rust
// Bad - one message per file; the UI event loop never drains the queue
for p in &paths { hash(p); tx.send(Progress::Item(p.clone()))?; }

// Good - coalesce on a time interval; the UI redraws at most ~20x/sec anyway
let mut last = Instant::now();
let done = AtomicUsize::new(0);
paths.par_iter().for_each(|p| {
    hash(p);
    let n = done.fetch_add(1, Ordering::Relaxed) + 1;
    if last.elapsed() >= Duration::from_millis(50) { let _ = tx.try_send(Progress::Count(n)); }
});
```

Two properties matter more than the exact interval: progress is a **latest-value** signal, not a stream that must be delivered in full, so `try_send` dropping an update on a full channel is correct behaviour rather than a lost message; and the final completion message is sent unconditionally, since dropping *that* one leaves a progress bar stuck at 97%.

Coalescing on the producer side beats throttling in the UI - the messages are never constructed, so the allocation and the channel traffic both disappear.

### Contention vs partitioned ownership

```rust
// Bad - the mutex is taken once per file; every worker serializes on it
let groups = Arc::new(Mutex::new(HashMap::<u64, Vec<PathBuf>>::new()));
paths.par_iter().for_each(|p| {
    groups.lock().unwrap().entry(size_of(p)).or_default().push(p.clone());
});

// Good - each worker builds its own map; one merge at the end
let groups = paths.par_iter()
    .fold(HashMap::<u64, Vec<PathBuf>>::new,
          |mut m, p| { m.entry(size_of(p)).or_default().push(p.clone()); m })
    .reduce(HashMap::new, merge_maps);
```

The rule generalizes: shared state touched once per item is contention; shared state touched once per worker is not. Where a shared counter is genuinely needed, an `AtomicUsize` with `Ordering::Relaxed` costs far less than a `Mutex`.

A `Mutex` held across an `.await` or across a blocking file read holds it for orders of magnitude longer than the critical section needs. Copy out what is needed, drop the guard, then do the slow work.

### Determinism

```rust
// Bad - group order and within-group order vary per run; the preview reshuffles,
// and a snapshot test fails intermittently
let groups: Vec<Group> = collect_parallel(&paths);

// Good - one sort makes the output a function of the input
let mut groups: Vec<Group> = collect_parallel(&paths);
for g in &mut groups { g.files.sort(); }
groups.sort_by(|a, b| b.wasted_bytes.cmp(&a.wasted_bytes).then(a.key.cmp(&b.key)));
```

The sort key includes a tiebreaker, or equal primary keys still permute. This matters beyond tests: a dedup UI whose "keep the first" default depends on completion order deletes a different file each run.

## Output Format

Two modes, chosen by whether something is being reviewed or authored.

**Authoring mode** - the request is to write concurrent code. Emit the code, then any `Deferred:` lines. No finding blocks and no status line.

**Review mode** - one block per finding:

```
### [Severity] {file:line | symbol, when source was supplied without paths | symptom, when no source was supplied}

- Category: {Backpressure | UnboundedSpawn | Cancellation | ProgressFlood | Contention | Determinism | WrongWorkload | PanicHandling}
- Evidence: {measured (name the tool, machine, and input scale) | estimated (stated item count and payload size) | inferred (no source read)}
- Code: {one-line citation, or `not supplied` when the finding is inferred}
- Failure: {the concrete state that goes wrong - queue depth, thread count, stall duration, a differing run}
- Fix: {the concrete change}
- Verify: {what to re-measure or re-check - peak RSS at the largest input, thread count, cancel-to-stop latency, two runs byte-identical}
```

`Severity: {Critical | High | Medium | Low}` - Critical = OOM, deadlock, or a UI that never recovers. High = an unresponsive cancel, a visible stall, or nondeterministic output on a destructive path. Medium = contention or overhead that costs throughput without a correctness risk. Low = a cheap win with no observed symptom.

`Category` takes exactly one value; where a defect fits two, pick the one `Fix` addresses and name the other in `Failure`. `estimated` and `inferred` bound the header at High, with `Failure` naming the uncapped band; neither ever raises a block.

A defect owned by a sibling named in the ownership blockquote is written after the findings as `Deferred: {defect} -> {owning skill}`, one per line. Omit when there are none.

In review mode, close with exactly one status line, after any `Deferred:` lines:

| Condition | Line |
| --- | --- |
| One or more findings emitted | none - the findings are the output |
| Source or a symptom report was available, nothing found | `No concurrency findings.` |
| No source, diff, or symptom supplied | `Concurrency check not run: no source supplied.` |

## Avoid

- Parallelising a directory walk on a single spinning disk and calling it an optimization
- `std::sync::mpsc::channel()` or any unbounded queue between a fast producer and a slow consumer
- Sizing a channel bound by item count without regard for payload size
- `tx.send(..).unwrap()` where a dropped receiver is a normal cancellation
- `thread::spawn` per work item
- Blocking file I/O on an async runtime's core threads
- A cancellation flag checked only between stages
- Reporting a cancelled run as an error
- One progress message per item
- Dropping the final completion message along with the throttled intermediate ones
- `Arc<Mutex<_>>` locked once per item inside a parallel loop
- Holding a lock across a file read or an `.await`
- Returning parallel results without a sort, then comparing or snapshotting them
- A sort key with no tiebreaker
- A worker panic that takes down the batch instead of failing one item
