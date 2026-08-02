---
name: task-unity-test
description: Unity test strategy and scaffolding - EditMode rules coverage, PlayMode limits, seeded determinism, NSubstitute, test asmdefs, batch-mode CI.
agent: unity-test-engineer
metadata:
  category: mobile
  tags: [unity, testing, edit-mode, play-mode, unity-test-framework, nsubstitute, determinism, asmdef, ci, workflow]
  type: workflow
user-invocable: true
---

> **Behavioral directive:** Load `Use skill: behavioral-principles` before executing this workflow.

# Unity Test

Unity-aware test strategy and scaffolding across EditMode and PlayMode, built on the engine-free rules assembly, with determinism and batch-mode CI treated as first-class concerns.

## When to Use

- Test strategy for a new Unity 2D game or feature
- Coverage-gap assessment across the rules core and the engine glue
- Scaffolding tests for an under-covered rules layer, save migration, or screen
- Test pyramid review
- Adding failure-path tests to happy-path-only tests
- Diagnosing a test that passes locally and fails in CI, or passes alone and fails in a suite

**Not for:** debugging a failing test whose production code is wrong (`unity-engineer`), general review (`task-unity-review`).

## Workflow

### STEP 1 - STACK AND PROJECT SHAPE

Accept the project shape from a parent workflow when invoked as a subagent. Otherwise read `ProjectSettings/ProjectVersion.txt` and `Packages/manifest.json` to confirm Unity and the test framework present.

Read `ProjectSettings/ProjectVersion.txt` and compare `m_EditorVersion` numerically against the **Unity 6.3 LTS floor (`6000.3.x`)**. Below the floor, state the detected version and the floor and **STOP** - guidance for a version the project cannot run is worse than none. Above the floor proceeds normally. Unreadable or absent file: say so and ask, rather than assuming.

Record from `Packages/manifest.json` and the project: Test Framework presence, substitution library (NSubstitute / hand-written fakes / none), whether Enter Play Mode Options is enabled and domain reload disabled, UI system, persistence store, and whether Addressables is in use.

Any of these that cannot be read is recorded as `unknown` and carried forward as `unknown` in the deliverable - never silently defaulted. Where a recorded value gates later guidance, `unknown` takes the safe branch and says so: unknown domain reload is treated as disabled, so statics are reset in `[SetUp]` regardless.

### STEP 2 - READ CODE UNDER TEST AND EXISTING TESTS

Ground output in project conventions, not generic templates.

- Read every `.asmdef`: which assemblies exist, what each references, and whether a rules assembly free of `UnityEngine` already exists. That answer determines the whole strategy - a project with no engine-free assembly cannot have a fast bulk layer until one exists, and that is the finding to lead with
- Read the target top-to-bottom: the rules types, the MonoBehaviours wiring them, the save DTO and its migration chain
- Glob `**/Tests/**/*.cs` and `**/*Tests.asmdef`; read one existing EditMode test, one PlayMode test, and any shared setup - learn the project's assertion style, fixture construction, and fake conventions
- Read CI configuration for how tests are invoked, whether EditMode and PlayMode run in separate jobs, and how results surface
- Check whether `Library/` is cached between CI runs

If no existing tests: say so and propose conventions explicitly rather than inventing them silently.

**When no engine-free assembly exists**, every target below P5 is blocked on creating one, and a whole-project re-layering is not the recommendation. Scope the extraction to the P1-P3 targets only (the save DTO and its migration chain, the rules the game is graded on, the monetized paths), leave the remaining scripts in `Assembly-CSharp`, and say which files move. State the rejected alternative - testing those rules in PlayMode where they currently sit - and why it loses: a domain reload and scene load per run, forever, and ambient time or randomness still cannot be injected. Write the extraction as the first entry in `Gaps to close`, not as a precondition that stops the deliverable.

### STEP 3 - THE UNITY 2D TEST PYRAMID

