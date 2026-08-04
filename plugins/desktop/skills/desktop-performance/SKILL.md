---
name: desktop-performance
description: Rust desktop throughput - two-tier size-then-hash grouping, fast non-cryptographic hashing, I/O ordering, allocation, startup cost, release profile.
metadata:
  category: desktop
  tags: [rust, performance, throughput, hashing, xxhash, blake3, two-tier, allocation, startup-latency, release-profile, benchmarking]
user-invocable: false
---

# Desktop Performance

> Confirm the target machine and the input scale before reporting any number - "fast" on an NVMe developer laptop over 500 files says nothing about a 200k-file external drive, which is the case users complain about.
>
> This skill owns **cost**: what a scan, a hash, and a cold start cost the user. Parallel execution mechanics belong to `desktop-concurrency-patterns`; decode sizing to `desktop-image-processing`; traversal correctness to `desktop-filesystem-patterns`; cache schema and invalidation to `desktop-data-persistence` (a missing or ineffective cache is still a `Caching` finding here); per-item failure typing to `rust-error-handling`; whether the optimization is warranted at all to `desktop-overengineering-review`.

## When to Use

- A scan, dedup, or batch that is slower than the user tolerates
- Reviewing a hot path: hashing, traversal, per-item allocation, comparison
- Slow app startup or a delayed first window
- Deciding whether an optimization is worth its complexity

## Rules

- **Attribute the cost to disk I/O, CPU, or allocation before changing anything.** The three have different fixes and only one of them is ever the bottleneck at a time
- **The algorithmic win comes first.** Two-tier grouping turns O(n) full reads into O(candidates) full reads; no micro-optimization inside a full read approaches that factor
- **Content identity uses a fast non-cryptographic hash.** BLAKE3 or xxHash3, not SHA-256. This is dedup, not signing
- **Never quote a number from a debug build.** Optimizations and SIMD are off; debug timings cannot even rank two approaches
- **Startup cost is the most visible latency in the app.** Work before the first paint is the only latency every user experiences on every run
- A hot loop does not allocate per item. Reuse the buffer
- Every reported number carries an evidence level and, for `measured`, the tool and the machine

## Patterns

### Attributing the cost

| Symptom | Likely owner | Typical cause |
| --- | --- | --- |
| CPU near idle, scan slow, disk busy | Disk I/O | reading every file in full; random access order |
| One core pinned, disk idle | CPU | hashing everything, or a cryptographic hash |
| All cores busy, throughput below single-threaded | Contention | shared lock per item (`desktop-concurrency-patterns`) |
| Memory climbs with input size, never falls | Allocation | unbounded queue or cache, per-item retained allocations |
| Slow only on the first run over a folder | Cold cache | expected; verify the second run is fast (`desktop-data-persistence`) |
| Window appears seconds after launch | Startup | work before first paint |
| Fine on SSD, unusable on a spinning or network drive | I/O ordering | seek-heavy access pattern, or a parallelised walk |

Attribute with a profiler, not by reading code: `cargo flamegraph` or `samply` for CPU, the OS activity monitor for disk-vs-CPU split, `hyperfine` for end-to-end wall time, `criterion` for a specific function.

### Two-tier hashing: the win that dominates

Comparing n files pairwise by content is the naive shape and it reads every byte of every file. The funnel does not:

| Tier | Reads | Typical survivors from 100k files |
| --- | --- | --- |
| 1. Group by file size | metadata only | ~5% - distinct sizes cannot be byte-identical |
| 2. Partial hash: first + last 64 KB | 128 KB per file, two seeks | ~1% - kills same-size-different-content sets |
| 3. Full hash, only within tier-2 collision groups | full read | the actual duplicate sets |

