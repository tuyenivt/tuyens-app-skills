---
name: csharp-async-patterns
description: C# async - Task vs ValueTask, CancellationToken, IProgress coalescing, bounded Channels, Dispatcher.UIThread marshalling, no async void.
metadata:
  category: desktop
  tags: [csharp, async, task, cancellation, iprogress, channels, backpressure, dispatcher, configureawait, valuetask]
user-invocable: false
---

# C# Async Patterns

> This skill owns **work that outlives one UI interaction and how its results get back**: async flow, cancellation, progress, and thread marshalling. Command wiring and ViewModel shape belong to `avalonia-mvvm-patterns`; rendering progress and cancel affordances to `avalonia-control-patterns`; `Parallel.ForEach` partitioning and degree-of-parallelism tuning to `desktop-concurrency-patterns`; per-item failure types and cancellation-as-control-flow semantics to `csharp-error-handling`.

## When to Use

- A scan, hash, or decode runs longer than a frame
- Progress must appear while a job is still running
- Adding cancellation, or a rescan that supersedes a running one
- A deadlock, a vanished exception, or UI state touched from a worker thread

## Rules

- **`async void` is only for event handlers.** Anywhere else, exceptions bypass every catch and crash the process, and completion cannot be awaited or tested. Commands are `async Task` methods - `[RelayCommand]` wraps them
- **Never `.Result`, `.Wait()`, or `.GetAwaiter().GetResult()` on the UI thread.** The continuation needs the thread the caller is blocking - the classic deadlock. Async flows all the way up; this app has no tolerated sync-over-async site
- The core library puts `ConfigureAwait(false)` on every await; the UI layer omits it, because ViewModel code after `await` must resume on the UI thread
- **`CancellationToken` threads through every long-running core API** as the last parameter, checked inside the work loop and passed to every call that accepts one. Granularity is set by the largest single item: hashing a 4 GB file checks between buffer reads, not between files
- `Task.Run` wraps CPU-bound sync work at the UI-layer call site. Never inside the core library - the caller owns the thread decision - and never around naturally-async I/O
- **Progress is coalesced before it reaches the dispatcher.** 100k per-item reports starve rendering; report a count per chunk (256 items) or per interval (100 ms) through `IProgress<T>`
- `Progress<T>` is constructed on the UI thread so its callback marshals automatically. Worker code that must touch bound state directly uses `Dispatcher.UIThread.Post` or `InvokeAsync`
- Producer/consumer pipelines use `Channel.CreateBounded`. An unbounded channel from a fast scanner to a slower consumer is a memory climb; a producer suspended on a full channel is the backpressure working
- `ValueTask` is for hot APIs that usually complete synchronously (a cache hit). It is awaited exactly once, never stored; everywhere else, `Task`

## Patterns

### async Task, not async void

```csharp
// Bad - an exception here crashes the process; nothing can await or test this
public async void StartScan() => Results = await _scanner.ScanAsync(Root);

// Good - awaitable; the exception flows to the awaiter
public async Task StartScanAsync(CancellationToken ct)
    => Results = await _scanner.ScanAsync(Root, ct);
```

An event handler is the one legitimate `async void` - the delegate demands it. Keep its body a single awaited call into an `async Task` method, so everything testable lives behind the signature that can fail loudly. The same trap in disguise: `_ = SomethingAsync();` fire-and-forget, whose exceptions vanish unobserved.

### Deadlock: sync-over-async

```csharp
// Bad - the UI thread blocks; the await's continuation needs the UI thread; deadlock
var report = _scanner.ScanAsync(root, ct).Result;

// Good - the thread is released while the scan runs
var report = await _scanner.ScanAsync(root, ct);
```

`ConfigureAwait(false)` in the library shrinks the deadlock window but is not the fix - blocking is the defect. `.Result` also wraps failure in `AggregateException` where `await` throws the original (`csharp-error-handling` owns that surface).

### ConfigureAwait: split by project

```csharp
// Core library - resumes on any pool thread; never assumes a UI context
var n = await stream.ReadAsync(buffer, ct).ConfigureAwait(false);

// ViewModel - no ConfigureAwait: the next line sets bound properties
Results = await Task.Run(() => _scanner.Scan(root, progress, ct), ct);
StatusText = $"Done: {Results.Groups.Count} groups";
```

