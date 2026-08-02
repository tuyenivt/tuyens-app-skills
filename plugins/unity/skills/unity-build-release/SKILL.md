---
name: unity-build-release
description: "Ship Unity builds safely: IL2CPP stripping reflection hazards, app bundles, iOS signing, WebGL hosting, Addressables catalogs, batch-mode CI."
metadata:
  category: mobile
  tags: [unity, build, release, il2cpp, code-stripping, addressables, android-app-bundle, ios-signing, webgl, ci, build-size]
user-invocable: false
---

# Unity Build Release

> Confirm the project's target platforms first - they decide which of the packaging, signing, and hosting guidance applies. This skill owns **the build and its shipped artifact**. Build-time configuration (what ships, how it is stripped, compressed, and packaged) is this skill; runtime budget (what it costs once running) is `unity-performance`. A texture that is too large ships here and costs there - report the import/packaging half, defer the memory-budget half. Runtime frame and memory cost belong to `unity-performance`; save durability to `unity-save-persistence`; secrets and tamper exposure to `unity-security-patterns`.

## When to Use

- Setting up build configuration, signing, or CI for a new project
- A bug appears only in a device build and never in the editor
- Preparing a store submission, or shipping content without a store release
- Cutting download size, or diagnosing where the size went
- Adding WebGL as a target

## Rules

- **Anything reflection-based must be preserved against managed code stripping.** IL2CPP strips code the linker cannot see referenced, and reflection is invisible to it. This is the single most common ship-breaking Unity bug - it never reproduces in the editor
- **Test every release-configuration change on a release build on a device.** The editor and a development build both differ from the shipped artifact in scripting backend, stripping, and optimization
- **Performance numbers come from a non-development build.** Development builds carry profiler instrumentation and disable optimizations; a frame time measured there is not the shipped frame time (`unity-performance`)
- Android ships as an **app bundle** to Play. A universal APK ships every architecture and locale to every device
- Version code and version string increment on every submitted build, and are set by CI, not by hand
- Signing credentials, keystores, and store API keys live in CI secrets. Never in the repo, never in `ProjectSettings`
- CI builds run headless, non-interactive, and fail loudly: `-batchmode -nographics -quit` with a non-zero exit on error, and licence activated and released around the build
- A remote content catalog and the build that loads it are a **matched pair**. Shipping one without the other breaks live players

## Patterns

### Build targets and build profiles

Build Profiles (which replaced the Build Settings window in the Unity 6 line, and are the shipped surface on the 6.3 LTS floor) hold per-target scene lists, scripting defines, and player-setting overrides as assets in the project. That makes configuration reviewable in a diff instead of living in one developer's editor state.

Keep at minimum a development profile (development build, profiler, deep-ish diagnostics) and a release profile (stripping on, no development flag, store signing) per target. The divergence between them is exactly where release-only bugs live, so the release profile must be built and smoke-tested regularly, not first on submission day.

CI selects the active profile with `-activeBuildProfile <asset path>`. Note this conflicts with passing `-buildTarget` in the same invocation - some CI templates still pass `-buildTarget` by default, and the combination errors. Confirm which your CI tooling supplies before wiring both.

### IL2CPP, stripping, and the reflection hazard

IL2CPP converts IL to C++ and is required on iOS, Android 64-bit, and WebGL. Managed code stripping runs with it and removes types and members the linker cannot prove are used.

| Stripping level | Behaviour |
| --- | --- |
| Disabled | nothing stripped; largest build |
| Minimal | strips the least; safest starting point |
| Low / Medium | progressively more assemblies searched for unreachable code |
| High | most aggressive; most likely to remove a reflection target |

The failure mode is specific and reliably confusing: **reflection-based JSON deserialization silently loses fields, or throws a missing-method/missing-type exception, only in the release build.** The type exists in the editor and in a Mono build; the linker saw no static reference to it and removed it.

