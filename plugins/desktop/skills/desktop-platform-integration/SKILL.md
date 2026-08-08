---
name: desktop-platform-integration
description: Wire Avalonia OS integration - IStorageProvider dialogs, drag in and out, TrayIcon, toasts, FileSystemWatcher, hotkeys, single instance, keychain.
metadata:
  category: desktop
  tags: [csharp, avalonia, storage-provider, drag-and-drop, tray-icon, notifications, filesystemwatcher, clipboard, single-instance, autostart, credentials]
user-invocable: false
---

# Desktop Platform Integration

> Confirm the target platforms and whether the app is packaged and signed before applying this skill - several of these APIs succeed under `dotnet run` and silently do nothing in a shipped build.
>
> This skill owns **the app's contact surface with the OS**. Signing, bundling, and installer layout belong to `desktop-build-release`; what the app does with a dropped or watched path to `desktop-filesystem-patterns`; when a delete goes to the recycle bin to `desktop-batch-operations` - the shell interop for it lives here; the channel and dispatcher mechanics behind the callback hand-off to `desktop-concurrency-patterns` and `csharp-async-patterns` (the hand-off rule itself lives here); what is safe to store in a credential store to `desktop-security-patterns`.

## When to Use

- Adding a file/folder picker, drag-and-drop, tray icon, notification, or global hotkey
- Watching a directory for external changes
- Reading or writing the clipboard, or deleting to the recycle bin
- Deciding launch-on-login, single-instance, or per-user install behaviour
- Storing a token or password on the user's machine
- A feature that works in development and does nothing after packaging

## Rules

- **Packaging and signing are prerequisites, not release polish.** Notifications, credential storage, and file-open activation all depend on the app having a stable OS identity. Feature work that assumes them is untestable until they exist
- Every integration has a **declared degraded path**. The OS refuses, the permission is denied, or the platform lacks the feature - the app states what it does then, in the UI, not in a log
- **No OS callback runs application logic on the thread it arrives on.** Watcher, hotkey, and pipe callbacks arrive on threadpool or native threads: hand off a message and return, and touch UI state only via `Dispatcher.UIThread`
- **File dialogs go through `IStorageProvider`.** `OpenFileDialog` and `SaveFileDialog` are obsolete; new code that uses them is a defect
- **`FileSystemWatcher` is a hint, never a source of truth.** Every consumer pairs it with a rescan path, because the watcher drops events by design under load
- A path arriving from the OS - drop, dialog, watcher, or forwarded argv - is handled by `desktop-filesystem-patterns` rules before anything acts on it

## Patterns

### Capability matrix

| Capability | API | Windows | macOS | Prerequisite that bites |
| --- | --- | --- | --- | --- |
| File/folder dialogs | `IStorageProvider` via `TopLevel` | yes | yes | Async only; `OpenFileDialog` is obsolete |
| Drag-and-drop **in** | `DragDrop` attached events | yes | yes | - |
| Drag-and-drop **out** | `DragDrop.DoDragDrop` | yes | yes | Needs the live pointer event; files must exist on disk |
| Tray icon + menu | Avalonia `TrayIcon` | yes | yes | Menu mutations on the UI thread |
| Notifications | OS toast interop (yours) | yes | yes | **Silently no-ops** without AUMID shortcut / `.app` bundle |
| File watching | `FileSystemWatcher` | yes | yes | 8 KB default buffer overflows and drops events |
| Clipboard | Avalonia `IClipboard` via `TopLevel` | yes | yes | Async; reachable only from a live window |
| Global hotkeys | P/Invoke `RegisterHotKey` / Carbon | yes | yes | No Avalonia API; macOS permission flow is yours |
| Single instance | named `Mutex` | yes | yes | Boolean only; argv forwarding is yours |
| Recycle-bin delete | `IFileOperation` / `NSFileManager` interop | yes | yes | No BCL API; `desktop-batch-operations` decides when |
| Launch on login | `Run` registry key / `LaunchAgents` | yes | yes | Per-user, and the user can revoke it |
| Credential storage | Credential Manager / Keychain interop | yes | yes | **Keychain fails on unsigned binaries** |

### Dialogs: `IStorageProvider`

```csharp
// Bad - obsolete API; compiles with a warning, gone in a future release
var files = await new OpenFileDialog { AllowMultiple = true }.ShowAsync(window);

// Good - the storage provider from the TopLevel, async by design
var top = TopLevel.GetTopLevel(view)!;
var files = await top.StorageProvider.OpenFilePickerAsync(new FilePickerOpenOptions {
    AllowMultiple = true,
    FileTypeFilter = [FilePickerFileTypes.ImageAll],
});
```

A cancelled picker returns an empty list - a normal outcome, not an error. `IStorageFile.TryGetLocalPath()` is nullable: virtual and remote locations have no local path, so handle `null` instead of assuming every pick is a file on disk. How the provider reaches a ViewModel without the ViewModel holding a window is `avalonia-mvvm-patterns`' concern.

