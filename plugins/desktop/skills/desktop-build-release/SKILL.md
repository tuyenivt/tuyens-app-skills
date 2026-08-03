---
name: desktop-build-release
description: Ship signed Rust desktop builds - release profile, panic strategy, cargo-packager, Windows and macOS signing, CI, per-user install, auto-update.
metadata:
  category: desktop
  tags: [rust, release-profile, lto, panic-abort, cargo-packager, code-signing, notarization, rcodesign, github-actions, aws-lc-rs, velopack, self-replace, auto-update, windows, macos]
user-invocable: false
---

# Desktop Build and Release

> **Distribution model is settled: closed-source commercial, sold as a perpetual license with a 12-month update window, alongside a free tier.** Buying a version owns it forever; updates are included for a year, after which the app keeps working and a new major is a discounted upgrade. Three consequences shape everything below - a private repository means macOS CI bills at the 10x multiplier rather than being free; signing is mandatory rather than optional, because a paid product carrying a SmartScreen or Gatekeeper warning loses the sale at the download; and licence enforcement is a local check with no server behind it, which bounds what enforcement can achieve.
>
> Confirm which platforms actually ship before applying this skill - macOS adds a paid membership, a signing identity, and a CI runner that Windows-only shipping does not need.
>
> This skill owns **turning a working binary into something a user can install and update**. Update signature verification as a threat belongs to `desktop-security-patterns`; the OS identity that packaging unlocks (AUMID, bundle id, keychain access) to `desktop-platform-integration`; runtime performance measurement to `desktop-performance`.

## When to Use

- Setting or changing the release profile
- Producing an installer or a distributable bundle
- Setting up signing, notarization, or CI for either platform
- Adding or changing auto-update
- A build that succeeds locally and fails in CI, or a binary that behaves differently in release

## Rules

- **`panic = "abort"` removes unwinding.** Any `catch_unwind` recovery path silently becomes a process termination in release while continuing to work in debug. Choose the strategy against whether the app recovers from panics, not against binary size alone
- **Signing is a prerequisite for features, not a release-day step.** Notifications, keychain access, and a non-hostile first-run experience all depend on it (`desktop-platform-integration`)
- **A GUI Rust binary cannot be cross-compiled to macOS from Windows or Linux.** Use CI with a macOS runner. Do not design a pipeline around a local cross-build
- **Signed does not mean working.** Signing and notarization prove provenance, not that the app launches. A build that has never run on real target hardware is unverified
- Install per-user, not machine-wide. `%LOCALAPPDATA%` on Windows avoids UAC on install and is what makes silent auto-update possible at all
- **Exactly one rustls crypto provider is linked.** Two active providers is a runtime panic, not a compile error
- The release pipeline is reproducible from a clean checkout in CI. A step that only exists on the maintainer's machine is a bus factor of one
- **The free/paid split is on scale, never on safety.** Dry-run preview and undo are free-tier features in every build. A free tier that can destroy files but cannot undo them is a data-loss tool wearing an evaluation label
- **An expired update window never disables the app.** The purchased version keeps working indefinitely; only access to newer builds lapses. An app that stops functioning when a licence date passes is a subscription, whatever the storefront called it

## Patterns

### Release profile

```toml
[profile.release]
lto = "fat"           # cross-crate inlining; the largest single size and speed win
codegen-units = 1     # slower build, better optimization
strip = "symbols"     # smaller binary; keep "debuginfo" if you symbolicate crashes
panic = "abort"       # see below - this is a behaviour change, not a size knob
opt-level = 3         # "s"/"z" only if measurement says size matters more than speed
```

`panic = "abort"` is the one entry with a correctness consequence:

```rust
// Bad - a decoder guard that works in debug and terminates the whole app in release
let result = panic::catch_unwind(|| decode_untrusted(bytes));   // never returns Err under abort

// Good - either keep unwinding, or make the boundary a process boundary
// profile.release: panic = "unwind"   (keep catch_unwind working)
// or: run the untrusted decode in a child process and read its exit status
```

