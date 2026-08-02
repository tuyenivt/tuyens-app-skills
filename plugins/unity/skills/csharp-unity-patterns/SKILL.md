---
name: csharp-unity-patterns
description: Write C# as Unity runs it - the destroyed-object == overload, allocation in Update, boxing, closures, coroutines vs async/await and Awaitable.
metadata:
  category: mobile
  tags: [unity, csharp, allocation, gc, async, coroutines, nullable, boxing]
user-invocable: false
---

# C# Unity Patterns

> This skill owns **C# language mechanics under the Unity runtime**. Where code lives and what it may depend on belongs to `unity-architecture-patterns`; callback timing and coroutine lifetime belong to `unity-monobehaviour-lifecycle`; what the serializer persists belongs to `unity-serialization-prefabs`; measured frame budget and profiling belong to `unity-performance`.

## When to Use

- Writing or reviewing C# that runs per-frame or per-entity
- A null check passes but the next line throws `MissingReferenceException`
- Choosing between a coroutine, `async`/`await`, and `Awaitable`
- GC spikes appear in the profiler with no obvious `new`

## Rules

- **Never use `?.`, `??`, or `??=` on a `UnityEngine.Object`.** Those operators use the runtime's real null, not Unity's overloaded `==`. A destroyed object is not real null, so `?.` invokes on it and throws. Compare with `== null` / `!= null`, or `if (obj)`
- Do not allocate in `Update`, `FixedUpdate`, `LateUpdate`, or any per-entity loop. LINQ, lambdas capturing locals, string concatenation, and `new` collections all allocate
- Reuse collections and buffers across frames; clear, do not reallocate
- A `struct` passed to a parameter typed `object` or a non-generic interface boxes. Keep hot-path generics constrained and concrete
- `async void` is unobservable and uncatchable outside a Unity event handler. Return `Task`, `Awaitable`, or use a coroutine
- Async work started from a MonoBehaviour must be cancelled when that object dies. Nothing cancels it for you
- Version-sensitive claims (C# language level, whether a given `foreach` boxes) are toolchain-determined - verify against the project's `ProjectVersion.txt` and editor rather than asserting

## Patterns

### The `==` overload: the one that bites hardest

`UnityEngine.Object` overloads `==` so a **destroyed but not garbage-collected** object compares equal to `null`. That overload is a static method on the type. The null-conditional and null-coalescing operators bypass operator overloads entirely and test for real reference null - the managed wrapper of a destroyed object is still a live reference, so it passes them and the call proceeds. It throws `MissingReferenceException` once that call touches native engine state, which is what any component member worth calling does.

```csharp
// Bad - the managed wrapper is alive, so `?.` calls into a destroyed object
target?.TakeDamage(10);            // MissingReferenceException
var t = cached ?? FindTarget();    // returns the destroyed object

// Good - goes through the overload, so destroyed reads as null
if (target != null) target.TakeDamage(10);
var t = cached != null ? cached : FindTarget();
```

The same trap applies to `is null`, `is not null`, and pattern matching against `null` - all bypass the overload. Cost note: the overload is not free, so a per-frame `!= null` on a cached reference is worth hoisting out of the loop.

Nullable reference types (`#nullable enable`) compound this. The compiler's flow analysis knows nothing about the overload, so it will mark a destroyed-but-assigned field as non-null and suppress warnings on a reference that is about to throw.

```csharp
// Bad - NRT says this is non-null; at runtime it is destroyed
[SerializeField] private Rigidbody2D body = null!;
body.linearVelocity = Vector2.zero;    // MissingReferenceException

// Good - the runtime check still runs, NRT or not
if (body != null) body.linearVelocity = Vector2.zero;
```

Treat NRT annotations on `UnityEngine.Object`-derived fields as documentation of intent, never as a runtime guarantee.

### LINQ and closures in hot paths

```csharp
// Bad - per frame: an enumerator, a closure capturing `range`, a delegate, a List
void Update() {
    var near = enemies.Where(e => e.Distance < range).ToList();
}

// Good - reused buffer, no closure, no delegate
private readonly List<Enemy> _near = new();
void Update() {
    _near.Clear();
    for (int i = 0; i < enemies.Count; i++)
        if (enemies[i].Distance < range) _near.Add(enemies[i]);
}
```

A lambda that captures nothing is cached by the compiler and allocates once. A lambda capturing a local or a field through `this` allocates a display class on every call. The second is the common case and the one that shows in the profiler as steady per-frame garbage, and the one that survives "we removed all the LINQ".

Deep Profile instruments every managed call and adds allocation of its own, so a `GC.Alloc` figure captured under it is an upper bound, not the shipping number. Attribute allocation from a non-deep capture on the target device before sizing a fix.

### Strings

```csharp
// Bad - two allocations per frame, 60/s, forever
scoreLabel.text = "Score: " + score;

// Good - assign only when the value actually changed
if (score != _lastScore) { _lastScore = score; scoreLabel.text = _scoreStrings[score]; }
```

Interpolation, `ToString()` on a numeric, `string.Format`, and `+` all allocate. For a small bounded range, precompute the strings; otherwise gate the assignment on change so the cost is per-change rather than per-frame.

### Boxing and value types

```csharp
// Bad - boxes the struct on every call
void Log(object payload) { }
Log(new Vector2Int(x, y));

// Good - generic, no boxing
void Log<T>(T payload) where T : struct { }
```

Other boxing sources: a `struct` implementing an interface then stored as that interface; `Enum` used as a `Dictionary` key without a comparer on older toolchains; `params object[]`. Passing a large `readonly struct` by value copies it - take it by `in` in hot code.

`foreach` over a `Dictionary` or `List` boxing its enumerator is **largely historical**: current Unity runtimes use the struct enumerator without boxing for these concrete types. It still boxes when the collection is iterated through an interface (`IEnumerable<T>`). Whether a given pattern allocates is version-and-backend-dependent - confirm with the profiler on the target backend rather than applying the old folklore.

### Coroutines, async/await, and Awaitable

| Need | Use |
| --- | --- |
| Frame-paced sequencing tied to a GameObject's life | Coroutine |
| Awaiting engine work (frame, seconds, scene load) with `await` syntax | `Awaitable` |
| I/O, file, or CPU work with a return value and cancellation | `async Task` / `Awaitable` with a `CancellationToken` |
| Anything that must survive the object being disabled | Neither on that object - move ownership |

`Awaitable` is Unity's engine-aware awaitable type: it resumes on the main thread by default and integrates with the player loop, which a bare `Task` does not. Coroutines cannot return a value or be awaited; `async` methods are not stopped by the engine when the object dies.

```csharp
// Bad - keeps running after the object is destroyed, then touches a dead transform
async void Start() { await Task.Delay(5000); transform.position = target; }

// Good - cancelled with the object, and the exception is observable
async Awaitable Start() {
    await Awaitable.WaitForSecondsAsync(5f, destroyCancellationToken);
    transform.position = target;
}
```

`destroyCancellationToken` is a `MonoBehaviour` property that cancels when the object is destroyed. `async void` swallows exceptions into an unobserved task and gives the caller nothing to await - the only defensible use is a UI event handler that catches internally. `Awaitable` API surface has moved across Unity 6 minors; check the version in `ProjectVersion.txt` before relying on a specific member.

### Collection reuse

```csharp
// Bad - a new array every physics query, every frame
var hits = Physics2D.OverlapCircleAll(pos, r);

// Good - preallocated, non-allocating overload
private readonly Collider2D[] _hits = new Collider2D[16];
int n = Physics2D.OverlapCircle(pos, r, _filter, _hits);
```

Unity's non-allocating overloads exist across physics, mesh, and component APIs; their exact names and signatures differ per API and have changed across versions - look up the concrete overload rather than assuming a `NonAlloc` suffix. Fixed-size buffers silently truncate when the result exceeds capacity, so size for the worst case and check the returned count.

## Output Format

Two modes, chosen by whether the request supplies code to judge or asks for code to be produced.

**Authoring mode** - the request is to write or design something. Emit the code or design, then any `Deferred:` lines. No finding blocks, no severity, no status line: nothing was reviewed, so a not-run line would misdescribe the work.

**Review mode** - source, a diff, or a symptom report was supplied. Emit one block per defect. highest severity first. One line tripping several categories is one block under the category that carries the fix; name the others in `Impact`.

```
### [Severity] {file:line | symbol or type.member, when source was supplied without paths | symptom, when no source was supplied}

- Category: {DestroyedObjectNullCheck | HotPathAllocation | Boxing | StringAllocation | ClosureCapture | AsyncLifetime | CollectionReuse | NullableAnnotation}
- Evidence: {source | inferred (state what was not seen)}
- Code: {one-line citation, or `not supplied` when the finding is inferred}
- Impact: {runtime consequence - "throws MissingReferenceException on a destroyed target", "1.2 KB/frame steady garbage"}
- Fix: {concrete change}
```

`Severity: {Critical | High | Medium | Low}` - Critical = `?.`/`??`/`is null` on a `UnityEngine.Object`, or `async void` touching engine state after destruction. High = per-frame allocation in `Update`/`FixedUpdate`, or uncancelled async holding a destroyed object. Medium = allocation on a per-interaction path, or boxing in a warm loop. Low = a cold-path allocation with no measured cost.

Severity that does not fit a listed band: assign the nearest lower band and state why in `Impact`. `Category` takes exactly one value - where a defect fits two, pick the one the `Fix` addresses and name the other in `Impact`; where it fits none, pick the closest and name the real concern in `Impact`.

`Evidence: inferred` is required whenever the source was not read. It bounds the header at High: a Critical-band defect is written High, and `Impact` names the uncapped band. It never raises a block - a Medium defect stays Medium. Among blocks sharing a band, order by what the reader must fix first: root cause before the symptoms it produces. A finding whose severity depends on an unseen declaration (whether a field is a `UnityEngine.Object`) is `inferred`, and `Fix` states the check that settles it.

A defect owned by a sibling named in the ownership blockquote is not emitted as a finding. Write those after the findings, one per line, as `Deferred: {defect} -> {owning skill}`, so the workflow routes rather than drops them. Omit entirely when there are none.

In review mode, close with exactly one status line, after any `Deferred:` lines:

| Condition | Line |
| --- | --- |
| One or more findings emitted | none - the findings are the output |
| No findings, and a symptom or report was available to reason from | `No C# findings.` |
| No source, symptom, or report of any kind was supplied | `C# check not run: no source supplied.` |

A symptom-only report (a crash signature, a profiler complaint, a verbal description) is checkable input: emit `Evidence: inferred` findings from it rather than the not-run line.

## Avoid

- `?.`, `??`, `??=`, `is null`, or `is not null` applied to any `UnityEngine.Object`
- Treating a nullable-reference annotation as proof a component reference is alive
- LINQ, string concatenation, or capturing lambdas inside per-frame code
- `async void` outside a UI event handler that catches internally
- Async work started on a `MonoBehaviour` without `destroyCancellationToken`
- Allocating a new collection or array per frame where a reused buffer fits
- Passing a `struct` as `object` or through a non-generic interface in a hot loop
- Asserting a C# language version or allocation behaviour without checking the project's toolchain