### Drag-and-drop, both directions

**In**: enable with `DragDrop.SetAllowDrop(control, true)`, set `e.DragEffects` in `DragOver`, read `e.Data.GetFiles()` in `Drop`. A drop may deliver multiple items, folders, and names in any encoding. Make the drop target visibly live on hover-enter, as a distinct state.

**Out** - dragging files to Explorer or Finder - works in this stack via `DragDrop.DoDragDrop`, unlike the previous one:

```csharp
// Drag out starts from a live pointer gesture; the event args prove it
private async void OnPointerPressed(object? sender, PointerPressedEventArgs e) {
    var data = new DataObject();
    data.Set(DataFormats.Files, new[] { storageFile });
    var result = await DragDrop.DoDragDrop(e, data, DragDropEffects.Copy);
}
```

Two constraints: the drag must start from an active pointer event - it cannot be launched from a command or a timer - and the dragged files must exist on disk at drag start, so content that lives only in memory is exported to a temp file first. The `await` completes when the drag ends; do not mutate the item's state assuming the drop succeeded until the result says so.

### Tray icon

Avalonia's built-in `TrayIcon` (with a `NativeMenu`) covers both platforms. Mutations go through the UI thread:

```csharp
// Bad - worker thread touches the tray; native menu APIs are not thread-safe
trayIcon.ToolTipText = $"Done: {count} duplicates";

// Good - the worker posts; the UI thread owns every tray mutation
Dispatcher.UIThread.Post(() => trayIcon.ToolTipText = $"Done: {count} duplicates");
```

Decide close-to-tray versus quit explicitly, persist it, and always leave a discoverable quit path in the tray menu. An app that vanishes into the tray with no visible exit is a support ticket.

### Notifications: the silent no-op

The most common "works on my machine" defect in this stack. The toast call returns and nothing appears.

- **Windows**: toasts require an **AUMID** backed by a Start Menu shortcut carrying that AppUserModelID. Without it the notification is dropped by the OS with no error. The installer creates the shortcut - which is why packaging is a prerequisite and not polish
- **macOS**: notifications require a real `.app` bundle with a bundle identifier. A bare binary run from `dotnet run` cannot post one
- Avalonia has no OS-toast API; the interop (a community package or P/Invoke) is yours, and so is detecting whether it can work

```csharp
// Bad - "posted" means handed to the OS, not that the user saw it
toast.Show(message);

// Good - the capability is resolved once at startup and the UI adapts
if (Notifications.Support is NotificationSupport.Available) toast.Show(message);
else statusBar.ShowCompletion(message);
```

Ship an in-app completion surface regardless. A batch job that finished is information the user needs whether or not the OS agreed to deliver it.

### File watching: `FileSystemWatcher`

```csharp
// Bad - default 8 KB buffer; a 500-file copy overflows it and events vanish silently,
// and the handler touches UI state from a threadpool thread
var w = new FileSystemWatcher(dir) { IncludeSubdirectories = true, EnableRaisingEvents = true };
w.Changed += (_, e) => viewModel.Refresh(e.FullPath);

// Good - bigger buffer, Error handled as "rescan now", events coalesced off-thread
var w = new FileSystemWatcher(dir) { InternalBufferSize = 64 * 1024, IncludeSubdirectories = true };
w.Error   += (_, _) => channel.Writer.TryWrite(WatchEvent.OverflowRescan);
w.Changed += (_, e) => channel.Writer.TryWrite(WatchEvent.Changed(e.FullPath));
w.EnableRaisingEvents = true;
```

The real reliability model: the internal buffer (default 8 KB, max 64 KB) overflows under bursts and the lost events are gone - `Error` fires with `InternalBufferOverflowException`, and the only correct response is a rescan of the watched tree. A single editor save emits several events and a folder copy emits thousands, so coalesce batches (a `Channel` drained on a short timer) rather than reacting per event; there is no built-in debouncer. Events arrive on threadpool threads - marshal to the UI via `Dispatcher.UIThread` or a message. Watchers also die silently when the watched directory is deleted or a network share disconnects - surface that state and offer a re-arm. Scope the watch to what the UI displays, not a whole drive.

### Clipboard, hotkeys, single instance, autostart

- **Clipboard**: `TopLevel.GetTopLevel(view)?.Clipboard`, async API, reachable only through a live window. Copying a path copies its display string; the app keeps its own canonical path
- **Global hotkeys**: no Avalonia API. On Windows, P/Invoke `RegisterHotKey` and receive `WM_HOTKEY` through a message window. Registration fails when another app holds the combination - surface the conflict and let the user rebind; never register a bare common key. On macOS the interop (Carbon `RegisterEventHotKey` or an `NSEvent` global monitor) and its permission flow are also yours: detect the denial, explain it, and deep-link to System Settings
- **Single instance**: a named `Mutex` (`new Mutex(true, @"Local\MyApp", out var isFirst)`) answers only "am I first". Forwarding the second instance's argv to the first (a `NamedPipeServerStream` on both platforms) and raising the existing window is application code - without it, double-clicking a file does nothing visible. Two OS caveats: Windows foreground-lock rules may flash the taskbar button instead of raising the window - accept that as the OS's answer - and macOS delivers file-opens to a running app as Apple Events through the application lifetime, not as a second process's argv
- **Launch on login**: `HKCU\Software\Microsoft\Windows\CurrentVersion\Run` on Windows, a `LaunchAgents` plist on macOS. Per-user, opt-in, revocable from the app's own settings, and the setting is read back from the OS rather than from the app's config - the user may have turned it off elsewhere

