---
name: task-desktop-review-security
description: C# desktop security review - path containment, Zip Slip, symlink escape, TOCTOU on destructive ops, P/Invoke audit, NuGet advisories.
agent: desktop-security-engineer
metadata:
  category: desktop
  tags: [csharp, dotnet, desktop, security, path-traversal, zip-slip, toctou, pinvoke, supply-chain, workflow]
  type: workflow
user-invocable: true
---

> **Behavioral directive:** Load `Use skill: behavioral-principles` before executing this workflow.

# C# Desktop Security Review

Security lens for a C# desktop change set. The threat model is **local-first**, not server-shaped: there is no attacker on the wire holding a session, and there is no privilege boundary between the app and the user running it.

## Threat Model

State this before filing anything, because a server threat model imported here produces noise.

**The user is not the adversary.** The app runs with the user's privileges and can already read and delete their files. "The user could modify the config" is not a finding.

Real adversaries and consequences:

| Adversary | Reaches the app through |
| --- | --- |
| A crafted file the user opens or scans | Archive entry paths, image dimensions, parsed metadata, filenames |
| A hostile directory tree | Symlinks, junctions, hardlinks, reparse points, deeply nested paths |
| A malicious or compromised dependency | The NuGet graph, a build-time props/targets script, a native library |
| A network response | Update checks, licence validation, any fetched content |
| Another local process | A race on a path between check and use, a world-writable temp file |

The dominant consequence is **irreversible data loss**, not information disclosure. A bug that deletes the wrong directory outranks one that leaks a filename.

## When to Use

- The umbrella review escalated `+sec`, or a security lens was requested directly
- A change adds a destructive filesystem operation, path resolution, archive handling, `unsafe`, P/Invoke, or a dependency

**Not for:** feature design (`task-desktop-implement`), correctness review (`task-desktop-review`), or throughput (`task-desktop-review-perf`).

## Depth

| Depth | Runs |
|-------|------|
| `standard` | Steps 4-10 |
| `deep` | Steps 4-10 + the dependency-graph audit |

When invoked as a subagent, **the parent's resolved depth wins** over this table.

## Control Types

Every finding names what its fix actually achieves:

| Control type | Meaning |
| --- | --- |
| `prevented` | The fix makes the defect impossible - a contained path that cannot escape its root |
| `cost-raising only` | The fix makes exploitation harder but not impossible - a check that narrows a race window without closing it |
| `accepted exposure` | The exposure is inherent to a local-first app and is documented rather than fixed |

`accepted exposure` is a legitimate outcome here, unlike in a server review. An app that can delete the user's files can be misused by whoever controls the machine; saying so plainly is better than inventing a control that does not exist.

The boundary test: `prevented` means the vulnerable code path no longer exists; a check that leaves the path reachable through a narrower window is `cost-raising only`. A fix whose parts differ in strength carries the weakest part's type in the finding's slot, with the per-part split stated in Fix.

## Excluded Surfaces

`bin/`, `obj/`, `publish/`, and generated code are never findings. Vendored third-party sources are excluded from style and structure findings but remain in scope for data-flow and advisory concerns.

## Invocation

| Form | Meaning |
|------|---------|
| `/task-desktop-review-security` | Standalone: resolve the change set, review, write the report |
| `/task-desktop-review-security --staged` | Standalone against the staged change set |
| `/task-desktop-review-security deep` | Force `deep` depth |

When invoked as a subagent (e.g. by `task-desktop-review`), the parent supplies the precondition handle, the pre-read diff, the depth level, the detected project shape, and the reviewable-surface table whose excluded rows are the excluded surfaces: Step 3 is skipped, no git is re-run, and Step 11 returns findings instead of writing - the parent owns the report.

## Workflow

### Step 1 - Behavioral Principles

Use skill: `behavioral-principles`. Accept the parent's confirmation if invoked as a subagent.

### Step 2 - Stack and Project Shape

Accept the project shape from the parent when invoked as a subagent. Otherwise read the `.csproj` files; if none exists, stop - this workflow reviews .NET projects only.

Record whether the app fetches anything over the network, whether it reads credentials, whether it ships an updater, and whether any P/Invoke, `unsafe` block, or native library binding is present. The parent's shape does not carry these four facts: in subagent mode establish them from the project files and source directly - file reads, not git. A source-wide token sweep (`HttpClient`, `DllImport`, `unsafe`, credential-store APIs) is the floor for asserting absence; state the method used.

