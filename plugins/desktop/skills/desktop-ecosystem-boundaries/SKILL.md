---
name: desktop-ecosystem-boundaries
description: Capability register for .NET/Avalonia desktop - hard gaps, silent no-op traps, the Avalonia paid-tier boundary, and escape hatches.
metadata:
  category: desktop
  tags: [csharp, dotnet, avalonia, ecosystem, nuget, feasibility, platform-gaps, silent-failure, nativeaot, escape-hatch]
user-invocable: false
---

# Desktop Ecosystem Boundaries

> Load this **before** feature design, not after. A requirement that lands on a hard gap must be renegotiated at design time; discovering it mid-implementation costs the whole implementation.
>
> This skill owns **whether a capability is reachable at all, and at what cost**. Which store holds a chosen capability's data belongs to `desktop-data-persistence`; how to call the OS once a capability is confirmed reachable to `desktop-platform-integration`; whether a reachable capability is worth building to `desktop-overengineering-review`; the packaging, signing, and publish prerequisites a silent-failure trap names to `desktop-build-release`.

## When to Use

- Scoping a feature that touches the OS shell, printing, OCR, notifications, drag and drop, credentials, or accessibility
- Reviewing a plan that assumes a capability exists because it exists in Electron, WPF, or Qt - or assumes one is missing because another stack lacked it
- Triaging a feature that "does nothing" with no error in the log
- Checking whether a dependency sits on the free or paid side of the Avalonia tier boundary

## Rules

- **A capability with `Verdict: Gap` is not planned around, scheduled, or estimated.** It is dropped from scope or replaced by its escape hatch, and the escape hatch is what gets estimated
- Every Gap verdict states whether the block is **stack-specific** or **universal**. `UserChoice` file associations are impossible for every stack including C++ and Electron; saying ".NET can't" invites a rewrite that would also fail
- **Silent-failure traps are prerequisites, not polish.** Notifications, Keychain, and AOT-safe serialization each depend on packaging, signing, or build configuration that no application code substitutes for. Schedule them before the feature they gate, never after
- **NativeAOT compatibility is a verdict input.** A library that works only under JIT reflection is `Workable` at best in an AOT-published app, and its trimming warnings are the evidence (`desktop-build-release` owns the publish setup)
- The Avalonia framework is MIT and free for commercial use; **paid tiers sell tooling and premium components, not the framework**. A plan that budgets $0 for a control the Pro tier owns has a gap, not a discount
- Verdicts are stated with the evidence that produced them - version, date, or documented OS behaviour. `Workable` with no citation is a guess, not a verdict. Where this register names an API without a version, run the NuGet or platform-docs check and cite what it returns

## Patterns

### What a verdict obliges the caller to do

| Verdict | Meaning | Design-time obligation |
| --- | --- | --- |
| `Strong` | First-party or mature API, no setup beyond a PackageReference | Build it. No spike |
| `Workable` | Reachable, with a named cost you accept up front | Record the caveat in the plan; schedule the prerequisite |
| `Gap` | Not reachable in this stack, or not at all | Renegotiate scope, or estimate the escape hatch instead of the feature |

### Present capabilities other stacks lack: do not design around a gap that is not there

Plans written against other desktop stacks routinely assume these are missing. Here they work:

| Capability | Verdict | API |
| --- | --- | --- |
| Screen readers | Strong | Avalonia automation peers: UIA on Windows, NSAccessibility on macOS (`desktop-accessibility` owns usage) |
| Drag OUT to Explorer/Finder | Strong | `DragDrop.DoDragDrop` with a `DataObject` carrying the files |
| OCR | Workable | `Windows.Media.Ocr` - WinRT, free, ships inside Windows. Windows-only; macOS needs a Vision-framework shim |
| Printing | Workable | `System.Drawing.Printing` / `System.Printing`. Windows-only - see the cross-platform row below |
| Rich text editing | Workable | Editable rich text is reachable, but full WYSIWYG remains limited - the polished Rich Text Editor is an Avalonia Pro-tier component |

### Hard gaps: do not design a feature on these

| Capability | Why it is blocked | Escape hatch |
| --- | --- | --- |
| **File associations / "Open With" default** | Windows 10+ `UserChoice` is hash-protected and guarded by the `UCPD.sys` driver. **Impossible by design for every stack - this is not a .NET limitation** | Register the ProgID and deep-link the user to the Windows default-apps Settings page to pick your app themselves; never attempt the write |
| **Cross-platform printing** | The printing story is Windows-only: `System.Drawing.Common` is Windows-only in modern .NET, and Avalonia has no print API | Render to PDF with SkiaSharp (`SKDocument.CreatePdf`), then hand it to the OS: the `print` verb on Windows, `open` on macOS |
| **Finder Sync / Quick Look extensions** | Require an Xcode-built `.appex` embedded in the bundle; no .NET path | Drop the in-shell surface; deliver the same value from your own window |
| **Windows shell extensions** (context menu, overlay, preview) | Arguably worse in .NET than elsewhere: loading the CLR into Explorer's process was historically discouraged. NativeAOT plus `ComWrappers` improves it, but it remains an expert path a native DLL is better suited to | Deliver the action from your own window, or ship a small native (C++/Rust) DLL as a separate artefact |

