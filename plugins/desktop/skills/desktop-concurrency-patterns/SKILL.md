---
name: desktop-concurrency-patterns
description: Run desktop work in parallel - Parallel.ForEach bounds, bounded Channels backpressure, cooperative cancellation, coalesced progress.
metadata:
  category: desktop
  tags: [csharp, dotnet, parallel-foreach, channels, backpressure, cancellation-token, iprogress, interlocked, determinism, dispatcher]
user-invocable: false
---

# Desktop Concurrency Patterns

> Confirm what the workload is bound by before parallelising - CPU-bound hashing and I/O-bound traversal on one spinning disk have opposite answers, and applying the CPU answer to the I/O case makes it slower.
>
> This skill owns **running work on more than one thread safely and usefully**. Whether the parallel version is actually faster belongs to `desktop-performance`; async/await shape and delivering results into ViewModels to `csharp-async-patterns`; what the workers touch on disk to `desktop-filesystem-patterns`; error type design to `csharp-error-handling`.

## When to Use

- Adding `Parallel.ForEach`, `Task.Run`, or a worker pipeline to a scan, hash, or decode path
- Reviewing a `Channel<T>`, a shared collection, or a counter touched by workers
- A cancel button that does not cancel, a UI that stutters during a scan, or memory that climbs during one
- Results that differ between runs on the same input

## Rules

- **Parallelism is applied to the CPU-bound stage, not the I/O-bound one.** Eight threads enumerating one spinning disk contend on a single arm and lose to one thread
- **Every producer-to-consumer channel is bounded.** An unbounded channel from a fast producer to a slow consumer is a memory defect that presents as an OOM on the largest input
- **Cancellation is cooperative and observed inside the work loop.** A token checked only between stages does not cancel a running stage
- **Progress is coalesced before it reaches the dispatcher.** One `IProgress<T>.Report` per item at 40k items/sec is a livelock of the UI event loop, not a progress bar
- Parallel output is sorted by a stable key before it is compared, displayed, or written. Completion order is not a specification
- Shared mutable state is partitioned per worker and merged once. A lock taken per item serializes the parallelism it sits inside
- **A worker exception fails one item, not the batch.** The rest continue and the item is recorded as failed

## Patterns

### Where parallelism helps, and where it hurts

| Stage | Bound by | Parallelise? |
| --- | --- | --- |
| Directory traversal, one local spinning disk | seek latency, one arm | No. One thread, streaming enumeration |
| Directory traversal, SSD or network share | per-call latency, deep queue | Yes, modestly - a small bounded pool |
| Metadata (`stat`) on a known path list | syscall latency | Yes, bounded to a small multiple of cores |
| Content hashing | CPU + sequential read | Yes, but bound concurrent readers so seeks do not thrash |
| Image decode | CPU | Yes. `Parallel.ForEach`, one item per iteration |
| Applying renames | filesystem serialization | No. Serial, ordered, journalled (`desktop-batch-operations`) |

The single most common wrong turn is parallelising the walk on a hard disk and reporting the result as an optimization. Measure both (`desktop-performance`); when the disk type is unknown, keep the walk serial and parallelise the hashing.

### Backpressure

```csharp
// Bad - the walker enumerates 400k paths in seconds; the hasher drains them in
// minutes. The queue holds every unprocessed path and memory climbs to OOM.
var ch = Channel.CreateUnbounded<string>();

// Good - the writer waits when the consumer is behind; memory is bounded by capacity
var ch = Channel.CreateBounded<string>(new BoundedChannelOptions(1024) {
    FullMode = BoundedChannelFullMode.Wait, SingleWriter = true,
});
foreach (var p in Walk(root)) await ch.Writer.WriteAsync(p, token);
ch.Writer.Complete();
```

`WriteAsync` throwing because the reader completed means the consumer is gone: the producer stops. That is a normal cancellation, not an error to rethrow as a failure.

Capacity is chosen so a stalled consumer costs bounded memory: 1024 path strings is kilobytes, 1024 decoded 4K bitmaps is gigabytes. Size the bound by the payload, and prefer sending a handle over sending pixels.

### Bounded parallelism

