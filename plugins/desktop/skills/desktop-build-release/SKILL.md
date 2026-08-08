---
name: desktop-build-release
description: Ship signed .NET/Avalonia builds - publish modes and NativeAOT, Velopack installers, Windows/macOS signing, CI cost, offline licence gating.
metadata:
  category: desktop
  tags: [csharp, dotnet, nativeaot, trimming, publish, velopack, code-signing, notarization, rcodesign, github-actions, auto-update, licensing, windows, macos]
user-invocable: false
---

# Desktop Build and Release

> **Distribution model is settled: closed-source commercial, sold as a perpetual licence with a 12-month update window, alongside a free tier.** Buying a version owns it forever; updates are included for a year, after which the app keeps working and a new major is a discounted upgrade. Three consequences shape everything below - a private repository means macOS CI bills at the 10x multiplier rather than being free; signing is mandatory rather than optional, because a paid product carrying a SmartScreen or Gatekeeper warning loses the sale at the download; and licence enforcement is a local check with no server behind it, which bounds what enforcement can achieve.
>
> This skill owns **turning a working build into something a user can install and update**. Update signature verification as a threat belongs to `desktop-security-patterns`; the OS identity that packaging unlocks (AUMID, bundle id, keychain access) to `desktop-platform-integration`; runtime performance measurement to `desktop-performance`.

## When to Use

- Choosing or changing the publish mode, or enabling trimming or NativeAOT
- Producing an installer or a distributable bundle
- Setting up signing, notarization, or CI for either platform
- Adding or changing auto-update, or wiring the licence check
- A build that succeeds locally and fails published, or behaves differently under AOT

## Rules

- **The publish mode is a behaviour decision, not a size knob.** NativeAOT removes runtime reflection: compiled bindings and source generators are mandatory, and trimming warnings are the early signal - a suppressed warning is a runtime failure deferred to a customer machine
- **NativeAOT cannot cross-compile across operating systems.** macOS artefacts need a macOS runner regardless; do not design a pipeline around a local cross-build
- **Signing is mandatory and budgeted from the first release.** Deferring the certificate builds a SmartScreen warning reputation that later signing must undo - the expensive order
- Install per-user to `%LOCALAPPDATA%`, never `Program Files` - avoids UAC on install and is what makes silent auto-update possible at all
- **The free/paid split is on scale, never on safety.** Dry-run preview and undo are free-tier features in every build. A free tier that can destroy files but cannot undo them is a data-loss tool wearing an evaluation label
- **An expired update window gates which builds install, never whether the installed app runs.** An app that stops working when a licence date passes is a subscription, whatever the storefront called it
- One binary with a capability check, not separate free and paid builds
- The updater executes nothing it downloaded before verifying a signature against a key embedded in the binary, and refuses downgrades
- **Signed does not mean working.** A platform never smoke-tested on real hardware is unverified; state the gap rather than closing it on paper
- The release pipeline is reproducible from a clean checkout in CI. A step that only exists on the maintainer's machine is a bus factor of one

## Patterns

### Publish modes

| Mode | Startup | Runtime dependency | Constraint |
| --- | --- | --- | --- |
| Framework-dependent | JIT | Requires .NET 10 installed | Wrong for consumer desktop - the install step becomes "first install .NET" |
| Self-contained | ~1960ms (JIT) | None | Large artefact; JIT startup cost |
| NativeAOT (`PublishAot`) | ~460ms | None | No runtime reflection; compiled bindings and source generators mandatory |

Set the posture on day one, not at release time - retrofitting AOT onto a reflection-bound codebase is a rewrite:

```xml
<PropertyGroup>
  <PublishAot>true</PublishAot>
  <AvaloniaUseCompiledBindingsByDefault>true</AvaloniaUseCompiledBindingsByDefault>
  <TrimmerSingleWarn>false</TrimmerSingleWarn>  <!-- full per-warning detail; each one is a future runtime failure -->
</PropertyGroup>
```

Publish with trimming in CI from the first commit even while the shipping decision is open: `PublishTrimmed` warnings name exactly the dependencies that will break under AOT, while there is still time to swap them.

### Windows signing

Unsigned Windows binaries get a SmartScreen warning most users read as malware.

