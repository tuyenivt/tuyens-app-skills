---
name: csharp-error-handling
description: C# error design - throw vs result at the core boundary, per-item batch outcomes, filesystem exceptions, cancellation flow, actionable messages.
metadata:
  category: desktop
  tags: [csharp, errors, exceptions, result-types, partial-failure, ioexception, cancellation, exception-filters, aggregate-exception]
user-invocable: false
---

# C# Error Handling

> This skill owns **how failure is typed, propagated, and told to the user**. Which project an outcome type lives in belongs to `desktop-core-architecture`; per-item journals and undo semantics to `desktop-batch-operations`; retry and path-existence races to `desktop-filesystem-patterns`; `Parallel.ForEach` partitioning and channel wiring to `desktop-concurrency-patterns`; token threading and who triggers cancellation to `csharp-async-patterns`; where the message is rendered to `avalonia-control-patterns`.

## When to Use

- Choosing between throwing and returning an outcome for a new core API
- Reviewing a diff containing `catch`, `throw`, or a new outcome type
- A batch operation must report which items failed and why
- An exception crosses `Parallel.ForEach`, a `Task`, or a channel before reaching the UI

## Rules

- **Exceptions are for the exceptional; expected per-item failures are values.** A locked file in a 100k-file batch is expected - it becomes an outcome. A missing root directory before the batch starts is exceptional - throw
- **A batch operation never throws on item failure.** Each item completes with an outcome; the loop body catches the expected kinds per item, records, and continues
- **Never catch `Exception` broadly on a destructive path.** An unexpected failure mid-rename means the operation's assumptions are already false; continuing to mutate files under them loses data. Stop with recorded state intact
- Catch exactly what the site can handle, where it can handle it. A catch that logs and rethrows adds no handling - delete it, or make logging an exception filter so the stack is preserved
- `OperationCanceledException` is control flow, not an error. It propagates to the operation's initiator; it is never converted to a failed outcome or an error dialog
- Filesystem work catches `IOException` and `UnauthorizedAccessException` specifically. `FileNotFoundException`, `DirectoryNotFoundException`, and `PathTooLongException` are `IOException` subclasses - catch them first or branch inside; `UnauthorizedAccessException` is not
- Exception filters (`when`) keep a catch specific without nesting, and the filter runs before the stack unwinds
- An awaited task throws the original exception; `.Result`, `.Wait()`, and `Parallel.ForEach` wrap failures in `AggregateException`. Design so it never carries batch item failures - the per-item catch lives inside the loop body
- Every error that reaches the user states what failed, which item, and what to do next. The runtime's `ex.Message` is diagnostic detail, not the headline

## Patterns

### Throw or return: the boundary rule

```csharp
// Bad - the first locked file unwinds the batch and discards completed work
public IReadOnlyList<RenameResult> ExecuteAll(RenamePlan plan) =>
    plan.Steps.Select(Execute).ToList();   // IOException on item 61,301 loses 61,300 results

// Good - preconditions throw; each item completes with an outcome
public BatchReport ExecuteAll(RenamePlan plan, CancellationToken ct)
{
    ArgumentNullException.ThrowIfNull(plan);
    var outcomes = new List<ItemOutcome>(plan.Steps.Count);
    foreach (var step in plan.Steps)
    {
        ct.ThrowIfCancellationRequested();
        try { Apply(step); outcomes.Add(new Renamed(step.Source, step.Target)); }
        catch (Exception ex) when (ex is IOException or UnauthorizedAccessException)
        { outcomes.Add(new Failed(step.Source, UserReason(ex), ex)); }
    }
    return new BatchReport(outcomes);
}
```

The good shape lets the UI render "98,412 renamed, 3 failed" with the three paths listed and a retry scoped to those three. The journal that makes the partial result undoable is `desktop-batch-operations`' concern - this skill's contract is that the outcome list exists and is complete.

### The outcome type at the core boundary

```csharp
public abstract record ItemOutcome(string Path);
public sealed record Renamed(string Path, string NewPath) : ItemOutcome(Path);
public sealed record Skipped(string Path, SkipReason Reason) : ItemOutcome(Path);
public sealed record Failed(string Path, string Reason, Exception Cause) : ItemOutcome(Path);
```

A closed record hierarchy the UI switches over exhaustively, each case carrying the item it belongs to - an outcome without its path cannot be reported or retried. This is the full extent of result types in this codebase: no general-purpose `Result<T, E>` monad as the app-wide error channel. Exceptions are C#'s channel; outcome types exist only where partial failure is the normal case, which is batch boundaries.

### Broad catch on a destructive path

```csharp
// Bad - disk full, a path race, or a bug is swallowed; the batch keeps mutating files
try { Apply(step); }
catch (Exception) { outcomes.Add(new Failed(step.Source, "error", null!)); }

// Good - expected kinds become outcomes; anything else stops the batch, state recorded
catch (Exception ex) when (ex is IOException or UnauthorizedAccessException)
{ outcomes.Add(new Failed(step.Source, UserReason(ex), ex)); }
```

The filter is the allowlist of failures the operation understands. An exception outside it proves the operation's model wrong, and the only safe response on a destructive path is to stop - with the outcomes recorded so far still reaching the caller (return the partial report, or carry it on the thrown exception), never discarded with a local list. Broad catch is legitimate in exactly one place: the top-level boundary (unhandled-exception hook, command wrapper) that reports and halts, never one that continues.

### Cancellation is not an error

