---
name: desktop-filesystem-patterns
description: Rust filesystem traversal and paths - walkdir/jwalk, non-UTF-8 OsStr, Windows MAX_PATH and reserved names, macOS NFC/NFD, symlinks, atomic writes.
metadata:
  category: desktop
  tags: [rust, filesystem, walkdir, jwalk, pathbuf, osstr, max-path, unc, nfc-nfd, symlink, junction, atomic-write, windows, macos]
user-invocable: false
---

# Desktop Filesystem Patterns

> Confirm the target platforms before applying the platform sections - Windows-primary and macOS-secondary have different failure sets, and applying both everywhere produces noise.
>
> This skill owns **reaching files correctly and naming them correctly**. What a batch does once it has the paths belongs to `desktop-batch-operations`; parallelising the walk to `desktop-concurrency-patterns`; symlink escape and TOCTOU as an attack to `desktop-security-patterns`; walk throughput to `desktop-performance`.

## When to Use

- Writing or reviewing directory traversal, filtering, or path construction
- A bug that appears only on one platform, only for some filenames, or only deep in a tree
- Persisting a path, comparing two paths, or displaying a path to the user
- Writing a file the user would lose if the process died mid-write

## Rules

- **Paths are not UTF-8 on either platform.** Use `Path`/`PathBuf`/`OsStr` end to end. `to_string_lossy` is for display only; a path that round-trips through it is corrupted
- **A permission error mid-walk continues the walk and is reported.** Aborting a 200k-file scan because one directory is unreadable throws away all completed work
- Traversal does not follow symlinks by default. Following requires a cycle guard and an explicit decision
- **Windows path length is handled before it is hit**, not after a mystery `os error 3`
- Never construct a path by string concatenation with `/` or `\`. `Path::join` and `push` only
- Filename comparison honours the filesystem's case rules and the platform's Unicode normalization. Byte equality is the wrong test on both platforms
- **Any write the user cares about is temp-file-plus-rename.** Truncating the real file and streaming into it loses the old contents on a crash

## Patterns

### Paths are `OsStr`, not `String`

Windows paths are UTF-16 that may contain unpaired surrogates; Unix paths are arbitrary bytes excluding NUL and `/`. Neither is guaranteed valid Unicode.

```rust
// Bad - drops or mangles any non-UTF-8 name, then renames the mangled result
let name = path.file_name().unwrap().to_string_lossy().to_string();
let target = parent.join(format!("{}_backup", name));

// Good - stays in OsString; lossless on both platforms
let mut name = path.file_name().unwrap().to_os_string();
name.push("_backup");
let target = path.with_file_name(name);
```

Store paths as `PathBuf`. When they must be serialized, keep the lossless form (`Vec<u8>` on Unix via `OsStrExt`, or a well-defined encoding) rather than assuming `String`, and state in the schema which you chose (`desktop-data-persistence`).

### Traversal with `walkdir`, and when `jwalk` earns its place

`walkdir` is the default: single-threaded, deterministic order, mature. `jwalk` parallelises directory reads and wins on wide trees on an SSD or a network share where the bottleneck is latency, not bandwidth. On a single spinning disk it is usually neutral or worse (`desktop-concurrency-patterns`).

```rust
// Bad - one unreadable directory ends the scan and the caller sees nothing
for entry in WalkDir::new(root) {
    let entry = entry?;
    files.push(entry.into_path());
}

// Good - errors are collected per-entry; the walk finishes
let mut skipped = Vec::new();
for result in WalkDir::new(root).follow_links(false) {
    match result {
        Ok(e) if e.file_type().is_file() => files.push(e.into_path()),
        Ok(_) => {}
        Err(e) => skipped.push(SkippedPath {
            path: e.path().map(Path::to_path_buf),
            reason: e.to_string(),
        }),
    }
}
```

`skipped` is part of the result, not a log line. A scan that silently omitted 4,000 files under a protected directory and reported "no duplicates found" is a wrong answer, not a partial one.

Filter with `filter_entry` to prune whole subtrees (`.git`, `node_modules`, `$RECYCLE.BIN`, `System Volume Information`) before descending - it avoids the traversal cost, where a post-filter does not.

### Windows specifics

| Trap | What happens | Handling |
| --- | --- | --- |
| Reserved device names | `CON`, `PRN`, `AUX`, `NUL`, `COM1`-`COM9`, `LPT1`-`LPT9` cannot be created as files - with or without an extension (`NUL.txt` too) | Reject at plan time with a named reason, not at apply time as an opaque error |
| Trailing dots and spaces | `report.` and `report ` are silently stripped by the Win32 layer, so the created name differs from the requested one | Normalize during planning and show the user the name that will actually exist |
| `MAX_PATH` = 260 | Paths over 260 chars fail unless long-path support is on and the manifest opts in | Prefix with `\\?\` for absolute paths, or keep operations relative to a short root |
| `\\?\` semantics | The prefix disables normalization entirely: no `/` translation, no `.`/`..` resolution, no relative paths | Canonicalize *before* prefixing, never after |
| UNC paths | `\\server\share\...` becomes `\\?\UNC\server\share\...`, not `\\?\\\server\...` | Use `dunce` or `std::fs::canonicalize` plus explicit UNC handling; do not hand-build the prefix |
| Case-insensitive, case-preserving | `Photo.jpg` and `photo.jpg` are one file with the stored casing preserved | Compare case-insensitively; preserve the user's casing when writing |
| Junctions and reparse points | A junction is a reparse point, not a symlink; `is_symlink` behaviour differs, and directory junctions create traversal cycles | Treat any reparse point as a boundary and do not descend by default |

`std::fs::canonicalize` on Windows returns a `\\?\`-prefixed path, which is correct for the API and wrong for display and for handing to other programs. Strip it for the UI (`dunce::simplified`).

### macOS: NFC/NFD normalization

HFS+ stores filenames in a decomposed form, and APFS preserves what it was given while comparing normalization-insensitively. Either way a name written as NFC (`é` = U+00E9) can read back as NFD (`e` + U+0301). The bytes differ; the file is the same file.

```rust
// Bad - the name just written is not found in the map keyed on the pre-write string
map.insert(intended_name.clone(), meta);
for entry in fs::read_dir(dir)? {
    let observed = entry?.file_name();       // NFD on macOS
    let meta = map.get(&observed);            // None - lookup misses
}