The split mirrors the UI-free core rule: core code must never need a UI context, so it says so on every await; ViewModel code depends on returning to it, so it stays silent.

### Cancellation that reaches the work

```csharp
// Bad - HashAsync accepts a ct but never consults it; Cancel waits for the whole file
while ((n = await s.ReadAsync(buf)) > 0) _hasher.Append(buf.AsSpan(0, n));

// Good - the token reaches the read; a 4 GB file cancels mid-read
while ((n = await s.ReadAsync(buf, ct).ConfigureAwait(false)) > 0)
    _hasher.Append(buf.AsSpan(0, n));
```

In the ViewModel, `[RelayCommand(IncludeCancelCommand = true)]` supplies the token and the cancel command; a hand-rolled path owns a `CancellationTokenSource`, disposes it, and replaces it per run. What the user is told after a cancel is `csharp-error-handling`'s contract.

### Superseded jobs: cancel and drain

```csharp
// Bad - the old scan finishes late and overwrites the new scan's results
_cts.Cancel();
_cts = new CancellationTokenSource();
_running = ScanAsync(_cts.Token);

// Good - the old run is cancelled and awaited before the new one starts
_cts.Cancel();
try { await _running; } catch (OperationCanceledException) { }
_cts = new CancellationTokenSource();
_running = ScanAsync(_cts.Token);
```

Draining the old task before starting the new one means stale results cannot arrive out of order - no generation counters to maintain. The await also surfaces any real failure the old run died with instead of losing it.

### Progress without flooding the dispatcher

```csharp
// Bad - one Report per file; 100k dispatcher posts starve the rendering they serve
for (var i = 0; i < paths.Count; i++) { Hash(paths[i]); progress.Report(new(i + 1, total)); }

// Good - coalesced: one report per chunk, plus the final one
if ((i + 1) % 256 == 0 || i + 1 == total) progress.Report(new ScanProgress(i + 1, total));
```

Construct the `Progress<T>` in the ViewModel - it captures the UI context at construction, so worker-side `Report` calls marshal automatically. A time-based interval (a `Stopwatch` and 100 ms) is the better coalescer when item costs vary wildly. Results follow the same rule as counts: accumulate a chunk, publish the chunk, extend the bound state once per chunk - never once per item.

### Bounded channels for pipelines

```csharp
// Bad - unbounded; enumeration outruns hashing and the queue eats the tree
var ch = Channel.CreateUnbounded<string>();

// Good - bounded; a full channel suspends the producer, which is the point
var ch = Channel.CreateBounded<string>(new BoundedChannelOptions(1024)
{ SingleWriter = true, FullMode = BoundedChannelFullMode.Wait });
// producer: await ch.Writer.WriteAsync(path, ct);
// consumer: await foreach (var path in ch.Reader.ReadAllAsync(ct)) ...
```

Complete the writer in a `finally` - `ch.Writer.Complete()`, or `Complete(ex)` to propagate failure - or consumers await forever. A channel earns its place when stages run at different speeds (enumerate -> hash -> report); a parallel map over a known list is `Parallel.ForEachAsync` with less machinery (`desktop-concurrency-patterns` owns its tuning).

### Task.Run: where and where not

```csharp
// Bad - the core library decides threading for its caller
public Task<BatchReport> ScanAsync(string root) => Task.Run(() => Scan(root));

// Bad - burns a pool thread to wait on I/O that was already async
var text = await Task.Run(() => File.ReadAllTextAsync(path));

// Good - the UI layer wraps the honest sync signature at the call site
var report = await Task.Run(() => _core.Scan(root, progress, ct), ct);
```

Core exposes sync signatures for CPU-bound work and true-async signatures for I/O-bound work, and the ViewModel chooses. Fake-async wrappers in core hide the cost model from the one place that must know it.

### Marshalling to the UI thread

```csharp
// Bad - a channel consumer on a pool thread sets a bound property
await foreach (var batch in ch.Reader.ReadAllAsync(ct))
    vm.Groups = Merge(vm.Groups, batch);   // wrong thread: bindings misbehave or throw

// Good - the mutation is posted to the dispatcher, one post per batch
await foreach (var batch in ch.Reader.ReadAllAsync(ct))
    await Dispatcher.UIThread.InvokeAsync(() => vm.Apply(batch));
```

