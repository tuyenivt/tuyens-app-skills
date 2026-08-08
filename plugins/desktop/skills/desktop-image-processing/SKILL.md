---
name: desktop-image-processing
description: Decode and dedup images with SkiaSharp - scaled thumbnail decode, bounded LRU cache, hash funnel, EXIF orientation, off-UI-thread decode.
metadata:
  category: desktop
  tags: [csharp, skiasharp, avalonia, thumbnail, decode, lru-cache, dedup, xxhash, perceptual-hash, exif, orientation, off-thread]
user-invocable: false
---

# Desktop Image Processing

> Confirm what the images are for before applying this skill - a preview grid, a dedup pass, and a conversion pipeline have different correctness bars.
>
> This skill owns **decoding pixels and comparing images**. Hash selection and the size-first grouping strategy belong to `desktop-performance`; a UI-thread decode is a finding here, while the worker, channel, and in-flight-cap mechanics of the fix belong to `desktop-concurrency-patterns`; acting on the match to `desktop-batch-operations`; reading the files to `desktop-filesystem-patterns`; video and audio to `desktop-media-processing`.

## When to Use

- Building or reviewing a thumbnail grid, preview pane, or image list
- Duplicate or near-duplicate image detection
- Any code path calling `SKBitmap.Decode`, `SKCodec`, or a resize
- Investigating memory growth or UI stalls in an image-heavy view

## Rules

- **Decode at thumbnail resolution. Never decode full then downscale.** A 6000x4000 JPEG is 96 MB of BGRA in memory; the 200 px thumbnail needed 0.16 MB
- **Every cache is bounded in bytes and disposes what it evicts.** An unbounded thumbnail map on a 50k-image folder is an OOM, not a slow path
- **Decoding never runs on the UI thread.** One 40 MP file freezes the window for a visible fraction of a second, and a grid does it hundreds of times
- **Hash equality alone never authorizes a destructive action.** Confirm bytes, or require explicit user selection
- Orientation is applied from EXIF before any comparison or display. Two files identical apart from their orientation tag are not the same picture
- `SKBitmap` and `SKCodec` hold native memory the GC cannot see. Dispose deterministically - a leak presents as process growth no managed profiler attributes
- A decode failure marks one item as failed. It does not end the scan

## Patterns

### Decode at the size you need

```csharp
// Bad - 96 MB decoded to produce a 200 px thumbnail
using var full = SKBitmap.Decode(path);                    // 6000x4000 BGRA = 96 MB
var thumb = full.Resize(new SKImageInfo(200, 133), SKSamplingOptions.Default);

// Good - the codec decodes near the target size; most of the work never happens
using var codec = SKCodec.Create(path);
var near = codec.GetScaledDimensions(200f / codec.Info.Width);
using var reduced = SKBitmap.Decode(codec,
    new SKImageInfo(near.Width, near.Height, SKColorType.Bgra8888, SKAlphaType.Premul));
var thumb = reduced.Resize(TargetInfo(reduced, 200), SKSamplingOptions.Default);
```

JPEG honours the requested scale natively - libjpeg-turbo emits 1/2, 1/4, or 1/8 size directly from the DCT coefficients, skipping most of the work rather than doing it and throwing it away. Formats without scaled decode return the full dimensions from `GetScaledDimensions`; the pattern still works, it just pays the full decode once before the downscale.

Guard untrusted input *before* decoding: `codec.Info` states the declared dimensions without touching pixels. Reject when `(long)Info.Width * Info.Height * 4` exceeds a budget - an image header declaring 65535x65535 otherwise attempts a 17 GB allocation. The guard is authored here; `desktop-security-patterns` supplies only the threat framing.

### Bounded thumbnail cache

```csharp
// Bad - grows until the process dies, and the GC never sees the native pixels
Dictionary<string, SKBitmap> cache = [];

// Good - byte budget, LRU eviction, identity key, disposal on evict
public sealed record CacheKey(string Path, long Size, long MtimeTicks);

public sealed class ThumbCache(long byteBudget) {
    private readonly Dictionary<CacheKey, LinkedListNode<Entry>> _map = [];
    private readonly LinkedList<Entry> _lru = new();   // front = most recent
    private long _bytes;
    // Add: _bytes += entry.ByteCount; evict from the back while _bytes > byteBudget,
    // disposing each evicted bitmap
}
```