### Credential storage

Windows Credential Manager (`CredRead`/`CredWrite` P/Invoke or a maintained wrapper) and the macOS Keychain. Two traps:

- **macOS returns `errSecMissingEntitlement` (-34018) for unsigned binaries.** This is a signing prerequisite, not a code bug
- **Ad-hoc signing changes identity on every rebuild**, so during development each build looks like a different app and cannot read what the previous one stored. Expect the miss; do not treat it as data loss

Store the smallest possible thing - a token, never a re-derivable secret - and treat the store as unavailable-by-default with a working degraded path (`desktop-security-patterns` owns what is safe to store).

## Output Format

Two modes, chosen by whether the request supplies code to judge or asks for code to be produced.

**Authoring mode** - the request is to write or design something. Emit the code or design per integration, each integration followed by its `Degraded:` line stating what the app does when the OS refuses or the capability is absent; close with any `Deferred:` lines. No finding blocks, no severity, no status line.

**Review mode** - source, a diff, or a symptom report was supplied. Emit one block per finding, ordered by severity, Critical first.

```
### [Severity] {file:line | file - symbol, when the line is unknown | symbol, when source was supplied without paths | symptom, when no source was supplied}

- Category: {Dialog | DragDrop | Tray | Notification | FileWatch | Clipboard | Hotkey | SingleInstance | Autostart | CredentialStore | ThreadAffinity}
- Platform: {Windows | macOS | both}
- Evidence: {source (a deployment or packaging fact stated in the request counts; quote it in Code:) | inferred (state what was not seen)}
- Code: {one-line citation, or `not supplied` when the finding is inferred}
- Prerequisite: {packaging | signing | notarization | OS permission | none; list every value that applies}
- Symptom: {what the user observes - "no toast appears and no error is logged"}
- Fix: {the concrete change, and the degraded path when the capability is absent}
```

`Severity: {Critical | High | Medium | Low}` - Critical = a capability the platform cannot provide is depended on with no alternative, or UI or native-menu state is mutated off the UI thread. High = the feature silently no-ops in a shipped build (missing AUMID or bundle, unsigned keychain access), an OS callback runs work on the delivering thread, or watcher events are consumed as truth with no rescan path. Medium = a missing degraded path for a denied permission or a conflicting hotkey, a default-buffer watcher with no `Error` handler, an unbounded recursive watch, or obsolete dialog APIs in new code. Low = a discoverability or persistence nit.

Severity that does not fit a listed band: assign the nearest lower band and state why in `Symptom`. A defect matching two bands takes the higher and names the other in `Symptom`. `Category` takes exactly one value - where a defect fits two, pick the one the `Fix` addresses and name the other in `Symptom`.

`Evidence: inferred` is required whenever the source was not read. It bounds the header at High and never raises a block.

A defect - or, in authoring mode, out-of-scope work the deliverable depends on - owned by a sibling named in the ownership blockquote is written after the findings or the authored output as `Deferred: {item} -> {owning skill}`, one per line. Omit when there are none.

In review mode, after any `Deferred:` lines, close per this table - when findings were emitted there is no separate closing line:

| Condition | Line |
| --- | --- |
| One or more findings emitted | none - the findings are the output |
| Source or a symptom report was available, nothing found | `No platform integration findings.` |
| No source, diff, or symptom supplied | `Platform integration check not run: no source supplied.` |

## Avoid

- `OpenFileDialog` or `SaveFileDialog` in new code
- Treating a cancelled picker as an error
- Assuming `TryGetLocalPath()` always returns a path
- Starting a drag-out without a live pointer event, or dragging files not yet on disk
- Treating a posted toast as proof the user saw it
- Shipping toasts with no Start Menu shortcut carrying the AUMID
- Testing notifications or keychain access from a bare `dotnet run` binary
- Mutating tray, menu, or UI state from a watcher, hotkey, or pipe callback thread
- Consuming `FileSystemWatcher` events with no `Error` handler and no rescan path
- Leaving `InternalBufferSize` at its 8 KB default over a bursty tree
- Treating watcher events as a source of truth rather than a hint to rescan
- Watching an entire drive or a network share recursively for one visible folder
- Registering a global hotkey with no conflict handling or rebind path
- Using the `Mutex` answer alone without forwarding argv and raising the existing window
- Reading launch-on-login state from app config instead of from the OS
- Reporting Keychain -34018 as a code defect rather than a signing prerequisite
