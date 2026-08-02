---
name: unity-save-persistence
description: "Design durable Unity saves: format choice, atomic writes, corruption recovery, schema migration, cloud conflicts, PlayerPrefs and WebGL limits."
metadata:
  category: mobile
  tags: [unity, save, persistence, serialization, migration, atomic-write, playerprefs, cloud-save, webgl, mobile]
user-invocable: false
---

# Unity Save Persistence

> Confirm the project's target platforms first - they decide which path and durability guidance applies. This skill owns **save format, durability, and migration**. Tamper resistance and integrity signing belong to `unity-security-patterns`; what Unity's serializer does to scene and prefab assets to `unity-serialization-prefabs`; what save state is shaped like in the rules layer to `unity-architecture-patterns`.

## When to Use

- Designing or reviewing a save system for a new game
- A player reports lost progress, a reset profile, or a save that will not load
- Shipping an update that changes the save model
- Adding cloud save, or a second device to an existing single-device save
- Targeting WebGL and needing persistence that survives a page reload

## Rules

- **PlayerPrefs is never the primary save store.** It is a registry key on Windows, a plist on macOS/iOS, and shared preferences on Android - trivially editable by the player, size-limited, and wiped by common uninstall/clear-data paths. It is for non-critical local preferences only: volume, last locale, tutorial-seen flags
- **Every save write is atomic.** Write to a temp file, flush, then replace the real file. A process kill mid-write must never leave a truncated save - the OS kills backgrounded mobile apps routinely and without warning
- **Every save carries an explicit schema version field**, written from the first release. A save without a version cannot be migrated later without guessing
- **An old save must load in a new build.** Players update from any prior version and old versions stay installed on devices for years. Migration runs forward from whatever version was read
- Load must survive a corrupt, truncated, empty, or unreadable file without crashing or silently starting a new game as if nothing was lost
- Save to `Application.persistentDataPath` and never to a hardcoded path or `Application.dataPath`, which is read-only on most platforms
- Save data is the rules layer's state, serialized through an explicit DTO. Never serialize MonoBehaviours, scene references, or engine types into a save
- Cloud save needs a stated conflict-resolution rule chosen before shipping, not discovered when two devices disagree
- **On mobile, `OnApplicationPause(true)` is the save point, not `OnApplicationQuit`** - the OS kills a backgrounded app without delivering quit. Mark state dirty on mutation and flush at checkpoints plus pause; a full write per mutation widens the kill window rather than narrowing it

## Patterns

### Choosing a format

| Store | Use for | Durability | Player-editable |
| --- | --- | --- | --- |
| JSON file in `persistentDataPath` | the primary save - progress, board state, currency, inventory | durable with atomic write | yes, plainly - integrity is `unity-security-patterns` |
| Binary file in `persistentDataPath` | large saves where parse time or size is measured as a problem | durable with atomic write | yes, with slightly more effort |
| `PlayerPrefs` | volume, locale, tutorial flags - anything a player losing costs nothing | not durable, wipeable, size-limited | yes, trivially |
| Cloud save service | cross-device continuity and reinstall recovery | depends on service and sign-in | server-side rules apply |

JSON is the default: human-readable when debugging a player's corrupted save, diffable, and trivially versionable. `JsonUtility` is engine-fast but limited - no dictionaries, no polymorphism, no nullable primitives, and it serializes fields, not properties. A third-party JSON serializer handles those at the cost of reflection, which **managed code stripping will break in an IL2CPP release build unless the types are preserved** (`unity-build-release`). That failure appears only in a release build, never in the editor.

Binary is not a security measure. It is a size and parse-time measure, and only after measuring.

### Atomic write and corruption recovery

```csharp
// Bad - a kill mid-write leaves a truncated file and the save is gone
File.WriteAllText(savePath, json);

// Good - the real file is only ever replaced by a complete temp file
var tmp = savePath + ".tmp";
File.WriteAllText(tmp, json);
File.Replace(tmp, savePath, savePath + ".bak");   // falls back to Move when no original exists
```

The invariant: the destination path always holds either the previous complete save or the new complete save, never a partial one. Keeping the displaced file as a `.bak` gives a recovery target for free.

Load order on start: try the main file; on parse failure or a failed integrity check, try the backup; if both fail, start fresh **and tell the player**, rather than silently presenting an empty profile as normal.

