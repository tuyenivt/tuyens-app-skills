---
name: rust-language-patterns
description: Idiomatic Rust - ownership over cloning, borrowing, iterators, newtypes, Cow, impl Trait, justified unsafe, AI-generated Rust smells.
metadata:
  category: desktop
  tags: [rust, ownership, borrowing, clone, iterators, newtype, cow, unsafe, generics, api-design]
user-invocable: false
---

# Rust Language Patterns

> This skill owns **Rust language mechanics and type-level API shape**. Which crate a type lives in belongs to `desktop-core-architecture`; `Result` design, error types, and `unwrap` policy to `rust-error-handling`; threads, channels, and `Send`/`Sync` reasoning to `desktop-concurrency-patterns`; measured allocation cost to `desktop-performance`; whether an abstraction earns its keep to `desktop-overengineering-review`.

## When to Use

- Writing or reviewing Rust in either the core or the UI crate
- A borrow checker error was resolved and the fix needs judging
- Designing a public function signature or a domain type
- `unsafe`, a raw pointer, or a `transmute` appears in a diff

## Rules

- **`clone()` is a design signal, not a fix.** Cloning to satisfy the borrow checker means the ownership model is wrong; cloning a cheap `Copy`-adjacent value or an `Arc` handle is correct. Judge it by what is cloned and why, never by the call count
- Take the least you need: `&str` over `&String`, `&[T]` over `&Vec<T>`, `&Path` over `&PathBuf`. Take ownership only when the function stores or consumes the value
- Return iterators or `impl Iterator` when the caller decides the collection; `collect()` at the point of use, not in the middle of a pipeline
- **A domain identity gets a newtype**, not a bare `String` or `u64`. Path, hash, and size are distinct types the compiler can keep apart
- `Arc<Mutex<T>>` requires shared mutation across threads. Anything less - single ownership, a channel, or `&mut` - uses that instead
- Generics and traits are introduced for call sites that exist. One caller and one implementer is a concrete type
- **`unsafe` carries a `// SAFETY:` comment stating the invariant the caller upholds.** No comment means the block is unreviewable and is rejected
- Version-sensitive claims (edition, MSRV, whether a std API is stable) are toolchain-determined - check `Cargo.toml` and `rust-toolchain.toml` rather than asserting

## Patterns

### Cloning: the signal versus the correct use

```rust
// Bad - clones a whole path list to end a borrow the compiler complained about
let files = state.files.clone();
for f in &files { state.mark_seen(f); }

// Good - split the borrow, or index; no copy of the data
for i in 0..state.files.len() {
    let f = state.files[i].clone();   // one PathBuf, not the whole Vec
    state.mark_seen(&f);
}
```

Three clones that are correct and should not be flagged: `Arc::clone` (a refcount bump, spell it `Arc::clone(&x)` so the reader sees it), a small `Copy`-like value, and a value an Iced `Message` must own because messages are `Clone` by contract. Three that are defects: cloning a collection to end a borrow, cloning inside a hot loop where a reference would do, and cloning a `String` that is immediately borrowed as `&str`.

### Cow for the mostly-unchanged case

```rust
// Bad - allocates a new String for every filename, including the ones unchanged
fn sanitize(name: &str) -> String { name.replace(':', "_") }

// Good - allocates only when a replacement actually happened
fn sanitize(name: &str) -> Cow<'_, str> {
    if name.contains(':') { Cow::Owned(name.replace(':', "_")) } else { Cow::Borrowed(name) }
}
```

`Cow` pays off where most inputs pass through untouched - filename sanitizing over 100k files is exactly that shape. It is noise where every input is transformed.

### Iterators over intermediate collections

```rust
// Bad - three Vecs materialized for one pass
let a: Vec<_> = entries.iter().filter(|e| e.is_file()).collect();
let b: Vec<_> = a.iter().map(|e| e.len()).collect();
let total: u64 = b.iter().sum();

// Good - one pass, no intermediate allocation
let total: u64 = entries.iter().filter(|e| e.is_file()).map(|e| e.len()).sum();
```

Return `impl Iterator<Item = T> + '_` from core functions that produce a sequence, so the caller chooses between streaming and collecting. Return a concrete `Vec<T>` when the value is stored, sent across a channel, or put in an Iced message - `impl Trait` cannot cross those boundaries.

### Newtypes for domain identity

```rust
// Bad - two u64s the compiler will happily swap
fn group(hash: u64, size: u64) -> GroupKey;

// Good - a transposed call is a compile error
#[derive(Clone, Copy, PartialEq, Eq, Hash)]
pub struct ContentHash(pub u64);
#[derive(Clone, Copy, PartialEq, Eq, PartialOrd, Ord)]
pub struct ByteSize(pub u64);

fn group(hash: ContentHash, size: ByteSize) -> GroupKey;
```

