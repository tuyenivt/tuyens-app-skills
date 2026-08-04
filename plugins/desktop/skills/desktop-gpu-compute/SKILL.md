---
name: desktop-gpu-compute
description: wgpu 29 compute shaders in Iced - consume wgpu via iced's re-export, one WGSL codebase for DX12/Vulkan/Metal, keep results GPU-side.
metadata:
  category: desktop
  tags: [rust, iced, wgpu, wgsl, compute-shader, gpu, image-processing, shader-widget]
user-invocable: false
---

# Desktop GPU Compute

> Confirm the exact `iced` version from `Cargo.lock` first - it determines which `wgpu` you compile against, and that is not a choice you get to make separately.
>
> This skill owns **moving pixel work onto the GPU and back**. CPU-side image algorithms belong to `desktop-image-processing`; whether the work is worth parallelizing at all to `desktop-performance`; thread and task structure around the dispatch to `desktop-concurrency-patterns`; whether the GPU path earns its keep to `desktop-overengineering-review`.

## When to Use

- Per-pixel or per-tile work over large images where the same operation runs on every element
- Deciding whether an image operation belongs on the GPU or stays in `rayon`
- Reviewing a diff that adds `wgpu`, WGSL, or an Iced `shader` widget

## Rules

- **Consume `wgpu` through `iced::widget::shader::wgpu`, never as a direct dependency.** Iced re-exports its exact version; a separate `wgpu` line in `Cargo.toml` compiles a second copy of the crate and every device, buffer, and texture type mismatches with an error that reads like nonsense
- **The Iced `Engine` owns the device and queue.** Your code borrows them; it does not create them. Anything requiring non-default features or limits must be checked at startup, because you cannot retroactively request them
- **`wgpu`'s version is Iced's, and this project tracks latest.** Never pin `wgpu` independently - that is rule 1 restated, since a version chosen apart from Iced's is a second crate copy. Three breaking `wgpu` majors shipped in roughly six months, so an Iced upgrade can rewrite every shader binding underneath you: treat a `Cargo.lock` bump of `iced` or `wgpu` as a migration to run and test, and check the shader path explicitly after one
- Compute output that feeds rendering stays in a GPU buffer or texture. A readback to CPU memory that exists only to hand data to the next GPU pass forfeits the entire reason for the GPU path
- **The `tiny-skia` CPU fallback is a tested path, not a theoretical one.** VMware guests and adapter-less machines land there; if it has never been run, it does not work
- A GPU port needs a measurement against the CPU implementation it replaces. Transfer cost is real and for small images it dominates

## Patterns

### Take wgpu from Iced, not from crates.io

```toml
# Bad - two wgpu crates in the graph; `device` from Iced will not accept your `BufferDescriptor`
[dependencies]
iced = "0.14"
wgpu = "29.0"

# Good - one wgpu, the one Iced already resolved
[dependencies]
iced = { version = "0.14", features = ["wgpu"] }
```

```rust
// Good - the single import path for every wgpu type you touch
use iced::widget::shader::wgpu;
```

`cargo tree -d | grep wgpu` returning nothing is the invariant. This is the single most common way a working shader refuses to compile.

### One WGSL source, three backends

DX12 and Vulkan on Windows, Metal on macOS, from one shader file. A C++ or C# equivalent maintains HLSL *and* Metal Shading Language, plus the translation layer between them. That is the concrete advantage of this stack and it is worth naming when the GPU path is being justified.

```wgsl
// grayscale.wgsl - compiles unchanged for DX12, Vulkan, and Metal
@group(0) @binding(0) var src: texture_2d<f32>;
@group(0) @binding(1) var dst: texture_storage_2d<rgba8unorm, write>;

@compute @workgroup_size(8, 8)
fn main(@builtin(global_invocation_id) id: vec3<u32>) {
    let c = textureLoad(src, vec2<i32>(id.xy), 0);
    let l = dot(c.rgb, vec3<f32>(0.2126, 0.7152, 0.0722));
    textureStore(dst, vec2<i32>(id.xy), vec4<f32>(l, l, l, c.a));
}
```

### The `shader` widget and its `Program`

The widget hands you the device and queue at prepare time and a render pass at draw time. Pipelines and bind group layouts are built once and stored, not rebuilt per frame.

