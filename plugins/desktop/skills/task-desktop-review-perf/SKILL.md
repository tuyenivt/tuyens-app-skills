---
name: task-desktop-review-perf
description: Rust desktop performance review - scan throughput, hashing and decode cost, UI-thread blocking, allocation, caching, startup time.
agent: desktop-performance-engineer
metadata:
  category: desktop
  tags: [rust, iced, performance, throughput, filesystem, hashing, ui-thread, allocation, workflow]
  type: workflow
user-invocable: true
---

> **Behavioral directive:** Load `Use skill: behavioral-principles` before executing this workflow.

# Rust Desktop Performance Review

Throughput and responsiveness lens for a Rust desktop change set. Two distinct budgets: **work throughput** (how fast a scan, hash, or decode completes) and **UI responsiveness** (whether the window stays interactive while it runs). A change can pass one and fail the other.

## When to Use

- The umbrella review escalated `+perf`, or a perf lens was requested directly
- A change adds traversal, hashing, decode, or a large in-memory collection
- A change adds work to `update`, a view function, or a subscription

**Not for:** feature design (`task-desktop-implement`), correctness review (`task-desktop-review`), or security (`task-desktop-review-security`).

## Depth

| Depth | Runs |
|-------|------|
| `standard` | Steps 4-10 |
| `deep` | Steps 4-10 + the measurement plan |

When invoked as a subagent, **the parent's resolved depth wins** over this table. A lens invoked at `deep` returns its deep-only section even where its own trigger did not fire.

## Measurement Discipline

Every finding states how its cost was established:

| Evidence | Meaning |
| --- | --- |
| `measured` | A profile, benchmark, or timing was actually read. Name the tool and the machine |
| `estimated` | Derived from the code's shape and a stated input size, with no profile |
| `inferred` | The source was not read - only the diff or a filename. The weakest form; never the sole basis for a `[Must]` |

**An `inferred` finding cannot be `[Must]`.** Raise it as `[Recommend]` and state what reading would confirm it. This cap exists because a perf claim with no evidence costs the reader more than silence.

State the input scale a finding assumes. "Slow" without a file count, image size, or item count is not a finding.

## Excluded Surfaces

`target/`, `dist/`, generated bindings, and vendored third-party sources are never findings. A debug-build timing is not evidence for a release-build claim - `image` decode in particular is dramatically slower in debug builds because its SIMD paths are disabled, so a debug measurement of decode cost is not a finding.

## Invocation

| Form | Meaning |
|------|---------|
| `/task-desktop-review-perf` | Standalone: resolve the change set, review, write the report |
| `/task-desktop-review-perf --staged` | Standalone against the staged change set |
| `/task-desktop-review-perf deep` | Force `deep` depth |

When invoked as a subagent (e.g. by `task-desktop-review`), the parent supplies the precondition handle, the pre-read diff, the depth level, the detected project shape, and the excluded-surface list: Step 3 is skipped, no git is re-run, and Step 11 returns findings instead of writing - the parent owns the report.

## Workflow

### Step 1 - Behavioral Principles

Use skill: `behavioral-principles`. Accept the parent's confirmation if invoked as a subagent.

### Step 2 - Stack and Project Shape

Accept the project shape from the parent when invoked as a subagent. Otherwise read `Cargo.toml`; if it is absent, stop - this workflow reviews Rust projects only.

Record the pinned Iced version, the async runtime, whether `rayon` is present, and the release profile settings (`opt-level`, `lto`, `codegen-units`, `debug-assertions`). A release profile with `opt-level = 0` or `debug-assertions = true` changes every cost claim in this review; state it if found.

### Step 3 - Resolve the Change Set

Use skill: `review-precondition-check`. Forward `--staged` if passed. Read `git diff <base>` once and reuse. Restrict analysis to the handle's `reviewable` paths, minus the excluded surfaces above.

**Skip entirely** when invoked as a subagent and the parent passed the handle plus the pre-read diff.

### Step 4 - Read the Performance Surface

Use skill: `desktop-performance`. Establish what the change actually costs before judging it: which paths are per-item, what the expected item count is, and where the work runs.

Classify every added path as one of: **per-item** (runs once per file, image, or row), **per-batch** (runs once per operation), or **per-frame** (runs in `update`, a view function, or a subscription tick). The same code costs differently in each.

