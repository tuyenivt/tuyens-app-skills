---
name: desktop-security-patterns
description: Harden local-first C# desktop apps - crafted files, Zip Slip, symlink escape, TOCTOU, P/Invoke, ArgumentList spawning, NuGet audit, signed updates.
metadata:
  category: desktop
  tags: [csharp, dotnet, security, local-first, zip-slip, toctou, symlink, pinvoke, nuget-audit, code-signing, auto-update]
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
- Adding `unsafe` code, a P/Invoke signature, or a `Process.Start` call
- Adding or changing the update mechanism
- Reviewing a dependency bump or a new dependency

## Rules

- **The user is not the adversary.** The app runs with their privileges on their machine. They may read its config, edit its database, and attach a debugger - none of that is a finding. Controls that defend the app against its own user are noise
- **The dominant consequence is irreversible data loss, not disclosure.** A crafted input that makes a rename or delete land on the wrong path outranks anything that reveals data the user already owns
- **No server threat model.** CSRF, session fixation, auth bypass, rate limiting, and SQL injection against a single-user local SQLite file the user owns are out of scope. Do not import them
- Every control is labelled with what it actually achieves: `prevented`, `cost-raising only`, or `accepted exposure`. An unlabelled control implies a guarantee it usually does not have
- **Untrusted input is anything the app did not write itself**: archive entry names, image headers, EXIF fields, filenames from a downloaded tree, and sidecar metadata. Validate shape and range before any of it reaches a filesystem path
- **A destructive plan is revalidated against the disk at apply time**, and the revalidation compares identity, not just existence
- `unsafe` blocks and P/Invoke signatures carry a comment stating the invariant the caller must uphold. A native boundary with no stated invariant is unreviewable
- **The update mechanism verifies a signature before executing anything it downloaded.** This is the one control whose absence converts a network position into code execution on the user's machine

## Patterns

### The threat model, stated once

| Adversary | Reaches the app via | Dominant consequence |
| --- | --- | --- |
| Crafted file | The user opens or scans a downloaded archive, image, or tree | Path escape, resource exhaustion, decoder crash |
| Hostile tree | Symlinks, junctions, hardlinks inside a scanned directory | Writes and deletes land outside the selected root |
| Concurrent mutation | Another process changes the tree between preview and apply | The wrong file is renamed or deleted |
| Compromised dependency | `dotnet restore` and the build | Arbitrary code, in-process |
| Network attacker on update | The update channel | Arbitrary code execution as the user |
| **The user** | - | **Not an adversary. Out of scope** |

Absent from this table on purpose: privilege escalation (the app has no privileges to escalate to), tenant isolation, and credential theft from a service the app does not have.

### Archive entry paths: Zip Slip

The highest-value crafted-file bug in a file utility. An entry named `../../../../Windows/System32/x.dll` writes outside the extraction root if the name is joined naively.

```csharp
// Bad - the archive chooses the destination
var outPath = Path.Combine(destRoot, entry.FullName);
entry.ExtractToFile(outPath, overwrite: false);

// Good - normalize the joined result, then prove containment against the root
var root = Path.GetFullPath(destRoot);
var outPath = Path.GetFullPath(Path.Combine(root, entry.FullName));
if (!outPath.StartsWith(root + Path.DirectorySeparatorChar, StringComparison.OrdinalIgnoreCase))
    return Rejected.EscapesRoot(entry.FullName);
```

Details the naive check misses: `Path.Combine` treats a rooted second argument (`C:\x`, `\x`) as absolute and silently discards the root, and it resolves no `..` at all - so containment is proved on `GetFullPath` of the *joined* result, never by canonicalizing the root and then trusting the join. The `StartsWith` includes the trailing separator so `C:\out` does not match `C:\out-evil`, and compares case-insensitively where the filesystem does. An entry may itself be a symlink whose target escapes - validate the links an archive creates, not just its regular entries. `ZipFile.ExtractToDirectory` performs this proof itself in modern .NET; the moment extraction is per-entry, the proof is yours again.