Also relevant: a panic in a Rayon or `tokio` worker under `unwind` poisons or propagates rather than killing the app; under `abort` it takes the process with it, including any unsaved journal (`desktop-batch-operations`). Decide which the app wants and write the reason next to the setting.

Keep a `[profile.release-debug]` inheriting release with `debug = true` for profiling - measuring a debug build measures nothing (`desktop-performance`).

### Packaging

Use **`cargo-packager`**: it produces MSI and NSIS on Windows and `.app`/`.dmg` on macOS, and it drives signing. **`cargo-bundle` is self-declared alpha** - do not build a release pipeline on it.

Packaging is what creates the artefacts other features depend on: the Start Menu shortcut carrying the AUMID that Windows toasts require, and the `.app` bundle with a bundle identifier that macOS notifications and the keychain require.

### Windows signing

Unsigned Windows binaries get a SmartScreen warning that most users read as malware.

| Route | Cost | Constraint |
| --- | --- | --- |
| Azure Trusted Signing, individual | ~$120/yr | **US and Canada only** for individual identity validation |
| Traditional OV certificate | several hundred $/yr | Hardware token or cloud HSM; reputation builds over time |
| EV certificate | higher | Immediate SmartScreen reputation |
| Unsigned | $0 | SmartScreen warning on every download; not shippable to non-technical users |

A solo maintainer outside US/Canada therefore faces the several-hundred-per-year traditional route, not the cheap one. Establish this before promising a shipping date.

**A paid product cannot ship unsigned.** A download from a web page is precisely the distribution shape SmartScreen and Gatekeeper warn about, and there is no app-store reputation standing behind it. Against zero revenue the certificate is a painful fixed cost; against even modest sales it is trivial - so the certificate is budgeted from the first release rather than deferred. Deferring it is the expensive order: shipping unsigned first builds a warning reputation that later signing has to undo.

### macOS signing and notarization without a Mac

Signing and notarizing macOS artefacts is possible from Windows or Linux, because Apple's Notary API is plain REST and `rcodesign` (from `indygreg/apple-platform-rs`) reimplements the signing format.

What is still unavoidable:

- **Apple Developer Program membership, $99/yr.** There is no free path to a Developer ID certificate
- **A Team App Store Connect API key.** Personal keys are rejected by the Notary API. This is a common first failure and reads as an authentication error
- **Notarization is a service round-trip**, so it needs network and takes minutes; staple the ticket to the artefact afterwards so Gatekeeper works offline

What is **not** possible is building the app there. `osxcross` requires the macOS SDK, whose Xcode SLA restricts use to Apple hardware, and the hard part is not the compiler but linking AppKit, Metal, and the frameworks an Iced/wgpu binary needs. Treat a local macOS cross-build as unavailable and use CI.

### CI

GitHub Actions macOS runners are the practical build host. Cost model: **free and unmetered for public repositories; a 10x minute multiplier on private ones.** A closed-source project is a private repository, so the multiplier applies and the free tier does not.

Budget accordingly rather than discovering it: a 20-minute macOS job costs 200 billed minutes, which is roughly ten runs per month on the Free plan and fifteen on Pro. Two practices keep this affordable and both belong in the workflow file rather than in a habit:

- **macOS runs on tag or on merge to the default branch, not on every push.** Windows is primary and runs on every push; macOS is the expensive one and does not need to.
- **Lint and test on Linux where the code is platform-neutral.** Only the macOS-specific build, signing, and notarization need the macOS runner - and per the section above, `rcodesign` can sign and notarize from Linux, so even that is partly avoidable.

Store the signing key, certificate, and API key as encrypted secrets; never in the repo. The pipeline builds on each platform's native runner, signs, notarizes on macOS, and uploads artefacts.

### `aws-lc-rs`: the highest-probability build failure

