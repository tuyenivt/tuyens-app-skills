---
name: desktop-security-patterns
description: Harden local-first Rust desktop apps - crafted files, symlink escape, TOCTOU on destructive ops, unsafe/FFI, cargo audit, signed updates.
metadata:
  category: desktop
  tags: [rust, security, local-first, zip-slip, toctou, symlink, hardlink, unsafe, ffi, cargo-audit, code-signing, auto-update]
user-invocable: false
---

# Desktop Security Patterns

> Confirm the app has no server and no multi-user model before applying this skill - it is written for a local-first tool running with the invoking user's own privileges.
>
> This skill owns **what a hostile input or a hostile tree can do to the user's data**. Generic traversal and path-string mechanics belong to `desktop-filesystem-patterns` - containment proofs against untrusted names stay here; preview, undo, and per-item outcomes to `desktop-batch-operations`; decoder limits and pixel budgets to `desktop-image-processing`; signing and packaging mechanics to `desktop-build-release`; credential storage APIs to `desktop-platform-integration`.

## When to Use

- Reading a file the app did not create: archive, image, video, sidecar, imported config
- Any destructive operation whose target is computed before it is applied
- Traversing a tree the user did not author, or one containing links
- Adding `unsafe`, a `-sys` crate, or a `Command` invocation
- Adding or changing the update mechanism
- Reviewing a dependency bump or a new dependency

## Rules

- **The user is not the adversary.** The app runs with their privileges on their machine. They may read its config, edit its database, and attach a debugger - none of that is a finding. Controls that defend the app against its own user are noise
- **The dominant consequence is irreversible data loss, not disclosure.** A crafted input that makes a rename or delete land on the wrong path outranks anything that reveals data the user already owns
- **No server threat model.** CSRF, session fixation, auth bypass, rate limiting, and SQL injection against a single-user local SQLite file the user owns are out of scope. Do not import them
- Every control is labelled with what it actually achieves: `prevented`, `cost-raising only`, or `accepted exposure`. An unlabelled control implies a guarantee it usually does not have
- **Untrusted input is anything the app did not write itself**: archive entry names, image headers, EXIF fields, filenames from a downloaded tree, and sidecar metadata. Validate shape and range before any of it reaches a filesystem path
- **A destructive plan is revalidated against the disk at apply time**, and the revalidation compares identity, not just existence
- `unsafe` blocks carry a comment stating the invariant the caller must uphold. An `unsafe` block with no stated invariant is unreviewable
- **The update mechanism verifies a signature before executing anything it downloaded.** This is the one control whose absence converts a network position into code execution on the user's machine

## Patterns

### The threat model, stated once

| Adversary | Reaches the app via | Dominant consequence |
| --- | --- | --- |
| Crafted file | The user opens or scans a downloaded archive, image, or tree | Path escape, resource exhaustion, decoder crash |
| Hostile tree | Symlinks, junctions, hardlinks inside a scanned directory | Writes and deletes land outside the selected root |
| Concurrent mutation | Another process changes the tree between preview and apply | The wrong file is renamed or deleted |
| Compromised dependency | `cargo build` | Arbitrary code, in-process |
| Network attacker on update | The update channel | Arbitrary code execution as the user |
| **The user** | - | **Not an adversary. Out of scope** |

Absent from this table on purpose: privilege escalation (the app has no privileges to escalate to), tenant isolation, and credential theft from a service the app does not have.

### Archive entry paths: Zip Slip

The highest-value crafted-file bug in a file utility. An entry named `../../../../Windows/System32/x.dll` writes outside the extraction root if the name is joined naively.

```rust
// Bad - the archive chooses the destination
let out = dest_root.join(entry.name());
fs::create_dir_all(out.parent().unwrap())?;

// Good - resolve, then prove containment against a canonical root
let root = dest_root.canonicalize()?;
let out = root.join(entry.name());
let out = normalize_lexically(&out);            // resolve `..` without touching disk
if !out.starts_with(&root) { return Err(Rejected::EscapesRoot(entry.name().into())); }
```