| Route | Cost | Constraint |
| --- | --- | --- |
| Azure Trusted Signing, individual | ~$120/yr | **US and Canada only** for individual identity validation |
| Traditional OV certificate | Several hundred $/yr | Hardware token or cloud HSM; reputation builds over time |
| EV certificate | Higher | Immediate SmartScreen reputation |
| Unsigned | $0 | Not shippable to paying users |

A solo maintainer outside US/Canada faces the several-hundred-per-year traditional route, not the cheap one. Establish this before promising a shipping date. Against zero revenue the certificate is a painful fixed cost; against even modest sales it is trivial - so it is budgeted from the first release, not deferred.

### macOS signing and notarization without a Mac

Signing and notarizing macOS artefacts works from Windows or Linux: Apple's Notary API is plain REST, and `rcodesign` reimplements the signing format. What is still unavoidable:

- **Apple Developer Program membership, $99/yr.** There is no free path to a Developer ID certificate
- **A Team App Store Connect API key.** Personal keys are rejected by the Notary API - a common first failure that reads as an authentication error
- **Notarization is a service round-trip**; staple the ticket afterwards so Gatekeeper works offline

What is not possible without a Mac is *building*: NativeAOT does not cross-compile across OSes, so the macOS build itself runs on a macOS CI runner.

### CI cost on a private repository

GitHub Actions macOS runners are free only for public repositories; **a private repo bills macOS at the 10x minute multiplier**, so a 20-minute macOS job costs 200 billed minutes. Two practices belong in the workflow file, not in a habit:

- **macOS runs on tag or on merge to the default branch, not on every push.** Windows is primary and runs on every push
- **Lint and test on Linux** where the code is platform-neutral; only build, signing, and notarization need the expensive runners

Signing keys, certificates, and API keys live in encrypted secrets, never in the repo.

### Installers and install location

**Velopack** is the default: .NET-native, delta updates, both platforms, and it handles the self-update executable swap. WiX/MSI or Inno Setup fit when corporate deployment conventions demand them. Avalonia's own Parcel packager is a Pro-tier product - budget it or do not plan on it (`desktop-ecosystem-boundaries` owns the tier register).

```
# Bad - Program Files: UAC prompt on install and on every self-update
C:\Program Files\MyApp\MyApp.exe

# Good - per-user: no elevation, silent update possible
%LOCALAPPDATA%\MyApp\MyApp.exe
```

On macOS the `.app` drag-installs to `/Applications` or `~/Applications`; both are user-writable, so the same silent-update property holds. Packaging is also what creates the identity other features depend on: the AUMID-bearing shortcut Windows toasts require, the bundle id macOS notifications and Keychain require (`desktop-platform-integration`).

### Licence enforcement and the update window

Offline verification: a signed licence file checked against a public key embedded in the binary. There is no backend, so a phone-home check would introduce the one network dependency this architecture avoids - and would break the app when the server eventually goes away.

```csharp
// Bad - the app stops working when the window closes; this is a subscription
if (licence.UpdatesUntil < DateOnly.FromDateTime(DateTime.UtcNow))
    throw new LicenceExpiredException();   // blocks startup

// Good - the purchased version runs forever; only newer builds are gated at install time
public static bool UpdateAllowed(DateOnly buildDate, Licence licence)
    => buildDate <= licence.UpdatesUntil;
```

- **Grace over lockout.** A missing or unreadable licence file falls back to the free tier rather than refusing to start. A paying user whose disk lost a file gets a working app, not a support ticket
- **Enforcement is a speed bump, not a wall.** A local check in a closed-source binary raises cost; it cannot stop a determined bypass. Do not spend engineering on hardening a paying user never sees
- The free/paid split is a capability check in the core project. Two binaries doubles the release matrix, and the one the user downloads is then wrong half the time

### The QA gap, stated plainly

Signing and notarizing from CI proves the artefact is well-formed and trusted, not that the app launches, renders, or survives a Gatekeeper first-run. **A maintainer with no Mac has an open QA gap on macOS.** Honest mitigations: a CI smoke test that launches the bundle, a beta tester with the hardware, or shipping macOS as explicitly experimental. Require the gap to be stated; never close it on paper.

## Output Format

