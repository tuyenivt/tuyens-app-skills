---
name: desktop-filesystem-patterns
description: Traverse and name files correctly in .NET - streaming enumeration, Windows reserved names and MAX_PATH, macOS NFC/NFD, links, atomic writes.
metadata:
  category: desktop
  tags: [csharp, dotnet, filesystem, enumeration, max-path, unc, reserved-names, nfc-nfd, symlink, junction, hardlink, atomic-write, windows, macos]
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

- **Enumeration streams.** `Directory.EnumerateFiles` with `EnumerationOptions`, never `GetFiles` - `GetFiles` materializes the whole tree before returning the first entry
- **A permission error mid-walk continues the walk and is reported.** Aborting a 200k-file scan because one directory is unreadable throws away all completed work
- Traversal does not descend into reparse points or follow symlinks by default. Following requires a cycle guard and an explicit decision
- **Windows path length is handled before it is hit**, not after a mystery `IOException` at depth 12
- Never construct a path by string concatenation with `/` or `\`. `Path.Combine` and `Path.Join` only
- Filename comparison honours the filesystem's case rules and the platform's Unicode normalization. Ordinal equality is the wrong test on both platforms
- **Any write the user cares about is temp-file-plus-move.** Truncating the real file and streaming into it loses the old contents on a crash

## Patterns

### Streaming enumeration that survives errors

```csharp
// Bad - materializes the whole tree before the first result, and the first
// unreadable directory throws UnauthorizedAccessException, ending the scan with nothing
foreach (var f in Directory.GetFiles(root, "*", SearchOption.AllDirectories))
    files.Add(f);

// Good - streams entries and keeps walking past protected directories
var opts = new EnumerationOptions {
    RecurseSubdirectories = true,
    IgnoreInaccessible = true,                       // skip, do not throw
    AttributesToSkip = FileAttributes.ReparsePoint,  // junctions and symlinks are a boundary
};
foreach (var f in Directory.EnumerateFiles(root, "*", opts))
    files.Add(f);
```

`IgnoreInaccessible` skips silently. When the report must name what was missed - and for a scan whose answer the user acts on, it must - walk directories with an explicit stack and one try/catch per directory:

```csharp
var pending = new Stack<string>([root]);
while (pending.TryPop(out var dir)) {
    try {
        foreach (var d in Directory.EnumerateDirectories(dir))
            if ((File.GetAttributes(d) & FileAttributes.ReparsePoint) == 0)   // links stay a boundary
                pending.Push(d);
        foreach (var f in Directory.EnumerateFiles(dir)) files.Add(f);
    }
    catch (Exception e) when (e is UnauthorizedAccessException or IOException) {
        skipped.Add(new SkippedPath(dir, e.Message));
    }
}
```

When the options overloads are used instead, `EnumerationOptions` defaults `AttributesToSkip` to `Hidden | System` - a file utility clears it to `FileAttributes.None`, or hidden user files silently vanish from the answer.

`skipped` is part of the result, not a log line. A scan that silently omitted 4,000 files under a protected directory and reported "no duplicates found" is a wrong answer, not a partial one.

Prune whole subtrees (`.git`, `node_modules`, `$RECYCLE.BIN`, `System Volume Information`) before descending - skipping the directory avoids the traversal cost, where a post-filter does not.

### Windows specifics

| Trap | What happens | Handling |
| --- | --- | --- |
| Reserved device names | `CON`, `PRN`, `AUX`, `NUL`, `COM1`-`COM9`, `LPT1`-`LPT9` cannot be created as files - with or without an extension (`NUL.txt` too), and .NET does not reject them for you | Reject at plan time with a named reason; the check applies to the segment before the first dot, case-insensitively, after trailing spaces are stripped |
| Trailing dots and spaces | `report.` and `report ` are silently stripped by the Win32 layer, so the created name differs from the requested one | Normalize during planning and show the user the name that will actually exist |
| `MAX_PATH` = 260 | Paths past 260 chars fail unless the machine enables long paths *and* the app manifest declares `longPathAware` | Ship the manifest entry, and still validate length at plan time - the registry opt-in is not guaranteed on user machines |
| UNC paths | `\\server\share\...` roots and normalizes differently from drive paths | `Path.GetPathRoot` understands UNC; never hand-build a `\\?\` prefix |
| Case-insensitive, case-preserving | `Photo.jpg` and `photo.jpg` are one file with the stored casing preserved | Compare with `StringComparer.OrdinalIgnoreCase`; preserve the user's casing when writing |
| Junctions and reparse points | A junction is a reparse point, created without elevation, and directory junctions create traversal cycles | Treat any `FileAttributes.ReparsePoint` as a boundary and do not descend by default |

### macOS: NFC/NFD normalization

HFS+ stores filenames decomposed (NFD). APFS is normalization-preserving but normalization-insensitive: it stores the bytes it was given and treats NFC and NFD as the same name on lookup. Either way, a name written as NFC (`é` = U+00E9) can be observed as NFD (`e` + U+0301). The strings differ; the file is the same file.

```csharp
// Bad - the name just written is not found in a map keyed on the pre-write string
map[intendedName] = meta;
var observed = Path.GetFileName(entry);   // NFD on HFS+; whatever was first written on APFS
map.TryGetValue(observed, out var m);     // miss - the strings differ

