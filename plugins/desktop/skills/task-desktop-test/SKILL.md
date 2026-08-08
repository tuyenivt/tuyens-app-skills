---
name: task-desktop-test
description: C# desktop test strategy and scaffolds - core unit tests, filesystem fixtures, destructive-op and migration coverage, xUnit, CI matrix.
agent: desktop-test-engineer
metadata:
  category: desktop
  tags: [csharp, testing, xunit, filesystem-fixtures, property-tests, migration, ci, workflow]
  type: workflow
user-invocable: true
---

> **Behavioral directive:** Load `Use skill: behavioral-principles` before executing this workflow.

# C# Desktop Test Strategy

Test strategy, coverage assessment, and scaffolds for a C# desktop project. The UI-free core is what makes this cheap - most of what matters is testable without a window.

## When to Use

- A feature needs a test strategy beyond what `task-desktop-implement` writes inline
- Coverage assessment across an existing codebase
- A destructive operation, a migration, or a path-handling routine needs exhaustive cases
- CI wiring for a cross-platform matrix

**Not for:** writing the feature itself (`task-desktop-implement`), reviewing existing tests for style (`task-desktop-review` covers test files for coverage only).

## Rules

- **No test touches the user's real filesystem.** Every filesystem test runs against a temporary directory or an injected seam, and cleans up even when it fails
- **A destructive operation is tested by asserting what survived**, not only what the return value said. A test that checks the outcome list and not the resulting tree does not test the operation
- Tests are deterministic: no wall-clock time (inject `TimeProvider` or a fixed instant), no unseeded randomness, no dependence on filesystem enumeration order
- A test that needs a GUI to run is a design finding about the code, not a test to write
- Platform-divergent behaviour is tested per platform, gated visibly (`[SkippableFact]`-style runtime skips, or `#if` where the case cannot compile elsewhere), not asserted once and assumed portable
- New suites default to xUnit; a project's existing framework (NUnit, MSTest) is kept. Assertion style follows the suite - FluentAssertions where it is already present, plain `Assert` otherwise; neither is a rewrite target

## Workflow

### Step 1 - Behavioral Principles

Use skill: `behavioral-principles`.

### Step 2 - Project Gate and Shape

Read the `.csproj` files (through the `.sln` where one exists). If none is found, stop - this workflow covers .NET projects only. Record the solution layout, whether the solution references Avalonia (the guidance assumes an Avalonia desktop app; where it does not, state that and keep only the plain-.NET surfaces the request names), whether a UI-free core project exists, the test framework and dependencies already present (xUnit, NUnit, MSTest, FluentAssertions, FsCheck or CsCheck, Verify, BenchmarkDotNet), and the CI configuration if any.

