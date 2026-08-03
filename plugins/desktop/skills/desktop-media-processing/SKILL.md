---
name: desktop-media-processing
description: Rust audio and video - symphonia decode, cpal/rodio I/O, FFI encode, and the FFmpeg LGPL-vs-GPL linking trap that decides commercial viability.
metadata:
  category: desktop
  tags: [rust, audio, video, symphonia, cpal, rodio, ffmpeg, licensing, codecs, ffi]
user-invocable: false
---

# Desktop Media Processing

> **The licensing decision is settled: LGPL.** FFmpeg is an LGPL build, dynamically linked, never `--enable-gpl`. This keeps the application's own licensing free and rules out x264 and x265, so software H.264/H.265 encode is replaced by hardware encoders. Treat this as a constraint to design within, not a choice to re-open.
>
> This skill owns **audio and video pipelines and their licensing**. Still-image work belongs to `desktop-image-processing`; GPU pixel passes to `desktop-gpu-compute`; reading and writing media files on disk to `desktop-filesystem-patterns`; the decode/encode threading model to `desktop-concurrency-patterns`.

## When to Use

- Adding audio playback, decode, analysis, or format conversion
- Scoping any video feature, including "just extract a thumbnail"
- Reviewing a diff that adds `symphonia`, `cpal`, `rodio`, `rsmpeg`, `ffmpeg-the-third`, or an FFI codec

## Rules

- **Audio decode is strong; video decode is FFI.** No pure-Rust H.264 or H.265 decoder exists. Any plan claiming otherwise is wrong
- **Static linking is Cargo's ergonomic default and is wrong for LGPL compliance.** FFmpeg is dynamically linked here, always. Rust makes this trap worse than C++ does, because the correct choice is the one you have to go out of your way to make
- **`--enable-gpl` is prohibited**, along with x264 and x265. It would make the whole application GPL, and there is no commercial FFmpeg license at any price. H.264/H.265 encode goes through **hardware encoders** - NVENC, QuickSync, VideoToolbox - with `rav1e` (AV1) as the software fallback
- **Video sits behind a trait boundary from day one.** The binding lineage `ffmpeg` -> `ffmpeg-next` -> `ffmpeg-the-third` is three abandonments deep; the current binding will be abandoned too
- `symphonia`'s **default features are royalty-free codecs only**. MP3 and AAC require explicit feature flags, and it is **decode-only by design** - no plan may assume it encodes
- Budget a **multi-day spike** for FFmpeg build setup on Windows before estimating any video feature. vcpkg or prebuilt binaries, bindgen and LLVM, `PKG_CONFIG_PATH` - none of it is a one-afternoon task

## Patterns

### Audio: strong across the board

| Need | Crate | Note |
| --- | --- | --- |
| Decode | `symphonia` 0.6 (2026-05-15) | Pure safe Rust: MP3, AAC, FLAC, Vorbis, WAV, MP4, MKV. **Decode only** |
| Device I/O | `cpal` | WASAPI and ASIO on Windows, CoreAudio on macOS |
| Playback | `rodio` 0.22 | Sits on `cpal`; mixing and control |
| Resampling | `rubato` | High quality, sample-rate conversion |
| Plugins | `nih-plug` | Exports VST3 and CLAP, ships an Iced adapter |

```toml
# Bad - MP3 and AAC files silently fail to decode; only the royalty-free set is compiled in
symphonia = "0.6"

# Good - the codecs you actually accept are named
symphonia = { version = "0.6", features = ["mp3", "aac", "isomp4"] }
```

Audio **encode** is FFI regardless: LAME for MP3, libopus (the cleanest license of the three), FDK-AAC (check the terms - **not GPL-compatible**).

### Decode is not playback, and neither is analysis

Three different needs get conflated into "play a file", and each has a different crate:

| Need | Reach for | Not |
| --- | --- | --- |
| Sample data for analysis, waveform, or conversion | `symphonia` directly | `rodio` - it owns an output stream you do not want |
| Play a file to the default device | `rodio` 0.22 | raw `cpal` - you would rewrite mixing |
| Capture input, or a custom output graph | `cpal` | `rodio` - it does not expose the device layer you need |
| Change sample rate between two of the above | `rubato` | naive nearest-sample resampling |

Pulling `rodio` in for a waveform display starts an audio output stream and a mixing thread the app never uses.

### Video: the licensing decision comes first

**This plugin's projects are LGPL: an FFmpeg LGPL build, dynamically linked, with no `--enable-gpl`.** The row below is the decided path, not a menu. It keeps the application's own licensing free - closed-source commercial or open-source, either remains available - which is exactly what `--enable-gpl` would foreclose.

| Distribution model | Decode | Encode |
| --- | --- | --- |
| **LGPL - this plugin's default** | FFmpeg LGPL build, **dynamically linked** | Hardware encoder (NVENC / QuickSync / VideoToolbox), or AV1 via `rav1e` |
| GPL-compatible / open source | FFmpeg any build | `--enable-gpl` with x264/x265 is available |
| AV1 only, pure Rust | `rav1d` | `rav1e` |