```csharp
// Bad - one task per file; 50k queued work items, and the pool thrashes before the disk does
foreach (var p in paths) _ = Task.Run(() => Hash(p));

// Good - the degree is explicit and threads are reused
Parallel.ForEach(paths,
    new ParallelOptions { MaxDegreeOfParallelism = 4, CancellationToken = token },
    p => Hash(p));
```

`Parallel.ForEachAsync` gives the same bound for async work. Keep blocking file I/O on this dedicated loop, not on async continuations the UI shares (`csharp-async-patterns`).

A body exception stops `Parallel.ForEach` from scheduling further items and surfaces an `AggregateException` - one unreadable file loses the batch. Isolate per item:

```csharp
p => { try { results.Enqueue(Hash(p)); }
       catch (Exception e) when (e is not OperationCanceledException)   // cancel is not an item failure
       { failures.Enqueue((p, e.Message)); } }
```

### Cancellation

```csharp
// Bad - checked only between stages; a scan already inside the hash phase runs to completion
if (!token.IsCancellationRequested) HashAll(paths);

// Good - observed inside the loop, at a granularity the user perceives as immediate
Parallel.ForEach(paths, new ParallelOptions { CancellationToken = token }, p => {
    token.ThrowIfCancellationRequested();
    Hash(p, token);   // a 2 GB file checks the token between read chunks too
});
```

`ParallelOptions.CancellationToken` stops scheduling new iterations; the per-item and per-chunk checks stop the running ones, so the cancel reaches the worker within a fraction of a second at human scale.

Cancellation is not failure. Catch `OperationCanceledException` at the orchestration boundary and return `Cancelled { Completed = n }` so the UI can say what was done, rather than an error the user reads as a crash. A cancelled destructive batch still leaves an accurate journal (`desktop-batch-operations`).

### Progress without flooding

```csharp
// Bad - one Report per file; each Report is a dispatcher post, and the UI
// event loop never drains 40k posts/sec
progress.Report(new(done, total));

// Good - workers race an atomic timestamp; at most ~20 reports/sec win
long done = 0, lastMs = 0; var sw = Stopwatch.StartNew();
Parallel.ForEach(paths, p => {
    Hash(p);
    var n = Interlocked.Increment(ref done);
    long now = sw.ElapsedMilliseconds, prev = Interlocked.Read(ref lastMs);
    if (now - prev >= 50 && Interlocked.CompareExchange(ref lastMs, now, prev) == prev)
        progress.Report(new(n, total));
});
progress.Report(new(Interlocked.Read(ref done), total));  // the final message is unconditional
```

Two properties matter more than the exact interval: progress is a **latest-value** signal, not a stream that must be delivered in full, so a throttled-away intermediate update is correct behaviour rather than a lost message; and the final completion message is sent unconditionally, since dropping *that* one leaves a progress bar stuck at 97%.

Coalescing on the producer side beats throttling in the UI - the messages are never constructed, so the allocations and the dispatcher posts both disappear. `Progress<T>` posts to the `SynchronizationContext` it was constructed on: construct it on the UI thread, or the reports land on the pool.

### Contention vs partitioned ownership

```csharp
// Bad - the dictionary is safe but List<T>.Add is not; workers race and corrupt it,
// and even a locked list would serialize every worker on one lock
var groups = new ConcurrentDictionary<long, List<string>>();
Parallel.ForEach(paths, p => groups.GetOrAdd(SizeOf(p), _ => []).Add(p));

// Good - each worker builds its own map; one merge per worker at the end
var groups = new Dictionary<long, List<string>>();
Parallel.ForEach(paths,
    () => new Dictionary<long, List<string>>(),
    (p, _, local) => {
        var size = SizeOf(p);
        if (!local.TryGetValue(size, out var list)) local[size] = list = [];
        list.Add(p);
        return local;
    },
    local => { lock (groups) Merge(groups, local); });
```

The rule generalizes: shared state touched once per item is contention (or, as above, a race); shared state touched once per worker is neither. Where a shared counter is genuinely needed, `Interlocked.Increment` costs far less than a lock. A lock or `SemaphoreSlim` held across a file read or an `await` is held for orders of magnitude longer than the critical section needs - copy out what is needed, release, then do the slow work.

### Determinism