// Good - compare on a normalized key, keep the on-disk name for I/O
var key = observed.Normalize(NormalizationForm.FormC);
map.TryGetValue(key, out var m);
```

Consequences to design around: a rename that "did nothing" actually changed only the encoding; a duplicate detector reports two files that are one; a path stored in a cache never matches on the next run. Normalize to a single form for comparison keys and cache keys, and always perform the actual I/O with the string the filesystem handed you.

### Symlinks, junctions, and hardlinks

- **Symlinks**: `new FileInfo(path).LinkTarget` is non-null for a link; `ResolveLinkTarget(returnFinalTarget: true)` follows the chain. Following links without a visited set keyed on file identity turns a cycle into an infinite walk. A link pointing outside the selected root is a security concern, not just a correctness one (`desktop-security-patterns`)
- **Windows junctions**: created without elevation and common in user profiles (`Documents and Settings`, `Application Data`). .NET surfaces them as links too; they produce loops in exactly the trees a file utility scans
- **Hardlinks**: two paths, one file, one set of bytes. A dedup tool that "removes a duplicate" by deleting one hardlink frees zero bytes and may be exactly what the user did not want. The BCL exposes no link count - it takes interop (`GetFileInformationByHandle` on Windows, `st_nlink` on macOS); report links separately from content duplicates

### Atomic writes

```csharp
// Bad - a crash between truncate and flush leaves an empty or half-written settings file
File.WriteAllBytes(settingsPath, bytes);

// Good - the real path only ever holds a complete file
var tmp = Path.Combine(Path.GetDirectoryName(settingsPath)!, $"settings-{Guid.NewGuid():N}.tmp");
using (var fs = new FileStream(tmp, FileMode.CreateNew, FileAccess.Write)) {
    fs.Write(bytes);
    fs.Flush(flushToDisk: true);                // durable before the swap
}
File.Move(tmp, settingsPath, overwrite: true);  // atomic replace on the same volume
```

Three details make this actually work: the temp file must be on the **same volume** (a cross-volume move is a copy and is not atomic - the same directory guarantees it); `Flush(flushToDisk: true)` must precede the move or the rename can land before the data; `File.Move` with `overwrite: true` maps to the platform's atomic replace, and `File.Replace` is the variant that also keeps a backup of the old file.

## Output Format

Two modes, chosen by whether something is being reviewed or authored.

**Authoring mode** - the request is to write traversal or path code. Emit the code, then any `Deferred:` lines. No finding blocks and no status line.

**Review mode** - one block per finding, ordered by severity, Critical first:

```
### [Severity] {file:line | file:line-line | symbol, when source was supplied without paths | symptom, when no source was supplied}

- Category: {PathConstruction | Traversal | WindowsPath | MacNormalization | LinkHandling | ErrorHandling | AtomicWrite | Comparison}
- Platform: {Windows | macOS | both}
- Evidence: {measured (name the tree or repro - a reported repro or a deterministic failure readable from the supplied source counts) | estimated (source read, no measurement; state the input shape assumed) | inferred (no source read; state what was not seen)}
- Code: {one-line citation, or `not supplied` when no source was read}
- Failure: {the concrete case that breaks - a name, a depth, a filesystem}
- Fix: {the concrete change}
```

`Severity: {Critical | High | Medium | Low}` - Critical = data loss or a wrong destructive target. High = files silently missed, a walk aborted by a single per-entry error, or an operation that fails on a common name. Medium = a platform-specific failure on an uncommon input. Low = correctness cleanup with no observed symptom.

`Category` takes exactly one value; where a defect fits two, pick the one `Fix` addresses and name the other in `Failure`. `inferred` bounds the header at High and never raises a block; when the cap lowers a would-be Critical, say so in `Failure`. Findings are ordered by the capped header; within a band, the finding whose uncapped band is higher leads.

A defect - or, in authoring mode, out-of-scope work the deliverable depends on - owned by a sibling named in the ownership blockquote is written after the findings or the authored code as `Deferred: {item} -> {owning skill}`, one per line. Omit when there are none.

In review mode, close with exactly one status line, after any `Deferred:` lines:

| Condition | Line |
| --- | --- |
| One or more findings emitted | none - the findings are the output |
| Source or a symptom report was available, nothing found | `No filesystem findings.` |
| No source, diff, or symptom supplied | `Filesystem check not run: no source supplied.` |

## Avoid

- `Directory.GetFiles` with `SearchOption.AllDirectories` on a tree of unknown size
- Letting one `UnauthorizedAccessException` abort the scan
- Dropping skipped paths instead of returning them with the results
- Building paths with string interpolation and slashes
- Assuming a 260-char path works on a user's machine, or hand-building a `\\?\` prefix
- Comparing filenames with ordinal equality on Windows or macOS
- Treating an NFD read-back as a different file from the NFC name just written
- Following symlinks without a visited-set cycle guard
- Descending into junctions and reparse points by default
- Counting hardlinks to one file as duplicate content to be deleted
- Creating a file named `CON`, `NUL`, or `COM1`, or one ending in a dot or a space, on Windows
- `File.WriteAllBytes` or `File.WriteAllText` on a settings or cache file the user would miss
- A temp file on a different volume from its move target
