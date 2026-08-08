---
name: desktop-overengineering-review
description: C#/Avalonia necessity review - one-implementer interfaces, MediatR in-process, premature DI and async, plus the absent-structure floor.
metadata:
  category: desktop
  tags: [csharp, dotnet, avalonia, code-review, overengineering, necessity, interfaces, dependency-injection, async, mvvm]
user-invocable: false
---

# Desktop Overengineering Review

> Confirm the solution layout, the MVVM toolkit, and the DI story already in use from the `.sln` and `.csproj` files first. An established convention is context, not a finding - review what the diff adds against it.
>
> This skill owns **whether a layer earns its keep**. Where code lives and which way references point belongs to `desktop-core-architecture`; Avalonia's MVVM shape to `avalonia-mvvm-patterns`; control composition to `avalonia-control-patterns`; language idiom to `csharp-language-patterns`; error type design to `csharp-error-handling`; `async` mechanics to `csharp-async-patterns`; measured cost to `desktop-performance`.

## When to Use

- Reviewing a C# desktop diff that adds interfaces, base classes, DI registrations, `async` signatures, or a framework (MediatR, AutoMapper, ReactiveUI)
- Catching code that compiles, passes analyzers, and passes tests but does not need to exist

## Rules

- Every finding names what makes the abstraction unnecessary: one implementer and no test double, one subclass, one call site, one handler, no measurement, the branch is unreachable. When several stack, comma-separate them in `Unnecessary because:`
- Intent:
  - **`[Recommend]`** (default). Name the constraint, recommend the edit. Escalate to **`[Must]`** when measurable or structural cost is present; cite it in `Cost:`. In a design-only review with nothing to measure, the structural triggers still escalate. Triggers: an abstraction that forces a test to build a container or handler pipeline where a plain call sufficed; a runtime reflection registry in a NativeAOT-published app; a dispatch layer whose latency is measurable; an async signature already propagated through callers for a method that touches no I/O; a branch presented as handling a case it can never reach
  - **`[Recommend]`** when justification is plausible but not visible in the diff - state the assumption and ask the author to confirm
- An abstraction with **visible** justification - a second implementer, a test double, a benchmark in the PR - is not a finding
- **Scale is the discriminator, and scale is not domain.** Price an abstraction against the variation it absorbs: maintainer count, shipped platforms, supported formats, locales. A solo-maintained two-platform file utility absorbs almost no variation and earns almost no layers; cite the project's actual numbers, not a general principle
- **This skill has a floor as well as a ceiling.** Where structure is absent rather than excessive - a God ViewModel holding traversal, hashing, and rename rules; static mutable state as the only channel between features; no seam to test a destructive operation - say so plainly and route it to `desktop-core-architecture`. Never read "solo project" or "small utility" as licence for no structure: a codebase where one rename rule costs edits in nine places has already paid more than the abstraction would have cost
- Never propose deleting a layer the diff's own tests bind to
- **Performance abstractions need a measurement, not an argument.** A cache, an index, object pooling, or `Parallel.ForEach` introduced without a benchmark is speculative regardless of how reasonable it sounds
- **Never propose migrating a project already on ReactiveUI.** The finding exists only when the diff *introduces* ReactiveUI where CommunityToolkit.Mvvm suffices

## Patterns

### Category 1: Type-System Ceremony

#### Interface with one implementer and no test double

```csharp
// Bad - one implementation, never substituted, never faked
public interface IFileScanner { IReadOnlyList<string> Scan(string root); }
public sealed class FileScanner : IFileScanner { /* ... */ }

// Good - the class
public sealed class FileScanner { public IReadOnlyList<string> Scan(string root) { /* ... */ } }
```

Justified when a fake or a second implementation exists, or arrives in the same PR. A `TimeProvider` or filesystem seam substituted in tests is genuine (`desktop-core-architecture`) - check for the substitution before flagging.

#### Abstract base class with one subclass

`abstract` members nobody else overrides, `protected virtual` hooks nobody else calls. Collapse into the subclass. Justified at a second subclass, or when a framework demands the base.

#### Generic repository over SQLite for a local cache

```csharp
// Bad - IRepository<T>, unit-of-work, and specifications over a local cache with four queries
public interface IRepository<T> { Task<T?> GetAsync(int id); Task AddAsync(T item); /* ... */ }

// Good - a class with the queries the app makes
public sealed class ScanCache { public DuplicateGroup? FindByHash(string hash) { /* ... */ } }
```

The repository pattern absorbs database-engine variation; a local cache has exactly one engine, chosen forever (`desktop-data-persistence` owns the store itself).

#### AutoMapper for two mappings

A hand-written mapping method is visible, debuggable, and AOT-safe. Justified at a mapping count where drift is a real maintenance cost - cite the count.

### Category 2: Framework Weight

#### MediatR or an event aggregator in a single-process desktop app

```csharp
// Bad - a request dispatched through a registry to its one handler in the same process
await _mediator.Send(new ApplyRenameCommand(plan));

// Good - a method call
await _renameService.ApplyAsync(plan);
```

`Cost:` every trace and debug step now routes through a dispatch layer, and the handler registry is reflection-based - hostile to a NativeAOT publish. Justified by genuine fan-out: multiple independent subscribers that must not know each other.

#### A DI container for three services a constructor could wire