Three details the naive check misses: on Windows an entry name may contain `\` or a drive letter (`C:\x`), which `Path::join` treats as absolute and silently discards the root; an entry may itself be a symlink whose target escapes, so extracted links are validated too, not just regular entries; and `canonicalize` on the output path fails before the file exists, so containment is proved lexically on the *normalized* path, not by canonicalizing the target.

Also bound the extraction: total uncompressed bytes, entry count, and per-entry size. An unbounded extractor turns a 1 MB archive into a full disk.

### Metadata that reaches a path

EXIF, ID3, and archive comments are attacker-controlled strings that tools cheerfully use to name output files.

```rust
// Bad - a camera-supplied field becomes a filename component
let name = exif_field("Model").unwrap_or_default();
let out = dir.join(format!("{name}_{n}.jpg"));

// Good - reduced to a known-safe token set before it can address anything
let name = sanitize_component(&exif_field("Model").unwrap_or_default());
if name.is_empty() { return Err(Rejected::UnusableMetadata); }
```

`sanitize_component` strips path separators on both platforms, `..`, control characters, leading/trailing dots and spaces, and Windows reserved device names (`desktop-filesystem-patterns` lists them). Applied after decoding, never before.

Dimension and length fields from a header are also untrusted - a declared 60000x60000 image is an allocation bomb. Budget from the declared header before decoding (`desktop-image-processing`).

### Symlink, junction, and hardlink escape

A user scanning `~/Downloads` is scanning a tree an attacker may have contributed to.

- **Escape**: a symlink or Windows junction inside the selected root pointing outside it means a delete or overwrite lands outside the scope the user approved. Do not follow links during a destructive walk; where following is a deliberate feature, canonicalize the target and prove `starts_with(root)`
- **Junctions need no elevation on Windows** and appear inside ordinary user profiles, so this is not an exotic case
- **Hardlinks and dedup**: two paths, one inode, one copy of the bytes. A dedup tool that "removes the duplicate" deletes one name of a single file, frees zero bytes, and can destroy both names of content the user has exactly one copy of. Detect via `nlink > 1` or the Windows file index and classify links as links, never as duplicate content

### TOCTOU on destructive operations, honestly labelled

Preview at T0, user reads it, apply at T1. Everything can change in between.

```rust
// Bad - existence at plan time, blind clobber at apply time
if !target.exists() { fs::rename(&src, &target)?; }

// Good - open by handle, verify identity, act through the handle where the API allows
let meta = fs::symlink_metadata(&src)?;         // does not follow a link planted since
if !same_file(&meta, &planned_identity) { return Ok(StepOutcome::Skipped(Changed)); }
if target.symlink_metadata().is_ok() { return Ok(StepOutcome::Skipped(TargetAppeared)); }
fs::rename(&src, &target)?;
```

State the honest label: on a single-user desktop with no `openat2`-style atomic path resolution in the standard library, **most of this is `cost-raising only`, not `prevented`**. The window narrows from minutes to microseconds; it does not close. Say that in the finding rather than claiming the race is fixed. What genuinely is prevented: `symlink_metadata` instead of `metadata` (a link planted after preview is no longer followed) and an identity check that catches a swapped file.

### `unsafe`, FFI, and `Command`

```rust
// Bad - a shell string built from a user path; spaces, quotes, and `&` all reachable
Command::new("cmd").args(["/C", &format!("ffmpeg -i {} out.mp4", input.display())]).spawn()?;

// Good - separate arguments, no shell, no interpolation
Command::new("ffmpeg").arg("-i").arg(input).arg("out.mp4").spawn()?;
```

`Command::new("ffmpeg")` also resolves through `PATH`, which on Windows searches the current directory first for some invocation styles - prefer an absolute path to a binary the app ships or located deliberately.

For `unsafe` and `-sys` crates: the boundary is where a C library parses attacker-controlled bytes. A memory-safety bug in an image or archive decoder is reachable by opening a file. Prefer a pure-Rust decoder where one exists; where a `-sys` crate is required, name it as `accepted exposure` with the reason.

### Dependencies

`cargo audit` against RustSec in CI, and a policy for what a finding does: fail the build, or file an issue. A supply-chain compromise runs at build time with the developer's privileges, which is a strictly larger blast radius than anything the shipped app can do. `cargo deny` additionally gates licences and duplicate versions.

### The update mechanism

The highest-severity item in this skill. An unsigned or unverified update converts any network position - a hostile hotspot, a hijacked CDN, a stale DNS record - into arbitrary code execution as the user.

Required, in order: TLS to a pinned host; a signature over the downloaded artifact verified against a **public key compiled into the binary**, not fetched alongside the artifact; a version check that refuses downgrades; and an atomic swap of the executable (`desktop-build-release`). A checksum published on the same server as the artifact is not a signature - whoever serves one serves the other.

## Output Format

Two modes, chosen by whether the request supplies code to judge or asks for code to be produced.

**Authoring mode** - the request is to write or design something. Emit the code or design, then any `Deferred:` lines. No finding blocks, no severity, no status line.

**Review mode** - source, a diff, a symptom report, or a question describing app surfaces without code was supplied (the last takes `Evidence: inferred` throughout). Emit one block per finding, ordered by severity, Critical first.

```
### [Severity] {file:line | symbol, when source was supplied without paths | symptom, when no source was supplied}

