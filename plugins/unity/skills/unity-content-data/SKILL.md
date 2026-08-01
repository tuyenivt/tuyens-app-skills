---
name: unity-content-data
description: Model large static content banks for Unity quiz games - authoring format, import validation, Addressables streaming, content updates post-release.
metadata:
  category: mobile
  tags: [unity, content, scriptableobject, addressables, json, csv, validation, quiz]
user-invocable: false
---

# Unity Content Data

> This skill owns **the structure, validation, loading, and versioning of static content banks**. Player progress and save files belong to `unity-save-persistence`; translating bank text belongs to `unity-i18n`; memory and load-time budgets belong to `unity-performance`; where the loading code lives belongs to `unity-architecture-patterns`.

## When to Use

- A game ships a question bank, word list, flag set, level pack, or balance table
- Deciding between JSON, CSV, and ScriptableObject for authored content
- Content is large enough that loading it all costs memory or startup time
- Content needs to change after release without a store submission
- Reviewing a diff that adds, reshapes, or loads content

## Rules

- **Content is data, never code and never prefabs.** A question, level, or item is a row in a bank. One prefab or one C# constant per question is unshippable at bank scale and unreviewable in a diff
- **A malformed bank fails at import or build, never at runtime.** Validation runs in the editor and in CI, and a validation failure fails the build. A user seeing a question with no correct answer is a shipped defect that validation would have caught
- Every content entry has a **stable identifier** that is independent of its array position, its text, and its locale. Reordering a bank must not invalidate saved progress or analytics
- The correct answer is identified by a stable answer id, not by an index into a shuffled list and not by string comparison against display text
- Content is versioned separately from the app binary, with an explicit `contentVersion` the app checks and can migrate across
- Loading strategy is chosen against a measured bank size. Load the whole bank only when the whole bank is small enough to state a number for; otherwise stream by group
- Remote or downloaded content is untrusted input: validate shape, size, and identifiers on arrival, and fall back to bundled content on failure

## Patterns

### Authoring format

| Format | Authoring | Diffs | Validation | Best for |
| --- | --- | --- | --- | --- |
| CSV | Spreadsheet, non-engineers, bulk edit | Clean line diffs | Import script | Flat rows at volume - questions, words, flags, balance tables |
| JSON | Text editor or tooling, nested shapes | Readable, noisy on reformat | Schema check at import | Nested or heterogeneous content; remote-delivered content |
| ScriptableObject | Unity inspector, one asset per entry | Binary-ish YAML, merge-hostile | Editor validation | Small hand-curated sets that reference other Unity assets |

The workable default for a quiz bank: **author in CSV or JSON, import into one ScriptableObject holding the whole bank as a serializable array.** Authors get spreadsheet ergonomics and clean diffs; the runtime gets a single asset load with no per-entry file overhead and no runtime parse.

One ScriptableObject asset per question is the failure mode to name explicitly: 2,000 assets means 2,000 `.meta` files, 2,000 GUIDs, a slow import, an unreviewable diff, and Addressables entries that dominate the catalog.

### Content schema

```csharp
// Bad - correct answer is an index into a list that gets shuffled for display
[Serializable] public class Question { public string text; public string[] answers; public int correctIndex; }

// Good - stable ids survive shuffling, reordering, and localization
[Serializable] public sealed class Question {
    public string id;               // stable, unique, never reused: "de-signs-0142"
    public string promptKey;        // localization key, not display text
    public Answer[] answers;        // each with its own id
    public string correctAnswerId;
    public string[] tags;           // category, difficulty, licence class
}
```

Shuffle the presentation order, resolve correctness by `correctAnswerId`. With `correctIndex`, any shuffle, any locale that reorders answers, and any authoring reorder produces a silently wrong grade.

Distractors (wrong answers) are authored content, not generated at runtime. A generated distractor can be accidentally correct, duplicate the right answer, or be trivially implausible. Where distractors are pooled across questions, validation must assert the pooled set never contains the correct answer for the question drawing from it.

`tags` carries the filtering axes the game needs - category, difficulty, licence class, region. Encoding those as separate banks instead means every new axis is a new file and a new loading path.

### Import-time validation

```csharp
// Bad - trusts the file; the failure surfaces in front of a player
var bank = JsonUtility.FromJson<QuestionBank>(text);

// Good - validated at import, build fails on error
foreach (var q in bank.questions) {
    Assert(!string.IsNullOrEmpty(q.id), $"empty id at row {i}");
    Assert(seen.Add(q.id), $"duplicate id {q.id}");
    Assert(q.answers.Any(a => a.id == q.correctAnswerId), $"{q.id}: correctAnswerId matches no answer");
}
```

The validation set that catches real bank defects:

| Check | Failure it prevents |
| --- | --- |
| Non-empty, unique ids | Save/analytics collisions; unresolvable progress |
| `correctAnswerId` resolves to an answer | Ungradeable question shown to a player |
| Exactly one correct answer (or `n` where multi-select is intended) | Ambiguous grading |
| Answer count within the range the UI renders | Truncated or clipped options |
| No duplicate answer text within a question | Two identical options, one of them "wrong" |
| Referenced assets (images, audio, flags) resolve | Missing sprite at runtime |
| Localization key exists for every shipped locale | Blank or key-shaped text in the UI |
| Prompt and answer lengths within layout limits | Overflow in fixed-width 2D layouts |

Run this as an editor validation entry point that CI invokes in batch mode, with a non-zero exit on failure. Validation that only runs when a developer clicks a menu item is not validation.

### Loading and streaming

| Bank size | Strategy |
| --- | --- |
| Small, fully needed | One Addressables asset load at session start; hold for the session |
| Large, sliced by category or level | One Addressables group per slice; load the slice, release on exit |
| Very large or update-driven | Remote Addressables group with a catalog; download on demand, cache on device |