```csharp
// Bad - a generic host and registrations for a graph the eye can hold
services.AddSingleton<IScanner, Scanner>();
services.AddSingleton<IRenamePlanner, RenamePlanner>();
services.AddSingleton<MainViewModel>();

// Good - a composition root; construction order is the dependency graph
var scanner = new Scanner();
var vm = new MainViewModel(scanner, new RenamePlanner(scanner));
```

Justified when lifetimes genuinely vary or the graph outgrows a screenful. Convention-based assembly scanning is a second finding on top - it is also AOT-hostile.

#### ReactiveUI introduced where CommunityToolkit.Mvvm suffices

Observable pipelines, schedulers, and a second binding idiom to absorb what `[ObservableProperty]` and `[RelayCommand]` already cover. Flag only on introduction; an existing ReactiveUI codebase is convention, and migration is never the recommendation.

#### Premature `IOptions<T>` for constants

`IOptions<ThumbnailSettings>` wrapping four values no deployment ever varies. A `static class` of constants, or a plain record loaded once, says what it is.

#### A service layer that only forwards

A `FooService` whose every method is one line calling `FooRepository`. Delete the layer; callers take the dependency the service was hiding.

### Category 3: Premature Async

#### `async` on a CPU-bound synchronous path

```csharp
// Bad - async signature over pure computation; every caller goes async for nothing
public async Task<int> CompareAsync(PerceptualHash a, PerceptualHash b) => Distance(a, b);

// Good - sync core; the ViewModel offloads at the boundary
public int Compare(PerceptualHash a, PerceptualHash b) => Distance(a, b);
// caller: await Task.Run(() => _comparer.Compare(a, b));
```

`Cost:` the async signature propagates through every caller and test for a method that touches no I/O. Offloading belongs at the UI boundary (`csharp-async-patterns`), not in the core signature.

#### A `Task`-returning method that never awaits

`async` with no `await` (the compiler already said so: CS1998), or `Task.FromResult` returned to satisfy a self-imposed interface. Make it synchronous; wrap at the call site that genuinely needs a `Task`.

### Category 4: Speculative Performance

A cache, a precomputed index, object pooling, or a `Parallel.ForEach` added "for performance" with no measurement cited. `desktop-performance` owns the measurement discipline; the finding here is the abstraction added on a guess. If a benchmark is cited, this is not a finding regardless of the outcome.

### Category 5: Absent Structure

The floor rule's category. A single ViewModel holding traversal, hashing, and rename rules; static mutable state as the only channel between features; a destructive operation with no seam a test can reach. The finding states what the absence already costs - the edit-site count, the untestable path - and routes the extraction to `desktop-core-architecture`.

## Output Format

One block per finding; the consuming workflow merges them:

```
### [Must | Recommend] {file:line | file - symbol, when the line is unknown | symbol, when source was supplied without paths | quoted design excerpt, when reviewing a design description | symptom, when no source was supplied}

- Category: {Type-System Ceremony | Framework Weight | Premature Async | Speculative Performance | Absent Structure}
- Code: {one-line citation, or `not supplied` when the finding is inferred}
- Unnecessary because: {what makes it dead, unread, or single-valued; comma-separate when stacked} -- OR, for Absent Structure -- Missing because: {what the absence costs}
- Cost: {required for [Must]; omit otherwise}
- Recommendation: {concrete C# or csproj edit; for Absent Structure, the extraction and its owning skill}
- Justified when: {one-line note if a legitimate reason might apply; otherwise omit}
```

`Absent Structure` blocks take `Cost:` as the edit-site count or regression count already being paid, and that count is what escalates the block to `[Must]`.

Output order: finding blocks, then `Justified as-is:` lines, then the per-category zero-finding lines, then `Deferred:` lines. An abstraction examined and found justified is written one per line, so the reader can tell a defended layer from an unexamined one:

```
Justified as-is: {abstraction} - {the visible justification: implementer count, the test double, the benchmark}
```

This is the required form whenever the request questions an existing layer, since `No <category> findings.` alone reads as "nothing was checked" rather than "this was checked and it holds".

For each category with zero findings, emit exactly: `No <category> findings.` (using the category name from the enum) so the workflow knows the check ran; append ` - not assessable from this input` when the input could not exercise that category. Omit this line for categories that have at least one finding. Emit `Necessity check not run: no source supplied.` instead of the per-category lines only when nothing at all was supplied - a prose description of the design is checkable input, and yields findings, `Justified as-is:` lines, or `Deferred:` lines like any other source.

A defect owned by a sibling named in the ownership blockquote is not emitted as a finding. Write those at the end, one per line, as `Deferred: {defect} -> {owning skill}`, so the workflow routes rather than drops them. Omit entirely when there are none.

## Avoid

- Flagging an interface that a test double or a second implementation substitutes
- Flagging `async` on a method that awaits real I/O - `Stream.ReadAsync` in the body is not premature async
- Flagging `IProgress<T>` or a channel that streams progress from a long-running operation
- Flagging a cache, pooling, or parallel path whose PR cites a benchmark
- Proposing migration off ReactiveUI, the DI container, or the error convention the solution already standardized on
- Treating file count, project count, or line count as a complexity metric
- Removing a layer the diff's own tests bind to
- Raising findings against generated code - `obj/`, `bin/`, `*.g.cs`, XAML-generated partials
- Reading "solo maintainer" as licence to skip the floor check
