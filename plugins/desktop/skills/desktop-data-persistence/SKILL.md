---
name: desktop-data-persistence
description: Persist local data with Microsoft.Data.Sqlite - user_version migrations with fixtures, per-platform dirs, atomic settings, cache invalidation.
metadata:
  category: desktop
  tags: [csharp, dotnet, sqlite, microsoft-data-sqlite, migration, user-version, special-folder, settings, source-generated-json, cache-invalidation, local-first]
user-invocable: false
---

# Desktop Data Persistence

> Confirm what the data is worth before choosing a store - a rebuildable scan cache and a user's saved rename presets have different durability requirements, and treating the cache as precious is wasted work.
>
> This skill owns **choosing a store, versioning its schema, and locating its files**. The temp-file-plus-move mechanics belong to `desktop-filesystem-patterns` (an `AtomicWrite` finding is still owned here); journal replay and undo semantics to `desktop-batch-operations`; query throughput and index cost to `desktop-performance`; whether a database is warranted at all to `desktop-overengineering-review`; what ships in the installer to `desktop-build-release`.

## When to Use

- Choosing or reviewing an embedded database, cache, or settings format
- Any schema change to a store that has already shipped
- Deciding where a file goes on Windows or macOS
- A cache that returns stale results, or settings that vanish

## Rules

- **`Microsoft.Data.Sqlite` is the default store.** Its default package bundles the native SQLite library, so there is no system dependency to hunt on either platform and no version skew between a developer machine and a user's
- Schema version lives in SQLite's `user_version` pragma. A version tracked in a side file drifts from the database it describes
- **Migrations are forward-only and each one ships with a fixture database from the version it upgrades.** A migration tested only against a freshly-created schema is untested
- Config and data directories come from `Environment.GetFolderPath` per platform. A hardcoded path, or one beside the executable, is wrong on at least one target
- **Settings are written atomically.** A crash during a settings write must not leave the user with no settings
- **A cache entry is invalid unless `(path, size, mtime)` all match.** Path alone is a stale-result generator
- Serialization is source-generated (`JsonSerializerContext`). Reflection-based `JsonSerializer` calls fail under NativeAOT - at runtime, on a user's machine
- A corrupt cache is deleted and rebuilt. A corrupt settings file is reported, backed up, and replaced with defaults - never silently overwritten

## Patterns

### Choosing the store

| Data | Store | Why |
| --- | --- | --- |
| Scan cache, hashes, thumbnail index | SQLite (`Microsoft.Data.Sqlite`) | Indexed lookup by path, survives restart, deletable and rebuildable |
| User settings and presets | JSON file, atomically written | Human-readable, diffable, editable when the app will not start |
| Window geometry, last folder | Same settings file | Small, low-stakes |
| Undo journal for a running batch | Append-only file or a SQLite table, in the data directory - never the cache directory | Must survive a kill mid-run and a cache wipe (`desktop-batch-operations`) |
| Anything the user cannot regenerate | SQLite, with a backup before migration | Durability is the requirement |

For a solo-maintained utility, one SQLite cache, one journal store, and one settings file cover everything - the journal is separate from the cache because they live in different directories with different corruption stories. Any store beyond those three needs its migration and corruption story named.

EF Core is available and is usually overkill here: for a scan cache with a handful of tables, hand-written SQL - or Dapper when the mapping grows tiresome - is the lighter path, with no model building at startup, no second migration mechanism competing with `user_version`, and no NativeAOT constraints to work around.

```xml
<!-- The default package bundles native SQLite; nothing to install on either platform -->
<PackageReference Include="Microsoft.Data.Sqlite" Version="10.0.0" />
```

### Schema versioning with `user_version`

```csharp
// Bad - version tracked beside the database; a restored .db with a stale version
// file runs the wrong migrations against the wrong schema
var v = int.Parse(File.ReadAllText(Path.Combine(dir, "schema_version")));

// Good - the version travels inside the file it describes, and each step is one
// transaction: schema change and version bump commit together or not at all
static void Migrate(SqliteConnection conn) {
    var v = Scalar<long>(conn, "PRAGMA user_version");
    if (v == 0) Step(conn, V1Sql, 1);
    if (v <= 1) Step(conn, V2Sql, 2);
}
static void Step(SqliteConnection conn, string sql, int to) {
    using var tx = conn.BeginTransaction();
    Exec(conn, tx, sql);
    Exec(conn, tx, $"PRAGMA user_version = {to}");
    tx.Commit();
}
```