### Step 3 - Resolve the Change Set

Use skill: `review-precondition-check`. Forward `--staged` if passed. Read `git diff <base>` once and reuse. Restrict analysis to the handle's `reviewable` paths, minus the excluded surfaces above.

**Skip entirely** when invoked as a subagent and the parent passed the handle plus the pre-read diff.

### Step 4 - Path Resolution

Use skill: `desktop-security-patterns`. The primary defect class in this app type.

- A path built by joining user-supplied or file-supplied components, then used for a read, write, or delete without a containment proof: `Path.GetFullPath` on the *joined* result, compared against the root with the trailing separator included, case-insensitively where the filesystem is - `C:\out` must not match `C:\out-evil`
- `Path.Combine` silently discarding the root when a later component is rooted (`C:\x`, `\x`) - the input chooses the destination
- `..` components surviving into a resolved path, or a canonicalization followed by a *second* join afterwards - the check is defeated by the later concatenation
- An operation whose root confinement is asserted in a comment rather than enforced by a check
- Windows specifics: reserved device names (`CON`, `PRN`, `AUX`, `NUL`, `COM1`-`COM9`, `LPT1`-`LPT9`), trailing dots and spaces, alternate data streams, the `\\?\` prefix and the 260-character limit, and UNC paths
- A filename normalized (NFC/NFD) for comparison and then used for I/O - the operation lands on a name the disk does not have

### Step 5 - Symlinks, Junctions, and Link Following

A hostile tree is the realistic delivery mechanism for a destructive-operation bug.

- Traversal that descends into reparse points or follows symlinks where it should not, letting a scan escape its root
- Windows directory junctions and reparse points, which behave differently from POSIX symlinks and are frequently missed
- A delete or overwrite that resolves through a link and destroys the target rather than the link
- Hardlink-aware dedup: deleting a "duplicate" that is a hardlink to the file being kept destroys both names of one file
- A recursion with no depth bound or cycle detection
- A clean traversal verdict states which walk it covers: a read-only scan and a destructive apply resolve links differently, and a clean scan proves nothing about the apply path

### Step 6 - TOCTOU on Destructive Operations

The window between deciding and acting. Local, real, and consequential here because the action is irreversible.

- A preview computed, then applied later against a tree that may have changed - the plan must be revalidated at apply time, and the revalidation compares identity (size, mtime), not just existence
- `File.Exists` checked, then a create or overwrite performed - use the atomic form (`FileMode.CreateNew`) instead
- A rename target checked for collision, then written non-atomically
- A temp file created with a predictable name in a shared directory, rather than created with `FileMode.CreateNew` in the app's own directory or beside its destination
- An operation that is not idempotent under retry, so a partial failure plus a retry compounds the damage
- A destructive apply with no undo is the umbrella's preview-and-undo finding, not this lens's: file the race that makes the loss irreversible here, and list the missing undo as a `routed:` line in Reviewed, Not Filed

State the control type honestly: most TOCTOU narrowing on a local filesystem is `cost-raising only`, not `prevented`.

### Step 7 - Untrusted Input Parsing

Every byte from a file the user did not author is attacker-controlled.

- Archive extraction without entry-path validation - the Zip Slip class, where an entry named `../../x.dll` escapes the extraction root. `ZipFile.ExtractToDirectory` performs the containment proof itself in modern .NET; the moment extraction is per-entry (`ZipArchiveEntry.ExtractToFile`), the proof is yours again. Also bound the extraction: total uncompressed bytes counted while copying (`ZipArchiveEntry.Length` is a declared value, not a guarantee), entry count, and per-entry size
- Image dimensions used to allocate before being bounds-checked - a crafted header declaring 60000x60000 requests a 14 GB buffer. Read the header alone (`SKCodec`) and budget width x height x 4 bytes before any decode
- Parsed metadata (EXIF, ID3, tags) trusted for a filesystem operation - a filename derived from file-supplied metadata is attacker-controlled input reaching a path; sanitize to a known-safe token set
- Deserialization of a persisted or downloaded structure without size or shape limits
- Integer overflow in a size or offset computation derived from file content
- A file-supplied string - a filename, a tag, parsed metadata - concatenated into SQL against the local store: parameterize. The adversary is the crafted file, not the user, so this stays in scope despite the server-import exclusion

### Step 8 - `unsafe`, P/Invoke, and Process Boundaries

- Every `unsafe` block and P/Invoke signature carries a comment stating the invariant that makes it sound. Absent that comment, it is a finding regardless of correctness
- P/Invoke boundaries: a buffer whose length is trusted from the native side, a pointer lifetime that outlives its owner, a string marshalling assumption (encoding, NUL termination), an error code or `Marshal.GetLastPInvokeError` ignored, a raw `IntPtr` where a `SafeHandle` should own the lifetime
- `Process.Start` invocations: arguments passed as separate `ArgumentList` entries, never assembled into one `Arguments` string with interpolated values - and never routed through a shell (`cmd /c`) with attacker-influenced input. Odd switch shapes stay one entry: `explorer.exe`'s `/select,<path>` is a single comma-joined `ArgumentList` entry, not a rebuilt shell line
- The PDF-print escape hatch (render, then hand the file to the OS `print`/`open` verb) and any other shell-out inherits this check
- A build step that downloads, or runs code from a path the build environment controls - an `Exec` task or a `.targets` file counts

### Step 9 - Credentials, Network, and Update Path

- Secrets in source or committed configuration
- Credential storage that assumes success: macOS Keychain fails for unsigned binaries, and every rebuild changes an ad-hoc signing identity - yesterday's credential is unreadable today. A stable signing identity is the prerequisite, not a retry
- TLS verification disabled or a certificate check bypassed - a `ServerCertificateCustomValidationCallback` returning `true`, in any form, including "just for local testing"
- An update mechanism that fetches and executes without verifying a signature against a key embedded in the binary, or that accepts downgrades - the highest-severity finding available in this app type, because it converts any network position into code execution
- A licence or activation check: there is no server behind it, so what it achieves is `cost-raising only` by construction. State that, rather than filing bypassability as a defect more obfuscation could fix

### Step 10 - Dependencies and Evidence

- New packages: what they pull in transitively (`dotnet list package --include-transitive`), whether the package is maintained, and whether `dotnet list package --vulnerable` reports an advisory. These commands restore, which writes `obj/` - build output, excluded and gitignored, so running them does not modify the change set under review. Where the check cannot run (no restore, no network), record advisory status as unverified and name which packages to check first - never report a supply chain clean on a check that did not run, and where it runs for some projects and not others, say which. An advisory the check surfaces on a dependency this change did not touch is still reported - on the Advisory check line, not as a finding against the change
- A package that ships build-time `.props`/`.targets` logic - code that runs at build time on the maintainer's machine and in CI
- `unsafe`-heavy or native-binding dependencies on a parsing path
- At `deep`: audit the added subtree rather than only the direct edge

Then assign a control type to every finding, and route anything that is not a security defect rather than filing it here.

### Step 11 - Write Report

**Standalone only.** A subagent run returns the `## Findings` sections, the `## Dependency Triage` and `## Reviewed, Not Filed` sections, and at `deep` the `## Dependency Graph Audit` - nothing else. No frontmatter, no Summary block, no Next Steps: the parent owns those and cannot merge two of them. Project-shape values the parent already supplied are not echoed back.

