---
name: task-desktop-review-security
description: Rust desktop security review - path traversal, symlink escape, TOCTOU on destructive ops, unsafe audit, FFI boundaries, dependency advisories.
agent: desktop-security-engineer
metadata:
  category: desktop
  tags: [rust, desktop, security, path-traversal, symlink, toctou, unsafe, ffi, supply-chain, workflow]
  type: workflow
user-invocable: true
---

> **Behavioral directive:** Load `Use skill: behavioral-principles` before executing this workflow.

# Rust Desktop Security Review

Security lens for a Rust desktop change set. The threat model is **local-first**, not server-shaped: there is no attacker on the wire holding a session, and there is no privilege boundary between the app and the user running it.

## Threat Model

State this before filing anything, because a server threat model imported here produces noise.

**The user is not the adversary.** The app runs with the user's privileges and can already read and delete their files. "The user could modify the config" is not a finding.

Real adversaries and consequences:

| Adversary | Reaches the app through |
| --- | --- |
| A crafted file the user opens or scans | Archive entry paths, image dimensions, parsed metadata, filenames |
| A hostile directory tree | Symlinks, junctions, hardlinks, reparse points, deeply nested paths |
| A malicious or compromised dependency | The crate graph, a `build.rs`, an FFI library |
| A network response | Update checks, license validation, any fetched content |
| Another local process | A race on a path between check and use, a world-writable temp file |

The dominant consequence is **irreversible data loss**, not information disclosure. A bug that deletes the wrong directory outranks one that leaks a filename.

## When to Use

- The umbrella review escalated `+sec`, or a security lens was requested directly
- A change adds a destructive filesystem operation, path resolution, archive handling, `unsafe`, FFI, or a dependency

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
| `prevented` | The fix makes the defect impossible - a canonicalized path that cannot escape its root |
| `cost-raising only` | The fix makes exploitation harder but not impossible - a check that narrows a race window without closing it |
| `accepted exposure` | The exposure is inherent to a local-first app and is documented rather than fixed |

`accepted exposure` is a legitimate outcome here, unlike in a server review. An app that can delete the user's files can be misused by whoever controls the machine; saying so plainly is better than inventing a control that does not exist.

## Excluded Surfaces

`target/`, `dist/`, and generated bindings are never findings. Vendored third-party sources are excluded from style and structure findings but remain in scope for data-flow and advisory concerns.

## Invocation

| Form | Meaning |
|------|---------|
| `/task-desktop-review-security` | Standalone: resolve the change set, review, write the report |
| `/task-desktop-review-security --staged` | Standalone against the staged change set |
| `/task-desktop-review-security deep` | Force `deep` depth |

When invoked as a subagent (e.g. by `task-desktop-review`), the parent supplies the precondition handle, the pre-read diff, the depth level, the detected project shape, and the excluded-surface list: Step 3 is skipped, no git is re-run, and Step 11 returns findings instead of writing - the parent owns the report.

## Workflow

### Step 1 - Behavioral Principles

Use skill: `behavioral-principles`. Accept the parent's confirmation if invoked as a subagent.

### Step 2 - Stack and Project Shape

Accept the project shape from the parent when invoked as a subagent. Otherwise read `Cargo.toml`; if it is absent, stop - this workflow reviews Rust projects only.

Record whether the app fetches anything over the network, whether it reads credentials, whether it ships an updater, and whether any FFI or `-sys` crate is present. The parent's shape does not carry these four facts: in subagent mode establish them from the manifest and source directly - file reads, not git.

### Step 3 - Resolve the Change Set

Use skill: `review-precondition-check`. Forward `--staged` if passed. Read `git diff <base>` once and reuse. Restrict analysis to the handle's `reviewable` paths, minus the excluded surfaces above.

**Skip entirely** when invoked as a subagent and the parent passed the handle plus the pre-read diff.

### Step 4 - Path Resolution