```csharp
// Bad - the whole 5,000-question bank plus every image resident for a 10-question round
var bank = Resources.Load<QuestionBank>("AllQuestions");

// Good - load the slice the round needs, release when the round ends
var handle = Addressables.LoadAssetAsync<QuestionBank>(categoryKey);
var bank = await handle.Task;
// ...
Addressables.Release(handle);
```

`Resources/` is the legacy path: everything under it is loaded into the build unconditionally, cannot be partially updated, and inflates startup. Use Addressables.

Every `LoadAssetAsync` needs a matching `Release`. Unreleased handles are the standard Addressables memory leak, and they compound across rounds. Handles held by a screen are released when the screen pops.

Text is cheap; referenced media is not. A 5,000-row question bank is a few megabytes of strings, but 5,000 referenced sprites resident at once is a memory failure on a low-end Android device. Reference media by Addressables key and load it per presented question, not per bank entry. See `unity-performance` for the budget, `unity-2d-rendering` for atlas grouping.

Group content so a group is the unit the game actually needs - by category, licence class, or level pack. Groups split by authoring convenience produce downloads that fetch content the player will not see.

### Content updates without a store release

Remote Addressables groups plus a hosted catalog let a bank change without a binary submission:

```
build -> remote catalog + content bundles -> CDN
app start -> check catalog hash -> download changed bundles -> use
```

What this can and cannot do: content bundles can change data, text, and referenced assets. They cannot change C# in an IL2CPP build, which is AOT-compiled - a rules change still needs a store release. Bundles built against a different Unity version or a changed serialized type layout will not load; content updates must be built from the same content-update workflow as the shipped player.

Every remote fetch needs the offline path: bundled content ships in the build and is the fallback when the catalog is unreachable, the download fails, or validation of the downloaded bank fails. A first-run experience that requires a download is a first-run failure on a plane.

Version explicitly:

```csharp
// Bad - swap the file and hope every install agrees
// Good - the app knows what it holds and what it needs
public sealed class BankManifest : ScriptableObject {
    public int contentVersion;      // bumped on every shipped bank change
    public int minAppVersion;       // below this, the app refuses the bank and keeps bundled content
}
```

`minAppVersion` is what stops a bank that uses a new field from loading into an old binary that will silently drop it. Content that references saved progress by id must never reuse an id for different content - retire ids, do not recycle them.

### Localization interaction

A bank multiplies by locale. Two shapes, and the choice is structural:

| Shape | Bank holds | Cost |
| --- | --- | --- |
| Keys into string tables | `promptKey`, answer keys | One bank; translations live in `unity-i18n` string tables; loads one locale |
| Per-locale banks | Full text per locale | N banks; simplest for bulk-translated static content; must load only the active locale |

Either way: **never ship every locale's text resident at once**, and never key correctness off translated text. The correct-answer id is locale-invariant; the display string is not. `unity-i18n` owns table structure, glyph coverage, and text expansion; this skill owns that the bank's identifiers stay stable across all of it.

## Output Format

When reviewing, emit one block per finding:

```
### [Severity] {file:line | asset path | symptom, when no source was supplied}

- Category: {Format | Schema | Validation | Identifier | Loading | Memory | Versioning | RemoteContent | LocaleStructure}
- Evidence: {source | inferred (state what was not seen)}
- Code: {one-line citation, or `not supplied` when the finding is inferred}
- Impact: {what a player or author hits - "shuffled answers grade wrong", "whole bank resident for a 10-question round"}
- Fix: {concrete change}
```

`Severity: {Critical | High | Medium | Low}` - Critical = content defect reaches a player (ungradeable or wrongly graded question, unvalidated remote bank, no offline fallback) or a content id collides with saved progress. High = whole-bank load where a slice is needed, unreleased Addressables handles, or validation that exists but does not fail the build. Medium = format or grouping choice that costs authoring or download efficiency with no correctness impact. Low = schema or naming nit.

Severity that does not fit a listed band: assign the nearest lower band and state why in `Impact`. `Category` takes exactly one value - where a defect fits two, pick the one the `Fix` addresses and name the other in `Impact`; where it fits none, pick the closest and name the real concern in `Impact`.

`Evidence: inferred` is required whenever the source was not read, and caps the block at High - write the capped severity in the header, not the uncapped one, and name the uncapped band in `Impact`.

A defect owned by a sibling named in the ownership blockquote is not emitted as a finding. Write those after the findings, one per line, as `Deferred: {defect} -> {owning skill}`, so the workflow routes rather than drops them. Omit entirely when there are none.

Close with exactly one status line, after any `Deferred:` lines:

| Condition | Line |
| --- | --- |
| The project ships no static content bank | `No content bank in scope.` and no findings |
| One or more findings emitted | none - the findings are the output |
| No findings, and a symptom or report was available to reason from | `No content findings.` |
| No source, diff, symptom, or report of any kind was supplied | `Content check not run: no source supplied.` |

A symptom-only report (a QA ticket, a verbal description) is checkable input: emit `Evidence: inferred` findings from it rather than the not-run line.

## Avoid

- One prefab or one ScriptableObject asset per content entry at bank scale
- Content entries hardcoded in C# arrays or switch statements
- `correctIndex` into a list that is shuffled, reordered, or localized
- Distractors generated at runtime with no correctness check
- Validation that runs only on a developer's menu click, or only at runtime
- Content ids reused, renamed, or derived from array position
- `Resources.Load` for content banks
- `LoadAssetAsync` with no matching `Release`
- Every referenced sprite or audio clip resident because the bank references them directly
- Remote content trusted without shape validation, or shipped with no bundled fallback
- Every locale's text loaded when one locale is active
