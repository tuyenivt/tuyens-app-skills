---
name: desktop-media-processing
description: .NET desktop audio and video - Media Foundation and AVFoundation first, FFmpeg LGPL dynamic fallback, hardware encode, NAudio limits.
metadata:
  category: desktop
  tags: [csharp, dotnet, audio, video, media-foundation, avfoundation, ffmpeg, licensing, codecs, naudio]
user-invocable: false
---

# Desktop Media Processing

> **The default path is settled: the OS media APIs.** Windows Media Foundation / `Windows.Media` and macOS AVFoundation / VideoToolbox are free, hardware-accelerated, and their H.264/H.265 patent licensing is already paid by Microsoft and Apple. FFmpeg is the fallback for coverage the OS lacks - an LGPL build, dynamically linked, never `--enable-gpl`. Treat both as constraints to design within, not choices to re-open.
>
> This skill owns **audio and video pipelines and their licensing**. Still-image work belongs to `desktop-image-processing`; reading and writing media files on disk to `desktop-filesystem-patterns`; the decode/encode threading model to `desktop-concurrency-patterns`; `async` mechanics to `csharp-async-patterns`; argument construction and spawning hygiene for external tool processes to `desktop-security-patterns`.

## When to Use

- Adding audio playback, decode, analysis, or format conversion
- Scoping any video feature, including "just extract a thumbnail"
- Reviewing a diff that adds `FFmpeg.AutoGen`, `FFMediaToolkit`, `NAudio`, `LibVLCSharp`, or a `Windows.Media` reference

## Rules

- **The OS media APIs are the default; FFmpeg is the fallback.** A plan that reaches for FFmpeg first names the format or need the OS APIs failed to cover. On Windows the WinRT APIs are high-level - this is not the raw COM other stacks would bind
- **The OS path costs two implementations** - Media Foundation and AVFoundation. That is the real reason FFmpeg dominates in every language: it trades a licensing problem for a portability problem. State the trade explicitly when choosing; do not discover it as scope creep
- **`--enable-gpl` with x264/x265 is prohibited.** It would make the whole application GPL. H.264/H.265 encode goes through hardware encoders - Media Foundation's vendor MFTs (NVENC / QuickSync / AMF) on Windows, VideoToolbox on macOS
- When FFmpeg appears, it is an **LGPL build, dynamically linked**. P/Invoke is dynamic linking by default, which makes .NET a better fit for LGPL compliance than a static-by-default toolchain - the compliant choice is the one you get without trying
- **Media sits behind an interface from day one.** Two OS implementations exist by design, so the seam has two implementers immediately - it is real, not ceremony
- **Decode, playback, and analysis are different needs.** Pulling a playback library to get sample data drags in machinery that plays nothing

## Patterns

### The default path: the OS already ships the codecs

| Need | Windows | macOS |
| --- | --- | --- |
| Transcode / convert | `MediaTranscoder` (`Windows.Media.Transcoding`) | `AVAssetExportSession` |
| Thumbnail / frame grab | `MediaComposition.GetThumbnailAsync` | `AVAssetImageGenerator` |
| Metadata, duration | `Windows.Media` properties / Source Reader | `AVAsset` |
| Hardware H.264/H.265 encode | Media Foundation MFTs - the OS picks NVENC / QuickSync / AMF | VideoToolbox |
| Playback | `MediaPlayer` | `AVPlayer` |

On Windows these are WinRT APIs, reached from .NET via CsWinRT with a `net10.0-windows10.0.19041.0`-style TFM. On macOS AVFoundation is reached via the macOS bindings or a thin native shim - that shim is part of the two-implementation cost, not a reason to abandon the path.

```csharp
// Bad - FFmpeg pulled in for an MP4 thumbnail on Windows; licensing discipline and DLL shipping bought for nothing
var frame = FfmpegHelper.ExtractFrame(path, TimeSpan.FromSeconds(1));

// Good - the OS API the codecs already live in
var clip = await MediaClip.CreateFromFileAsync(file);
var composition = new MediaComposition { Clips = { clip } };
var thumb = await composition.GetThumbnailAsync(TimeSpan.FromSeconds(1), 0, 0, VideoFramePrecision.NearestFrame);
```

### When the fallback is earned

| Situation | Path |
| --- | --- |
| Format the OS decodes (MP4, MOV, H.264, AAC, MP3) | OS media APIs |
| Format coverage the OS lacks (full MKV, exotic codecs) | FFmpeg fallback |
| One pipeline that must behave identically on both OSes | FFmpeg fallback, trade named |
| Remux - trim or repackage without re-encode | FFmpeg stream copy; check whether remux covers the requirement before budgeting any encoder work |

### The FFmpeg fallback, and the discipline that still applies

Bindings: `FFmpeg.AutoGen` (raw generated bindings) or `FFMediaToolkit` (higher-level frame access). Either way:

- LGPL build, dynamically linked - P/Invoke already is. **Most convenient prebuilt Windows archives are GPL "full" builds; ship an LGPL shared build**, and record the source offer the LGPL requires
- No `--enable-gpl`, no x264, no x265. Hardware encoders remain the encode path even inside FFmpeg (`h264_mf`, `hevc_videotoolbox`)
- The native DLLs ship with the app; verify they load from the installed layout, not just from `bin/Debug`