Record the request's scope: whole-project (a strategy or coverage assessment) or targeted (a named operation or module). A targeted request runs Steps 3-8 against the named surface only, and Step 9 only when the request includes CI; out-of-scope report sections are written `n/a - outside the request's scope`.

**No core project, or one that references Avalonia:** report it as the primary finding. Test strategy for logic embedded in ViewModels or code-behind is a rewrite recommendation, not a test plan - say so rather than proposing elaborate UI harnessing. That run still completes the report: Step 3 records where the logic lives, the Findings name the extraction targets - destructive and path-handling routines first, since Step 4 rates them Critical - each with the destination core type and what stays behind, and the plan covers only what is already UI-free, with sections that have nothing UI-free to cover written `n/a - blocked on core extraction`.

### Step 3 - Assess Current Coverage

Read the existing tests before proposing new ones. Classify what exists:

| Layer | What belongs there |
| --- | --- |
| Core unit tests | Plan generation, collision resolution, undo, outcome reporting, path logic, migrations |
| Property tests | Invariants over generated input (FsCheck or CsCheck) - round-trips, idempotence, ordering independence |
| Integration tests | Project composition, a real temporary filesystem end to end |
| Snapshot tests | Stable rendered output (Verify) - a plan's textual preview, a generated report |

State coverage as gaps against risk, not as a percentage. A 90%-covered codebase with no test for the delete path is worse-covered than a 40% one that tests it.

### Step 4 - Prioritize by Blast Radius

Order what to test by what an untested failure costs:

| Priority | Surface |
| --- | --- |
| Critical | Destructive apply paths, undo, migration, path confinement, and the selection logic that decides what a destructive operation touches |
| High | Collision and auto-suffix naming, partial-failure reporting, cancellation |
| Medium | Traversal filters, sorting and grouping, settings round-trips |
| Low | Formatting, display mapping |

A Critical-row surface with no test is a `[Must]` finding in the report, not a suggestion.

### Step 5 - Core Unit Tests

Use skill: `desktop-core-architecture` for what belongs in the core. Use skill: `desktop-batch-operations` for the operation semantics under test.

Cover per operation:

- **Plan generation** produces the same plan for the same input, and the plan is inspectable without applying it
- **Preview equals apply**: the plan the preview showed is the plan apply consumes. This is the single highest-value test in this app type, because drift between the two is invisible until it destroys data
- **Collision resolution**: an existing target, a target that collides with another item in the same batch, a target colliding with a name the batch will create later, auto-suffix generation past `(1)` into `(2)` and beyond, a case-only rename on a case-insensitive filesystem gaining no spurious suffix, and a step whose target is a later step's source - ordered correctly, or routed through a temporary name for a true cycle
- **Undo round-trip**: apply then undo restores the prior state, including for a partially-applied batch
- **Partial failure**: an item that fails mid-batch leaves the remaining items with their own reported outcomes, and the aggregate reflects the mix
- **Idempotence under retry** where the operation may be retried

### Step 6 - Filesystem Fixtures

Use skill: `desktop-filesystem-patterns` for the cases that matter per platform.

Build fixtures on a per-test temporary directory - `Directory.CreateTempSubdirectory()` wrapped in an `IDisposable` fixture - so cleanup runs even when the test fails. Prefer a helper that constructs a tree from a declarative description over hand-written `CreateDirectory` chains - the fixture should read as the scenario.

Cover, gated where the case is platform-specific:

- Filenames outside ASCII, including combining characters and names an encoding round-trip would corrupt
- Windows: reserved device names, trailing dots and spaces, paths past 260 characters, and case-insensitive collision where two names differ only in case
- macOS: Unicode normalization, where a filename written as NFC reads back as NFD - a rename tool hits this directly, and a test asserting byte equality on a round-tripped name will fail here for a correct implementation
- Symlinks and, on Windows, directory junctions - including a cycle, to prove the traversal bound holds
- Hardlinks, for a dedup path that must not delete both names of one file
- Permission-denied entries mid-traversal, proving the walk continues and reports rather than aborting
- An empty directory, a single file, and a deeply nested tree

Where a case cannot run in the environment (symlink creation needs Developer Mode or elevation on some Windows configurations), gate it with a runtime skip and say it is gated rather than silently skipping - a gated case reports as skipped, never as absent.

### Step 7 - Property and Migration Tests

Use FsCheck or CsCheck where an invariant is stronger than an example:

- Apply-then-undo returns the original tree, for any generated batch
- Auto-suffix naming never produces a duplicate within one batch, for any generated collision set
- Plan generation is order-independent where the operation claims to be
- A path confined to a root stays confined, for any generated component sequence including `..` and separators

A test-only package the suite lacks (FsCheck, CsCheck, a runtime-skip package, Verify) is proposed in the Test Plan with the package named, never added by this run - scaffolds ship against what the project already references, designed around the gap where a case allows it.

Use skill: `desktop-data-persistence` for migrations. **Every shipped schema version gets a fixture database and a test that migrates it forward to current.** Construct a version fixture programmatically - the old version's DDL plus its `user_version` - unless a shipped binary artifact is itself the regression surface. These accumulate; an old version's fixture is never deleted, because an installed user may still be on it. A migration test that starts from the current schema tests nothing. The empty database is version 0's fixture - a test that walks an empty file 0 -> current covers the fresh-install path.

### Step 8 - Concurrency and Cancellation

Use skill: `desktop-concurrency-patterns`.

- Cancellation observed mid-operation leaves a consistent state, and the reported outcome says what was applied before the stop
- Progress reporting arrives in order and totals correctly
- A parallel scan produces the same result set as a sequential one over the same tree - run both and compare, since completion-order differences are the usual source of flakiness here
- Backpressure: a fast producer with a slow consumer does not grow unbounded

Determinism note: a parallel test that asserts on ordering will flake. Assert on the set, or sort before comparing.

### Step 9 - CI Matrix

Use skill: `desktop-build-release` for the packaging and signing side.

The matrix runs Windows and macOS. Windows is primary, so it runs on every push; macOS runs at least on every merge to the default branch.

```yaml
strategy:
  matrix:
    os: [windows-latest, macos-latest]
```

Per job: `dotnet format --verify-no-changes`, `dotnet build -warnaserror`, `dotnet test`, and a Release publish. Add `dotnet list package --vulnerable` on a schedule rather than per push.

Two practical notes: macOS runners bill at a 10x multiplier on private repos and are free on public ones, so a private-repo matrix that runs macOS on every push burns the allowance quickly - gate the macOS leg there by building the `os` list conditionally (`fromJSON` on `github.ref`) or splitting macOS into its own job with an `if:` on the default branch. And a NativeAOT publish needs the platform's native toolchain on the runner - the C++ build tools on Windows, Xcode Command Line Tools on macOS - so a publish leg that fails at link time in CI and nowhere else is usually this.

Where Step 2 found the core boundary broken, the matrix also enforces the fix: a per-job core-purity gate that fails when the core project's transitive dependency tree contains Avalonia (`dotnet list <core>.csproj package --include-transitive` piped through a grep for Avalonia).

### Step 10 - Report

Write the strategy as the report below, delivered inline in the reply - this workflow writes no report file unless the request names a destination. Write scaffolds when the request asked for tests to be written; a strategy or assessment question gets the plan and no unrequested files. Scaffolds are runnable test methods with real assertions, not `NotImplementedException` stubs - a stub that compiles gives false coverage confidence. Scaffolds live in the solution's test projects, mirroring the namespaces they cover; create `<Core>.Tests` when none exists rather than putting tests in a production project.

A test that pins intended behaviour against a defect this run found is written against the intent and skipped (`[Fact(Skip = "...")]`) with the defect and its finding named in the skip reason - never asserted green against the bug, never left failing in the suite.

## Output Format

```markdown
## Project Shape
- Solution: {layout}
- Avalonia app: {yes | no - stated, desktop-specific guidance dropped}
- Core project: {name | none - primary finding | references Avalonia - primary finding}
- Test framework and dependencies present: {list | none}
- CI: {configured | absent}
- Request scope: {whole-project | targeted: <surface>}