Also bound the extraction: total uncompressed bytes counted while copying (`ZipArchiveEntry.Length` is a declared value, not a guarantee), entry count, and per-entry size. An unbounded extractor turns a 1 MB archive into a full disk.

### Metadata that reaches a path

EXIF, ID3, and archive comments are attacker-controlled strings that tools cheerfully use to name output files.

```csharp
// Bad - a camera-supplied field becomes a filename component
var model = exif.GetString("Model") ?? "";
var outFile = Path.Combine(dir, $"{model}_{n}.jpg");

// Good - reduced to a known-safe token set before it can address anything
var model = SanitizeComponent(exif.GetString("Model") ?? "");
if (model.Length == 0) return Rejected.UnusableMetadata;
```

`SanitizeComponent` strips path separators on both platforms, `..`, control characters, leading and trailing dots and spaces, and Windows reserved device names (`desktop-filesystem-patterns` lists them). Applied after decoding, never before.

Dimension and length fields from a header are also untrusted - a declared 60000x60000 image is a 14 GB allocation bomb. Read the header alone (`SKCodec.Create(path).Info`) and budget width x height x 4 bytes before any decode; `desktop-image-processing` owns the budget numbers.

### Symlink, junction, and hardlink escape

A user scanning `~/Downloads` is scanning a tree an attacker may have contributed to.

- **Escape**: a symlink or Windows junction inside the selected root pointing outside it means a delete or overwrite lands outside the scope the user approved. Skip reparse points during a destructive walk - `new EnumerationOptions { AttributesToSkip = FileAttributes.ReparsePoint }` - and where following is a deliberate feature, resolve with `File.ResolveLinkTarget(path, returnFinalTarget: true)` and prove containment against the root
- **Junctions need no elevation on Windows** and appear inside ordinary user profiles, so this is not an exotic case
- **Hardlinks and dedup**: two paths, one file, one copy of the bytes. A dedup tool that "removes the duplicate" deletes one name of a single file, frees zero bytes, and can destroy both names of content the user has exactly one copy of. The BCL exposes no link count - P/Invoke `GetFileInformationByHandle` (link count and file index) on Windows, `stat`'s `st_nlink` on macOS - and classify links as links, never as duplicate content

### TOCTOU on destructive operations, honestly labelled

Preview at T0, user reads it, apply at T1. Everything can change in between.

```csharp
// Bad - existence at plan time, blind clobber at apply time
if (!File.Exists(target)) File.Move(src, target);

// Good - re-verify identity at apply time, without following a planted link
var info = new FileInfo(src);
if (info.LinkTarget is not null) return new Skipped(src, SkipReason.LinkAppeared);
if (info.Length != planned.Length || info.LastWriteTimeUtc != planned.LastWriteTimeUtc)
    return new Skipped(src, SkipReason.Changed);
if (Path.Exists(target)) return new Skipped(src, SkipReason.TargetAppeared);
File.Move(src, target);
```

State the honest label: the BCL has no handle-anchored path resolution, so **most of this is `cost-raising only`, not `prevented`**. The window narrows from minutes to microseconds; it does not close. Say that in the finding rather than claiming the race is fixed. What genuinely is prevented: the `LinkTarget` check stops the apply from following a link planted after preview, and the identity comparison catches a swapped file.

### Process spawning, P/Invoke, and `unsafe`

```csharp
// Bad - a shell string built from a user path; spaces, quotes, and & all reachable
Process.Start("cmd.exe", $"/C ffmpeg -i {input} out.mp4");

// Good - separate arguments, no shell, no interpolation
var psi = new ProcessStartInfo(ffmpegPath) { UseShellExecute = false };
psi.ArgumentList.Add("-i");
psi.ArgumentList.Add(input);
psi.ArgumentList.Add("out.mp4");
Process.Start(psi);
```

A bare `ProcessStartInfo("ffmpeg")` resolves through `PATH` - prefer an absolute path to a binary the app ships or located deliberately.