### Audio: name the need before the library

| Need | Windows | Cross-platform |
| --- | --- | --- |
| Decode for analysis or waveform | NAudio `MediaFoundationReader` | FFmpeg fallback |
| Playback | NAudio (WASAPI) | `LibVLCSharp` - heavyweight but LGPL, dynamically linked, fits the discipline |
| Capture | NAudio (WASAPI capture) | Per-OS APIs; no strong cross-platform story - state it, do not imply one |

- **NAudio is Windows-centric** - WASAPI, WaveOut, `MediaFoundationReader` do not exist on macOS. It is the right Windows-primary answer and the wrong cross-platform one
- **`ManagedBass` is a licensing trap**: the BASS library it wraps requires a paid licence for commercial use. Name that before any technical merit
- **Audio encode follows the same split as video**: the OS transcoder covers AAC on both platforms. MP3 or Opus encode means an FFI codec (LAME, libopus) - each names its own licence in the `Licensing:` line, and libopus carries the cleanest terms of the set
- **`LibVLCSharp` ships FFmpeg-derived native libraries**, so the LGPL discipline and the `Licensing:` predicate apply to it exactly as they do to a direct FFmpeg binding

### Silent-failure modes specific to media

- **Windows N editions lack the Media Feature Pack**, so Media Foundation is absent entirely. Catch the failure on first use and degrade with a message, not a crash
- **H.265 decode is not guaranteed on Windows** - the HEVC Video Extensions package is a paid Store item. Probe, then route H.265 files to the fallback or report clearly
- **FFmpeg DLLs absent from the installed layout**: works in `dotnet run`, dies on the user's machine at first video open
- **Hardware encode is not guaranteed**: Media Foundation silently falls back to a software MFT when the vendor encoder is unavailable (VMs, stale drivers) - same output, minutes slower. Log which encoder ran

### The interface boundary

```csharp
// Bad - WinRT types threaded through the app; the macOS implementation now has no seam
public Task<SoftwareBitmap> GetThumbnailAsync(string path);

// Good - one owned type crosses; each implementation is the only class naming its API
public interface IVideoSource
{
    Task<RgbaFrame> GetFrameAsync(TimeSpan at);
    TimeSpan Duration { get; }
}
// MediaFoundationSource, AvFoundationSource, FfmpegSource
```

This is also what keeps the FFmpeg fallback a swap instead of a rewrite, and confines each OS API to one project.

## Output Format

Per finding, when reviewing code:

```
[Must | Recommend] {file:line | csproj | symbol, when source was supplied without paths}
Area: {Licensing | Codec Coverage | Platform Coverage | Binding Longevity | Runtime Failure}
Issue: {the defect, named}
Exposure: {for Licensing: the obligation triggered and who it binds; otherwise the user-visible failure}
Fix: {concrete code, csproj, or packaging edit}
```

`Area: Licensing` findings are always `[Must]`. No other area is escalated without a measurement or a reproduced failure - and a deterministic silent failure readable from the project (a GPL build bundled, DLLs absent from the installed layout, H.265 accepted with no decoder probe) counts as reproduced. A missing interface boundary files as `Binding Longevity`.

A defect owned by a sibling named in the ownership blockquote is written after the findings or the scoping form as `Deferred: {defect} -> {owning skill}`, one per line. Omit when there are none. A review that produces no finding closes with exactly `No media findings.`

When scoping a media feature rather than reviewing code, produce instead the form below - one form per scope; when several needs coexist, a field carries one line per need:

```
Media type: {audio | video | both}
Path: {OS media APIs | FFmpeg fallback | mixed - which need takes which path}
Licensing: {LGPL - dynamic link via P/Invoke, no --enable-gpl, source offer recorded - when FFmpeg or any FFI codec is in the path | n/a - OS media APIs only}
Decode path: {OS API per platform, or binding + FFmpeg build variant}
Encode path: {OS transcoder, hardware encoder, or `none - decode only`}
Platform coverage: {per target, with the fallback or degradation when one OS lacks the codec}
Interface boundary: {the owned type crossing it, or `absent - defect`}
```

`Licensing:` is a predicate, not a fixed value: the LGPL discipline line applies exactly when FFmpeg or an FFI codec library is in the path, and is `n/a - OS media APIs only` otherwise. A non-FFmpeg FFI codec (LAME, libopus) names its own licence on the same line.

## Avoid

- Reaching for FFmpeg first when the OS API covers the format - and reaching for it silently, without naming the trade
- `--enable-gpl`, x264, or x265 anywhere in any build
- Bundling a prebuilt GPL "full" FFmpeg archive because it was the first download link
- Statically linking FFmpeg or an FFI codec into a closed-source binary
- Treating NAudio as the cross-platform audio answer
- `ManagedBass` in a commercial app without a purchased BASS licence
- Assuming H.265 decode or Media Foundation itself is present on every Windows machine
- OS-specific media types (`SoftwareBitmap`, `AVAsset`) appearing outside their adapter implementation
- Shipping FFmpeg DLLs without verifying they load from the installed layout
- Conflating decode, playback, and analysis into one library choice
