---
name: desktop-ecosystem-boundaries
description: Rust desktop capability register - what is a hard gap (printing, shell extensions, drag-out), what silently no-ops, and the escape hatch for each.
metadata:
  category: desktop
  tags: [rust, iced, ecosystem, crates, feasibility, platform-gaps, silent-failure, escape-hatch]
user-invocable: false
---

# Desktop Ecosystem Boundaries

> Load this **before** feature design, not after. A requirement that lands on a hard gap must be renegotiated at design time; discovering it mid-implementation costs the whole implementation.
>
> This skill owns **whether a capability is reachable at all, and at what cost**. Which crate to hold a chosen capability's data belongs to `desktop-data-persistence`; how to call the OS once a capability is confirmed reachable to `desktop-platform-integration`; whether a reachable capability is worth building to `desktop-overengineering-review`.

## When to Use

- Scoping a feature that touches the OS shell, printing, media, credentials, notifications, or the GPU
- Reviewing a plan or issue that assumes a capability exists because it exists in Electron, .NET, or Qt
- Triaging a feature that "does nothing" with no error in the log

## Rules

- **A capability with `Verdict: Gap` is not planned around, scheduled, or estimated.** It is either dropped from scope or replaced by its escape hatch, and the escape hatch is what gets estimated
- Every Gap verdict states whether the block is **Rust-specific** or **universal**. `UserChoice` file associations are impossible for every stack including C# and C++; saying "Rust can't" invites a rewrite that would also fail
- **Silent-failure traps are prerequisites, not polish.** Notifications, Keychain, and GPU selection each depend on packaging, signing, or process-start environment that no amount of application code substitutes for. Schedule them before the feature they gate, never after
- A crate whose last release predates the current year by more than two years is Gap regardless of download count. `sled` at 0.34.7 from 2021 is dead; `rusqlite` replaces it
- License is a verdict input, not a footnote. AGPL and GPL dependencies disqualify a closed-source app outright and appear in `Caveat:` before any technical merit
- **Verdicts are stated with the evidence that produced them** - version number, issue number, or date. `Workable` with no cited version is not a verdict, it is a guess. Where this register names a crate without a version, run the crates.io check and cite what it returns rather than guessing one

## Patterns

### What a verdict obliges the caller to do

| Verdict | Meaning | Design-time obligation |
| --- | --- | --- |
| `Strong` | Mature crate, current release, no setup beyond `cargo add` | Build it. No spike |
| `Workable` | Reachable, with a named cost you accept up front | Record the caveat in the plan; schedule the prerequisite |
| `Gap` | Not reachable in this stack, or not at all | Renegotiate scope, or estimate the escape hatch instead of the feature |
| `Unknown` | Not in this register | Run the named check before the feature is estimated |

### Hard gaps: do not design a feature on these

| Capability | Why it is blocked | Escape hatch |
| --- | --- | --- |
| **Printing** | Unsolved in Rust; Tauri was still only *proposing* a printing plugin as of March 2026 | Generate a PDF with `typst` or `printpdf`, then hand it to the OS print dialog: `ShellExecute` with the `print` verb on Windows, `open` on macOS |
| **File associations / "Open With" default** | Windows 10+ `UserChoice` is hash-protected and guarded by the `UCPD.sys` driver. **Impossible by design for every stack** - not a Rust limitation | Register the ProgID and *ask the user* to pick your app in the Windows Settings dialog. Deep-link them there; do not attempt the write |
| **Shell extensions / Finder Sync / Quick Look** | Windows needs COM plus sparse MSIX packaging; macOS needs an Xcode-built `.appex`. Zero working Rust examples exist | Drop the in-shell surface. Deliver the same value from your own window, or ship a separate non-Rust helper bundle |
| **Drag-OUT to Explorer/Finder** | Confirmed winit gap, open since 2020. Drag-**IN** works fine | Export to a user-chosen folder via `rfd`, or copy paths to the clipboard with `arboard` |
| **Rich text / WYSIWYG editing** | Iced issue #156. Markdown *rendering* and plain-text editing both work | Author in markdown with Iced 0.14's native `markdown` widget plus `text_editor`; render the formatted result |
| **OCR** | No credible pure-Rust option | Shell out to Tesseract; treat its presence as an optional runtime dependency and degrade cleanly when absent |
| **`sled` as the embedded store** | 0.34.7 dates to 2021; the 1.0 alpha stalled in 2024 | `rusqlite` 0.40 |

### Silent-failure traps: no error, no crash, feature just does nothing

These are the expensive ones. Each fails by doing nothing observable, so it survives development and dies on a user's machine.