For P/Invoke and `unsafe`: the boundary that matters is where a native library parses attacker-controlled bytes - a memory-safety bug in an image or archive decoder is reachable by opening a file. Prefer a managed decoder where one exists; where a native one is required (SkiaSharp is native Skia), name it as `accepted exposure` with the reason. A wrong `DllImport` signature - a length in characters where the API takes bytes, a buffer the callee outlives - is memory corruption in an otherwise memory-safe app, which is why the invariant comment is mandatory.

### Dependencies

NuGet audit runs on restore in modern SDKs; keep its warnings loud, and run `dotnet list package --vulnerable --include-transitive` in CI with a policy for what a finding does: fail the build, or file an issue. A supply-chain compromise runs at build time with the developer's privileges, which is a strictly larger blast radius than anything the shipped app can do.

### The update mechanism

The highest-severity item in this skill. An unsigned or unverified update converts any network position - a hostile hotspot, a hijacked CDN, a stale DNS record - into arbitrary code execution as the user.

Required, in order: TLS to a pinned host; a signature over the downloaded artifact verified against a **public key compiled into the binary**, not fetched alongside the artifact; a version check that refuses downgrades; and an atomic swap of the installed build (`desktop-build-release`). A checksum published on the same server as the artifact is not a signature - whoever serves one serves the other.

## Output Format

Two modes, chosen by whether the request supplies code to judge or asks for code to be produced.

**Authoring mode** - the request is to write or design something. Emit the code or design, then any `Deferred:` lines. No finding blocks, no severity, no status line. The `prevented | cost-raising only | accepted exposure` labels appear as comments on the controls they describe.

**Review mode** - source, a diff, a symptom report, or a question describing app surfaces without code was supplied (the last takes `Evidence: inferred` throughout). Emit one block per finding, ordered by severity, Critical first.

```
### [Severity] {file:line | symbol, when source was supplied without paths | symptom, when no source was supplied}

- Category: {CraftedInput | PathEscape | LinkHandling | TOCTOU | UnsafePInvoke | ProcessSpawn | DependencyAdvisory | UpdateIntegrity | CredentialStorage}
- Evidence: {source | incident (the reported event already demonstrates the attack) | inferred (state what was not seen)}
- Code: {one-line citation, or `not supplied` when no source was read - inferred or incident}
- Attack: {the mechanism, concretely - "an archive entry named ../../ writes outside the extraction root"; a link or a concurrent change needs no adversary - state what the tree does, not who}
- Consequence: {what the user loses - name the files or the bytes, not "compromise"}
- Control type: {prevented | cost-raising only | accepted exposure} - labels what the Fix achieves; when the Fix's parts differ, label each part inside Fix
- Fix: {the concrete change}
```

`Severity: {Critical | High | Medium | Low}` - Critical = code execution on the user's machine (unverified update, a decoder reached through an unaudited native boundary) or irreversible data loss outside the approved root. High = data loss inside the root from crafted input or a link, or a destructive op applied without apply-time revalidation. Medium = a resource-exhaustion or crash path from untrusted input, or an unaudited dependency surface. Low = a cost-raising control worth adding with no demonstrated path.

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
- Joining an archive entry name onto the extraction root without normalizing the joined result
- Proving containment with a `StartsWith` that lacks the trailing separator, so `C:\out` matches `C:\out-evil`
- Validating archive entries but not the symlinks the archive creates
- Extracting without a byte, entry-count, or per-entry size budget
- Trusting `ZipArchiveEntry.Length` as the bytes an entry will actually decompress to
- EXIF, ID3, or archive-comment strings used as filename components unsanitized
- Trusting declared image dimensions before allocating
- Following reparse points during a destructive walk
- Deleting one hardlink as a "duplicate"
- Claiming a TOCTOU race is fixed when the window was only narrowed
- Building a command string instead of adding to `ArgumentList`
- An `unsafe` block or `DllImport` with no stated invariant
- No vulnerability audit in CI, or one whose findings do nothing
- An update that runs a downloaded artifact without verifying a signature against a key in the binary
- Treating a checksum served beside the artifact as integrity