Any dependency pulling `rustls` with default features pulls `aws-lc-rs`, which needs a C toolchain, **CMake, and NASM on Windows**. It appears through `reqwest`, `ureq`, and most update crates - transitively, with nothing in `Cargo.toml` naming it.

```
# Diagnose first - do not guess which dependency pulled it in
cargo tree -i aws-lc-rs
```

Two fixes, in order of preference: switch the dependency to its `rustls-ring` / `ring` feature so the pure-Rust backend is used, or install CMake and NASM on the runner. **Never end up with both `ring` and `aws-lc-rs` providers active** - `rustls` panics at runtime when no single default provider can be resolved, which surfaces as a crash on the first TLS connection in a build that compiled cleanly.

### Install location and auto-update

```
# Bad - Program Files: UAC prompt on install and on every self-update
C:\Program Files\MyApp\myapp.exe

# Good - per-user: no elevation, silent update possible
%LOCALAPPDATA%\MyApp\myapp.exe
```

A machine-wide install cannot silently self-update, because writing to `Program Files` requires elevation the running app does not have. Per-user install is what makes the update story work at all.

For the update itself: **`velopack`** (delta updates, both platforms, handles the installer side) or **`self_update`** (simpler, GitHub Releases-shaped). Either way the executable swap goes through **`self-replace`** - a running executable cannot overwrite itself directly on Windows, and hand-rolling the rename dance is where update systems corrupt installs.

The update must verify a signature against a key compiled into the binary before executing anything it downloaded (`desktop-security-patterns`). Version comparison refuses downgrades.

### Licence enforcement and the update window

The update window is enforced by comparing a build's release date against the licence's expiry, and it gates **which builds install**, never whether the installed app runs.

```rust
// Bad - the app stops working when the window closes; this is a subscription
if licence.expires_at < now { return Err(Licence::Expired); }   // blocks startup

// Good - the purchased version runs forever; only newer builds are gated
fn update_allowed(build_date: Date, licence: &Licence) -> bool {
    build_date <= licence.updates_until
}
```

Three properties keep this honest and keep it cheap for a solo maintainer:

- **Offline verification.** A signed licence file or key, checked against a public key in the binary. There is no backend (`desktop-core-architecture`), so a phone-home check would introduce the one network dependency the architecture avoids - and would break the app when the server eventually goes away.
- **Enforcement is a speed bump, not a wall.** A local check in a closed-source binary raises cost; it cannot prevent a determined bypass. Treat it as `cost-raising only` in the sense `desktop-security-patterns` uses, and do not spend engineering on hardening that a paying user never sees.
- **Grace over lockout.** A missing or unreadable licence file falls back to the free tier rather than refusing to start. A paying user whose disk lost a file gets a working app, not a support ticket.

The free/paid split is a capability check in the core crate, not a separate build. Two binaries doubles the release matrix, and the one the user downloads is then the wrong one half the time.

### The QA gap, stated plainly

Signing and notarizing from CI proves the artefact is well-formed and trusted. It does not prove the app launches, renders, finds its fonts, or survives a Gatekeeper first-run on real hardware. **A maintainer with no Mac has an open QA gap on macOS**, and the honest mitigations are a CI smoke test that launches the bundle headlessly, a beta tester with the hardware, or shipping macOS as explicitly experimental. Do not close this gap on paper.

## Output Format

Two modes, chosen by whether the request supplies configuration to judge or asks for a pipeline to be produced.

**Authoring mode** - the request is to write or design a profile, pipeline, or packaging setup. Emit the configuration or design, then any `Deferred:` lines. No finding blocks, no severity, no status line.

A pipeline or packaging design opens with this line, because every cost below follows from it:

```
Distribution: closed-source commercial, perpetual licence + 12-month update window, free tier - private repo (macOS CI at 10x), signing mandatory, offline licence check
```

It is a fixed value, not a question. A design departing from it states why on the same line.

When the design targets a platform the maintainer cannot run, it carries one `QA gap:` line naming the unverified platform and the chosen mitigation - a headless CI smoke test, a beta tester with the hardware, or shipping that platform as experimental.