```rust
// Bad - hashes 400 GB to find 2 GB of duplicates
let mut by_hash: HashMap<Hash, Vec<PathBuf>> = HashMap::new();
for p in &paths { by_hash.entry(full_hash(p)?).or_default().push(p.clone()); }

// Good - each tier only sees what the previous one could not separate
let by_size = group_by(&paths, |p| metadata(p).map(|m| m.len()));
let candidates = by_size.into_values().filter(|g| g.len() > 1);
let by_partial = candidates.flat_map(|g| group_by(&g, partial_hash));
let dupes = by_partial.filter(|g| g.len() > 1).flat_map(|g| group_by(&g, full_hash));
```

Two details decide whether the win is real. A group of size 1 is discarded at every tier, never carried forward. And the partial hash must include the **tail** as well as the head: files sharing a size and a container header (media, archives, office documents) are exactly the case a head-only prefix fails to separate, which is the case this tier exists for.

Skip tier 2 for files below roughly its own read size - reading 128 KB from a 40 KB file to avoid reading 40 KB is a loss.

### Choosing the hash

| Hash | Throughput class | Use for |
| --- | --- | --- |
| BLAKE3 | GB/s, parallel-friendly, cryptographic | Default. Fast enough that its cryptographic strength is free |
| xxHash3 | Fastest available | Cache keys, partial-hash tiers, where a collision costs a byte comparison |
| SHA-256 | Roughly an order of magnitude slower | Nothing here. Signing and integrity attestation, which this app does not do |
| MD5 | Slow *and* broken | Nothing |

```rust
// Bad - SHA-256 for dedup; the hash becomes the bottleneck ahead of the disk
let digest = Sha256::digest(&bytes);

// Good - streamed, no whole-file buffer, and not the bottleneck
let mut hasher = blake3::Hasher::new();
let mut buf = vec![0u8; 256 * 1024];
loop {
    let n = file.read(&mut buf)?;
    if n == 0 { break; }
    hasher.update(&buf[..n]);
}
```

Reading with `fs::read` into a `Vec` before hashing allocates the whole file. Stream through a reused buffer; a 4 GB video must not become a 4 GB allocation.

Record which hash produced a cached digest, so a later change of algorithm invalidates rather than mismatches (`desktop-data-persistence`).

### I/O ordering

On a spinning disk, sequential access is roughly two orders of magnitude faster than random. Even on SSD, ordering reduces queue thrash.

- **Batch metadata before content.** One pass collecting sizes, then a second pass reading only the candidates - not interleaved stat-and-read per file
- **Order reads by directory**, so files stored near each other are read near each other
- **Do not parallelise reads on a single spinning disk.** Concurrent readers turn a sequential pattern into a seek storm (`desktop-concurrency-patterns`)
- **Never read a file twice** for two different purposes. If size, hash, and thumbnail are all needed, take them from one open

Memory-mapping is not a free speedup: it moves failure to a signal on I/O error, and on Windows an mmapped file blocks operations on it. Use it for large sequential hashing after measuring, not by default.

### Allocation on the per-item path

```rust
// Bad - two allocations per file across 200k files
for p in &paths {
    let name = p.file_name().unwrap().to_string_lossy().to_string();
    let key = format!("{}:{}", name, size);
}

// Good - one reused buffer, no per-item String
let mut key = String::with_capacity(64);
for p in &paths {
    key.clear();
    write!(key, "{}:{}", p.file_name().unwrap().to_string_lossy(), size)?;
}
```

Where to look, in payoff order: `to_string`/`format!` inside a loop, `clone()` on a `PathBuf` that could be a `&Path`, collecting an iterator that is immediately consumed, and a `Vec` grown without `with_capacity` when the count is known. `Cow<str>` avoids the allocation when the borrowed form is usually sufficient.

This is a real but second-order win. It is worth doing after the algorithmic tiering, not instead of it.

### Startup cost

Time to first paint is the latency the user notices most, because it precedes anything they asked for.

