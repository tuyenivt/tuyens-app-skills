---
name: desktop-image-processing
description: Rust image 0.25 decode and dedup - thumbnail-resolution decode, bounded LRU cache, exact vs perceptual matching, EXIF orientation, off-thread decode.
metadata:
  category: desktop
  tags: [rust, image, thumbnail, decode, lru-cache, dedup, perceptual-hash, exif, orientation, simd, off-thread]
user-invocable: false
---

# Desktop Image Processing

> Confirm what the images are for before applying this skill - a preview grid, a dedup pass, and a conversion pipeline have different correctness bars.
>
> This skill owns **decoding pixels and comparing images**. Hash selection and the size-first grouping strategy belong to `desktop-performance`; getting the decode off the UI thread to `desktop-concurrency-patterns`; acting on the match to `desktop-batch-operations`; reading the files to `desktop-filesystem-patterns`.

## When to Use

- Building or reviewing a thumbnail grid, preview pane, or image list
- Duplicate or near-duplicate image detection
- Any code path calling `image::open`, `ImageReader`, or a resize
- Investigating memory growth or UI stalls in an image-heavy view

## Rules

- **Decode at thumbnail resolution. Never decode full then downscale.** A 6000x4000 JPEG is 72 MB of RGBA in memory; the 200px thumbnail needed 0.16 MB
- **Every cache is bounded.** An unbounded thumbnail map on a 50k-image folder is an OOM, not a slow path
- **Decoding never runs on the UI thread.** One 40 MP file freezes the window for a visible fraction of a second, and a grid does it hundreds of times
- **Hash equality alone never authorizes a destructive action.** Confirm bytes, or require explicit user selection
- Orientation is applied from EXIF before any comparison or display. Two files identical apart from their orientation tag are not the same picture
- **Never benchmark `image` in a debug build.** SIMD paths are disabled and optimizations are off; debug decode is commonly an order of magnitude slower and the number is meaningless
- A decode failure marks one item as failed. It does not end the scan

## Patterns

### Decode at the size you need

```rust
// Bad - 72 MB allocated and fully decoded to produce a 200px thumbnail,
// with no limits against a hostile header
let img = image::open(path)?;
let thumb = img.thumbnail(200, 200);

// Good - limits capped before any pixel is allocated
let reader = ImageReader::open(path)?.with_guessed_format()?;
let mut decoder = reader.into_decoder()?;
decoder.set_limits(limits_for_thumbnails())?;   // cap width, height, and alloc
let img = DynamicImage::from_decoder(decoder)?;
let thumb = img.thumbnail(200, 200);            // preserves aspect; `resize_exact` does not
```

Where the pinned `image` version exposes reduced-scale decode for a format (JPEG can emit 1/2, 1/4, or 1/8 scale directly from the DCT coefficients), request the smallest scale at or above the target before decoding - that skips most of the work rather than doing it and throwing it away. The API for this has moved across `image` versions and decoder backends; verify it exists in the resolved version's docs before relying on it, and fall back to decode-with-limits-then-thumbnail when it does not.

Set decoder limits on any file the user did not author. An untrusted image declaring 65535x65535 will happily attempt a 17 GB allocation (`desktop-security-patterns`).

`thumbnail` is a fast box filter and is the right choice for a grid. `resize` with `FilterType::Lanczos3` is for output the user will inspect; it is several times the cost for a difference invisible at 200px.

### Bounded thumbnail cache

```rust
// Bad - grows until the process dies; the user scrolled through 50k images
struct Cache { map: HashMap<PathBuf, RgbaImage> }

// Good - fixed capacity, LRU eviction, keyed on identity not just path
struct Cache { lru: LruCache<CacheKey, Arc<RgbaImage>> }

#[derive(Hash, PartialEq, Eq)]
struct CacheKey { path: PathBuf, size: u64, mtime_ns: i128 }
```

The key includes `size` and `mtime` so an edited file is not served from a stale entry - the same invalidation triple the scan cache uses (`desktop-data-persistence`). Capacity is set in **bytes**, not entries: 500 entries of 200px thumbnails is ~80 MB, but 500 entries of 800px previews is ~1.2 GB. Track decoded byte size and evict against a memory budget.

Store `Arc<RgbaImage>` so the UI can hold a handle while the cache evicts the slot.

### Dedup: cheap tests first, destructive action last

The pipeline is a funnel, and every stage exists to keep the expensive stage from running:

| Stage | Cost | Eliminates |
| --- | --- | --- |
| Group by file size | metadata only, no read | almost everything; distinct sizes cannot be byte-identical |
| Partial hash (first + last 64 KB) | two seeks | files sharing a size but differing early |
| Full content hash | one full read | everything but true byte-identical sets |
| Perceptual hash (optional) | decode + downscale | nothing - it *adds* candidates, so it runs on its own axis |

Size grouping and hash choice are `desktop-performance`. What belongs here is the distinction the UI must make:

- **Exact duplicates** are byte-identical. The same file, copied. Safe to present as interchangeable
- **Perceptual near-duplicates** are visually similar: a re-encode, a resize, a crop, a different quality setting. They are *not* interchangeable - one is usually higher quality, and which one the user wants is a judgment the tool cannot make

```rust
// Bad - a perceptual match is treated as a duplicate and auto-deleted
if dhash(a) == dhash(b) { fs::remove_file(b)?; }

// Good - separate verdicts; only the exact path can be automated, and even then
// only after a byte comparison
enum Match {
    Exact { verified: bool },          // hash equal; `verified` set by a byte compare
    Similar { distance: u32 },         // Hamming distance on the perceptual hash
}
```