```csharp
// Bad - group order and within-group order vary per run; the preview reshuffles,
// and a snapshot test fails intermittently
var groups = CollectParallel(paths);

// Good - one sort makes the output a function of the input
foreach (var g in groups) g.Files.Sort(StringComparer.Ordinal);
groups.Sort((a, b) => {
    var c = b.WastedBytes.CompareTo(a.WastedBytes);
    return c != 0 ? c : string.CompareOrdinal(a.Key, b.Key);
});
```

The sort key includes a tiebreaker, or equal primary keys still permute. This matters beyond tests: a dedup UI whose "keep the first" default depends on completion order deletes a different file each run.

### Reaching the UI thread

`Dispatcher.UIThread.Post` is fire-and-forget delivery - right for progress and completion notifications. `await Dispatcher.UIThread.InvokeAsync(..)` is for when the worker needs the result or the completion. Never block a worker on `.Result` or `.Wait()` of an `InvokeAsync` - a worker the UI thread is awaiting deadlocks the pair. And a `Post` per item is exactly the flood the coalescing pattern exists to prevent.

## Output Format

Two modes, chosen by whether something is being reviewed or authored.

**Authoring mode** - the request is to write concurrent code. Emit the code, then any `Deferred:` lines. No finding blocks and no status line.

**Review mode** - one block per finding, ordered by severity, Critical first:

```
### [Severity] {file:line | symbol, when source was supplied without paths | symptom, when no source was supplied}

- Category: {Backpressure | UnboundedSpawn | Cancellation | ProgressFlood | Contention | Determinism | WrongWorkload | ExceptionHandling}
- Evidence: {measured (name the tool, machine, and input scale - a deterministic failure readable from the supplied source counts, with the line as the repro) | estimated (source read, no measurement; state the item count and payload size assumed) | inferred (no source read; state what was not seen and the scale assumed)}
- Code: {one-line citation, or `not supplied` when the finding is inferred}
- Failure: {the concrete state that goes wrong - queue depth, thread count, stall duration, a differing run}
- Fix: {the concrete change}
- Verify: {what to re-measure or re-check - peak private bytes at the largest input, thread count, cancel-to-stop latency, two runs byte-identical}
```

`Severity: {Critical | High | Medium | Low}` - Critical = OOM, deadlock, a crash or corrupted collection that loses the batch, or a UI that never recovers. High = an unresponsive cancel, a visible stall, a normal shutdown path that surfaces an exception instead of stopping cleanly, or nondeterministic output on a destructive path. Medium = contention or overhead that costs throughput without a correctness risk. Low = a cheap win with no observed symptom.

`Category` takes exactly one value; where a defect fits two, pick the one `Fix` addresses and name the other in `Failure`. `estimated` and `inferred` bound the header at High, with `Failure` naming the uncapped band; neither ever raises a block. Findings are ordered by the capped header; within a band, the finding whose uncapped band is higher leads. A multi-symptom report takes one block per symptom.

A defect - or, in authoring mode, out-of-scope work the deliverable depends on - owned by a sibling named in the ownership blockquote is written after the findings or the authored code as `Deferred: {item} -> {owning skill}`, one per line. Omit when there are none.

In review mode, close with exactly one status line, after any `Deferred:` lines:

| Condition | Line |
| --- | --- |
| One or more findings emitted | none - the findings are the output |
| Source or a symptom report was available, nothing found | `No concurrency findings.` |
| No source, diff, or symptom supplied | `Concurrency check not run: no source supplied.` |

## Avoid

- Parallelising a directory walk on a single spinning disk and calling it an optimization
- `Channel.CreateUnbounded` between a fast producer and a slow consumer
- Sizing a channel bound by item count without regard for payload size
- `Task.Run` per work item with no bound on in-flight work
- A cancellation token checked only between stages
- Reporting a cancelled run as an error
- One progress `Report` per item
- Dropping the final completion message along with the throttled intermediate ones
- A shared collection mutated once per item inside a parallel loop
- A `ConcurrentDictionary` holding non-thread-safe values that workers mutate
- Holding a lock or semaphore across a file read or an `await`
- Returning parallel results without a sort, then comparing or snapshotting them
- A sort key with no tiebreaker
- A body exception that takes down the batch instead of failing one item
- Blocking a worker on `Dispatcher.UIThread.InvokeAsync(..).Result` or `.Wait()`