| Layer | Tooling | What belongs | Speed |
|-------|---------|--------------|-------|
| EditMode - rules | Unity Test Framework, rules asmdef only | move legality, state transitions, scoring, cascade termination, undo round-trips, economy math, save migration | milliseconds; no scene, no Play mode |
| EditMode - editor tooling | UTF + editor APIs | content-bank import validation, asset validation, build-config assertions | fast |
| PlayMode | UTF + `[UnityTest]` | lifecycle order, scene load, prefab wiring, coroutine sequences, a screen's state rendering | seconds each |

**The bulk is EditMode on the rules assembly.** This is the payoff of the engine-free architecture rule: the same logic tested in PlayMode costs a domain reload and a scene load per run, so a project that pushes rules into MonoBehaviours pays for it here, every run, forever.

PlayMode count stays deliberately low. A rule tested in PlayMode is a rule in the wrong assembly - fix the layering rather than the test.

### STEP 4 - DETERMINISM

Non-determinism in a Unity suite is not flakiness to retry; it is a design defect in the code under test.

- **Seeded randomness.** The rules layer takes an injected `IRandom`; tests construct it with a fixed seed. A test touching `UnityEngine.Random` is unreproducible, and its failure cannot be replayed. Assert on a specific sequence only where the PRNG is one the project owns - `System.Random`'s exact sequence is not guaranteed stable across runtime versions
- **Injected clock.** Time-dependent logic takes an `IClock`; tests advance a fake clock rather than waiting. Offline-progress and cooldown tests that sleep are slow and still wrong
- **No frame timing in logic tests.** A rule that needs `Time.deltaTime` to be tested belongs in the rules layer taking a fixed step as a parameter. `yield return null` is for tests of engine behaviour, never for letting a rule "settle"
- **No wall-clock assertions.** `DateTime.UtcNow` inside a test makes it fail at a timezone boundary or on a slow agent

Determinism cases worth having explicitly: same seed produces the same board, replay of a move list reproduces the final state, undo returns exactly the prior state, and a cascade terminates at or below its stated bound.

### STEP 5 - SUBSTITUTION

**The rules layer needs no mocking framework.** Its dependencies are interfaces the test implements directly - a `FakeClock` advancing on command, a `SeededRandom` with a fixed seed, an in-memory save store. Hand-written fakes are clearer in the assertion and cheaper than a mock configuration.

NSubstitute is for the seams where a hand-written fake is more work than the test: a third-party SDK interface, an analytics sink, a remote-config source, or asserting that a call happened at all.

**NSubstitute's presence in the manifest is not a reason to use it.** The test is cost, not category: reach for it only when the fake would be longer than the substitute configuration. A seam the test can implement in a handful of lines - clock, RNG, in-memory save store - is hand-written even when the seam would also qualify by category, and even when the assertion is that a call happened.

```csharp
// Bad - a mock configured over three lines to return a value a fake returns in one
var clock = Substitute.For<IClock>();
clock.UtcNow.Returns(new DateTime(2026, 1, 1));

// Good - a fake the test can also advance
var clock = new FakeClock(new DateTime(2026, 1, 1));
clock.Advance(TimeSpan.FromHours(3));
```

`UnityEngine.Object`-derived types cannot be substituted usefully - they are engine-constructed. Test against an interface the MonoBehaviour delegates to, which is the same seam the production composition root uses.

### STEP 6 - PLAYMODE, COROUTINES, AND ASYNC

`[UnityTest]` returns `IEnumerator` and yields frames, which is what makes it PlayMode-capable:

- `yield return null` advances one frame; use it to let a lifecycle callback or a coroutine step run
- Prefer yielding a **bounded** number of frames over waiting for a condition with no cap. An uncapped wait on a condition that never becomes true hangs the run rather than failing it usefully
- `yield return new WaitForSeconds(...)` uses scaled time and never completes at `Time.timeScale = 0`; use the realtime variant in any test that pauses
- Async methods returning `Task` are supported as test bodies by the Test Framework; verify the supported signature against the installed package version rather than assuming. Every awaited call in a test carries a timeout, so a hung await fails instead of stalling the job
- A PlayMode test that instantiates prefabs or loads scenes tears them down in `[TearDown]` or `[UnityTearDown]`. Leftover objects leak into the next test in the same run

