---
name: csharp-language-patterns
description: Idiomatic C# - nullable reference types, records vs structs, Span slicing, LINQ hot-path cost, disposal, NativeAOT limits, AI-generated smells.
metadata:
  category: desktop
  tags: [csharp, nullable, records, structs, span, linq, disposal, pattern-matching, nativeaot, api-design]
user-invocable: false
---

# C# Language Patterns

> This skill owns **C# language mechanics and type-level API shape**. Which project a type lives in belongs to `desktop-core-architecture`; throw-versus-result design to `csharp-error-handling`; `async`/`await`, `Task`, and cancellation to `csharp-async-patterns`; measured allocation cost to `desktop-performance`; whether an abstraction earns its keep to `desktop-overengineering-review`.

## When to Use

- Writing or reviewing C# in the core library or the Avalonia project
- Designing a public method signature or a domain type
- A `!` suppression, `dynamic`, or reflection call appears in a diff
- A LINQ chain sits on a per-item path over a large file set

## Rules

- **Nullable reference types are enabled everywhere** (`<Nullable>enable</Nullable>`). A `!` suppression is a design signal: the type says null cannot happen and the code disagrees. Fix the type or the flow; a surviving `!` carries a comment naming the invariant that makes it sound
- **A domain identity gets its own type.** `readonly record struct ContentHash(ulong Value)` and `ByteSize(long Value)`, not bare `ulong`/`long` the compiler will happily swap
- Records for immutable data with value equality; classes for identity, mutation, and lifecycles; `readonly record struct` for small domain values used as keys. Mutable structs are a defect
- Take the least you need: `ReadOnlySpan<char>` for parse-only inputs, `IReadOnlyList<T>` over `List<T>` in parameters. Return the concrete type the caller stores
- LINQ reads well at the orchestration layer; on a per-item path over 100k files, enumerator and delegate allocation are real cost and a `foreach` is both faster and easier to review
- An `IEnumerable<T>` is enumerated once. Materialize with `ToList()` before a second pass, or the pipeline re-executes - including its I/O
- **Everything `IDisposable` is created in a `using`** or owned by a type that disposes it in its own `Dispose`. `SKBitmap`, `FileStream`, and `SqliteConnection` hold native resources the GC does not reclaim promptly
- `dynamic`, `Type.GetType` on a name string, `Activator.CreateInstance`, and reflection-based serialization break NativeAOT. Source generators and explicit factories replace them
- Interfaces and generics are introduced for call sites that exist. One implementer and one caller is a concrete type; a test stub that exists and is used counts as the second implementer
- A type owns its invariants: mutable collections and fields stay private, and callers see read-only views or query methods

## Patterns

### Nullable: fix the flow, not the warning

```csharp
// Bad - the warning is silenced; a null here is an NRE on the user's machine
var name = entry.DisplayName!;

// Good - the flow handles the null instead of denying it
if (entry.DisplayName is not { Length: > 0 } name) return Skip(entry);
```

`!` on every access of one property means the property should be non-nullable and enforced at construction - validate once in the constructor (`ArgumentNullException.ThrowIfNull` at public core entry points), then the type carries the guarantee everywhere.

### Records, classes, structs

```csharp
// Bad - reference equality; grouping duplicates by this key silently never matches
public class FileKey { public long Size; public ulong Hash; }

// Good - value equality for free, immutable, allocation-free as a dictionary key
public readonly record struct FileKey(ByteSize Size, ContentHash Hash);
```

| Shape | Use for |
| --- | --- |
| `record` (class) | Immutable domain data with several fields: `RenamePlan`, `DuplicateGroup` |
| `readonly record struct` | Small identity values used as keys: `ContentHash`, `FileKey`, `GroupId` |
| `class` | Services, ViewModels, anything with mutation or a lifecycle |
| Mutable `struct` | Never - copy semantics plus mutation is a bug generator |

Derive nothing extra: records already carry the equality a key needs, and `IComparable` is added only where ordering is meaningful (`ByteSize`, not `ContentHash`).

### Span and Memory for allocation-free slicing

```csharp
// Bad - two substrings per filename; 200k allocations across a 100k rename preview
var stem = name.Substring(0, name.LastIndexOf('.'));
var ext = name.Substring(name.LastIndexOf('.'));

// Good - slices view the same memory; one allocation at the final string
ReadOnlySpan<char> stem = Path.GetFileNameWithoutExtension(name.AsSpan());
ReadOnlySpan<char> ext = Path.GetExtension(name.AsSpan());
return string.Concat(prefix, stem, ext);
```

`Span<T>` is stack-only: it cannot be a field, cross an `await`, or be captured by a lambda. When a slice must cross an async boundary, use `Memory<T>` and slice to `Span` at the point of use.

### LINQ cost on hot paths

```csharp
// Bad - the directory walk runs twice: once for Count(), once for the loop
IEnumerable<FileEntry> entries = ScanTree(root);
total = entries.Count();
foreach (var e in entries) Hash(e);

// Good - materialized once; both passes read the list
List<FileEntry> entries = ScanTree(root).ToList();
```

```csharp
// Bad - a pipeline rebuilt per item inside the scan loop: allocations per file
foreach (var f in files)
    if (rules.Where(r => r.Enabled).Any(r => r.Matches(f))) Include(f);

// Good - the invariant is hoisted; the inner check is a plain loop
var active = rules.Where(r => r.Enabled).ToArray();
foreach (var f in files)
    if (Matches(active, f)) Include(f);
```

Deferred execution is the feature and the trap: a query is a recipe, not a result. Name variables for what they are - `IEnumerable<T>` recipes get piped or materialized immediately, never handed around.