```rust
// Bad - a pipeline created inside the per-frame hook
fn prepare(&self, device: &wgpu::Device, ...) {
    let pipeline = device.create_compute_pipeline(&desc); // recompiled every frame
}

// Good - built once into shared storage, looked up per frame
fn prepare(&self, device: &wgpu::Device, queue: &wgpu::Queue, storage: &mut shader::Storage, ...) {
    if !storage.has::<Pipeline>() {
        storage.store(Pipeline::new(device, format));
    }
    storage.get_mut::<Pipeline>().unwrap().upload(device, queue, &self.params);
}
```

### Keep the result on the GPU

```rust
// Bad - compute, download, upload, render; two full PCIe transfers for nothing
let out = download_buffer(&device, &queue, &result_buf).await;   // 4K RGBA -> ~33 MB
let tex = upload_texture(&device, &queue, &out);

// Good - the compute pass writes a storage texture the render pass samples directly
// result_texture is bound to both pipelines; no CPU round-trip
```

Read back only when the CPU genuinely needs the bytes - saving a file, or feeding a CPU-only algorithm.

### Features and limits are decided before you are

```rust
// Bad - assumes a feature that Iced's device was never asked for; panics at pipeline creation
device.create_compute_pipeline(&pipeline_using_f16); // SHADER_F16 not enabled

// Good - probe at startup and pick a path
let supports_f16 = device.features().contains(wgpu::Features::SHADER_F16);
```

Check this at startup, not at first dispatch. Discovering an unavailable feature after the pipeline is written means rewriting the shader.

### When the GPU is not worth it

| Situation | Verdict |
| --- | --- |
| Inputs under roughly 1 MP - thumbnails, icons | CPU - transfer setup exceeds the compute |
| One pass over a few images | CPU - `rayon` already saturates the cores |
| Multi-megapixel image, several chained passes | GPU - transfer amortizes across passes |
| Per-pixel independent work, batch of large images | GPU |
| Branch-heavy, data-dependent per-pixel logic | CPU - divergence erases the parallelism |
| The result must be inspected on the CPU every frame | CPU - readback stalls the pipeline |

## Output Format

Per finding:

```
[Must | Recommend] {file:line | Cargo.toml | symbol, when source was supplied without paths | symptom, when no source was supplied}
Area: {Version Coupling | Device Ownership | Dispatch Shape | Transfer Cost | Fallback Path}
Issue: {the defect, named}
Evidence: {the measurement, the `cargo tree -d` result, or the API constraint}
Fix: {concrete edit}
```

`[Must]` for Version Coupling, Device Ownership, and an untested Fallback Path - broken compiles, startup panics, and a fallback that has never run are ship-blocking. Dispatch Shape and Transfer Cost escalate to `[Must]` only with a measurement cited in `Evidence`; `[Recommend]` otherwise.

A defect owned by a sibling named in the ownership blockquote is written after the findings as `Deferred: {defect} -> {owning skill}`, one per line. Omit when there are none.

When assessing whether a workload belongs on the GPU rather than reviewing code, produce instead:

```
Workload: {operation and input size}
Verdict: {GPU | CPU | Measure first}
Passes: {count of chained GPU passes, or `1`}
Readback: {none | per-frame | on-save}
Baseline: {the CPU timing this is compared against | not measured}
Fallback tested: {yes | no}
```

`Verdict: GPU` requires a `Baseline:` that is not `not measured` - absent one, the verdict is `Measure first`. `Verdict: CPU` stands without a baseline when a table row above justifies it.

Close with `wgpu source: {iced re-export | direct dependency - defect}` so the version-coupling check is visibly done.

## Avoid

- A `wgpu` entry in `Cargo.toml` alongside `iced`
- Creating a `wgpu::Device` or `Instance` when Iced's `Engine` already holds one
- A `wgpu` version chosen independently of Iced's, whether pinned or not
- Treating an Iced or `wgpu` lockfile bump as routine when it can rewrite every shader binding
- Building pipelines, bind group layouts, or shader modules inside a per-frame hook
- A CPU readback whose only consumer is the next GPU pass
- Requesting features or limits at dispatch time instead of probing at startup
- Setting the backend or power preference from inside `main()` rather than the process environment
- Shipping a `tiny-skia` fallback path that has never been executed
- Porting an operation to the GPU without timing the CPU implementation it replaces