Intent (`[Must]` / `[Recommend]`) is stated per finding so the parent can order Next Steps without re-deriving it.

Standalone: write to `review-security-<branch>.md` at the reviewed repository's root, overwriting if it exists. Sanitize `<branch>` from the handle's `current_branch`: replace `/` and any character outside `[A-Za-z0-9_-]` with `-`, collapse consecutive `-`, strip leading and trailing `-`.

```yaml
---
branch: <branch>
scope_mode: working-tree | staged-only | last-commit
files: <N>
scope: security
depth: standard | deep
generated_at: <ISO 8601 UTC timestamp>
---
```

After writing, print exactly one confirmation line:

```
Report written to review-security-<branch>.md (<N> files, scope: security)
```

## Output Format

The fence below delimits the template for display only. Emit the report body as raw Markdown; never wrap the whole report in a code fence.

```markdown
## C# Desktop Security Review Summary

**Assessment:** Approve | Request Changes | Discuss
**Threat Surfaces Touched:** [which of: crafted file, hostile tree, dependency, network, local race | none]
**Worst Case:** [the most severe realistic consequence of this change set, in one line; when consequences compete, least reversible wins, then furthest reach outside the operation's approved root]

## Findings

[Findings forming one causal chain - a planted link that both escapes traversal and is the TOCTOU payload - stay separate findings, each citing the other's `file:line`]

### [Must] file:line
- **Issue:** [the defect, named]
- **Adversary:** [which adversary from the threat model reaches it, and how]
- **Consequence:** [what happens - data loss, code execution, disclosure]
- **Control type:** prevented | cost-raising only | accepted exposure
- **Fix:** [concrete C# change]

### [Recommend] file:line
[same shape]

## Dependency Triage

[Each dependency the change adds or bumps: what it is for, what it pulls in, maintenance status, and any advisory. `none - no dependency change` when there is none. A top-level section, so a subagent returns it without its parent]
- Advisory check: {clean | every advisory the check surfaced as `<package> <version> - <advisory>`, including on dependencies this change did not touch | unverified - which projects could not be restored or scanned, and why}

## Reviewed, Not Filed

[Surfaces examined that produced no finding, and why - so the reader can tell a clean path from an unexamined one. e.g. "archive extraction: entry paths validated against the root at ArchiveExtractor.cs:44". Concerns examined and routed to another owner appear as `routed: <concern> -> <owning skill or lens>`. A top-level section, so a subagent returns it without its parent]

## Dependency Graph Audit _(deep depth only)_

[The added transitive subtree, `unsafe` and native-binding usage on parsing paths, and build-time script behaviour. `none - no dependency added or bumped; graph unchanged` when the change set adds none]

## Next Steps

[Standalone only - the parent owns this section in subagent mode. One numbered line per finding, `[Must]` before `[Recommend]`, each as `<label> file:line - <action>`]
```