Choosing LGPL has one consequence to design around rather than discover: **x264 and x265 are unavailable**, so software H.264/H.265 encode is off the table. Hardware encoders cover it on both targets - NVENC and QuickSync on Windows, VideoToolbox on macOS - at the cost of a per-vendor code path and a software fallback that is AV1 (`rav1e`) rather than H.264. Decode is unaffected: the LGPL build decodes H.264 and H.265 normally.

```toml
# Bad - static FFmpeg in a closed-source app; LGPL obligations unmet, and it builds cleanly
ffmpeg-the-third = { version = "*", features = ["static"] }

# Good - dynamic link, pinned binding, and the decision recorded
ffmpeg-the-third = { version = "3", default-features = false }   # links system LGPL FFmpeg
```

It builds and ships either way. Nothing in the toolchain warns you. That is what makes it expensive.

### Put video behind a trait on day one

```rust
// Bad - rsmpeg types threaded through the app; the next binding abandonment is a rewrite
pub fn thumbnail(path: &Path) -> Result<AVFrame> { /* rsmpeg all the way up */ }

// Good - one owned type crosses the boundary; the binding lives behind it
pub trait VideoSource {
    fn frame_at(&mut self, ts: Duration) -> Result<RgbaFrame, MediaError>;
    fn duration(&self) -> Duration;
}
// impl VideoSource for FfmpegSource - the only module that names the binding crate
```

This also isolates the FFmpeg build-environment pain to one crate rather than the whole workspace, and it makes a hardware-encoder swap a new impl instead of a refactor.

### The Windows FFmpeg build spike

Before any video estimate, run and record the outcome of:

1. Acquire FFmpeg - vcpkg or a prebuilt LGPL shared build. Confirm shared, not static
2. Install LLVM; `bindgen` needs `libclang` and will not say so clearly when it is missing
3. Set `PKG_CONFIG_PATH` (or `FFMPEG_DIR`) so the `-sys` crate resolves headers and import libs
4. Confirm the DLLs ship with the app and load from the installed layout, not just from the dev tree

A spike that has not produced a running binary on a clean machine has not de-risked anything.

### Silent-failure modes specific to media

- A `symphonia` feature omitted: the file opens, the probe finds no decoder, and the app shows an empty track rather than an error - surface the probe failure explicitly
- Dynamic FFmpeg with DLLs absent from the installed layout: works in `cargo run`, fails on the user's machine at first video open
- `cpal` device disconnection mid-stream: the stream stops silently unless the error callback is wired

## Output Format

Per finding:

```
[Must | Recommend] {file:line | Cargo.toml}
Area: {Licensing | Codec Coverage | Binding Longevity | Build Environment | Runtime Failure}
Issue: {the defect, named}
Exposure: {for Licensing: the obligation triggered and who it binds; otherwise the user-visible failure}
Fix: {concrete manifest, trait, or packaging edit}
```

`Area: Licensing` findings are always `[Must]`. No other area is escalated without a measurement or a reproduced failure.

When scoping a media feature rather than reviewing code, produce instead:

```
Media type: {audio | video | both}
Licensing: {LGPL - dynamic link, no --enable-gpl | n/a - no FFmpeg or FFI codec in this path}
Decode path: {crate + features, or FFI binding + link mode}
Encode path: {crate, FFI library, hardware encoder, or `none - decode only`}
Link mode: {dynamic | n/a} - required whenever FFmpeg appears; `static` is a defect, not an option
License obligations: {each triggered obligation - the LGPL relink provision, the shared-library shipping layout, the source offer}
Hardware encoder coverage: {per target, with the software fallback | `n/a - decode only`}
Trait boundary: {the owned type crossing it, or `absent - defect`}
Build spike: {completed on {date} | required, {estimate}}
```

`Licensing` is fixed, not a question, whenever FFmpeg or any FFI codec appears in either path - a scope emitting anything other than the LGPL line then has departed from the plugin's decision and states why in `License obligations`. A pure-Rust path (`symphonia` decode, `rav1e`/`rav1d`) writes the `n/a` form instead.

## Avoid

- Assuming a pure-Rust H.264 or H.265 decoder exists
- Statically linking FFmpeg
- `--enable-gpl`, x264, or x265 anywhere in the build
- Planning software H.264/H.265 encode, which LGPL rules out - hardware encoders and `rav1e` are the available paths
- Shopping for a commercial FFmpeg license - none exists
- `symphonia` with default features when MP3 or AAC input is accepted
- Planning encode on `symphonia`
- FFmpeg binding types appearing in application code outside the adapter module
- Estimating a video feature before the Windows build spike has produced a running binary
- Treating FDK-AAC as license-equivalent to libopus
- Shipping dynamic FFmpeg without verifying the DLLs load from the installed layout