### Disposal

```csharp
// Bad - SKBitmap wraps native memory; 100k undisposed decodes exhaust it
var bmp = SKBitmap.Decode(path);
return MakeThumbnail(bmp);

// Good - deterministic release, exception-safe
using var bmp = SKBitmap.Decode(path);
return MakeThumbnail(bmp);
```

A type that stores a disposable implements `IDisposable` and disposes what it owns; a type that stores an `IAsyncDisposable` implements both and callers use `await using`. Ownership is singular - the creator disposes unless it explicitly hands the value off.

### Pattern matching over type checks

```csharp
// Bad - cast-and-check chains; adding an outcome silently falls through
if (o is RenameOutcome) { var r = (RenameOutcome)o; if (r.Skipped) label = "skipped"; }

// Good - one switch expression; the compiler flags unhandled shapes
var label = outcome switch
{
    Skipped s => $"skipped: {s.Reason}",
    Failed f => $"failed: {f.Reason}",
    Renamed => "renamed",
    _ => throw new UnreachableException(),
};
```

Property patterns (`is not { Length: > 0 }`), relational patterns, and list patterns replace guard-clause pyramids. Collection expressions do the same for construction: `List<Rule> rules = [defaultRule, .. userRules];` replaces `Concat().ToList()` ceremony.

### NativeAOT compatibility

```csharp
// Bad - reflection serialization; works under JIT, trimmed or unsupported under AOT
var json = JsonSerializer.Serialize(settings);

// Good - source-generated context; AOT-safe and faster
var json = JsonSerializer.Serialize(settings, AppJsonContext.Default.Settings);
```

The same rule kills `dynamic` (needs runtime codegen), `Type.GetType(nameString)`, and `Activator.CreateInstance` over unreferenced types - each works in a JIT debug run and fails in the shipped AOT build, the worst place to discover it. Name-based ViewModel-to-View resolution is this failure's most common carrier; the explicit registry that replaces it belongs to `avalonia-mvvm-patterns`.

### AI-generated C# smells

| Smell | What it usually means | Fix direction |
| --- | --- | --- |
| Interface + one implementation + DI ceremony for a pure function | Enterprise template reflex | Concrete class or static method |
| `async void` outside an event handler | Fire-and-forget by accident | `csharp-async-patterns` |
| `catch (Exception)` that logs and continues | Error path never designed | `csharp-error-handling` |
| Five-operator LINQ chain on a per-item path | Written clause by clause | `foreach` with early exits |
| `!` sprinkled until warnings stop | Types not designed for null | Non-nullable at construction |
| Base class abstracting three shared lines | Symmetry, not need | Duplicate, or a static helper |
| `#region` and XML doc on private members | Filler | Delete |
| Mutable struct, or a record with settable key properties | Value semantics misunderstood | `readonly record struct` or class |

## Output Format

When this skill produces a finding, emit one block per finding, `[Must]` first:

```
[Must|Recommend] {file:line | file:line-line | symbol, when source was supplied without paths}
Category: <nullability | type-choice | span | linq | multiple-enumeration | disposal | pattern-matching | aot-compat | over-abstraction | api-signature>
Issue: <the defect, named>
Why it matters: <the concrete cost - an NRE on the user's machine, a re-executed directory walk, native memory never freed>
Fix: <the concrete signature or restructure>
```

After the findings come the `Suppression:` lines, then any `Deferred:` lines. A defect - or, in design mode, out-of-scope work the deliverable depends on - owned by a sibling named in the ownership blockquote (a throw-versus-result question, a `Task.Run` placement question, a project-placement question) is written as `Deferred: {item} -> {owning skill}`, one per line. Omit when there are none.

When designing an API or a type rather than reviewing, produce the form below - repeated per designed item:

```
Signature: <the proposed method or type>
Nullability: <which inputs and outputs admit null, and what enforces the rest>
Type choice: <record | readonly record struct | class, and why>
Allocation: <what allocates, per call and per item>
Interfaces: <none - concrete types | the abstraction and the second implementer that warrants it>
AOT: <no dynamic or reflection | the reflection use and its source-generated replacement>
```

Every `!` suppression and every `dynamic`/reflection use reviewed gets a line, findings or not: `Suppression: <file:line> - <invariant stated | stated but unsound - finding filed | INVARIANT MISSING>`.

`[Must]` marks a defect with a concrete cost - a `!` hiding a reachable null, a key type with reference equality, a twice-enumerated query with I/O behind it, an undisposed native resource, reflection in an AOT-shipped path. `[Recommend]` marks a working construct with a better shape - a missed span slice, a hoistable LINQ invariant, a derive the type does not need.

A review that produces no finding closes with exactly `No C# language findings.` after any `Suppression:` lines - a `!` whose stated invariant actually holds is not a defect; a stated invariant the code contradicts still files a finding.

## Avoid

- `!` suppression without a comment naming the invariant
- `#nullable disable` or `<Nullable>disable</Nullable>` in new code
- Bare `long`/`ulong`/`string` for size, hash, or path identity in core signatures
- Mutable structs, or records with settable properties used as dictionary keys
- `Substring` chains where `AsSpan` slicing avoids the allocations
- Enumerating an `IEnumerable<T>` twice, or storing one as if it were a result
- LINQ pipelines rebuilt inside per-item loops
- `SKBitmap`, streams, or connections outside a `using` or an owning `Dispose`
- `dynamic` anywhere; reflection where a source generator or explicit factory exists
- Interfaces, base classes, or generics with a single implementer and caller
- Cast-and-check chains where a switch expression is exhaustive