### Silent-failure traps: no error, no crash, feature just does nothing

These are the expensive ones. Each fails by doing nothing observable, so it survives development and dies on a user's machine.

- **Toast notifications no-op without an install identity.** Windows requires a Start Menu shortcut carrying an AUMID (or MSIX identity); macOS requires a real signed `.app` bundle. Under `dotnet run` they can appear fine - the installed build without its own identity is what drops them silently. Verify from an installed build, never from `bin/`
- **macOS Keychain fails for unsigned binaries**, and every rebuild changes the ad-hoc signing identity, so a credential stored yesterday is unreadable today. A stable signing identity is a prerequisite for any credential feature
- **`FileSystemWatcher` silently drops events when its internal buffer overflows.** Wire the `Error` event, raise `InternalBufferSize`, and rescan on overflow - treat the watcher as a hint, not a ledger
- **NativeAOT breaks reflection-based code**: reflection JSON serializers, convention-based DI registration, and `{Binding}` runtime XAML binding. Use source generators, explicit registration, and compiled bindings; trimming warnings are the early signal, and a suppressed warning is a runtime failure deferred to a customer machine

```csharp
// Bad - works under JIT, fails or returns nothing under NativeAOT; the trim warning was ignored
var settings = JsonSerializer.Deserialize<Settings>(json);

// Good - source-generated context; AOT-safe and trim-safe
var settings = JsonSerializer.Deserialize(json, SettingsContext.Default.Settings);
```

### The Avalonia tier boundary

The MIT framework is complete for a closed-source commercial app at $0: ~70 built-in controls, no runtime licence checks. What is paid is tooling and premium components:

| Tier | What it holds |
| --- | --- |
| Free (MIT framework) | The framework, all ~70 built-in controls, commercial use included |
| Plus | The new DevTools and IDE extensions |
| Pro | Premium control suite, including charts and the Rich Text Editor |
| Community | **Strictly non-commercial since its 2026 narrowing** - not available to this distribution model |

Avalonia 12 removed the free `Avalonia.Diagnostics` DevTools from the default template; the legacy package still exists, is free, and is re-added manually. State the risk plainly in any plan leaning on Avalonia tooling: free developer experience has moved behind paywalls twice in 18 months.

### Strong: build freely

| Need | API / package |
| --- | --- |
| File and folder dialogs | Avalonia `IStorageProvider` |
| Tray icon | Avalonia `TrayIcon` |
| Clipboard | Avalonia `IClipboard` |
| File watching | `FileSystemWatcher` (with the buffer-overflow caveat above) |
| Embedded store | `Microsoft.Data.Sqlite` |
| Logging | `Microsoft.Extensions.Logging` / Serilog |
| Archives | `System.IO.Compression` |
| JSON | `System.Text.Json`, source-generated |
| HTTP | `HttpClient` |

Nothing here needs a feasibility spike. Pick one and build.

## Output Format

One block per capability assessed:

```
Capability: {name}
Verdict: {Strong | Workable | Gap}
API/approach: {API or package + version; `none` when Verdict is Gap}
Caveat: {the trap, tier, licence, or prerequisite; `none` when Strong}
Escape hatch: {required when Verdict is Gap; omit otherwise}
Blocked for: {stack-specific | universal} - required when Verdict is Gap; omit otherwise. A paid-tier or licence block is stack-specific
```

`Escape hatch:` is mandatory on every `Gap` block. A Gap without one is an unanswered requirement, not a finding.

When the assessed capability sits in the silent-failure set, add one line after `Caveat:`:

```
Prerequisite: {packaging, signing, or build-configuration step that must ship first}
```

A capability outside this register gets no verdict from memory: run the NuGet or platform-docs check first and cite the result in `Caveat:`; until the check has run, name the check as the next step instead of emitting a block. A capability the register names without a specific API takes its documented verdict now, with the package or docs check named inside the block as the next step.

Close with one line so the caller knows the sweep ran:

```
Blocking gaps: {count} | Silent-failure prerequisites: {count}
```

A defect owned by a sibling named in the ownership blockquote is written after the blocks as `Deferred: {defect} -> {owning skill}`, one per line. Omit when there are none.

## Avoid

- Estimating a feature whose verdict is `Gap` instead of estimating its escape hatch
- Describing a universal platform block as a .NET shortcoming - or attempting the `UserChoice` write at all
- Designing around a gap that is not there: assuming screen readers, drag-out, or Windows OCR are missing
- Testing notifications, Keychain, or credential storage from `dotnet run` instead of an installed, signed build
- Adopting a reflection-dependent library in an AOT-published app without checking its trimming warnings
- Budgeting the Avalonia Community tier for a commercial project
- Loading managed code into Explorer's process for a shell extension
- Reporting a verdict with no version, date, or documented behaviour behind it
- Scheduling packaging and signing after the features that depend on them
