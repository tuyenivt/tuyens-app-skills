---
name: desktop-data-persistence
description: Local storage for Rust desktop apps - rusqlite bundled, user_version migrations with fixtures, per-platform dirs, atomic settings, cache invalidation.
metadata:
  category: desktop
  tags: [rust, rusqlite, sqlite, sled, redb, migration, user-version, directories, settings, cache-invalidation, local-first]
user-invocable: false
---

# Desktop Data Persistence

> Confirm what the data is worth before choosing a store - a rebuildable scan cache and a user's saved rename presets have different durability requirements, and treating the cache as precious is wasted work.
>
> This skill owns **choosing a store, versioning its schema, and locating its files**. The temp-file-plus-rename mechanics belong to `desktop-filesystem-patterns`; query throughput and index cost to `desktop-performance`; whether a database is warranted at all to `desktop-overengineering-review`; what ships in the installer to `desktop-build-release`.

## When to Use

- Choosing or reviewing an embedded database, cache, or settings format
- Any schema change to a store that has already shipped
- Deciding where a file goes on Windows or macOS
- A cache that returns stale results, or settings that vanish

## Rules

- **`rusqlite` with the `bundled` feature is the default store.** It compiles SQLite from source, so there is no system dependency to hunt on either platform and no version skew between a developer machine and a user's
- **`sled` is dead. Do not introduce it.** The published release is 0.34.7 from 2021 and the 1.0 alpha line stalled in 2024. `redb` is alive but has broken its on-disk format twice, so adopting it means owning format-migration code the app did not ask for. `sled` found already shipped is a `StoreChoice` finding, not an emergency: the fix is a one-time import into SQLite - High when the store holds data the user cannot regenerate, Medium otherwise
- Schema version lives in SQLite's `user_version` pragma. A version tracked in a side file drifts from the database it describes
- **Migrations are forward-only and each one ships with a fixture database from the version it upgrades.** A migration tested only against a freshly-created schema is untested
- Config and data directories come from the `directories` crate. A hardcoded `~/.myapp` or `%APPDATA%\myapp` is wrong on at least one target
- **Settings are written atomically.** A crash during a settings write must not leave the user with no settings
- **A cache entry is invalid unless `(path, size, mtime)` all match.** Path alone is a stale-result generator
- A corrupt cache is deleted and rebuilt. A corrupt settings file is reported, backed up, and replaced with defaults - never silently overwritten

## Patterns

### Choosing the store

| Data | Store | Why |
| --- | --- | --- |
| Scan cache, hashes, thumbnail index | SQLite (`rusqlite`, `bundled`) | Indexed lookup by path, survives restart, deletable and rebuildable |
| User settings and presets | TOML or JSON file, atomically written | Human-readable, diffable, editable when the app will not start |
| Window geometry, last folder | Same settings file | Small, low-stakes |
| Undo journal for a running batch | Append-only file or a SQLite table | Must survive a kill mid-run (`desktop-batch-operations`) |
| Anything the user cannot regenerate | SQLite, with a backup before migration | Durability is the requirement |

For a solo-maintained utility, one SQLite file plus one settings file covers everything. A second store is a second migration story, a second corruption story, and a second backup story.

```toml
# app/Cargo.toml - no system SQLite required on Windows or macOS
rusqlite = { version = "0.40", features = ["bundled"] }
```

Without `bundled`, the build links the system SQLite, which does not exist on a stock Windows machine and varies by version on macOS. That is a support burden a solo maintainer pays for repeatedly.

### Schema versioning with `user_version`

```rust
// Bad - version tracked beside the database; a restored .db with a stale .json
// version file runs the wrong migrations against the wrong schema
let v: u32 = fs::read_to_string(dir.join("schema_version"))?.parse()?;

// Good - the version travels inside the file it describes
fn migrate(conn: &Connection) -> Result<()> {
    let v: u32 = conn.query_row("PRAGMA user_version", [], |r| r.get(0))?;
    if v == 0 { conn.execute_batch(V1)?; conn.pragma_update(None, "user_version", 1)?; }
    if v <= 1 { conn.execute_batch(V2)?; conn.pragma_update(None, "user_version", 2)?; }
    Ok(())
}
```

Each step runs in a transaction, so a failed migration leaves the database at its previous version rather than half-upgraded. Migrations are **forward-only**: a downgrade path doubles the test matrix to serve a case a desktop utility does not have, since the user does not install an older version over a newer one.

Refuse to open a database whose `user_version` exceeds the versions the binary knows. That is a newer app's file, and running old code against it corrupts it.

### The fixture per shipped version

This is the step that is skipped and the one that catches real defects.

```
tests/fixtures/
  v1.db      # created by the binary that shipped as v1
  v2.db      # created by the binary that shipped as v2
```

```rust
#[test]
fn migrates_every_shipped_version_to_current() {
    for fixture in ["v1.db", "v2.db"] {
        let db = copy_to_temp(fixture);
        let conn = Connection::open(&db).unwrap();
        migrate(&conn).unwrap();
        assert_eq!(current_version(&conn), CURRENT);
        assert!(scan_cache_readable(&conn));       // data survived, not just the schema
    }
}
```

A migration validated only against `migrate(fresh_db())` passes while silently dropping every existing row, because a fresh database has no rows to drop. The fixture is committed when the version ships; recreating it later from current code defeats the purpose.

### Per-platform directories