- **`notify-rust` no-ops without an install identity.** Windows requires a Start Menu shortcut carrying an AUMID; macOS requires a real `.app` bundle. Run from `cargo run`, notifications silently vanish. Verify from an installed build, never from the target directory
- **macOS Keychain returns `-34018` for unsigned binaries**, and every rebuild changes the ad-hoc signing identity, so a credential stored yesterday is unreadable today. A stable signing identity is a prerequisite for any credential feature
- **`aws-lc-rs` needs CMake and NASM on Windows**, and rustls **panics** when both the `ring` and `aws-lc-rs` providers are active. Audit with `cargo tree -i aws-lc-rs` before adding any TLS-touching crate
- **wgpu finds no adapter under VMware** and defaults to the low-power iGPU on hybrid laptops. Iced's backend selection is **environment-variable-only**, so a launcher script or shim must set `ICED_BACKEND` / `WGPU_POWER_PREF` *before process start* - setting them in `main()` is too late. Test the `tiny-skia` CPU fallback deliberately
- **Packaging and signing gate the three above.** Sequence them first

```rust
// Bad - "fixing" GPU selection from inside the process; the adapter is already chosen
fn main() {
    std::env::set_var("WGPU_POWER_PREF", "high");
    iced::run(...)
}

// Good - a launcher sets it before the process that reads it starts
// run.cmd:  set WGPU_POWER_PREF=high && myapp.exe
```

### Strong: build freely

| Need | Crate |
| --- | --- |
| File watching | `notify` 8.2 |
| Clipboard | `arboard` |
| File dialogs | `rfd` 0.17 |
| Embedded store | `rusqlite` 0.40 |
| Logging / diagnostics | `tracing` |
| Excel and CSV | `calamine` (read) + `rust_xlsxwriter` (write) |
| Archives | `zip` 8.6, `sevenz-rust2`, `tar` + `flate2` |
| Localization | `i18n-embed` + `fluent-templates` + `icu4x` 2.0 |
| User scripting | `rhai` |
| HTTP | `ureq` 3.3 - avoids the tokio pull `reqwest` brings |
| Windows OS integration | `windows-registry`, `windows-service` |
| Markdown, code editing, highlighting | Iced 0.14 native `markdown`, `text_editor`, `highlighter` |

Nothing here needs a feasibility spike. Pick one and build.

### Verifying a trap instead of assuming it

A silent-failure trap is confirmed or cleared by a command, not by reading code:

```
cargo tree -d                 # duplicate crates - the wgpu/iced version-mismatch class
cargo tree -i aws-lc-rs       # second TLS provider - rustls panics with two active
cargo tree -i tokio           # an unintended runtime pulled in by a transitive dep
```

Run these before the feature is written. Each one that returns unexpected output converts an assumed-`Strong` capability into a `Workable` with a named caveat.

### Workable with caveats

| Capability | Approach | Caveat |
| --- | --- | --- |
| Auto-update | `velopack` or `self_update`, always `self-replace` for the exe swap | No de-facto standard; the choice is yours to own and maintain |
| Installers | `cargo-packager` | `cargo-bundle` is self-declared **alpha** |
| Tray icon | `tray-icon` + `muda` | macOS requires main-thread construction |
| Global hotkeys | `global-hotkey` | You write the macOS accessibility-permission flow yourself |
| Credential storage | `keyring` 4.1.6 | Deprecated itself toward `keyring-core`; 4.1.3 was yanked; needs a **stable** signing identity on macOS |
| Single instance | Available, boolean only | Argv forwarding to the running instance is yours to implement |
| PDF | `hayro` 0.7 (render), `lopdf` (edit), `typst` + `printpdf` (generate) | **`mupdf` is AGPL-3.0 - disqualifying for closed source** |
| Crash reporting | `sentry` 0.48 covers panics | Native minidumps need a second supervising process |
| Charting | `plotters-iced2` fork | Fragile; a fork tracking a moving Iced version |

## Output Format

One block per capability assessed:

```
Capability: {name}
Verdict: {Strong | Workable | Gap}
Blocked for: {Rust-specific | universal} - required when Verdict is Gap; omit otherwise
Crate/approach: {crate + version, or the mechanism; `none` when Verdict is Gap}
Caveat: {the trap, license, or prerequisite; `none` when Strong}
Escape hatch: {required when Verdict is Gap; omit for Strong and Workable}
```

`Escape hatch:` is mandatory on every `Gap` block. A Gap without one is an unanswered requirement, not a finding.

When the assessed capability sits in the silent-failure set, add one line after `Caveat:`:

```
Prerequisite: {packaging, signing, or launcher-environment step that must ship first}
```

When a capability is outside this register, emit `Capability: {name}` / `Verdict: Unknown` / `Evidence needed: {the crates.io check, issue search, or spike that would resolve it}` rather than guessing a verdict.

Close with one line so the caller knows the sweep ran:

```
Blocking gaps: {count} | Silent-failure prerequisites: {count} | Unknown: {count}
```

## Avoid

- Estimating a feature whose verdict is `Gap` instead of estimating its escape hatch
- Describing a universal platform block as a Rust shortcoming
- Testing notifications, Keychain, or GPU selection from `cargo run` instead of an installed, signed build
- Setting `ICED_BACKEND` or `WGPU_POWER_PREF` inside `main()`
- Adding a second TLS provider without running `cargo tree -i aws-lc-rs`
- Introducing `sled`, `mupdf`, or `cargo-bundle` into a new project
- Reporting a verdict with no version, issue number, or date behind it
- Scheduling packaging and signing after the features that depend on them