```csharp
// Bad - fields populated by a reflection-based serializer, nothing statically references them
public class SaveData { public int coins; public int[] board; }

// Good - the type and its members survive stripping
[Preserve] public class SaveData { public int coins; public int[] board; }
```

The two preservation mechanisms:

- `[UnityEngine.Scripting.Preserve]` on the type or member - precise, lives next to the code it protects
- `link.xml` - preserves whole assemblies, namespaces, or types by name. The file lives anywhere under `Assets/` and every `link.xml` under `Assets/` merges automatically, so a package's own file can sit beside yours. `preserve="all"` on an assembly is the blunt fix; scope it down once the build is green

```xml
<linker>
  <assembly fullname="Game.Save" preserve="all"/>
</linker>
```

What needs preserving: save/DTO types deserialized by reflection, types resolved by `Type.GetType` or by name from data, enum values parsed from strings, generic instantiations only ever created via reflection (AOT has no runtime JIT to create them), and third-party SDK entry points invoked by native code.

`[Preserve]` and `link.xml` stop an *existing* instantiation being removed; neither can create one. Under AOT a generic instantiation that is only ever built via reflection - `Dictionary<string, ItemState>` constructed by a JSON deserializer, say - was never compiled, so there is nothing to preserve. It needs a real static reference somewhere in shipped code (a field, or a dummy method that names the closed type) for the compiler to emit it at all.

Raising the stripping level is a build-size lever with a correctness cost. Raise it one step at a time and **re-run a full smoke test of save/load, IAP, ads, and analytics on a device build after each step** - these are exactly the reflection-heavy surfaces.

### Android packaging

| Artifact | Use | Note |
| --- | --- | --- |
| App Bundle (`.aab`) | Play store submission | required by Play; the store generates per-device APKs, cutting download size |
| APK | direct distribution, testing, other stores | a universal APK carries every ABI and every density |
| Split APKs by ABI | direct distribution where a bundle is unavailable | one artifact per architecture |

- **Target API level** is set by store policy and moves annually; Play rejects submissions below the current floor. Read the current requirement rather than trusting a number written here or in an older project
- **Minimum API level** is a market-reach decision. Lower reaches more devices and drags in older, slower hardware that will fail the performance targets - decide it with the device tier the game is actually built for
- ARM64 is mandatory for Play. Shipping ARMv7 alongside it doubles native code size, so include it only if the min API and target market genuinely need 32-bit
- **Split application binary** moves assets out of the base and is a separate concern from the ABI split
- Keystore in CI secrets; a lost upload keystore is recoverable only through Play's key-reset process, so back it up outside the repo

### iOS signing and provisioning

The Unity build produces an Xcode project; signing happens in the Xcode/`xcodebuild` step, not in Unity. Consequences:

- The bundle identifier, team, and provisioning profile must agree across Unity player settings, the generated Xcode project, and the profile. A mismatch fails at archive time, not at Unity build time
- CI needs the signing certificate and provisioning profile installed into a keychain it unlocks non-interactively; the automation-friendly route is a dedicated CI keychain plus API-key-based upload rather than an Apple ID password
- Post-process build steps (added frameworks, capabilities, plist entries such as usage-description strings and tracking permission) must be scripted, because the Xcode project is regenerated each build. A manual Xcode edit is lost on the next CI run
- Push, IAP, and Sign in with Apple are capabilities that must be enabled on the App ID *and* present in the profile; missing entitlements fail at upload or, worse, at runtime on a real device

### WebGL