```rust
// Bad - wrong on Windows, wrong on macOS, and unwriteable next to the exe under Program Files
let cfg = PathBuf::from(std::env::var("HOME")?).join(".myapp/config.toml");

// Good
let dirs = ProjectDirs::from("com", "Example", "MyApp")
    .ok_or(Error::NoHomeDirectory)?;
let config = dirs.config_dir();   // %APPDATA%\Example\MyApp   |  ~/Library/Application Support/com.Example.MyApp
let data   = dirs.data_dir();     // %APPDATA%\Example\MyApp\data
let cache  = dirs.cache_dir();    // %LOCALAPPDATA%\Example\MyApp\cache  |  ~/Library/Caches/...
```

The split is load-bearing: `cache_dir` is a location the OS and cleanup tools may empty, which is exactly right for a rebuildable scan cache and exactly wrong for settings. Putting the database in `cache_dir` and the settings in `config_dir` means a cache wipe costs a rescan and nothing else.

`ProjectDirs::from` returns `None` when no home directory can be determined. Handle it as an error with a clear message rather than unwrapping into a panic at startup.

Create directories with `create_dir_all` before the first write; they do not exist on a fresh install.

### Atomic settings writes

```rust
// Bad - a crash between truncate and write loses every setting the user had
fs::write(&path, toml::to_string(&settings)?)?;

// Good - the live file is only ever replaced by a complete one
let tmp = NamedTempFile::new_in(path.parent().unwrap())?;
tmp.as_file().write_all(toml::to_string_pretty(&settings)?.as_bytes())?;
tmp.as_file().sync_all()?;
tmp.persist(&path)?;
```

Deserialize with every field defaulted (`#[serde(default)]`) so a settings file written by an older version still loads, and an unknown field from a newer version is ignored rather than fatal. A settings struct that fails to parse on a missing field turns an added setting into a startup crash for every existing user.

On a parse failure: rename the bad file to `settings.toml.bak`, start with defaults, and tell the user where the old file went. Silently overwriting destroys the evidence they would need to recover a long-tuned configuration.

### Scan-cache invalidation

```rust
// Bad - path is the only key; an edited file serves its old hash forever
"SELECT hash FROM files WHERE path = ?1"

// Good - the entry is only valid if the file is still the file that was hashed
"SELECT hash FROM files WHERE path = ?1 AND size = ?2 AND mtime_ns = ?3"
```

Store `mtime` at the highest precision the platform gives (nanoseconds where available) - second granularity misses a file edited twice within the same second, which is common under an editor's save. Size is the cheap discriminator that catches most edits; mtime catches same-size edits.

Two further conditions must invalidate the cache: a **schema or algorithm change** (bump a `hash_algo_version` column and treat mismatches as misses, so switching hash functions does not compare new hashes against old ones), and **path case or normalization differences** on Windows and macOS, where the same file can arrive under two spellings (`desktop-filesystem-patterns`).

Prune rows for paths that no longer exist during a scan, or the cache grows forever across a user's reorganizations.

## Output Format

Two modes, chosen by whether something is being reviewed or authored.

**Authoring mode** - the request is to write storage, migration, or settings code. Emit the code, then any `Deferred:` lines. No finding blocks and no status line.

**Review mode** - one block per finding:

```
### [Severity] {file:line | symbol, when source was supplied without paths | symptom, when no source was supplied}

- Category: {StoreChoice | Migration | FixtureCoverage | Paths | AtomicWrite | CacheInvalidation | Corruption | SettingsCompat}
- Evidence: {measured (name the version or fixture reproduced against) | estimated (source read, no fixture run; state the schema history assumed) | inferred (no source read; state what was not seen)}
- Code: {one-line citation, or `not supplied` when the finding is inferred}
- Failure: {the concrete case that breaks - which upgrade path, which platform, which stale read}
- Fix: {the concrete change}
- Verify: {what to re-run or re-check - the fixture migration test, a fresh-install path, a same-second edit}
```

`Severity: {Critical | High | Medium | Low}` - Critical = user data lost or corrupted on upgrade, or a store that cannot be opened. High = stale results presented as current, a startup failure on an existing config, or a hardcoded path wrong on a target platform. Medium = a defect on an uncommon upgrade or platform path. Low = hygiene with no observed symptom.

`Category` takes exactly one value; where a defect fits two, pick the one `Fix` addresses and name the other in `Failure`. `estimated` and `inferred` bound the header at High, with `Failure` naming the uncapped band; neither ever raises a block.

A defect owned by a sibling named in the ownership blockquote is written after the findings as `Deferred: {defect} -> {owning skill}`, one per line. Omit when there are none.

In review mode, close with exactly one status line, after any `Deferred:` lines:

| Condition | Line |
| --- | --- |
| One or more findings emitted | none - the findings are the output |
| Source or a symptom report was available, nothing found | `No persistence findings.` |
| No source, diff, or symptom supplied | `Persistence check not run: no source supplied.` |

## Avoid

- Introducing `sled`, or adopting `redb` without owning its format-migration cost
- `rusqlite` without the `bundled` feature in a shipped desktop app
- A schema version in a file beside the database instead of `user_version`
- A migration that runs outside a transaction
- Opening a database whose `user_version` is newer than the binary understands
- Testing migrations only against a freshly-created schema
- Regenerating an old-version fixture from current code
- Writing a downgrade path for a desktop utility
- Hardcoding `~/.myapp`, `%APPDATA%\myapp`, or a path beside the executable
- `ProjectDirs::from(..).unwrap()`
- Putting settings in `cache_dir`, or a rebuildable cache in `config_dir`
- `fs::write` for settings
- A settings struct without `#[serde(default)]` on its fields
- Silently overwriting a settings file that failed to parse
- Keying a scan cache on path alone, or on second-granularity mtime
- Serving cached hashes across a hash-algorithm change
- A cache that never prunes rows for deleted paths