**Why hash equality is not sufficient authorization**: a 64-bit perceptual hash collides between genuinely different images at scale, and even a strong content hash is a claim about bytes read at a past instant on a tree that may have changed. For a delete, either compare the bytes of the final candidates or require the user to have selected them explicitly. The cost of a byte comparison across a handful of confirmed candidates is negligible against the cost of deleting the wrong photo.

Present the group with a preview and a size, and default the retained file to the largest or oldest by a stated rule rather than by iteration order. The `Similar` distance cutoff ships as a user-facing sensitivity with a stated default, not a hardcoded constant - and no distance, however small, authorizes an automatic action.

### EXIF orientation

A JPEG from a phone stores landscape pixels plus an orientation tag. Ignoring it shows sideways thumbnails and, worse, makes a rotated re-save look like a different image to a perceptual hash.

```rust
// Bad - the tag is dropped; portrait photos render sideways
let img = DynamicImage::from_decoder(decoder)?;

// Good - orientation applied before anything sees the pixels
let orientation = decoder.orientation()?;
let mut img = DynamicImage::from_decoder(decoder)?;
img.apply_orientation(orientation);
```

Apply orientation before hashing, before thumbnailing, and before display, so all three agree. Note that this changes the pixel buffer but not the file: a re-save must either write the corrected pixels with orientation reset to `NoTransforms`, or preserve the original tag. Doing neither produces a double rotation.

### Decode off the UI thread, with cancellation

A grid decodes on scroll, and the user scrolls faster than the decoder finishes. Two consequences: work must be cancellable when its item scrolls out of view, and completed decodes must be applied only if still wanted.

```rust
// Bad - blocks the Iced update loop for the whole decode
Message::Visible(path) => { let t = decode_thumb(&path)?; self.thumbs.insert(path, t); }

// Good - hand to a worker, apply on the response, drop stale generations
Message::Visible(path) => Task::perform(decode_thumb(path, self.generation), Message::Thumb),
Message::Thumb(Ok(t)) if t.generation == self.generation => { self.thumbs.insert(t.path, t.image); }
```

Bound the number of in-flight decodes; an unbounded spawn per visible cell is a fork bomb on a fast scroll (`desktop-concurrency-patterns`). Prioritize visible cells over prefetch.

### Debug builds lie about image performance

`image` relies on optimized and SIMD-accelerated inner loops that only exist in a release build. Debug decode and resize timings are not proportional to release timings, so a debug measurement cannot even rank two approaches reliably.

```toml
# Makes a debug-profile run usable for spot checks without a full release build
[profile.dev.package."*"]
opt-level = 3
```

This optimizes dependencies while keeping your own crate debuggable. Any number reported as a measurement still comes from a release build (`desktop-performance`).

## Output Format

Two modes, chosen by whether something is being reviewed or authored.

**Authoring mode** - the request is to write image or thumbnail code. Emit the code, then any `Deferred:` lines. No finding blocks and no status line.

**Review mode** - one block per finding:

```
### [Severity] {file:line | symbol, when source was supplied without paths | symptom, when no source was supplied}

- Category: {DecodeSize | CacheBound | UIThread | MatchSemantics | Orientation | DecodeLimits | FailureHandling | Benchmarking}
- Evidence: {measured (release build, name the machine and image set) | estimated (source read, no measurement; state the image count and dimensions assumed) | inferred (no source read; state what was not seen)}
- Code: {one-line citation, or `not supplied` when the finding is inferred}
- Cost: {with units - "72 MB decoded per 200px thumbnail", "unbounded cache over 50k files", "180 ms UI stall per cell"}
- Fix: {the concrete change}
- Verify: {what to re-measure or re-check - peak RSS, decode ms in a release build, cache byte ceiling, orientation on a portrait phone photo}
```

`Severity: {Critical | High | Medium | Low}` - Critical = data loss from an unverified match, or unbounded memory on a realistic library. High = a UI freeze on a common path, or full-resolution decode for thumbnails. Medium = cost or wrongness on an uncommon input. Low = a cheap win with no observed symptom.

`Category` takes exactly one value; where a defect fits two, pick the one `Fix` addresses and name the other in `Cost`. `estimated` and `inferred` bound the header at High, with `Cost` naming the uncapped band; neither ever raises a block. A number from a debug build is not `measured` - it is `estimated`, and the block says so.

A defect owned by a sibling named in the ownership blockquote is written after the findings as `Deferred: {defect} -> {owning skill}`, one per line. Omit when there are none.

In review mode, close with exactly one status line, after any `Deferred:` lines:

| Condition | Line |
| --- | --- |
| One or more findings emitted | none - the findings are the output |
| Source or a symptom report was available, nothing found | `No image-processing findings.` |
| No source, diff, or symptom supplied | `Image check not run: no source supplied.` |

## Avoid

- `image::open` followed by a downscale, for a thumbnail
- A thumbnail cache with no eviction, or one bounded by entry count rather than bytes
- A cache key that is the path alone, with no size or mtime
- Decoding on the UI thread, or spawning one unbounded decode task per visible cell
- Applying a completed decode from a superseded scroll generation
- Deleting a file on hash equality alone, without a byte comparison or an explicit user selection
- Presenting perceptual near-duplicates and byte-identical duplicates as one verdict
- Auto-selecting which of a near-duplicate pair to delete
- Hashing or thumbnailing before applying EXIF orientation
- Re-saving corrected pixels while leaving the original orientation tag in place
- Decoding an untrusted image with no dimension or allocation limits
- Aborting a scan because one file failed to decode
- Quoting a decode or resize timing taken in a debug build
- `resize` with Lanczos3 for a 200px grid cell