The key includes `Size` and `MtimeTicks` so an edited file is not served from a stale entry - the same invalidation triple the scan cache uses (`desktop-data-persistence`). An entry a visible cell still binds is not disposed by eviction - skip pinned entries, or hand ownership to the view when it binds. Capacity is set in **bytes**, not entries: 500 entries of 200 px thumbnails is ~80 MB, but 500 entries of 800 px previews is ~1.2 GB. Track decoded byte size and evict against a memory budget, disposing what leaves.

### SKBitmap to Avalonia Bitmap

```csharp
// Decode as Bgra8888 premultiplied (above), copy once into what Avalonia composites
using var skia = DecodeThumb(path);
var bmp = new Bitmap(PixelFormat.Bgra8888, AlphaFormat.Premul,
    skia.GetPixels(), new PixelSize(skia.Width, skia.Height),
    new Vector(96, 96), skia.RowBytes);
```

The Avalonia `Bitmap` copies the pixels, so the `SKBitmap` is disposed immediately after and the cache holds the `Bitmap` the views bind to. Whichever side the cache holds, the byte budget and eviction-disposal rules apply - `Bitmap` is native memory too.

### Dedup: cheap tests first, destructive action last

The pipeline is a funnel, and every stage exists to keep the expensive stage from running:

| Stage | Cost | Eliminates |
| --- | --- | --- |
| Group by file size | metadata only, no read | almost everything; distinct sizes cannot be byte-identical |
| Partial hash (first + last 64 KB, `XxHash3`) | two seeks | files sharing a size but differing early |
| Full content hash (`XxHash128`) | one full read | everything but true byte-identical sets |
| Perceptual hash (optional) | decode + downscale | nothing - it *adds* candidates, so it runs on its own axis |

`System.IO.Hashing`'s XxHash3 and XxHash128 are SIMD-accelerated and non-cryptographic - right for dedup, where the adversary is chance, not an attacker. Size grouping and hash choice are `desktop-performance`. What belongs here is the distinction the UI must make:

- **Exact duplicates** are byte-identical. The same file, copied. Safe to present as interchangeable
- **Perceptual near-duplicates** are visually similar: a re-encode, a resize, a crop, a different quality setting. They are *not* interchangeable - one is usually higher quality, and which one the user wants is a judgment the tool cannot make

```csharp
// Bad - a perceptual match is treated as a duplicate and auto-deleted
if (DHash(a) == DHash(b)) File.Delete(b);

// Good - separate verdicts; only the exact path can be automated, and even then
// only after a byte comparison
public abstract record Match;
public sealed record Exact(bool Verified) : Match;    // hashes equal; Verified set by a byte compare
public sealed record Similar(int Distance) : Match;   // Hamming distance on the perceptual hash
```

**Why hash equality is not sufficient authorization**: a 64-bit perceptual hash collides between genuinely different images at scale, and even a full content hash is a claim about bytes read at a past instant on a tree that may have changed. For a delete, either compare the bytes of the final candidates or require the user to have selected them explicitly. The cost of a byte comparison across a handful of confirmed candidates is negligible against the cost of deleting the wrong photo.

Present the group with a preview and a size, and default the retained file to the largest or oldest by a stated rule rather than by iteration order. The `Similar` distance cutoff ships as a user-facing sensitivity with a stated default, not a hardcoded constant - and no distance, however small, authorizes an automatic action.

### EXIF orientation

A JPEG from a phone stores landscape pixels plus an orientation tag. Ignoring it shows sideways thumbnails and, worse, makes a rotated re-save look like a different image to a perceptual hash.

```csharp
// Bad - the origin is dropped; portrait photos render sideways
using var bmp = SKBitmap.Decode(path);

// Good - orientation applied before anything sees the pixels
using var codec = SKCodec.Create(path);
using var raw = SKBitmap.Decode(codec);
using var upright = ApplyOrigin(raw, codec.EncodedOrigin);  // rotate/flip on a canvas per origin
```

