---
name: desktop-performance
description: Tune C# desktop throughput - two-tier size-then-hash dedup, XxHash3, I/O ordering, GC and allocation, SIMD, startup latency, evidence discipline.
metadata:
  category: desktop
  tags: [csharp, dotnet, performance, throughput, xxhash3, two-tier, gc, allocation, simd, startup-latency, benchmarkdotnet]
user-invocable: false
---

# Desktop Performance

> Confirm the target machine and the input scale before reporting any number - "fast" on an NVMe developer laptop over 500 files says nothing about a 200k-file external drive, which is the case users complain about.
>
> This skill owns **cost**: what a scan, a hash, and a cold start cost the user. Parallel execution mechanics belong to `desktop-concurrency-patterns`; decode sizing to `desktop-image-processing`; traversal correctness to `desktop-filesystem-patterns`; cache schema and invalidation to `desktop-data-persistence` (a missing or ineffective cache is still a `Caching` finding here); per-item failure typing to `csharp-error-handling`; whether the optimization is warranted at all to `desktop-overengineering-review`.

## When to Use

- A scan, dedup, or batch that is slower than the user tolerates
- Reviewing a hot path: hashing, traversal, per-item allocation, comparison
- Slow app startup or a delayed first window
- Deciding whether an optimization is worth its complexity

## Rules

- **Attribute the cost to disk I/O, CPU, GC, or allocation before changing anything.** They have different fixes and only one of them is the bottleneck at a time
- **The algorithmic win comes first.** Two-tier grouping turns O(n) full reads into O(candidates) full reads; no micro-optimization inside a full read approaches that factor
- **Content identity uses XxHash3 from `System.IO.Hashing`.** It is SIMD-accelerated and faster than any disk can feed it - the disk is the bottleneck, not the hash. SHA-256 is for signing, which this app does not do
- **Never quote a number from a Debug build.** Debug code is unoptimized by design; a Debug timing is never evidence for a Release claim and cannot even rank two approaches
- **Startup cost is the most visible latency in the app.** Work before the first paint is the only latency every user experiences on every run
- A hot loop does not allocate per item. Rent from `ArrayPool<T>`, slice with `Span<T>`, reuse the buffer
- Every reported number carries an evidence level and, for `measured`, the tool and the machine

## Patterns

### Attributing the cost

| Symptom | Likely owner | Typical cause |
| --- | --- | --- |
| CPU near idle, scan slow, disk busy | Disk I/O | reading every file in full; random access order |
| One core pinned, disk idle | CPU | hashing everything, or decode work on the scan path |
| All cores busy, throughput below single-threaded | Contention | shared lock per item (`desktop-concurrency-patterns`) |
| High "% Time in GC", pauses during a scan | GC pressure | per-item allocations; large short-lived buffers on the LOH |
| Memory climbs with input size, never falls | Allocation | unbounded queue or cache; 500k records held to the end |
| Slow only on the first run over a folder | Cold cache | expected; verify the second run is fast (`desktop-data-persistence`) |
| Window appears seconds after launch | Startup | work before first paint |
| Fine on SSD, unusable on a spinning or network drive | I/O ordering | seek-heavy access pattern, or a parallelised walk |

Attribute with a profiler, not by reading code: `dotnet-counters` for GC and allocation rate, `dotnet-trace` or PerfView for CPU, the OS activity monitor for the disk-vs-CPU split, BenchmarkDotNet for a specific method (it forces Release, warms the JIT, and `[MemoryDiagnoser]` reports allocation per op), a Release-build stopwatch or `hyperfine` for end-to-end wall time.

### Two-tier hashing: the win that dominates

Comparing n files pairwise by content is the naive shape and it reads every byte of every file. The funnel does not:

| Tier | Reads | Typical survivors from 100k files |
| --- | --- | --- |
| 1. Group by file size | metadata only | ~5% - distinct sizes cannot be byte-identical |
| 2. Partial hash: first + last 64 KB | 128 KB per file, two seeks | ~1% - kills same-size-different-content sets |
| 3. Full hash, only within tier-2 collision groups | full read | the actual duplicate sets |

```csharp
// Bad - hashes 400 GB to find 2 GB of duplicates
var dupes = paths.GroupBy(FullHash).Where(g => g.Count() > 1);

// Good - each tier only sees what the previous one could not separate
var bySize    = paths.GroupBy(p => new FileInfo(p).Length).Where(g => g.Count() > 1);
var byPartial = bySize.SelectMany(g => g.GroupBy(PartialHash)).Where(g => g.Count() > 1);
var dupes     = byPartial.SelectMany(g => g.GroupBy(FullHash)).Where(g => g.Count() > 1);
```

Two details decide whether the win is real. A group of size 1 is discarded at every tier, never carried forward. And the partial hash must include the **tail** as well as the head: files sharing a size and a container header (media, archives, office documents) are exactly the case a head-only prefix fails to separate, which is the case this tier exists for.