Derive what the type genuinely needs. A hash needs `Hash` and `Eq`; a size needs `Ord`. Blanket-deriving everything invites uses the type was not designed for.

### Ownership before `Arc<Mutex<T>>`

```rust
// Bad - shared mutable state for a value that has one writer
let results = Arc::new(Mutex::new(Vec::new()));
paths.par_iter().for_each(|p| results.lock().unwrap().push(hash(p)));

// Good - each worker owns its output; the collection is the join
let results: Vec<_> = paths.par_iter().map(|p| hash(p)).collect();
```

The bad version serializes every worker on one lock, which erases the parallelism it was written to get. Reach for `Arc<Mutex<T>>` only when several threads genuinely mutate one value and no channel or reduction expresses it.

### `impl Trait` and premature generics

```rust
// Bad - a type parameter, a trait bound, and a where clause for one caller passing &[PathBuf]
pub fn scan<I, P>(inputs: I) -> Vec<Entry>
where I: IntoIterator<Item = P>, P: AsRef<Path> { }

// Good - the concrete signature the one caller needs
pub fn scan(inputs: &[PathBuf]) -> Vec<Entry> { }
```

`AsRef<Path>` and `Into<String>` are worth it on a widely-used public API where callers hold different types. In a two-crate solo app with one call site, they cost readability and compile time and buy nothing. Generalize when the second caller with a different type arrives.

### `unsafe` and its invariant comment

```rust
// Bad - no statement of what makes this sound; unreviewable
let s = unsafe { std::str::from_utf8_unchecked(bytes) };

// Good - the invariant is named and checkable against the caller
// SAFETY: `bytes` came from `read_header`, which returns only after
// `from_utf8` validated the same slice. No mutation between the two.
let s = unsafe { std::str::from_utf8_unchecked(bytes) };
```

For this app class, `unsafe` is justified in roughly three places: a documented FFI call into a platform API, a memory-mapped file whose backing may not change while mapped, and a measured hot-loop bound-check elision with a benchmark to show for it. Anything else - and any `unsafe` used to escape a borrow error - is a defect. `unsafe` never fixes a lifetime problem; it hides it.

### AI-generated Rust smells

| Smell | What it usually means | Fix direction |
| --- | --- | --- |
| `.clone()` sprinkled until it compiles | Borrow structure never designed | Restructure ownership; split borrows |
| `Arc<Mutex<T>>` with no second thread | Copied from a concurrent example | Plain ownership or `&mut` |
| `.unwrap()` on every `Result` | Error type never chosen | `rust-error-handling` |
| Generic params with one caller | Symmetry, not need | Concrete types |
| `String` where `&str` is passed straight through | Signature defaults to owned | Take `&str` |
| `Box<dyn Trait>` for one implementer | Trait added for shape | Concrete type |
| `collect()` mid-pipeline then re-iterated | Pipeline written stepwise | Single chain |
| `.iter().cloned().collect()` to end a borrow | Borrow checker silenced | Fix the borrow |

## Output Format

When this skill produces a finding:

```
[Must|Recommend] <file:line>
Category: <ownership | borrowing | clone | iterator | newtype | cow | generics | unsafe | api-signature>
Issue: <the defect, named>
Why it matters: <the concrete cost - allocation per item, lost parallelism, a swap the compiler cannot catch>
Fix: <the concrete signature or restructure>
```

When designing an API or a type rather than reviewing:

```
Signature: <the proposed function or type>
Ownership: <borrowed | owned, and why the callee needs it>
Allocation: <what allocates, per call and per item>
Newtypes: <domain identities given their own type | none - all bare primitives, justified>
Unsafe: <none | the block, its invariant, and what upholds it>
```

Every `unsafe` block reviewed gets a line, findings or not: `Unsafe: <file:line> - <invariant stated | INVARIANT MISSING>`.

## Avoid

- `clone()` introduced to end a borrow the compiler rejected
- `.iter().cloned().collect()` or `to_vec()` as a borrow-checker escape
- `Arc<Mutex<T>>` where one owner, a channel, or `&mut` fits
- Locking inside a parallel loop that the parallelism exists to avoid
- `&String`, `&Vec<T>`, or `&PathBuf` in a parameter position
- `collect()` into a `Vec` that is immediately iterated once
- Type parameters, trait bounds, or `Box<dyn Trait>` introduced for one call site
- Bare `u64`/`String` for hash, size, or path identity in core signatures
- `unsafe` without a `// SAFETY:` comment naming the invariant
- `unsafe` used to resolve a lifetime or borrow error
- `transmute` where a `From`/`TryFrom` impl or a cast expresses the conversion
- Deriving every trait on a domain type regardless of what it needs
- Asserting an edition, MSRV, or std stability without reading `Cargo.toml`