- **Compression**: Brotli gives the smallest payload and gzip the broader compatibility. Either requires the **server** to send the matching `Content-Encoding` header. Serving Brotli-compressed files with no header is the single most common WebGL deployment failure - the browser downloads bytes it cannot decode and the loader errors. Some hosts (including itch.io-style hosts) handle this and some do not; decompression fallback exists at a size and startup cost when the host cannot be configured
- **Memory ceiling**: the WebGL heap is a bounded allocation, and exceeding it is an unrecoverable out-of-memory in the tab, not a degraded frame rate. Texture memory dominates - the mobile-oriented import settings in `unity-performance` matter more here, not less. Verify the heap size setting against the project's Unity version, since the configuration surface has changed across releases
- IL2CPP is the only backend; there is no runtime JIT, so reflection-created generic instantiations fail at runtime unless preserved
- Threads are not available by default, and `System.IO` durability does not apply (`unity-save-persistence`)
- Initial download is the whole loader plus data payload before anything renders; Addressables with a remote catalog is how a WebGL build stops shipping all content up front

### Addressables and remote catalogs

Addressables separates *what is in the build* from *what is downloaded at runtime*, which is what makes content updates without a store release possible.

- **Local groups** ship inside the build. **Remote groups** are hosted and downloaded on demand
- The **catalog** maps addresses to bundles. A remote catalog lets an already-installed build discover new or changed content, provided the build was configured to check for a catalog update
- Content update requires the **addressables build state from the shipped build** to be preserved (checked into version control or stored as a CI artifact). Without it, a content update rebuilds bundle identity and installed players break. This is the classic Addressables mistake and it is not recoverable after the fact
- Code cannot be shipped this way. A content update delivers assets and data; anything requiring new C# needs a store release
- Bundle a fallback: a remote fetch that fails must leave the game playable
- Version the remote catalog path per build version so an old build keeps loading the content it was built against, rather than pulling a catalog that references bundles it cannot use

Addressables' build and content-update API has changed across package major versions - check the version in `Packages/manifest.json` before scripting the pipeline.

### CI on batch mode

```bash
# Headless build; -quit is required or the editor stays open and the job hangs
Unity -batchmode -nographics -quit \
  -projectPath "$PROJECT" \
  -executeMethod BuildScript.BuildAndroid \
  -logFile - \
  || exit 1
```

- `-nographics` skips GPU initialization on a headless agent. It is not appropriate where the build step needs graphics (some asset processing), so a failure here is a signal to check, not to remove the flag reflexively
- `-logFile -` streams to stdout so CI captures it. A build that writes only to the default log file leaves you with no failure evidence
- **Batch mode does not fail the job by itself** - a thrown exception inside `-executeMethod` can still exit zero. The build method must catch, log, and call an explicit failure exit, and the job must check the build report result rather than trusting the process exit code alone
- **Licence activation** runs before the build and **return** runs after, including on failure - an unreleased seat leaks and blocks the next run. Credentials come from CI secrets
- Domain reload and asset import on a cold CI agent dominate build time; a persisted `Library` cache between runs is the main lever
- Determinism: pin the Unity version to the exact internal version from `ProjectSettings/ProjectVersion.txt` (for example `6000.3.x`) rather than a floating major

### Build size analysis

The Editor log written by the build contains a build report: size by asset type and a list of the largest individual assets. Read it rather than guessing.

Typical order of payoff for a casual 2D game: oversized source textures and import max-size settings (usually the largest single line), duplicated assets pulled into multiple bundles by overlapping dependencies, uncompressed or unnecessarily long audio, fonts carrying full glyph coverage for scripts the game does not ship, and unused packages left in `manifest.json`. Stripping level and ABI/architecture choices are code-side levers and are usually smaller than the asset side in these genres.

The number to optimize is the **store download size**, not the artifact on disk - Play and the App Store recompress and, for a bundle, deliver a per-device subset. Compare against the store-reported size after upload.

### Development versus release builds

| Aspect | Development build | Release build |
| --- | --- | --- |
| Profiler / deep profiling | available | not available |
| Script debugging | attachable | off |
| Managed code stripping | often relaxed | at the configured level - where reflection breaks |
| Optimization | reduced | full |
| `Debug.Log` output | visible on device | still executes unless compiled out |
| Frame timings | instrumented, slower | the number that ships |