Prefer designs where no manual marshalling exists at all: results return as the awaited value of an async command (already back on the UI thread), progress arrives through `Progress<T>`. Manual `Dispatcher.UIThread` is the fallback for worker callbacks - `Post` for fire-and-forget updates, `InvokeAsync` when completion must be awaited, and never a synchronous `Invoke` from a worker while the UI thread might be waiting on that worker.

### ValueTask discipline

```csharp
// Bad - stored and awaited twice; a ValueTask is invalid after its first await
var vt = _cache.GetThumbnailAsync(key);
var a = await vt; var b = await vt;

// Good - awaited once, immediately; stored futures use .AsTask()
var thumb = await _cache.GetThumbnailAsync(key);
```

The thumbnail cache is the fitting case: hits complete synchronously with no `Task` allocation, misses decode. Everywhere the completion is stored, combined, or awaited by more than one consumer, `Task` is the correct type.

## Output Format

When this skill produces a finding, emit one block per finding, `[Must]` first:

```
[Must|Recommend] {file:line | symbol, when source was supplied without paths}
Category: <async-void | sync-over-async | configureawait | cancellation-gap | cancel-granularity | stale-results | progress-flooding | unbounded-channel | task-run-misuse | dispatcher-marshalling | valuetask-misuse>
Issue: <the defect, named>
Consequence: <what the user experiences - "Cancel does nothing until the current 4 GB file finishes", "the queue grows to the size of the tree">
Fix: <the concrete change>
```

`[Must]` for async-void, sync-over-async, cancellation-gap, stale-results, and dispatcher-marshalling findings - crashes, deadlocks, dead Cancel buttons, and wrong results are user-visible breakage. `[Recommend]` otherwise. `Category` takes exactly one value - where a defect fits two, pick the one the `Fix` addresses and name the other in `Consequence`. `_ = SomethingAsync()` fire-and-forget files as `async-void`; a never-completed channel writer files as `unbounded-channel`. Repeated instances of the same defect in one file merge into one block with every location named.

A defect - or, in design mode, out-of-scope work the deliverable depends on - owned by a sibling named in the ownership blockquote is written after the findings or the design form as `Deferred: {item} -> {owning skill}`, one per line. Omit when there are none.

When designing an async path rather than reviewing:

```
Job: <what runs, and whether it is CPU-bound or I/O-bound>
Mechanism: <async command + Task.Run | true-async I/O | bounded Channel pipeline | a composite, primary first>
Cancellation: <the token's source, where it is checked, and the worst-case latency>
Progress: <what is reported and the coalescing interval - indeterminate until the total is known | none - job is sub-frame>
Backpressure: <bounded capacity and FullMode | producer-side batching | N/A - single stage>
UI updates: <awaited return value | Progress<T> | Dispatcher.UIThread, and why manual>
Supersession: <cancel-and-drain before restart | job cannot be restarted>
```

When a symptom is reported without source ("the app freezes when I scan"), emit no finding blocks. The deliverable is two labelled sections: `Diagnosis:` - candidate causes in likelihood order, re-ranked by the reported evidence (for a freeze: sync-over-async on the UI thread, CPU work before the first await in a command, per-item dispatcher posts, an unbounded queue); `Request:` - the code needed to file findings, typically the command body and the worker code. Severity labels and the closing line belong to filed findings only.

A review that produces no finding closes with exactly `No async findings.` - an `async void` event handler with a one-line awaited body is not a defect.

## Avoid

- `async void` outside an event handler; handler bodies beyond one awaited call
- `.Result`, `.Wait()`, or `.GetAwaiter().GetResult()` on the UI thread
- Missing `ConfigureAwait(false)` in core; `ConfigureAwait(false)` in ViewModel code that touches bound state after the await
- Accepting a `CancellationToken` and never consulting it
- Cancellation checked only between items when one item can run for seconds
- Restarting a job without cancelling and draining the previous run
- One progress report or one dispatcher post per item over a large set
- `Channel.CreateUnbounded` fed by a producer faster than its consumer
- A channel writer never completed, leaving consumers awaiting forever
- `Task.Run` around async I/O, or inside core library methods
- Setting bound properties from a pool thread
- Storing a `ValueTask` or awaiting it twice
- `_ = SomethingAsync()` fire-and-forget with no exception observer