### Step 5 - UI-Thread Blocking

The highest-value finding this lens makes. Filesystem I/O, hashing, decode, a lock acquisition, or any unbounded loop inside `update` or a view function freezes the window - the user sees a hung app, and Windows will paint it "Not Responding".

Use skill: `iced-async-patterns` for the correct dispatch. Check that long work is a `Task` or subscription, that progress arrives as messages rather than a polled block, and that cancellation is observed rather than awaited.

A view function that formats, sorts, or filters a large collection on every redraw is this defect in a slower form.

### Step 6 - Traversal and I/O Cost

Use skill: `desktop-filesystem-patterns` for traversal shape. Check:

- A directory walked more than once per operation, where one walk would serve
- `metadata()` or `symlink_metadata()` called repeatedly for a path already stated
- Traversal that reads file contents when only size or mtime is needed
- Per-entry allocation of a `PathBuf` where a borrowed `Path` would do
- I/O ordering that ignores physical placement on a spinning disk, where the scan is large enough to matter
- A watcher firing per-event without debouncing

### Step 7 - Hashing and Comparison Strategy

The dominant cost in a dedup workload, and the place where an algorithmic fix beats any micro-optimization.

Check for the **two-tier prefilter**: group by size first, then hash only within collision groups, and read a partial prefix before the full file. A change that full-hashes every file in a tree has an algorithmic defect, not a constant-factor one - state the file-count scale at which it becomes user-visible.

Check the hash choice: a cryptographic hash where a fast non-cryptographic one suffices is wasted throughput. Content identity for dedup does not require collision resistance against an adversary; state the exposure if the app treats hash equality as proof for a destructive action.

### Step 8 - Decode, Thumbnails, and Caching

Use skill: `desktop-image-processing`. Check:

- Full-resolution decode where a thumbnail-sized decode would serve
- Decode on the UI thread
- A thumbnail cache with no eviction policy - unbounded growth is a memory defect that presents as a slowdown
- Re-decoding on every scroll rather than caching the decoded texture
- A cache keyed so that it never hits

Use skill: `desktop-gpu-compute` when pixel work is heavy enough that the GPU is the right answer, and check that the wgpu device is Iced's rather than a second one.

### Step 9 - Allocation, Cloning, and Collections

Use skill: `rust-language-patterns` for the mechanics. Check:

- `clone()` on a large owned value inside a per-item loop
- `to_vec()`, `to_owned()`, or `collect()` materializing a collection an iterator would stream
- A `String` built by repeated `push_str` in a loop without capacity
- `Vec` growth without `with_capacity` where the size is known
- A collection holding every scanned entry when the UI only renders a window of them
- `Arc<Mutex<>>` contention on a per-item path where partitioned ownership would avoid the lock

### Step 10 - Concurrency, Startup, and Evidence

Use skill: `desktop-concurrency-patterns`. Check parallelism actually helps: a `rayon` pool over an I/O-bound walk can be slower than a sequential one on a single spinning disk, and thread-per-item is a defect at any scale. Check backpressure - an unbounded channel from a fast producer to a slow consumer is a memory defect.

Check startup cost: work done before the first frame is the most visible latency the app has. A scan, a migration, or a cache warm on the startup path belongs behind the first paint.

**Windows GPU startup.** wgpu may find no adapter under a VM, and defaults to the low-power iGPU on hybrid laptops. Iced's backend configuration is environment-variable-only, so a change that assumes a discrete GPU or a specific backend without a `tiny-skia` fallback path is a finding.

Then assign evidence to every finding per the Measurement Discipline table.

### Step 11 - Write Report

**Standalone only.** A subagent run returns the `## Findings` sections and, at `deep`, the `## Measurement Plan` - nothing else. No frontmatter, no Summary block, no Recommendations, no Next Steps: the parent owns those and cannot merge two of them. Project-shape values the parent already supplied are not echoed back.

A subagent that has something for `Unattributed` - a reported symptom the change set does not explain, or a profile-revealed condition it did not cause - returns it as a final `## Unattributed` section for the parent to fold into its Summary. Intent (`[Must]` / `[Recommend]`) is stated per finding so the parent can order Next Steps without re-deriving it.