### STEP 7 - TEST ASSEMBLY DEFINITIONS

This is the mechanism that keeps the fast tests fast, and getting it wrong silently converts the whole suite into slow tests.

| Assembly | References | Platforms | Runs in |
| --- | --- | --- | --- |
| `Game.Tests.EditMode` | `Game.Rules`, `nunit.framework.dll`, `UnityEngine.TestRunner`, `UnityEditor.TestRunner` | Editor only | EditMode |
| `Game.Tests.PlayMode` | `Game.Rules`, `Game.Runtime`, `UnityEngine.TestRunner` | all target platforms | PlayMode |

- The EditMode test asmdef references the **rules assembly only**. Adding a reference to the runtime assembly pulls the engine into the fast layer and undoes the point of the split
- A test asmdef needs `"Test Assemblies"` enabled (the `UNITY_INCLUDE_TESTS` define constraint) so it is excluded from player builds - otherwise tests and NSubstitute ship in the shipped artifact
- Editor-only test assemblies list `Editor` as their only platform; a PlayMode assembly must not reference `UnityEditor` types or it will not compile for a device run
- Precompiled references (NSubstitute) are declared on the assemblies that use them, not globally

### STEP 8 - CI ON BATCH MODE

```bash
Unity -batchmode -nographics -runTests \
  -projectPath "$PROJECT" \
  -testPlatform EditMode \
  -testResults "$RESULTS/editmode.xml" \
  -logFile -
```

- `-testPlatform` takes `EditMode` or `PlayMode`; run them as **separate invocations**, so the fast EditMode job gates every pull request and the slower PlayMode job can run on a narrower trigger
- `-runTests` implies the editor quits after the run - do not add `-quit`, which can terminate before results are written
- **Read the `-testResults` XML for the verdict.** The process exit code is not a reliable pass/fail signal in batch mode; a run that never started can exit zero
- `-logFile -` streams to stdout so CI captures the failure evidence
- **Licence activation** runs before the tests and **return** runs after, including on the failure path - an unreleased seat blocks the next run
- **Cache `Library/`** between runs. A cold agent pays a full asset import and script compile before a single test executes, which usually dominates the job
- `-nographics` is appropriate for EditMode; a PlayMode test that renders or reads back from the GPU may need it removed. Removing it reflexively hides a real failure, so change it only against a specific symptom
- Pin the Unity version to the exact internal version from `ProjectVersion.txt`, not a floating major

### STEP 9 - FLAKE CONTROL

Four causes account for nearly all Unity test flake. Diagnose to a named cause before touching the test:

| Symptom | Cause | Fix |
| --- | --- | --- |
| Passes alone, fails in a suite; second run differs from first | **Static state leaking between tests** - domain reload is commonly disabled, so statics and static event subscriptions survive across tests in one run | Reset statics in `[SetUp]`; the production reset via `[RuntimeInitializeOnLoadMethod]` does not run per test (`unity-monobehaviour-lifecycle`) |
| Passes locally, fails on a slower CI agent | **Frame-timing dependence** - the test waits a fixed frame count for work that takes longer under load | Assert on a state change with a bounded wait, not on a frame count |
| Fails depending on which test ran before it | **Scene load order and leftover objects** - a prior test left objects, a `DontDestroyOnLoad` instance, or an additively loaded scene alive | Tear down every instantiated object and unload every additive scene; do not depend on test execution order |
| Fails only in a device or release run | **Stripping or backend divergence** - reflection-dependent code the linker removed | Reproduce on a release build; preserve the type (`unity-build-release`) |

Retrying a flaky test hides one of these four. Quarantine with an issue and a named cause, never with a bare retry.

**One symptom set commonly has several of these at once** - differing failure values across runs are the fingerprint of two causes, not one. Diagnose each independently, report every cause that fires, and order the fixes so the cheapest one that unblocks the others goes first.

Two causes sit outside the four rows:

- **Global engine state left non-default by a prior test** - `Time.timeScale` at 0 from an unclosed pause popup, a changed fixed timestep, a loaded additive scene. It behaves like row 1 but the state is the engine's, not the project's: reset it in `[SetUp]` and fix the test that left it. `WaitForSeconds` never completes at `timeScale = 0`, so this reads as a hang rather than a failure.
- **Harness false-green** - the suite reports success when it did not run: a `-quit` that truncates before results are written, exit-code gating, or a job whose `-testPlatform` does not include the failing test. Diagnose against STEP 8. When the reported symptom is a false pass rather than a failure, this is the cause.

Where one fix closes two causes, report both causes and state the shared fix once.

### STEP 10 - COVERAGE TARGETS

State targets by layer, not as one global number, and say the split out loud:

| Layer | Target | Why |
| --- | --- | --- |
| Rules assembly | High - this is where the coverage number means something | Pure, fast, deterministic; every branch is cheap to reach |
| Save migration and content validation | High | Each shipped save version is a fixture; a gap here is player data loss |
| MonoBehaviour glue and presenters | **Low by design** | Delegating wiring with no branches. Chasing coverage here buys slow PlayMode tests that assert the engine works |
| Generated code, editor tooling, third-party | Excluded | Not the project's logic |

**Say the glue target explicitly in the deliverable.** A reviewer who sees 40% overall without the split reads it as under-tested, and the reflex fix - PlayMode tests over the glue - makes the suite slower and no safer.

Measure rather than guess: run the project's coverage tooling scoped to the rules assembly when it runs locally; when it cannot, estimate from test-file density and label the number an estimate.

**Prioritization when coverage is low** - run this before scaffolding, since it decides what is written first:

| Priority | Targets |
|----------|---------|
| P1 - Data integrity | Save migration from each shipped version, corrupt-save recovery, content-bank import validation |
| P2 - Rules correctness | Move legality, cascade termination, undo/replay, scoring, economy accrual and its clamps |
| P3 - Monetized and irreversible | IAP grant and restore, prestige reset, currency spend paths |
| P4 - High-churn | Files with frequent recent commits (`git log --since="3 months ago"`) or bug-fix history |
| P5 - Presentation glue | Screen wiring and presenters - low risk, low target, can wait |

**Multi-band rule.** When a target qualifies for multiple bands, file it under the highest (lowest number) and note the secondary so the plan covers both axes.

## Output Format

**Which output to produce:**

- "What tests are missing?" -> Coverage Assessment
- "Write tests for X" / "scaffold" -> Test Scaffolds
- "Why does this test fail in CI / only in a suite?" -> Flake Diagnosis
- "Test strategy" / "test plan" -> Strategy Doc
- Unclear -> default to Strategy Doc

More than one rule matching is the normal case, not a conflict: produce every deliverable that matched, in this order, separated by `---`: Coverage Assessment -> Flake Diagnosis -> Strategy Doc -> Test Scaffolds. A coverage question against a project with no engine-free assembly always matches both Coverage Assessment and Strategy Doc, because the extraction plan is the answer and only the Strategy Doc holds it.

Where two deliverables share a field, state it once in the deliverable that owns it and reference it from the other rather than restating. The prioritized list is owned by the Strategy Doc's `Gaps to close`: a Coverage Assessment emitted alongside a Strategy Doc replaces its `Prioritization` section with a one-line pointer to `Gaps to close`.

Every deliverable opens with the engine gate line:

```
**Engine:** {m_EditorVersion} | Floor 6000.3.x | {Above floor | Below floor - stopped | Unknown - asked}
```

**Coverage Assessment:**