## Coverage Assessment
| Surface | Priority | Current | Gap |
[one row per concrete surface the project contains - an operation, store, or routine (`RenameExecutor apply`, `ScanHistoryStore migrations`) - tagged with its Step 4 priority. A surface living outside the core (ViewModel-resident) is still a row; its Current cell is `untestable - lives in <file>`. A surface class the project does not contain is omitted, never written `n/a`. A targeted run lists only surfaces inside the request's scope]

## Findings
[`[Must]` for an untested Critical surface, `[Recommend]` otherwise. Each with `file:line` and the case to cover. A production defect found while reading is named inside the finding covering its surface - whether or not that surface is also a coverage gap - with a pointer to `task-desktop-review` for the review pass; a defect in a file outside a targeted request's scope gets its own one-line `[Recommend]` marked `out of scope - observed`, pointer only. A finding whose cases this run's scaffolds cover stays listed with `- closed by scaffolds written this run` appended, or `- partly closed by scaffolds written this run: <what remains uncovered, and why>` where coverage is partial. `none - all Critical surfaces covered` when there are none]

## Test Plan
| Layer | Count proposed | Covers |
[one row per Step 3 layer plus a `Concurrency` row for Step 8's cases; a row with nothing to propose writes `0 - {reason}` (`0 - no parallel path`, `0 - blocked on core extraction`) rather than being omitted, and a row whose cases split writes the count it proposes and names the uncovered cases with their reason]

## Scaffolds Written
[file -> what it covers. `none - request did not ask for tests to be written` when it did not]

## Fixture Strategy
- Temp directory: {helper and cleanup approach}
- Platform-gated cases: {which, and why gated}
- Cases that cannot run here: {which, and what would run them | none}

## Migration Coverage
| Schema version | Fixture | Test |
[`n/a - no persisted schema` when there is none. In a strategy run the Fixture and Test cells describe what is proposed, each marked `(proposed)`, not files that exist]

## CI Matrix
[the matrix, per-job commands, and the cost note where the repo is private or its visibility cannot be determined locally. `not proposed - CI already configured` when it is; `n/a - outside the request's scope` for a targeted request that does not include CI]

## Determinism Risks
[anything in the proposed suite that could flake, and what makes it deterministic. `none` when there are none]
```

Every slot is written. A section that does not apply is written as `n/a - {reason}` rather than omitted.

## Self-Check

- [ ] `behavioral-principles` loaded
- [ ] `.csproj` read; absence stopped the workflow
- [ ] Request scope recorded; a targeted request narrowed the run, out-of-scope sections written `n/a`
- [ ] Missing or Avalonia-referencing core project reported as the primary finding rather than harnessed around
- [ ] Existing tests read before new ones proposed
- [ ] Coverage stated as gaps against risk, not as a percentage
- [ ] Surfaces prioritized by blast radius; untested Critical surfaces raised as `[Must]`
- [ ] Preview-equals-apply covered for every destructive operation
- [ ] Collision (including case-only rename and step-ordering cycles), auto-suffix past `(2)`, undo round-trip, and partial failure covered
- [ ] Fixtures use a temp directory that cleans up even on failure; no test touches real user files
- [ ] Non-ASCII names, Windows reserved names and long paths, macOS normalization, symlinks, junctions, hardlinks, and permission-denied covered or explicitly gated
- [ ] Property tests proposed where an invariant beats an example
- [ ] Every shipped schema version has a fixture and a forward-migration test
- [ ] Cancellation, progress ordering, parallel-equals-sequential, and backpressure covered, or the Test Plan's `Concurrency` row states why not
- [ ] CI matrix covers Windows and macOS with format, build, test, and a Release publish
- [ ] Scaffolds written only when the request asked for tests, and contain real assertions, not stubs
- [ ] Determinism risks named

## Avoid

- Tests that touch the user's real filesystem
- A destructive-operation test that asserts only the return value and not the resulting tree
- `NotImplementedException` or empty-body scaffolds presented as coverage
- Coverage reported as a percentage without naming which Critical surfaces are uncovered
- A migration test that starts from the current schema
- Deleting an old schema fixture because the version is old - installed users are still on it
- Asserting byte equality on a macOS round-tripped filename
- Asserting ordering on a parallel result set
- Wall-clock time or unseeded randomness in a test
- Proposing a UI harness for logic that should move to the core project
- Testing Avalonia views instead of keeping them thin
- Recommending a macOS-on-every-push matrix for a private repo without naming the 10x cost