```csharp
// Bad - a corrupt file reads as "no save", and the player's progress vanishes without a word
if (!File.Exists(path)) return NewGame();
return JsonUtility.FromJson<SaveData>(File.ReadAllText(path));

// Good - distinguish "no save" from "unreadable save"
if (!File.Exists(path)) return NewGame();
if (TryLoad(path, out var s) || TryLoad(path + ".bak", out s)) return s;
return NewGameAfterCorruption();   // surfaces a message; keeps the bad file for diagnosis
```

Whether `File.Replace` and flush semantics guarantee durability against a device power loss is filesystem- and platform-dependent; the pattern protects against process kill, which is the mobile case that actually happens. Preserve the unreadable file rather than deleting it - it is the only evidence for the bug report.

### Schema versioning and migration

```csharp
// Bad - no version; every future change is a guess about what shape this is
[Serializable] public class SaveData { public int coins; public int[] board; }

// Good - version is the first field and is written from release 1
[Serializable] public class SaveData {
    public int schemaVersion = CurrentVersion;   // bump on every shape change
    public int coins;
    public int[] board;
}
```

Migration runs as an ordered chain from the file's version up to current, each step handling one bump. Chained single-version steps stay testable; a single branch per old-version-to-current does not.

| Change | Migration needed |
| --- | --- |
| Added a field with a usable default | Deserializer default may suffice - still bump and state the default explicitly |
| Renamed a field | Yes - read the old name, write the new one. A rename with no migration reads as data loss |
| Changed a field's type or units (seconds to milliseconds, int to long) | Yes - silent misinterpretation is worse than a load failure |
| Removed a field | Bump; ignore the old key on read |
| Restructured (flat currency fields into a dictionary) | Yes, explicitly |

Rules that hold regardless of the change: never renumber existing versions; keep migration steps forever, because a player can return after a year on an ancient version; test migration with a real saved file from each shipped version, kept as a fixture; and treat a save whose version is **newer than the build** as a distinct case - it happens when a player downgrades or restores a backup from a newer install, and the correct behaviour is to refuse and explain, never to load it as if the fields matched.

### Storage paths per platform

`Application.persistentDataPath` is the only correct root, and it resolves per platform: an app-container path on iOS, internal app storage on Android, a user-profile application-data path on desktop, and a virtual IndexedDB-backed path on WebGL. Never build a path by string concatenation with platform assumptions baked in, and never write to `Application.dataPath` (read-only in a build on most platforms) or `StreamingAssets` (read-only, and compressed inside the APK on Android).

Two consequences worth designing around: the path is not stable across reinstall on all platforms, so it is not a device identity; and on iOS, files in the wrong container subdirectory are backed up to iCloud when they should not be, or excluded when they should not be - see below.

### Cloud save and conflict resolution

Two devices playing offline will produce two divergent saves. The resolution rule must be chosen deliberately:

| Rule | Fits | Cost |
| --- | --- | --- |
| Last-write-wins by server timestamp | simple linear progress | silently discards the other device's session |
| Highest-progress-wins (a monotonic score: level, total currency earned) | most casual/puzzle progression | needs a genuinely monotonic metric; loses non-monotonic state |
| Field-level merge | idle games with independent counters | most work; needs per-field merge semantics |
| Ask the player | anything where either side may be the one they want | a UI surface, and a player who picks wrong |

Never trust the **device clock** for last-write-wins - a player can change it, and a wrong device clock will hand victory to the stale save permanently. Use the service's server-assigned timestamp or a monotonic save counter incremented on every write.

Local save stays authoritative during play; cloud sync happens at defined points (launch, resume, deliberate save), and a sync failure is not a reason to block play. Cloud save is also not a backup of a corrupt local save - validate before upload, or corruption propagates to every device.

### WebGL persistence

WebGL writes go to a browser-backed virtual filesystem in memory, persisted to **IndexedDB**, and the flush is asynchronous. A write that returned is not necessarily persisted - close the tab before the sync completes and it is gone.

- File writes to `persistentDataPath` need an explicit filesystem sync to reach IndexedDB. Unity syncs automatically for some paths and not for arbitrary file writes; the manual route is a JS call (`FS.syncfs` / the `JS_FileSystem_Sync` surface) from a `.jslib` plugin. The exact automatic-sync behaviour varies by Unity version - verify against the project's version rather than assuming
- IndexedDB is per-origin and cleared by ordinary browser privacy actions, private browsing, and storage-pressure eviction. Treat WebGL local persistence as a convenience, not durable storage; anything valuable needs a server-side save
- `System.IO` durability guarantees do not apply. The atomic write-then-replace pattern still helps against a mid-write reload but does not survive an origin wipe
- Save at meaningful checkpoints and sync explicitly rather than relying on unload handlers, which browsers do not reliably run