Standalone: write to `review-perf-<branch>.md` in the current working directory, overwriting if it exists. Sanitize `<branch>` from the handle's `current_branch`: replace `/` and any character outside `[A-Za-z0-9_-]` with `-`, collapse consecutive `-`, strip leading and trailing `-`.

```yaml
---
branch: <branch>
scope_mode: working-tree | staged-only | last-commit
files: <N>
scope: perf
depth: standard | deep
generated_at: <ISO 8601 UTC timestamp>
---
```

After writing, print exactly one confirmation line:

```
Report written to review-perf-<branch>.md (<N> files, scope: perf)
```

## Output Format

The fence below delimits the template for display only. Emit the report body as raw Markdown; never wrap the whole report in a code fence.

```markdown
## Rust Desktop Performance Review Summary

**Assessment:** Approve | Request Changes | Discuss
**Release Profile:** opt-level=<N>, lto=<setting>, debug-assertions=<bool> | not read
**Dominant Cost:** [the single largest cost this change adds, in one line]
**Scale Assumed:** [the item count, file size, or tree size the findings assume]
**Unattributed:** [reported symptoms this change set does not explain, and any profile-revealed condition the diff did not cause - each with what would attribute it | none]

## Findings

### High Impact

#### [Must] file:line
- **Issue:** [the pattern, named]
- **Cost:** [what it costs, at the stated scale]
- **Budget:** [work throughput | UI responsiveness | both]
- **Evidence:** measured (<tool>, <machine>, release) | estimated (no profile) | inferred (no source read)
- **Fix:** [concrete Rust change]

### Medium Impact / Low Impact
[same shape, `[Recommend]`]

## Recommendations

[Ordered by cost removed per unit of change. Algorithmic fixes before constant-factor ones.]

## Measurement Plan _(deep depth only)_

[Which input scale to measure at, which profile or benchmark to take, and the number that decides whether the fix worked. A top-level section, so a subagent returns it without its parent]

## Next Steps

[Standalone only - the parent owns this when invoked as a subagent]
```

**Omit empty sections.** No High Impact heading if there are none.

## Self-Check

- [ ] `behavioral-principles` loaded (or accepted from parent)
- [ ] Project shape accepted from parent, or `Cargo.toml` read; release profile recorded
- [ ] Step 3: handle resolved (or received); diff read once; excluded surfaces skipped
- [ ] Every added path classified per-item / per-batch / per-frame
- [ ] UI-thread blocking checked in `update`, view functions, and subscriptions
- [ ] Traversal cost, repeated walks, and per-entry allocation checked
- [ ] Two-tier hashing prefilter checked; algorithmic defects named as algorithmic
- [ ] Decode resolution, decode thread, and cache eviction checked
- [ ] Allocation, cloning, and collection materialization checked
- [ ] Parallelism checked for actual benefit; backpressure checked; startup path checked
- [ ] Windows GPU adapter and `tiny-skia` fallback checked where the change touches rendering setup
- [ ] Every finding states evidence, budget, and the scale it assumes
- [ ] No `inferred` finding raised as `[Must]`
- [ ] No debug-build timing used as evidence for a release claim
- [ ] Step 11: standalone: report written to `review-perf-<branch>.md` with the sanitized branch name and complete frontmatter; confirmation line printed; subagent: Findings (plus the deep-only plan, plus `Unattributed` where there is one) returned to parent, no Summary or Next Steps, no file written
- [ ] Reported symptoms the change set does not explain recorded in `Unattributed` rather than dropped

## Avoid

- Writing a report when invoked as a subagent - the parent owns it
- Returning the whole Output Format as a subagent - only `## Findings`, the deep-only `## Measurement Plan`, and `## Unattributed` are returned
- A `[Must]` resting on `inferred` evidence
- A cost claim with no stated input scale
- A debug-build measurement presented as a release-build cost
- Micro-optimizations recommended ahead of an algorithmic fix in the same change set
- Flagging `clone()` that is not on a per-item path
- Recommending `rayon` for an I/O-bound walk without stating the disk assumption
- Correctness or security findings - route them to the umbrella or the security lens
- Raising findings against `target/`, `dist/`, or vendored sources