- Category: {CraftedInput | PathEscape | LinkHandling | TOCTOU | UnsafeFFI | ProcessSpawn | DependencyAdvisory | UpdateIntegrity | CredentialStorage}
- Evidence: {source | incident (the reported event already demonstrates the attack) | inferred (state what was not seen)}
- Code: {one-line citation, or `not supplied` when no source was read - inferred or incident}
- Attack: {the mechanism, concretely - "an archive entry named ../../ writes outside the extraction root"; a link or a concurrent change needs no adversary - state what the tree does, not who}
- Consequence: {what the user loses - name the files or the bytes, not "compromise"}
- Control type: {prevented | cost-raising only | accepted exposure} - labels what the Fix achieves; when the Fix's parts differ, label each part inside Fix
- Fix: {the concrete change}
```

`Severity: {Critical | High | Medium | Low}` - Critical = code execution on the user's machine (unverified update, decoder reached through an unaudited FFI boundary) or irreversible data loss outside the approved root. High = data loss inside the root from crafted input or a link, or a destructive op applied without apply-time revalidation. Medium = a resource-exhaustion or crash path from untrusted input, or an unaudited dependency surface. Low = a cost-raising control worth adding with no demonstrated path.

Severity that does not fit a listed band: assign the nearest lower band and state why in `Consequence`. `Category` takes exactly one value - where a defect fits two, pick the one the `Fix` addresses and name the other in `Consequence`.

`Evidence: inferred` is required whenever the source was not read. It bounds the header at High and never raises a block. `Evidence: incident` carries no cap - the reported event is the evidence, so a confirmed compromise takes its true band.

A finding whose only adversary is the user themselves, or that imports a server threat model, is not emitted. Where the request asks for one, write `Out of scope: {concern} - {user is not the adversary | no server threat model}.` - one line per requested concern, before the findings.

A defect owned by a sibling named in the ownership blockquote is written after the findings as `Deferred: {defect} -> {owning skill}`, one per line. Omit when there are none.

In review mode, close with exactly one status line, after any `Deferred:` lines:

| Condition | Line |
| --- | --- |
| One or more findings emitted | none - the findings are the output |
| Source or a symptom report was available, nothing found | `No security findings.` |
| No source, diff, or symptom supplied | `Security check not run: no source supplied.` |

## Avoid

- Reporting "the user could edit the config/database/save file" as a finding
- Importing CSRF, session, auth, or SQL-injection concerns into a local single-user app
- Joining an archive entry name onto the extraction root without a containment proof
- Proving containment by canonicalizing a path that does not exist yet
- Validating archive entries but not the symlinks the archive creates
- Extracting without a byte, entry-count, or per-entry size budget
- EXIF, ID3, or archive-comment strings used as filename components unsanitized
- Trusting declared image dimensions before allocating
- Following symlinks or junctions during a destructive walk
- Deleting one hardlink as a "duplicate"
- Claiming a TOCTOU race is fixed when the window was only narrowed
- `fs::metadata` where `symlink_metadata` is the correct call
- Building a shell command string instead of passing separate `arg()` values
- An `unsafe` block with no stated invariant
- No `cargo audit` in CI, or a `cargo audit` whose findings do nothing
- An update that runs a downloaded artifact without verifying a signature against a key in the binary
- Treating a checksum served beside the artifact as integrity