### Platform backup interaction

iCloud (iOS) and Android Auto Backup will, by default, include app data and restore it onto a new device. Two failure modes follow:

- **Unwanted backup**: large caches and re-downloadable content inflate the user's backup quota. Mark caches as excluded from backup rather than storing them alongside the save
- **Unwanted restore**: a restored save arrives on a device that never played, and can arrive *older* than a cloud save, or newer than the installed build (the newer-than-build case above). Restore-aware games validate the save's version and reconcile against cloud state on first launch rather than assuming local is current

Backup restore also duplicates any device-local identifier stored in the save. Anything meant to be per-install must be derived at first run, not persisted and restored.

## Output Format

Two modes, chosen by whether the request supplies code to judge or asks for code to be produced.

**Authoring mode** - the request is to write or design something. Emit the code or design, then any `Deferred:` lines. No finding blocks, no severity, no status line: nothing was reviewed, so a not-run line would misdescribe the work.

**Review mode** - source, a diff, or a symptom report was supplied. Emit one block per finding.

```
### [Severity] {file:line | symbol or type.member, when source was supplied without paths | asset path | symptom, when no source was supplied}

- Category: {FormatChoice | PlayerPrefsMisuse | AtomicWrite | CorruptionRecovery | SchemaVersion | Migration | StoragePath | CloudConflict | WebGLPersistence | BackupInteraction}
- Evidence: {source | inferred (state what was not seen)}
- Code: {one-line citation, or `not supplied` when the finding is inferred}
- Impact: {what the player loses - "kill during autosave truncates the file and wipes progress", "v1 save fails to load in v2 with no migration path"}
- Fix: {concrete change}
```

`Severity: {Critical | High | Medium | Low}` - Critical = progress can be lost, silently reset, or made unloadable (non-atomic write, missing migration for a shipped shape change, primary progress in PlayerPrefs). High = recoverable data loss or a corrupt save handled as a fresh start with no signal. Medium = a durability or conflict gap on a path that is reachable but uncommon. Low = a hygiene issue with no data-loss path.

Severity that does not fit a listed band: assign the nearest lower band and state why in `Impact`. `Category` takes exactly one value - where a defect fits two, pick the one the `Fix` addresses and name the other in `Impact`; where it fits none, pick the closest and name the real concern in `Impact`.

`Evidence: inferred` is required whenever the source was not read. It bounds the header at High: a Critical-band defect is written High, and `Impact` names the uncapped band. It never raises a block - a Medium defect stays Medium. Among blocks sharing a band, order by what the reader must fix first: root cause before the symptoms it produces.

A defect owned by a sibling named in the ownership blockquote is not emitted as a finding. Write those after the findings, one per line, as `Deferred: {defect} -> {owning skill}`, so the workflow routes rather than drops them. Omit entirely when there are none.

In review mode, close with exactly one status line, after any `Deferred:` lines:

| Condition | Line |
| --- | --- |
| One or more findings emitted | none - the findings are the output |
| No findings, and a symptom or report was available to reason from | `No persistence findings.` |
| No source, diff, symptom, or report of any kind was supplied | `Persistence check not run: no source supplied.` |

A symptom-only report (a QA ticket, a verbal description) is checkable input: emit `Evidence: inferred` findings from it rather than the not-run line.

## Avoid

- PlayerPrefs holding progress, currency, inventory, or anything a player would file a ticket about losing
- `File.WriteAllText` straight over the live save file
- A save model with no `schemaVersion` field
- Migration written as one branch from every old version to current, rather than an ordered chain
- Deleting or overwriting a save that failed to parse, instead of preserving it for diagnosis
- Treating "file missing" and "file unreadable" as the same outcome
- Hardcoded paths, `Application.dataPath`, or writes into `StreamingAssets`
- Device-clock timestamps deciding a cloud save conflict
- Uploading a local save to cloud without validating it first
- Assuming a WebGL write is persisted when the call returned
- Serializing MonoBehaviours, scene references, or engine types into save data
- Relying on reflection-based serialization in an IL2CPP build without preserving the types