A failed migration therefore leaves the database at its previous version rather than half-upgraded - a version bump outside the step's transaction re-runs a completed step after a crash. Migrations are **forward-only**: a downgrade path doubles the test matrix to serve a case a desktop utility does not have, since the user does not install an older version over a newer one.

Refuse to open a database whose `user_version` exceeds the versions the binary knows. That is a newer app's file, and running old code against it corrupts it.

### The fixture per shipped version

This is the step that is skipped and the one that catches real defects.

```
tests/fixtures/
  v1.db      # created by the binary that shipped as v1
  v2.db      # created by the binary that shipped as v2
```

```csharp
[Theory]
[InlineData("v1.db"), InlineData("v2.db")]   // grows per release; old entries never leave
public void Migrates_every_shipped_version_to_current(string fixture) {
    using var conn = Open(CopyToTemp(fixture));
    Migrate(conn);
    Assert.Equal(Current, Scalar<long>(conn, "PRAGMA user_version"));
    Assert.True(ScanCacheReadable(conn));   // data survived, not just the schema
}
```

A migration validated only against `Migrate(freshDb)` passes while silently dropping every existing row, because a fresh database has no rows to drop. The fixture is committed when the version ships; recreating it later from current code defeats the purpose. Never delete an old fixture - installed users are still on it.

### The undo journal's store

Append-only, one row per completed step, committed before the next destructive step executes:

```sql
CREATE TABLE journal (seq INTEGER PRIMARY KEY, op TEXT NOT NULL,
                      from_path TEXT NOT NULL, to_path TEXT, at_utc TEXT NOT NULL);
PRAGMA journal_mode = WAL;   -- a mid-run kill leaves committed rows readable
```

One transaction per record, WAL mode, in the data directory - that is the durability floor for a store whose purpose is surviving a kill. The running batch never updates or deletes rows; replay and undo semantics belong to `desktop-batch-operations`.

### Per-platform directories

```csharp
// Bad - unwriteable beside the exe under Program Files, wrong inside a macOS bundle
var cfg = Path.Combine(AppContext.BaseDirectory, "settings.json");

// Good - platform conventions via SpecialFolder, split by what survives a cleanup
var config = Path.Combine(Environment.GetFolderPath(
    Environment.SpecialFolder.ApplicationData), "MyApp");
    // %APPDATA%\MyApp  |  ~/Library/Application Support/MyApp
var cache = OperatingSystem.IsMacOS()
    ? Path.Combine(Environment.GetFolderPath(Environment.SpecialFolder.UserProfile),
        "Library", "Caches", "MyApp")
    : Path.Combine(Environment.GetFolderPath(
        Environment.SpecialFolder.LocalApplicationData), "MyApp");
Directory.CreateDirectory(config);
Directory.CreateDirectory(cache);
```

Since .NET 8, `ApplicationData` resolves to the Apple-conventional `~/Library/Application Support` on macOS (earlier runtimes returned `~/.config`); there is no `SpecialFolder` for macOS caches, so that one is built from `UserProfile`.

The split is load-bearing: the cache directory is a location the OS and cleanup tools may empty, which is exactly right for a rebuildable scan cache and exactly wrong for settings and the undo journal. Putting the database in cache and the settings in config means a cache wipe costs a rescan and nothing else. Create the directories before the first write; they do not exist on a fresh install.

### Atomic settings writes, source-generated serialization

```csharp
// Bad - reflection-based serialization fails under NativeAOT, and a crash
// mid-write loses every setting the user had
File.WriteAllText(path, JsonSerializer.Serialize(settings));

// Good - source-generated context; temp-then-move mechanics per desktop-filesystem-patterns
[JsonSerializable(typeof(Settings))]
public partial class SettingsContext : JsonSerializerContext { }

var json = JsonSerializer.Serialize(settings, SettingsContext.Default.Settings);
AtomicFile.Replace(path, json);   // temp file, flush to disk, File.Move overwrite
```

Every settings property carries a default so a file written by an older version still loads, and an unknown field from a newer version is ignored rather than fatal. Ignoring is not enough on re-save: an older binary that saves the file strips fields it does not know - round-trip them so an installed older build does not silently drop a newer version's settings:

```csharp
public sealed class Settings {
    public int ThumbnailSize { get; set; } = 200;   // defaulted: an older file still loads
    [JsonExtensionData]                             // a newer version's keys survive
    public Dictionary<string, JsonElement>? Unknown { get; set; }
}
```