**Omit empty sections** other than `Dependency Triage` and `Reviewed, Not Filed`, which are always written - `none` is itself the useful signal.

## Self-Check

- [ ] `behavioral-principles` loaded (or accepted from parent)
- [ ] Project shape accepted from parent, or `.csproj` read; network, credential, updater, and native-boundary facts established
- [ ] Threat model applied - the user is not treated as the adversary
- [ ] Step 3: handle resolved (or received); diff read once; excluded surfaces skipped
- [ ] Path containment proved on the joined result, including Windows reserved names, long paths, rooted-component `Path.Combine` behaviour, and normalization
- [ ] Symlinks, junctions, hardlinks, and recursion bounds checked
- [ ] TOCTOU checked on every destructive operation; plan revalidation at apply time checked
- [ ] Archive entry paths, extraction bounds, image dimensions, and parsed metadata checked as untrusted input
- [ ] Every `unsafe` block and P/Invoke signature checked for a stated invariant
- [ ] `Process.Start` argument passing checked - `ArgumentList`, never a concatenated string
- [ ] Credentials, TLS verification, and the update path checked; signing-identity prerequisite named where credential storage is touched
- [ ] New dependencies triaged; `dotnet list package --vulnerable` run or recorded as unverified
- [ ] Every finding names its adversary, consequence, and control type
- [ ] `Dependency Triage` and `Reviewed, Not Filed` written, `none` where empty
- [ ] Step 11: standalone: report written to `review-security-<branch>.md` with the sanitized branch name and complete frontmatter; confirmation line printed; subagent: Findings, `Dependency Triage`, `Reviewed, Not Filed` (plus the deep-only audit) returned to parent, no Summary or Next Steps, no file written

## Avoid

- Writing a report when invoked as a subagent - the parent owns it
- Returning the whole Output Format as a subagent - only `## Findings`, `## Dependency Triage`, `## Reviewed, Not Filed`, and the deep-only audit are returned
- Treating the user as the adversary - the app already runs with their privileges
- Server threat model imports: session fixation, CSRF, rate limiting, SQL injection that models the user as the adversary - a file-supplied string reaching SQL is Step 7 untrusted input and stays in scope
- Filing "the config file is user-editable" as a finding
- Claiming `prevented` for a TOCTOU narrowing that only shrinks the window
- Filing a hardcoded user-facing string as a security finding unless the string is a secret
- Recommending a control that does not exist on a local-first app instead of stating accepted exposure
- Reporting a dependency clean of advisories when `dotnet list package --vulnerable` did not run
- Performance findings - route them to the perf lens
- Raising findings against `bin/`, `obj/`, `publish/`, or generated code
- Omitting `Reviewed, Not Filed` - an unexamined surface and a clean surface must be distinguishable