- **Nothing scans, migrates, or indexes before the window is shown.** Paint, then start the work with a visible indicator
- Restore the last window geometry from settings synchronously (it is one small file read) and defer everything else
- Open the SQLite connection lazily, on first use, not in `main`
- Run schema migration behind a visible state rather than a blank window
- Measure with `hyperfine` from process start to window-visible, on a cold OS file cache, not from a warm second run

A one-second delay before the window appears reads as a broken app; the same second spent with a visible spinner reads as work.

### Release profile

```toml
[profile.release]
opt-level = 3
lto = "thin"          # "fat" for a final ship build; slower to link
codegen-units = 1     # better optimization, slower build
panic = "abort"       # smaller binary; drop it if a panic must be caught
strip = "symbols"     # smaller shipped binary

[profile.dev.package."*"]
opt-level = 3         # dependencies optimized, own crate still debuggable
```

`codegen-units = 1` and `lto` together cost build time and buy single-digit-percent runtime, so they belong in the ship profile rather than the daily one. The `dev.package."*"` override is the higher-leverage setting during development: it makes `image`, hashing, and compression usable in a debug run without making the number quotable.

Binary size and installer packaging belong to `desktop-build-release`.

## Output Format

Two modes, chosen by whether something is being reviewed or authored.

**Authoring mode** - the request is to write or design something. Emit the code or design, then any `Deferred:` lines. No finding blocks, no severity, no status line: nothing was measured, so a not-run line would misdescribe the work.

**Review mode** - one block per finding:

```
### [Severity] {file:line | symbol, when source was supplied without paths | symptom, when no source was supplied}

- Category: {Algorithmic | HashChoice | IOOrdering | Allocation | Startup | BuildProfile | Caching | Memory}
- Evidence: {measured (tool, machine, input scale) | estimated (source read, no measurement; state the input scale assumed) | inferred (no source read; state what was not seen)}
- Code: {one-line citation, or `not supplied` when the finding is inferred}
- Cost: {with units - "400 GB read to find 2 GB of duplicates", "1.8 s before first paint", "2 allocations x 200k files", "SHA-256 at 400 MB/s vs 3 GB/s disk"}
- Fix: {the concrete change}
- Verify: {what to re-measure - hyperfine wall time on the same tree, bytes read, peak RSS, time to first paint on a cold cache}
```

`Severity: {Critical | High | Medium | Low}` - Critical = unusable at a realistic input scale, or unbounded memory growth. High = a measurable regression on a primary path, or an algorithmic shape one tier away from a large win. Medium = cost on a rare path or only at unlikely input sizes. Low = a cheap win with no observed symptom.

**Evidence gating.** `measured` requires a release-build number, and the block names the tool and the machine. `estimated` covers a finding read from source without a measurement, and names the input scale it assumes ("100k files, 400 GB"). `inferred` covers a finding from a bug report or a stated fact with no source read, and states what was not seen. A debug-build timing is never `measured`.

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
| Cold start to window visible | < 400 ms | migration and scan before first paint | paint first, defer work behind an indicator |
```

## Avoid

- Optimizing before attributing the cost to disk, CPU, or allocation
- Full-hashing every file instead of grouping by size first
- A partial hash that reads only the head, so same-header files all collide
- Running the partial-hash tier on files smaller than the partial read
- Carrying a group of size 1 to the next tier
- SHA-256 or MD5 for content identity
- `fs::read` into a `Vec` before hashing a large file
- Interleaving stat and read per file instead of batching metadata first
- Parallel reads on a single spinning disk
- Reading one file twice for size, hash, and thumbnail
- `format!` or `to_string` per item in a 200k-item loop
- Scanning, migrating, or indexing before the first paint
- Opening the database in `main` rather than on first use
- Quoting a timing from a debug build, or from a warm second run presented as cold
- Reporting a number with no tool, machine, or input scale
- Letting an `inferred` finding carry a Critical severity
- Micro-optimizing a loop whose enclosing algorithm reads every byte unnecessarily