```csharp
// Bad - cancel is reported as failure; pending items marked failed, an error dialog shown
catch (OperationCanceledException) { outcomes.Add(new Failed(step.Source, "cancelled", null!)); }

// Good - no catch in the loop; the initiator catches it once and reports honestly
try { Report = await Task.Run(() => _core.ExecuteAll(plan, ct), ct); }
catch (OperationCanceledException) { Status = "Cancelled - 61,300 of 100,000 done. Completed renames kept."; }
```

`catch (OperationCanceledException) when (ct.IsCancellationRequested)` distinguishes the user's cancel from a dependency timing out with the same exception type. What the user is told after a cancel follows the partial-failure shape: what completed, what did not, what remains undoable.

### Filesystem exceptions to user actions

```csharp
// Bad - locale-dependent string matching; silently wrong on non-English Windows
if (ex.Message.Contains("being used by another process")) { /* retry */ }

// Good - type and HResult behind a named helper
internal static bool IsSharingViolation(this IOException ex) =>
    ex.HResult == unchecked((int)0x80070020);   // ERROR_SHARING_VIOLATION, Windows
```

| Exception | Meaning | UI action |
| --- | --- | --- |
| `FileNotFoundException` / `DirectoryNotFoundException` | Gone since the scan | Drop the item, note it in the summary |
| `UnauthorizedAccessException` | ACL denies, or read-only attribute | Skip; "check permissions on {path}" |
| `PathTooLongException` | Target exceeds the path limit | Ask for a shorter name; name the limit |
| `IOException` + sharing-violation `HResult` | File open in another program | "Close the program using it" + retry |
| `IOException` (other) | Disk-level failure | Stop the batch; report |

Windows-specific `HResult` values stay behind named helpers so the mapping is testable and macOS branches have one place to differ (`desktop-platform-integration` owns platform divergence).

### AggregateException

```csharp
// Bad - .Result wraps the failure; the catch unwraps one layer and misses nested ones
try { var r = HashAllAsync(paths).Result; }
catch (AggregateException ex) { Log(ex.InnerException); }

// Good - await surfaces the original exception type directly
var r = await HashAllAsync(paths);
```

`Parallel.ForEach` aggregates exceptions from iterations - but a body that catches its expected kinds per item (the batch rule above) lets nothing escape except `OperationCanceledException`. Where an `AggregateException` must be handled anyway, call `Flatten()` and handle every inner exception, not just the first.

### The message that reaches the user

```csharp
// Bad - the headline is the runtime's diagnostic
"System.IO.IOException: The process cannot access the file 'C:\\Users\\...\\report.pdf' because it is being used by another process."

// Good - item, cause, action
"Could not rename 'report.pdf': the file is open in another program. Close it and retry."
```

Build this by keeping the path in the outcome and mapping exception type to an action sentence at the UI edge. The exception itself stays in the outcome for a details expander and the log - it is not the headline. A batch summary follows the same shape: counts first, then the failing items, then one retry affordance scoped to those items.

## Output Format

When this skill produces a finding, emit one block per finding, `[Must]` first:

```
[Must|Recommend] {file:line | symbol, when source was supplied without paths}
Category: <throw-vs-result | batch-abort | broad-catch | rethrow-noise | cancellation-as-error | exception-mapping | aggregate-unwrap | user-message>
Issue: <the defect, named>
Consequence: <what the user or caller loses - "the batch aborts at the first locked file", "cancel renders as an error dialog">
Fix: <the concrete type or call change>
```

In review mode, after the findings come the `Catch:` lines, then any `Deferred:` lines; design mode has no `Catch:` lines. A defect - or, in design mode, out-of-scope work the deliverable depends on - owned by a sibling named in the ownership blockquote is written as `Deferred: {item} -> {owning skill}`, one per line. Omit when there are none.

When designing an error path rather than reviewing, produce the form below - one form per layer, and per operation where their policies differ:

```
Layer: <core | ui>
Failure channel: <exceptions, types listed | per-item outcome type, and why this boundary carries one>
Per-item: <what each failed item carries | not a batch operation>
Cancellation: <where OperationCanceledException propagates to, and who catches it>
Unexpected-exception policy: <stop with recorded state | the boundary that reports and halts>
User message: <the sentence the user sees, and the action it offers | none at this layer - typed outcomes only, mapped at the UI edge>
```

Every `catch (Exception)` reviewed - bare or filtered - gets a line, findings or not: `Catch: <file:line> - <boundary handler, reports and stops | filtered to expected kinds | TOO BROAD - state what it swallows, wrongly continues past, or blindly rethrows>`.

`[Must]` marks a defect the Rules name - e.g. a batch aborting on its first item, a broad catch that continues on a destructive path, cancellation converted to an error, a message the user cannot act on; the list is examples, the Rules govern. `[Recommend]` marks a working path with a better shape - a filter worth adding, a summary type over bare counts, a helper to replace an inline `HResult`.

A review that produces no finding closes with exactly `No error handling findings.` after the `Catch:` lines - a boundary handler that reports and stops is not a finding.

## Avoid

- `Result<T, E>` monad libraries as the app-wide error channel - outcome types live at batch boundaries only
- A batch method whose one exception discards completed work
- `catch (Exception)` that continues on a path that mutates files
- Empty catch blocks outside a crash handler's own double-fault guard
- Catch-log-rethrow blocks that add no handling
- `throw ex;` - it resets the stack trace; `throw;` preserves it
- Converting `OperationCanceledException` into a failed outcome or an error dialog
- Matching `ex.Message` text instead of the exception type or `HResult`
- Catching `IOException` above its subclasses, making the specific catches unreachable
- An outcome type that omits the path of the item that failed
- `.Result`/`.Wait()` wrapping failures in `AggregateException` where `await` surfaces the original (the deadlock this also causes is `csharp-async-patterns`')
- Showing the user `ex.ToString()` or a raw runtime message as the headline