Apply orientation before hashing, before thumbnailing, and before display, so all three agree. Note that this changes the pixel buffer but not the file: a re-save must either write the corrected pixels with the orientation reset to top-left, or preserve the original tag. Doing neither produces a double rotation.

### Decode off the UI thread, with cancellation

A grid decodes on scroll, and the user scrolls faster than the decoder finishes. Two consequences: work must be cancellable when its item scrolls out of view, and completed decodes must be applied only if still wanted.

```csharp
// Bad - blocks the UI thread for the whole decode
void OnVisible(string path) => Thumbs[path] = DecodeThumb(path);

// Good - a worker decodes; the result applies only if its generation is still current
async Task OnVisibleAsync(string path) {
    var gen = _generation;
    var thumb = await Task.Run(() => DecodeThumb(path), _cts.Token);
    if (gen == _generation) Thumbs[path] = thumb; else thumb.Dispose();
}
```

Bound the number of in-flight decodes; an unbounded `Task.Run` per visible cell is a memory spike and a thread-pool flood on a fast scroll (`desktop-concurrency-patterns` owns the worker and cap mechanics). Prioritize visible cells over prefetch, and dispose results from superseded generations - they are native memory.

## Output Format

Two modes, chosen by whether something is being reviewed or authored.

**Authoring mode** - the request is to write image or thumbnail code. Emit the code, then any `Deferred:` lines. No finding blocks and no status line.

**Review mode** - one block per finding, ordered by severity, Critical first:

```
### [Severity] {file:line | symbol, when source was supplied without paths | symptom, when no source was supplied}

- Category: {DecodeSize | CacheBound | UIThread | MatchSemantics | Orientation | DecodeLimits | FailureHandling | NativeMemory}
- Evidence: {measured (name the machine and image set) | estimated (source read, no measurement; state the image count and dimensions assumed) | inferred (no source read; state what was not seen)}
- Code: {one-line citation, or `not supplied` when the finding is inferred}
- Cost: {with units - "96 MB decoded per 200 px thumbnail", "unbounded cache over 50k files", "180 ms UI stall per cell"}
- Fix: {the concrete change}
- Verify: {what to re-measure or re-check - peak private bytes, decode ms, cache byte ceiling, orientation on a portrait phone photo}
```

`Severity: {Critical | High | Medium | Low}` - Critical = data loss from an unverified match, or unbounded memory on a realistic library. High = a UI freeze on a common path, full-resolution decode for thumbnails, or wrong pixels or files on a common input (sideways rendering, a re-save that double-rotates, aspect distortion). Medium = cost or wrongness on an uncommon input. Low = a cheap win with no observed symptom.

`Category` takes exactly one value; where a defect fits two, pick the one `Fix` addresses and name the other in `Cost`. `estimated` and `inferred` bound the header at High, with `Cost` naming the uncapped band; neither ever raises a block.

A defect owned by a sibling named in the ownership blockquote is written after the findings as `Deferred: {defect} -> {owning skill}`, one per line. Omit when there are none.

In review mode, close with exactly one status line, after any `Deferred:` lines:

| Condition | Line |
| --- | --- |
| One or more findings emitted | none - the findings are the output |
| Source or a symptom report was available, nothing found | `No image-processing findings.` |
| No source, diff, or symptom supplied | `Image check not run: no source supplied.` |

## Avoid

- `SKBitmap.Decode` of the full image followed by `Resize`, for a thumbnail
- A thumbnail cache with no eviction, or one bounded by entry count rather than bytes
- A cache key that is the path alone, with no size or mtime
- Evicting without disposing, or relying on finalizers to release Skia memory
- Decoding on the UI thread, or spawning one unbounded decode task per visible cell
- Applying a completed decode from a superseded scroll generation, or leaking it undisposed
- Deleting a file on hash equality alone, without a byte comparison or an explicit user selection
- Presenting perceptual near-duplicates and byte-identical duplicates as one verdict
- Auto-selecting which of a near-duplicate pair to delete
- Hashing or thumbnailing before applying EXIF orientation
- Re-saving corrected pixels while leaving the original orientation tag in place
- Decoding an untrusted image without checking its declared dimensions against a budget
- Aborting a scan because one file failed to decode