Skip tier 2 for files below roughly its own read size - reading 128 KB of a 40 KB file to avoid reading 40 KB is a loss. Record which hash produced a cached digest, so a later change of algorithm invalidates rather than mismatches (`desktop-data-persistence`).

### Choosing and streaming the hash

| Hash | Throughput class | Use for |
| --- | --- | --- |
| XxHash3 (`System.IO.Hashing`) | SIMD-accelerated; saturates any disk | Default: content identity, partial tiers, cache keys |
| SHA-256 | An order of magnitude slower | Signing and integrity attestation, which this app does not do |
| MD5 | Slow *and* broken | Nothing |

```csharp
// Bad - the whole file becomes one allocation; a 4 GB video is a 4 GB byte[]
var digest = XxHash3.Hash(File.ReadAllBytes(path));

// Good - streamed through a rented buffer; memory is constant in file size
var hasher = new XxHash3();
var buf = ArrayPool<byte>.Shared.Rent(256 * 1024);
try {
    using var f = File.OpenRead(path);
    int n;
    while ((n = f.Read(buf, 0, buf.Length)) > 0) hasher.Append(buf.AsSpan(0, n));
    return hasher.GetCurrentHash();
} finally { ArrayPool<byte>.Shared.Return(buf); }
```

The rented buffer matters twice over: a fresh 256 KB array per file sits over the 85,000-byte Large Object Heap threshold, so the naive loop creates one LOH allocation per file and leaves Gen2 to clean up all of them.

### I/O ordering

On a spinning disk, sequential access is roughly two orders of magnitude faster than random. Even on SSD, ordering reduces queue thrash.

- **Batch metadata before content.** One pass collecting sizes, then a second pass reading only the candidates - not interleaved stat-and-read per file
- **Order reads by directory**, so files stored near each other are read near each other
- **Do not parallelise reads on a single spinning disk.** Concurrent readers turn a sequential pattern into a seek storm (`desktop-concurrency-patterns`)
- **Never read a file twice** for two different purposes. If size, hash, and thumbnail are all needed, take them from one open
- `MemoryMappedFile` is not a free speedup: I/O errors surface as exceptions at arbitrary read sites, and on Windows a mapped file blocks operations on it. Measure before adopting

### GC pressure and allocation

```csharp
// Bad - 500k record instances alive for the whole scan; they promote to Gen2
// and every Gen2 collection walks them
var records = new List<FileRecord>();      // class with string fields
foreach (var p in paths) records.Add(Scan(p));

// Good - results stream to the consumer; only collision groups are retained
foreach (var p in paths) channel.Writer.TryWrite(Scan(p));
```

Where to look, in payoff order: per-item `string` and array allocations in the scan loop (`Span<T>` slicing instead of `Substring` and `ToArray`), buffers at or over 85,000 bytes allocated instead of rented, whole result sets held alive when the consumer streams, and a `List<T>` grown without a capacity when the count is known. Small hot-path values become `readonly record struct` to stay off the heap - measure before converting; large structs pay in copies.

Workstation GC is the desktop default and the right one until measured otherwise; `<ServerGarbageCollection>` raises batch throughput at a memory footprint users notice in Task Manager - a measured decision, not a default.

### SIMD: use the BCL's, prove your own

The BCL's vectorized paths come first: XxHash3 itself, `SequenceEqual` and `IndexOf` over spans, and string search are already SIMD. Hand vectorization is for an inner loop that survives profiling after the algorithmic fix:

```csharp
// Portable form; compiles to AVX2 or NEON where available, scalar elsewhere.
// Caller guarantees a.Length == b.Length.
static bool BlocksEqual(ReadOnlySpan<byte> a, ReadOnlySpan<byte> b) {
    int i = 0;
    for (; i + Vector<byte>.Count <= a.Length; i += Vector<byte>.Count)
        if (!Vector.EqualsAll(new Vector<byte>(a[i..]), new Vector<byte>(b[i..]))) return false;
    for (; i < a.Length; i++) if (a[i] != b[i]) return false;
    return true;
}
```

`Vector<T>` is the portable tier; `System.Runtime.Intrinsics` (`Vector128`/`Vector256`, `Avx2`) is the platform tier below it, guarded by `IsSupported` with a scalar fallback. Either requires a BenchmarkDotNet result beating the BCL call it replaces - the usual finding is that it does not, because the disk is the bottleneck.

### Startup cost

Time to first paint is the latency the user notices most, because it precedes anything they asked for.