Two modes, chosen by whether the request supplies configuration to judge or asks for a pipeline to be produced.

**Authoring mode** - the request is to write or design a publish setup, pipeline, or packaging. Emit the configuration or design, then any `Deferred:` lines. No finding blocks, no severity, no status line. The design opens with this line, because every cost below follows from it:

```
Distribution: closed-source commercial, perpetual licence + 12-month update window, free tier - private repo (macOS CI at 10x), signing mandatory, offline licence check
```

It is a fixed value, not a question; a design departing from it states why on the same line. When the design targets a platform the maintainer cannot run, it carries one `QA gap:` line naming the unverified platform and the chosen mitigation, placed after the design body and before any `Deferred:` lines.

**Review mode** - a csproj, workflow file, or build symptom was supplied. One block per finding, ordered by severity, Critical first:

```
### [Severity] {file:line | symbol, when source was supplied without paths | symptom, when no source was supplied}

- Category: {PublishMode | Trimming | Packaging | WindowsSigning | MacSigning | Notarization | CrossCompile | CIConfig | InstallLocation | AutoUpdate | Licensing | ReleaseQA}
- Platform: {Windows | macOS | both | CI} - `CI` is for workflow-configuration defects; a build failing on a platform's runner takes that platform; a symptom whose platform is unknown takes `both`
- Evidence: {source (a setup described in the request counts; quote the description in Code:) | incident (the reported event already demonstrates the failure) | inferred (state what was not seen)}
- Code: {one-line citation, or `not supplied` when no source was read}
- Failure: {where it breaks - "publishes clean, throws on first deserialize under AOT"}
- Cost: {the money, time, or hardware the fix requires, or `none`}
- Fix: {the concrete change}
```

`Severity: {Critical | High | Medium | Low}` - Critical = the shipped artefact is unsafe or unusable (update applied without signature verification, licence check blocking startup on expiry, AOT binary shipped with unresolved trim warnings on a shipping code path). High = the release cannot be produced or installed as designed (macOS cross-build assumed, unsigned binary shipped to paying users, machine-wide install with silent auto-update promised). Medium = a CI or publish setting that fails for a fixable reason or works against the app's goal. Low = a size or build-time optimization with no behaviour change.

The band definitions govern; parenthesized items are examples. A defect fitting no band takes the nearest lower band, with the reason in `Failure`. `Evidence: inferred` caps severity at High and never raises a block; when the cap lowers a would-be Critical, say so in `Failure`. `Evidence: incident` carries no cap. When a fix requires a paid membership, certificate, or hardware, `Cost` names it explicitly rather than presenting the fix as free.

A defect - or, in authoring mode, out-of-scope work the deliverable depends on - owned by a sibling named in the ownership blockquote is written after the findings or the design as `Deferred: {item} -> {owning skill}`, one per line. Omit when there are none.

In review mode, close with exactly one status line, after any `Deferred:` lines:

| Condition | Line |
| --- | --- |
| One or more findings emitted | none - the findings are the output |
| Source or a symptom report was available, nothing found | `No build or release findings.` |
| No source, diff, or symptom supplied | `Build and release check not run: no source supplied.` |

## Avoid

- Treating the publish mode as a size decision with no behaviour consequence
- Enabling `PublishAot` at release time on a codebase never published with trimming warnings on
- Suppressing trim warnings instead of replacing the reflection-bound dependency
- Planning a local macOS cross-build - NativeAOT does not cross-compile across OSes
- Assuming Azure individual signing is available outside the US and Canada
- Submitting a Personal App Store Connect API key to the Notary API
- Notarizing without stapling the ticket
- Shipping an unsigned Windows binary to paying users, or deferring the certificate past the first release
- Ignoring the 10x macOS multiplier, or running the macOS job on every push
- Building the pipeline on the Pro-tier Parcel packager with a $0 tooling budget
- Installing to `Program Files` while promising silent auto-update
- Blocking startup on an expired update window, or refusing to start on a missing licence file
- Putting dry-run preview or undo behind the paywall
- Shipping separate free and paid binaries rather than one binary with a capability check
- Applying an update without verifying a signature against an embedded key, or accepting downgrades
- Committing signing keys, certificates, or API keys to the repo
- Claiming macOS is verified when nothing has run on a real Mac