On a parse failure: rename the bad file to `settings.json.bak`, start with defaults, and tell the user where the old file went. Silently overwriting destroys the evidence they would need to recover a long-tuned configuration.

### Scan-cache invalidation

```sql
-- Bad - path is the only key; an edited file serves its old hash forever
SELECT hash FROM files WHERE path = $path

-- Good - the entry is only valid if the file is still the file that was hashed
SELECT hash FROM files WHERE path = $path AND size = $size AND mtime_ticks = $mtime
```

Store mtime as `File.GetLastWriteTimeUtc(path).Ticks` - 100 ns resolution. Second granularity misses a file edited twice within the same second, which is common under an editor's save. Size is the cheap discriminator that catches most edits; mtime catches same-size edits.

Two further conditions must invalidate the cache: a **schema or algorithm change** (bump a `hash_algo_version` column and treat mismatches as misses, so switching hash functions does not compare new hashes against old ones), and **path case or normalization differences** on Windows and macOS, where the same file can arrive under two spellings (`desktop-filesystem-patterns`).

Prune rows for paths that no longer exist during a scan, or the cache grows forever across a user's reorganizations.

## Output Format

Two modes, chosen by whether something is being reviewed or authored.

**Authoring mode** - the request is to write storage, migration, or settings code. Emit the code, then any `Deferred:` lines. No finding blocks and no status line.

**Review mode** - one block per finding, ordered by severity, Critical first:

```
### [Severity] {file:line | symbol, when source was supplied without paths | symptom, when no source was supplied}

- Category: {StoreChoice | Migration | FixtureCoverage | Paths | AtomicWrite | CacheInvalidation | Corruption | SettingsCompat}
- Evidence: {measured (name the version or fixture reproduced against - a deterministic failure readable from the supplied source counts, with the line as the repro) | estimated (source read, no fixture run; state the schema history assumed) | inferred (no source read; state what was not seen)}
- Code: {one-line citation, or `not supplied` when the finding is inferred}
- Failure: {the concrete case that breaks - which upgrade path, which platform, which stale read}
- Fix: {the concrete change}
- Verify: {what to re-run or re-check - the fixture migration test, a fresh-install path, a same-second edit}
```

`Severity: {Critical | High | Medium | Low}` - Critical = user data lost or corrupted (on upgrade, on a crash mid-write, or by a migration), or a store that cannot be opened. High = stale results presented as current, a startup failure on an existing config, or a hardcoded path wrong on a target platform. Medium = a defect on an uncommon upgrade or platform path. Low = hygiene with no observed symptom.

`Category` takes exactly one value; where a defect fits two, pick the one `Fix` addresses and name the other in `Failure`. AOT-unsafe reflection serialization files as `SettingsCompat`. `estimated` and `inferred` bound the header at High, with `Failure` naming the uncapped band; neither ever raises a block. Severity that does not fit a listed band: assign the nearest lower band and state why in `Failure`.

A defect owned by a sibling named in the ownership blockquote is written after the findings as `Deferred: {defect} -> {owning skill}`, one per line. Omit when there are none.

In review mode, close with exactly one status line, after any `Deferred:` lines:

| Condition | Line |
| --- | --- |
| One or more findings emitted | none - the findings are the output |
| Source or a symptom report was available, nothing found | `No persistence findings.` |
| No source, diff, or symptom supplied | `Persistence check not run: no source supplied.` |

## Avoid

- EF Core for a two-table scan cache, or a second store beside SQLite without naming its migration and corruption story
- A schema version in a file beside the database instead of `user_version`
- A migration step outside a transaction, or a version bump outside the step's transaction
- Opening a database whose `user_version` is newer than the binary understands
- Testing migrations only against a freshly-created schema
- Regenerating an old-version fixture from current code, or deleting one
- Writing a downgrade path for a desktop utility
- Hardcoding `%APPDATA%`, `~/.myapp`, or a path beside the executable
- Putting settings or the undo journal in the cache directory, or a rebuildable cache in config
- `File.WriteAllText` for settings
- Reflection-based `JsonSerializer` calls in a NativeAOT build
- A settings property without a default, or a re-save that strips unknown keys
- Silently overwriting a settings file that failed to parse
- Keying a scan cache on path alone, or on second-granularity mtime
- Serving cached hashes across a hash-algorithm change
- A cache that never prunes rows for deleted paths