- **NativeAOT reaches the first window in roughly 460 ms where the same app under the JIT takes roughly 1960 ms.** Publish mechanics belong to `desktop-build-release`; the gap is a performance fact, and the app stays NativeAOT-compatible by design
- **Nothing scans, migrates, or indexes before the window is shown.** Paint, then start the work with a visible indicator
- Restore the last window geometry synchronously (one small file read) and defer everything else
- Open the SQLite connection on first use, not in `Program.Main`, and run schema migration behind visible state rather than a blank window
- Measure from process start to window-visible on a cold OS file cache, not a warm second run
- First-run wall time under the JIT includes warmup - that is the number the user gets; report the NativeAOT number separately when both are shipped

## Output Format

Two modes, chosen by whether something is being reviewed or authored.

**Authoring mode** - the request is to write or design something. Emit the code or design, then any `Deferred:` lines. No finding blocks, no severity, no status line: nothing was measured, so a not-run line would misdescribe the work.

**Review mode** - one block per finding:

```
### [Severity] {file:line | file - symbol, when the line is unknown | symbol, when source was supplied without paths | symptom, when no source was supplied}

- Category: {Algorithmic | HashChoice | IOOrdering | Allocation | GCPressure | Vectorization | Startup | Caching | Memory}
- Evidence: {measured (tool, machine, input scale) | estimated (source read, no measurement; state the input scale assumed) | inferred (no source read; state what was not seen)}
- Code: {one-line citation, or `not supplied` when the finding is inferred}
- Cost: {with units - "400 GB read to find 2 GB of duplicates", "1.8 s before first paint", "one 256 KB LOH allocation x 200k files"}
- Fix: {the concrete change}
- Verify: {what to re-measure - wall time on the same tree, bytes read, `dotnet-counters` % Time in GC, peak working set, time to first paint on a cold cache}
```

`Severity: {Critical | High | Medium | Low}` - Critical = unusable at a realistic input scale, or unbounded memory growth. High = a measurable regression on a primary path, or an algorithmic shape one tier away from a large win. Medium = cost on a rare path or only at unlikely input sizes. Low = a cheap win with no observed symptom.

**Evidence gating.** `measured` requires a Release-build number, and the block names the tool and the machine. `estimated` covers a finding read from source without a measurement, and names the input scale it assumes ("100k files, 400 GB"). `inferred` covers a finding from a bug report or a stated fact with no source read, and states what was not seen. A Debug-build timing is never `measured`.

`inferred` can never alone justify a Critical here, nor a `[Must]` in a consuming review workflow: both `estimated` and `inferred` bound the header at High, with `Cost` naming the uncapped band, and neither ever raises a block. Never report a number that was not measured as if it were.

`Category` takes exactly one value; where a defect fits two, pick the one `Fix` addresses and name the other in `Cost`. Among blocks sharing a band, order by what must be fixed first: the algorithmic shape before the micro-optimizations inside it.

A defect owned by a sibling named in the ownership blockquote is written after the findings as `Deferred: {defect} -> {owning skill}`, one per line. Omit when there are none.

In review mode, close with exactly one status line, after any `Deferred:` lines:

| Condition | Line |
| --- | --- |
| One or more findings emitted | none - the findings are the output |
| Source or a symptom report was available, nothing found | `No performance findings.` |
| No source, diff, or symptom of any kind supplied | `Performance check not run: no source supplied.` |

A symptom-only report is checkable input: emit `inferred` findings from it rather than the not-run line.

When invoked from an implementation workflow rather than a review, emit a budget table instead of finding blocks (`Deferred:` lines still follow it; no status line - budgets are authoring-shaped). Budget numbers are design targets, `estimated` until measured; say so once above the table:

```
| Surface | Budget | Risk | Mitigation |
|---------|--------|------|------------|
| Scan 100k files, external HDD | < 60 s to first result | full read per file | size group, then partial hash, then full |
| Thumbnail grid scroll | 60 fps, no dropped frame | full-resolution decode on the UI thread | decode at target size, off-thread, bounded LRU |
| Cold start to window visible | < 1 s | migration and scan before first paint | NativeAOT publish; paint first, defer work behind an indicator |
```

## Avoid

- Optimizing before attributing the cost to disk, CPU, GC, or allocation
- Full-hashing every file instead of grouping by size first
- A partial hash that reads only the head, so same-header files all collide
- Running the partial-hash tier on files smaller than the partial read
- Carrying a group of size 1 to the next tier
- SHA-256 or MD5 for content identity
- `File.ReadAllBytes` before hashing a large file
- A fresh 256 KB buffer allocated per file instead of rented from `ArrayPool<T>`
- Holding every scanned record alive in a `List<T>` when the consumer streams
- Interleaving stat and read per file instead of batching metadata first
- Parallel reads on a single spinning disk
- Hand-rolled intrinsics with no BenchmarkDotNet result beating the BCL call they replace
- Scanning, migrating, or indexing before the first paint
- Quoting a Debug-build timing, or a warm second run presented as cold
- Reporting a number with no tool, machine, or input scale
- Letting an `inferred` finding carry a Critical severity