```markdown
## Unity Test Coverage Assessment

**Engine:** {version} | Floor 6000.3.x | {result}
**Engine-free rules assembly:** {name, or "none - no fast layer exists"}
**Lead finding:** {the one sentence that gates the plan - when no fast layer exists, that it must be extracted and which files move; otherwise the largest untested surface}
**Substitution:** NSubstitute | hand-written fakes | none
**Domain reload:** disabled | enabled | unknown
**Coverage gaps:**

- **EditMode - rules:** [rule types, transitions, or bounds without coverage]
- **EditMode - data:** [save versions without a migration fixture; banks without import validation]
- **PlayMode:** [lifecycle, scene-load, or screen-state paths without coverage]
- **Determinism:** [logic reading ambient time or randomness, so it cannot be tested]
- **Assembly structure:** [test asmdefs referencing more than they need; missing test-assembly flag]

**Recommended balance:** EditMode rules [target - the bulk] / EditMode data [target] / PlayMode [target - keep small] _(targets for the suite to reach, never a measurement of it)_
**Measured coverage:** {tool, scope, and figure | "not measured - <reason>"; an estimate from test-file density is labelled an estimate}
**Coverage targets:** rules [high] / migration [high] / MonoBehaviour glue [low by design]

**Prioritization** _(include when coverage is low or gaps exceed 5)_:

1. **P1 - Data integrity:** [migrations, corrupt-save recovery, bank validation]
2. **P2 - Rules correctness:** [legality, termination, undo, economy]
3. **P3 - Monetized and irreversible:** [IAP grant and restore, prestige, currency spend]
4. **P4 - High-churn:** [files with frequent recent commits or bug-fix history]
5. **P5 - Presentation glue:** [screen wiring, presenters]
```

**Flake Diagnosis:**

```markdown
## Flake Diagnosis: {test name}

**Engine:** {version} | Floor 6000.3.x | {result}
**Causes found:** {n}

### Cause {n} - {name from the STEP 9 table, or one of its two out-of-table causes}

- **Evidence:** [the symptom detail that identifies this cause - a differing failure value, a static that survives, the exact CI flag]
- **Fix:** [the change, in code or CI configuration]

### Order to apply

1. [cheapest fix that unblocks the others first; say what each unblocks]

**Verification:** [how to prove the flake is gone - repeat count, shuffled order, the CI run that must now fail on a broken test]
```

No strategy doc, no coverage assessment, no pyramid. Every cause found gets its own section.

**Test Scaffolds:** ready-to-run files using project conventions:

- Right layer for the behaviour - EditMode unless engine behaviour is genuinely under test
- The test asmdef stated or created, with its references listed and the test-assembly flag set
- Grouped cases with descriptive names, not copy-pasted bodies
- Fixture builders with sensible defaults and named overrides, over repeated literals
- Fixed seeds and a fake clock; no ambient time or randomness
- Failure paths alongside the happy path - illegal move, corrupt save, cascade at its bound, clock moved backwards
- Bounded waits in `[UnityTest]`, with teardown for every instantiated object and additive scene
- **Migration fixtures come from real shipped saves** - one captured file per shipped version, kept under the test assembly. A hand-written payload encodes the shape the team *believes* shipped, which is the same assumption the migration already makes, so it cannot catch a mismatch. Obtain one by running a build of the old tag once, pulling a QA device's profile from `persistentDataPath`, or taking a support-ticket attachment; capture the current version's file at each release tag so the next migration is never in this position. Where none can be obtained, mark the synthetic fixture an unverified shape and make a missing fixture **skip loudly rather than pass** - a green suite over absent fixtures is what ships the data loss

Every scaffold deliverable ends with this block:

```markdown
**Verification:** {ran - N passed, read from <results path> | not run - <reason>}
{when not run: what is therefore unverified - compilation, the assertions, and the project symbols the scaffold assumes exist}
```

Run the EditMode suite in batch mode and read the `-testResults` XML wherever an editor exists; that is the only way `ran` may be written. Where none exists, `not run - <reason>` is the complete and correct value and the scaffold ships with it, labelled a scaffold rather than a passing suite. Delivering with an unrun block filled in is the defect; delivering with `not run` is not.

**Strategy Doc:**