Use skill: `desktop-security-patterns`. The primary defect class in this app type.

- A path built by joining user-supplied or file-supplied components without canonicalization, then used for a read, write, or delete
- `..` components surviving into a resolved path
- A canonicalization that happens, then a *second* join afterwards - the check is defeated by the later concatenation
- An operation whose root confinement is asserted in a comment rather than enforced by a check
- Windows specifics: reserved device names (`CON`, `PRN`, `AUX`, `NUL`, `COM1`-`COM9`, `LPT1`-`LPT9`), trailing dots and spaces, alternate data streams, the `\\?\` prefix and the 260-character limit, and UNC paths
- Non-UTF-8 path components handled by lossy conversion, which silently changes the path that is then operated on

### Step 5 - Symlinks, Junctions, and Link Following

A hostile tree is the realistic delivery mechanism for a destructive-operation bug.

- Traversal that follows symlinks by default where it should not, letting a scan escape its root
- Windows directory junctions and reparse points, which behave differently from POSIX symlinks and are frequently missed
- A delete or overwrite that resolves through a link and destroys the target rather than the link
- Hardlink-aware dedup: deleting a "duplicate" that is a hardlink to the file being kept destroys both names of one inode
- A recursion with no depth bound or cycle detection

### Step 6 - TOCTOU on Destructive Operations

The window between deciding and acting. Local, real, and consequential here because the action is irreversible.

- A preview computed, then applied later against a tree that may have changed - the plan must be revalidated at apply time, not trusted
- `exists()` checked, then a create or overwrite performed - use the atomic form (`create_new`, `OpenOptions`) instead
- A rename target checked for collision, then written non-atomically
- A temp file created predictably in a shared directory rather than with `tempfile`'s atomic, permission-restricted creation
- An operation that is not idempotent under retry, so a partial failure plus a retry compounds the damage

State the control type honestly: most TOCTOU narrowing on a local filesystem is `cost-raising only`, not `prevented`.

### Step 7 - Untrusted Input Parsing

Every byte from a file the user did not author is attacker-controlled.

- Archive extraction without entry-path validation - the Zip Slip class, where an entry named `../../etc/passwd` escapes the extraction root. Also check entry size for decompression bombs and entry count for resource exhaustion
- Image dimensions used to allocate before being bounds-checked - a crafted header requesting an enormous buffer
- Parsed metadata (EXIF, ID3, tags) trusted for a filesystem operation - a filename derived from file-supplied metadata is user-controlled input reaching a path
- Deserialization of a persisted or downloaded structure without size or shape limits
- Integer overflow in a size or offset computation derived from file content
- A file-supplied string - a filename, a tag, parsed metadata - concatenated into SQL against the local store: parameterize. The adversary is the crafted file, not the user, so this stays in scope despite the server-import exclusion

### Step 8 - `unsafe`, FFI, and Process Boundaries

- Every `unsafe` block carries a comment stating the invariant that makes it sound. Absent that comment, it is a finding regardless of correctness
- FFI boundaries: a pointer whose lifetime is not obviously owned, a buffer whose length is trusted from the other side, a string assumed NUL-terminated, an error code ignored
- `Command` invocations with interpolated arguments - and specifically, that arguments are passed as separate `arg()` values rather than assembled into a shell string
- The PDF-print escape hatch and any other shell-out inherits this check
- A `build.rs` that downloads, or that runs code from a path the build environment controls

### Step 9 - Credentials, Network, and Update Path

- Secrets in source or committed configuration
- Credential storage: `keyring` usage that assumes success, given it fails for unsigned binaries on macOS and returns `-34018`
- TLS verification disabled or a certificate check bypassed, in any form, including "just for local testing"
- An update mechanism that fetches and executes without verifying a signature - the highest-severity finding available in this app type, because it converts any network position into code execution
- A license or activation check that trusts a client-side value for a server-side decision, where one exists

### Step 10 - Dependencies and Evidence

- New dependencies: what they pull in transitively, whether the crate is maintained, and whether `cargo audit` reports an advisory
- A dependency with a `build.rs` that runs arbitrary code at build time
- `unsafe`-heavy dependencies on a parsing path
- Known-dead crates: `sled` is unmaintained since 2021
- At `deep`: audit the added subtree rather than only the direct edge

Then assign a control type to every finding, and route anything that is not a security defect rather than filing it here.

### Step 11 - Write Report

**Standalone only.** A subagent run returns the `## Findings` sections, the `## Dependency Triage` and `## Reviewed, Not Filed` sections, and at `deep` the `## Dependency Graph Audit` - nothing else. No frontmatter, no Summary block, no Next Steps: the parent owns those and cannot merge two of them. Project-shape values the parent already supplied are not echoed back.