// Good - compare on a normalized key, keep the on-disk name for I/O
let key = normalize_nfc(&observed.to_string_lossy());
let meta = map.get(&key);
```

Consequences to design around: a rename that "did nothing" actually changed only the encoding; a duplicate detector reports two files that are one; a path stored in a cache never matches on the next run. Normalize to a single form for comparison keys and cache keys, and always perform the actual I/O with the `OsString` the filesystem handed you.

### Symlinks, junctions, and hardlinks

- **Symlinks**: do not follow by default. Following without a visited-set on device+inode (Unix) or file index (Windows) turns a cycle into an infinite walk. A symlink pointing outside the selected root is a security concern, not just a correctness one (`desktop-security-patterns`)
- **Windows junctions**: created without elevation and common in user profiles (`Documents and Settings`, `Application Data`). They produce loops in exactly the trees a file utility scans
- **Hardlinks**: two paths, one inode, one set of bytes. A dedup tool that "removes a duplicate" by deleting one hardlink frees zero bytes and may be exactly what the user did not want. Detect via `nlink > 1` (Unix) or the file index (Windows) and report links separately from content duplicates

### Atomic writes

```rust
// Bad - a crash between truncate and flush leaves an empty or half-written settings file
let mut f = File::create(&settings_path)?;
f.write_all(&bytes)?;

// Good - the real path only ever holds a complete file
let dir = settings_path.parent().unwrap();
let mut tmp = NamedTempFile::new_in(dir)?;   // same volume: rename is atomic
tmp.write_all(&bytes)?;
tmp.as_file().sync_all()?;                   // durable before the swap
tmp.persist(&settings_path)?;                // atomic replace
```

Three details that make this actually work: the temp file must be on the **same volume** (a cross-device rename is a copy and is not atomic); `sync_all` must precede the rename or the rename can land before the data; on Windows the rename over an existing file needs `ReplaceFile`-style semantics, which `tempfile::NamedTempFile::persist` provides.

## Output Format

Two modes, chosen by whether something is being reviewed or authored.

**Authoring mode** - the request is to write traversal or path code. Emit the code, then any `Deferred:` lines. No finding blocks and no status line.

**Review mode** - one block per finding:

```
### [Severity] {file:line | symbol, when source was supplied without paths | symptom, when no source was supplied}

- Category: {PathEncoding | Traversal | WindowsPath | MacNormalization | LinkHandling | ErrorHandling | AtomicWrite | Comparison}
- Platform: {Windows | macOS | both}
- Evidence: {measured (name the tree or repro) | estimated (stated input shape) | inferred (no source read)}
- Code: {one-line citation, or `not supplied` when the finding is inferred}
- Failure: {the concrete case that breaks - a name, a depth, a filesystem}
- Fix: {the concrete change}
```

`Severity: {Critical | High | Medium | Low}` - Critical = data loss or a wrong destructive target. High = files silently missed or an operation that fails on a common name. Medium = a platform-specific failure on an uncommon input. Low = correctness cleanup with no observed symptom.

`Category` takes exactly one value; where a defect fits two, pick the one `Fix` addresses and name the other in `Failure`. `inferred` bounds the header at High and never raises a block.

A defect owned by a sibling named in the ownership blockquote is written after the findings as `Deferred: {defect} -> {owning skill}`, one per line. Omit when there are none.

In review mode, close with exactly one status line, after any `Deferred:` lines:

| Condition | Line |
| --- | --- |
| One or more findings emitted | none - the findings are the output |
| Source or a symptom report was available, nothing found | `No filesystem findings.` |
| No source, diff, or symptom supplied | `Filesystem check not run: no source supplied.` |

## Avoid

- `to_string_lossy().to_string()` on any path that is then used for I/O
- `path.to_str().unwrap()` in a traversal path
- `?` inside the walk loop, aborting the scan on the first unreadable directory
- Dropping skipped paths instead of returning them with the results
- Building paths with `format!("{}/{}", a, b)`
- Assuming a 260-char path works, or hand-building a `\\?\` prefix for a UNC path
- Prefixing `\\?\` onto a path that has not been canonicalized
- Showing the user a `\\?\`-prefixed path
- Comparing filenames byte-for-byte on Windows or macOS
- Treating an NFD read-back as a different file from the NFC name just written
- Following symlinks without a visited-set cycle guard
- Descending into Windows junctions and reparse points by default
- Counting hardlinks to one inode as duplicate content to be deleted
- Creating a file named `CON`, `NUL`, or `COM1`, or one ending in `.` or a space, on Windows
- `File::create` on a settings or cache file the user would miss
- A temp file on a different volume from its rename target