**Review mode** - a manifest, workflow file, or build symptom was supplied. Emit one block per finding.

```
### [Severity] {file:line | symbol, when source was supplied without paths | symptom, when no source was supplied}

- Category: {ReleaseProfile | PanicStrategy | Packaging | WindowsSigning | MacSigning | Notarization | CrossCompile | CIConfig | DependencyToolchain | CryptoProvider | InstallLocation | AutoUpdate | Licensing | ReleaseQA}
- Platform: {Windows | macOS | both | CI}
- Evidence: {source | inferred (state what was not seen)}
- Code: {one-line citation, or `not supplied` when the finding is inferred}
- Failure: {where it breaks - "compiles clean, panics on the first TLS connection"}
- Cost: {the money, time, or hardware the fix requires, or `none`}
- Fix: {the concrete change}
```

`Severity: {Critical | High | Medium | Low}` - Critical = the shipped artefact is unsafe or unusable (unverified update, two rustls providers linked, `panic = "abort"` under a live `catch_unwind` recovery path). High = the release cannot be produced or installed as designed (macOS cross-build assumed, unsigned Windows binary shipped to end users, machine-wide install with silent auto-update promised). Medium = a build that fails in CI for a fixable toolchain reason, or a profile setting working against the app's goal. Low = a size or build-time optimization with no behaviour change.

Severity that does not fit a listed band: assign the nearest lower band and state why in `Failure`. `Category` takes exactly one value - where a defect fits two, pick the one the `Fix` addresses and name the other in `Failure`.

`Evidence: inferred` is required whenever the source was not read. It bounds the header at High and never raises a block.

When a fix requires a paid membership, a certificate, or hardware the maintainer may not have, `Cost` names it explicitly rather than presenting the fix as free.

A defect owned by a sibling named in the ownership blockquote is written after the findings as `Deferred: {defect} -> {owning skill}`, one per line. Omit when there are none.

In review mode, close with exactly one status line, after any `Deferred:` lines:

| Condition | Line |
| --- | --- |
| One or more findings emitted | none - the findings are the output |
| Source or a symptom report was available, nothing found | `No build or release findings.` |
| No source, diff, or symptom supplied | `Build and release check not run: no source supplied.` |

## Avoid

- Setting `panic = "abort"` while a `catch_unwind` recovery path exists
- Treating `panic = "abort"` as a size knob with no behaviour consequence
- Profiling or benchmarking a debug build
- Building a release pipeline on `cargo-bundle`
- Planning a local macOS cross-build with `osxcross` for a GUI binary
- Assuming Azure individual signing is available outside the US and Canada
- Submitting a Personal App Store Connect API key to the Notary API
- Notarizing without stapling the ticket
- Shipping an unsigned Windows binary to non-technical users
- Ignoring the 10x macOS runner multiplier when the repo is private
- Assuming free macOS CI runners, which apply to public repositories and not to a closed-source project
- Running the macOS job on every push rather than on tag or default-branch merge
- Deferring the signing certificate past the first release, which builds a warning reputation later signing must undo
- Blocking startup on an expired update window - the purchased version runs forever; only newer builds are gated
- Putting dry-run preview or undo behind the paywall, which makes the free tier a data-loss tool
- A licence check that phones home, introducing the one network dependency this architecture avoids
- Refusing to start on a missing or unreadable licence file instead of falling back to the free tier
- Shipping separate free and paid binaries rather than one binary with a capability check
- Guessing which dependency pulled in `aws-lc-rs` instead of running `cargo tree -i`
- Linking both `ring` and `aws-lc-rs` rustls providers
- Installing to `Program Files` while promising silent auto-update
- Swapping a running executable without `self-replace`
- Applying an update without verifying a signature against an embedded key
- Committing signing keys, certificates, or API keys to the repo
- Claiming macOS is verified when nothing has run on a real Mac