```markdown
## Unity Test Strategy

**Engine:** {version} | Floor 6000.3.x | {result}
**Objective:** [what this strategy achieves]
**Pyramid balance:** EditMode rules {x}% / EditMode data {y}% / PlayMode {z}%
**Assembly plan:** [rules asmdef -> EditMode test asmdef; runtime asmdef -> PlayMode test asmdef]
**Tooling:** Unity Test Framework, NSubstitute [where a fake is insufficient], hand-written fakes for IClock/IRandom
**Determinism policy:** [seed source, clock injection, what may never be asserted on frame timing]
**Coverage targets:** rules [high] / migration [high] / glue [low by design]
**CI:** [EditMode and PlayMode invocations, licence handling, Library cache, where results are read]
**Flake policy:** [quarantine requires a named cause from the four classes; no bare retries]
**Gaps to close (prioritized):**

1. [Highest risk - typically save migration or an untestable rules layer]
2. ...
```

## Self-Check

**Always:**

- [ ] `behavioral-principles` loaded before the workflow ran
- [ ] Engine version compared numerically to `6000.3.x`; below-floor stopped rather than degraded
- [ ] Substitution library, domain-reload setting, and project surfaces recorded; anything unreadable carried as `unknown` on its safe branch
- [ ] `.asmdef` layout read; presence or absence of an engine-free rules assembly stated, and where absent, the extraction scoped to the P1-P3 targets
- [ ] Code under test, existing tests, and CI configuration read directly

**Strategy / Coverage:**

- [ ] Pyramid mapped to Unity layers: EditMode rules as the bulk, EditMode data, PlayMode kept small
- [ ] Determinism policy stated: seeded `IRandom`, injected `IClock`, no frame-timing or wall-clock assertions
- [ ] Substitution boundary stated: fakes for the rules layer, NSubstitute only where a fake costs more
- [ ] PlayMode and async conventions covered: bounded waits, timeouts, teardown
- [ ] Test asmdef plan stated, with the EditMode assembly referencing the rules assembly only and the test-assembly flag set
- [ ] CI invocations given per platform, with licence handling, `Library/` caching, and results read from the XML rather than the exit code
- [ ] Every flake cause present diagnosed and reported, not just the first; harness false-green diagnosed against STEP 8 where the symptom is a false pass
- [ ] Coverage targets split by layer, with the low glue target said explicitly
- [ ] Risk-based prioritization when coverage is low (P1 integrity, P2 rules, P3 monetized, P4 churn, P5 glue)

**Scaffolds:**

- [ ] Placed in the right layer, in an asmdef that compiles for that platform
- [ ] Grouped and descriptive, not copy-pasted
- [ ] Fixture builders over repeated literals
- [ ] Fixed seeds and a fake clock throughout
- [ ] Failure paths covered alongside the happy path
- [ ] Statics reset in setup where domain reload is disabled
- [ ] `[UnityTest]` waits bounded; instantiated objects and additive scenes torn down
- [ ] Verification block present and honest - `ran` only where a suite actually ran, `not run - <reason>` plus the unverified list otherwise

## Avoid

- Scaffolding without reading the `.asmdef` layout and existing tests
- Testing rules logic in PlayMode when an engine-free assembly could test it in EditMode
- An EditMode test asmdef referencing the runtime assembly
- A test assembly without the test-assembly flag, shipping tests into the player build
- `UnityEngine.Random`, `DateTime.UtcNow`, or `Time.deltaTime` inside a test
- Asserting a specific `System.Random` sequence as if it were stable across runtime versions
- Substituting a `UnityEngine.Object`-derived type instead of the interface behind it
- Mocking a clock or RNG that a three-line fake expresses better
- Uncapped `yield` loops waiting for a condition
- `WaitForSeconds` in a test that sets `Time.timeScale` to 0
- Tests that depend on execution order, or leave objects and additive scenes behind
- Treating a suite-only failure as flake instead of static state surviving a disabled domain reload
- Retrying a flaky test without naming its cause
- Trusting the batch-mode process exit code instead of reading `-testResults`
- `-quit` alongside `-runTests`
- Running EditMode and PlayMode in one invocation
- A CI job with no `Library/` cache and no licence return on the failure path
- Chasing a coverage number on MonoBehaviour glue
- Reporting one global coverage target with no per-layer split
- Presenting a scaffold as verified when no suite ran, or leaving the Verification block off