Intent (`[Must]` / `[Recommend]`) is stated per finding so the parent can order Next Steps without re-deriving it.

Standalone: write to `review-security-<branch>.md` in the current working directory, overwriting if it exists. Sanitize `<branch>` from the handle's `current_branch`: replace `/` and any character outside `[A-Za-z0-9_-]` with `-`, collapse consecutive `-`, strip leading and trailing `-`.

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
## Rust Desktop Security Review Summary

**Assessment:** Approve | Request Changes | Discuss
**Threat Surfaces Touched:** [which of: crafted file, hostile tree, dependency, network, local race | none]
**Worst Case:** [the most severe realistic consequence of this change set, in one line]

## Findings

### [Must] file:line
- **Issue:** [the defect, named]
- **Adversary:** [which adversary from the threat model reaches it, and how]
- **Consequence:** [what happens - data loss, code execution, disclosure]
- **Control type:** prevented | cost-raising only | accepted exposure
- **Fix:** [concrete Rust change]

### [Recommend] file:line
[same shape]

## Dependency Triage

[Each dependency the change adds or bumps: what it is for, what it pulls in, maintenance status, and any advisory. `none - no dependency change` when there is none. A top-level section, so a subagent returns it without its parent]

## Reviewed, Not Filed

[Surfaces examined that produced no finding, and why - so the reader can tell a clean path from an unexamined one. e.g. "archive extraction: entry paths validated against the root at extract.rs:44". A top-level section, so a subagent returns it without its parent]

## Dependency Graph Audit _(deep depth only)_

[The added transitive subtree, `unsafe` usage on parsing paths, and build-script behaviour]

## Next Steps

[Standalone only - the parent owns this section in subagent mode. One numbered line per finding, `[Must]` before `[Recommend]`, each as `<label> file:line - <action>`]
```

**Omit empty sections** other than `Dependency Triage` and `Reviewed, Not Filed`, which are always written - `none` is itself the useful signal.

## Self-Check

- [ ] `behavioral-principles` loaded (or accepted from parent)
- [ ] Project shape accepted from parent, or `Cargo.toml` read
- [ ] Threat model applied - the user is not treated as the adversary
- [ ] Step 3: handle resolved (or received); diff read once; excluded surfaces skipped
- [ ] Path resolution checked, including Windows reserved names, long paths, and non-UTF-8 components
- [ ] Symlinks, junctions, hardlinks, and recursion bounds checked
- [ ] TOCTOU checked on every destructive operation; plan revalidation at apply time checked
- [ ] Archive entry paths, image dimensions, and parsed metadata checked as untrusted input
- [ ] Every `unsafe` block checked for a stated invariant; FFI boundaries checked
- [ ] Shell-out argument passing checked
- [ ] Credentials, TLS verification, and the update path checked
- [ ] New dependencies triaged; `sled` flagged if present
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
- Performance findings - route them to the perf lens
- Raising findings against `target/`, `dist/`, or generated bindings
- Omitting `Reviewed, Not Filed` - an unexamined surface and a clean surface must be distinguishable