Use development builds to locate cost and to reproduce logic bugs; confirm both the final frame numbers and every reflection-dependent surface in a release build. A release-only bug that "makes no sense" is stripping until proven otherwise.

## Output Format

Two modes, chosen by whether the request supplies code to judge or asks for code to be produced.

**Authoring mode** - the request is to write or design something. Emit the code or design, then any `Deferred:` lines. No finding blocks, no severity, no status line: nothing was reviewed, so a not-run line would misdescribe the work.

**Review mode** - source, a diff, or a symptom report was supplied. Emit one block per finding.

```
### [Severity] {file:line | symbol or type.member, when source was supplied without paths | setting | asset path | symptom, when no source was supplied}

- Category: {Stripping | ScriptingBackend | AndroidPackaging | iOSSigning | WebGLHosting | Addressables | CIPipeline | BuildSize | ProfileConfig | Secrets}
- Evidence: {source (name the file, setting path, or CI step cited) | inferred (state what was not seen)}
- Platform: {Android | iOS | WebGL | Desktop | All}
- Impact: {shipped consequence - "release build fails to deserialize saves", "content update breaks installed players", "universal APK ships every ABI"}
- Fix: {concrete change}
```

`Severity: {Critical | High | Medium | Low}` - Critical = the shipped artifact is broken or unshippable, or a change breaks already-installed players (stripped reflection target, discarded addressables build state, signing credential in the repo). High = a store rejection, a build that cannot be reproduced by CI, or a size/config defect on the primary target. Medium = a maintainability or size gap with a known workaround. Low = a hygiene issue with no shipped consequence.

Severity that does not fit a listed band: assign the nearest lower band and state why in `Impact`. `Category` takes exactly one value - where a defect fits two, pick the one the `Fix` addresses and name the other in `Impact`; where it fits none, pick the closest and name the real concern in `Impact`.

`Evidence: inferred` is required whenever the source was not read. It bounds the header at High: a Critical-band defect is written High, and `Impact` names the uncapped band. It never raises a block - a Medium defect stays Medium. Among blocks sharing a band, order by what the reader must fix first: root cause before the symptoms it produces.

A defect owned by a sibling named in the ownership blockquote is not emitted as a finding. Write those after the findings, one per line, as `Deferred: {defect} -> {owning skill}`, so the workflow routes rather than drops them. In authoring mode the same line routes a design decision the sibling owns (`Deferred: memory budget of remote content -> unity-performance`). Omit entirely when there are none.

In review mode, close with exactly one status line, after any `Deferred:` lines:

| Condition | Line |
| --- | --- |
| One or more findings emitted | none - the findings are the output |
| No findings, and a symptom or report was available to reason from | `No build findings.` |
| No source, diff, symptom, or report of any kind was supplied | `Build check not run: no source supplied.` |

A symptom-only report (a QA ticket, a CI log, a verbal description) is checkable input: emit `Evidence: inferred` findings from it rather than the not-run line.

## Avoid

- Reflection-based serialization with no `[Preserve]` or `link.xml` coverage
- Raising the stripping level without re-smoke-testing save, IAP, ads, and analytics on a device build
- Testing only in the editor or only in a development build before submission
- Reporting a frame time from a development build as the shipped number
- A universal APK where an app bundle is available
- Manual Xcode project edits that the next build regenerates away
- Serving Brotli or gzip WebGL builds without the matching server `Content-Encoding` header
- Discarding the addressables build state from a shipped build
- Shipping a remote catalog update that the installed build cannot load
- `-batchmode` without `-quit`, or trusting the process exit code instead of the build report result
- Licence activation with no matching return on the failure path
- Keystores, certificates, or store API keys committed to the repo
- Hand-incremented version codes
- Guessing which asset is large instead of reading the build report
- Asserting a store API level requirement or an Addressables API from memory instead of checking the current policy and package version
