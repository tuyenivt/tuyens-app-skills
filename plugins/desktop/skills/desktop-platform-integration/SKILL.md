---
name: desktop-platform-integration
description: Wire Rust desktop OS integration - rfd dialogs, drag-and-drop, tray-icon, notify-rust, file watching, arboard, hotkeys, single-instance, keyring.
metadata:
  category: desktop
  tags: [rust, rfd, drag-and-drop, tray-icon, muda, notify-rust, file-watching, arboard, global-hotkey, single-instance, keyring, autostart, windows, macos]
user-invocable: false
---

# Desktop Platform Integration

> Confirm the target platforms and whether the app is packaged and signed before applying this skill - several of these APIs succeed in `cargo run` and silently do nothing in a shipped build.
>
> This skill owns **the app's contact surface with the OS**. Signing, bundling, and installer layout belong to `desktop-build-release`; what the app does with a dropped or watched path to `desktop-filesystem-patterns`; keeping OS callbacks off the UI thread to `desktop-concurrency-patterns` and `iced-async-patterns`; what is safe to store in a credential store to `desktop-security-patterns`.

## When to Use

- Adding a file/folder picker, drag-and-drop, tray icon, notification, or global hotkey
- Watching a directory for external changes
- Reading or writing the clipboard
- Deciding launch-on-login, single-instance, or per-user install behaviour
- Storing a token or password on the user's machine
- A feature that works in development and does nothing after packaging

## Rules

- **Packaging and signing are prerequisites, not release polish.** Notifications, credential storage, and hotkey permissions all depend on the app having a stable OS identity. Feature work that assumes them is untestable until they exist
- Every integration has a **declared degraded path**. The OS refuses, the permission is denied, or the platform lacks the feature - the app states what it does then, in the UI, not in a log
- **No OS callback runs application logic on the thread it arrives on.** Tray, hotkey, watcher, and dialog callbacks hand off a message and return
- macOS tray and menu APIs are **main-thread only**. Constructing or mutating them from a worker thread is undefined at the AppKit level
- A path arriving from the OS - drop, dialog, watcher, or forwarded argv - is a `PathBuf` handled by `desktop-filesystem-patterns` rules, never a `String`
- **Never claim a platform capability the framework does not have.** Where the gap is upstream and open, name it and design around it rather than proposing an integration that cannot be built

## Patterns

### Capability matrix

| Capability | Crate | Windows | macOS | Prerequisite that bites |
| --- | --- | --- | --- | --- |
| Native dialogs | `rfd` 0.17 | yes | yes | Async variant off the UI thread |
| Drag-and-drop **in** | winit/Iced events | yes | yes | - |
| Drag-and-drop **out** | none | **no** | **no** | Confirmed winit gap, open since 2020 |
| Tray icon + menu | `tray-icon` + `muda` | yes | yes | macOS main-thread only |
| Notifications | `notify-rust` | yes | yes | **Silently no-ops** without AUMID / `.app` bundle |
| File watching | `notify` 8.2 + debouncer | yes | yes | Raw events are bursty; debounce required |
| Clipboard | `arboard` | yes | yes | Contents die with the process on some paths |
| Global hotkeys | `global-hotkey` | yes | yes | macOS accessibility permission flow is yours |
| Single instance | `single-instance` | yes | yes | Boolean only; argv forwarding is yours |
| Launch on login | registry / `LaunchAgents` | yes | yes | Per-user, and the user can revoke it |
| Credential storage | `keyring` 4.1.6 | yes | yes | macOS `-34018` on unsigned binaries |

### Notifications: the silent no-op

The most common "works on my machine" defect in this stack. `notify-rust` returns `Ok` and nothing appears.

- **Windows**: toasts require an **AUMID** backed by a Start Menu shortcut carrying that AppUserModelID. Without it the notification is dropped by the OS with no error. The installer creates the shortcut - which is why packaging is a prerequisite and not polish
- **macOS**: notifications require a real `.app` bundle with a bundle identifier. A bare binary run from `target/debug` cannot post one

```rust
// Bad - Ok(()) means "handed to the OS", not "the user saw it"
Notification::new().summary("Done").body(&msg).show()?;

// Good - the capability is resolved once at startup and the UI adapts
match platform::notification_support() {
    NotificationSupport::Available => { /* post */ }
    NotificationSupport::Unavailable(reason) => status_bar.show_completion(&msg, reason),
}
```

Ship an in-app completion surface regardless. A batch job that finished is information the user needs whether or not the OS agreed to deliver it.

### Dialogs and drag-and-drop

`rfd`'s blocking pickers stall the event loop; use `AsyncFileDialog` and deliver the result as a message (`iced-async-patterns`). A cancelled dialog returns `None` - that is a normal outcome, not an error.

Drag-and-drop **in** arrives as window events and may deliver multiple paths, directories, and non-UTF-8 names. Handle hover/enter as a distinct state so the drop target is visibly live before the drop.

Drag-and-drop **out** to Explorer or Finder is **not available**: winit has no drag-source API, the issue has been open since 2020, and Iced inherits the gap. Do not design a flow that requires it. The substitutes are a "Reveal in Explorer/Finder" action, a copy-paths-to-clipboard action, or an explicit export-to-folder picker.

### Tray, menus, and the macOS main thread

```rust
// Bad - AppKit menu objects touched from a worker; undefined behaviour on macOS
thread::spawn(move || tray.set_menu(Some(Box::new(rebuild_menu(state)))));

// Good - the worker sends state; the main thread owns every tray mutation
thread::spawn(move || tx.send(Msg::MenuState(compute(state))).ok());
// on the main thread, in the event loop:
Msg::MenuState(s) => tray.set_menu(Some(Box::new(build_menu(&s)))),
```

Decide close-to-tray versus quit explicitly, persist it, and always leave a discoverable quit path in the tray menu. An app that vanishes into the tray with no visible exit is a support ticket.

### File watching

`notify` 8.2 raw events are bursty: a single save from an editor commonly emits create, modify, and rename events, and a copy of 500 files emits thousands. Use the debouncer, not a hand-rolled timer.

```rust
// Bad - rescans the tree per raw event; a folder copy triggers thousands of rescans
watcher.watch(&dir, RecursiveMode::Recursive)?;
for ev in rx { rescan(&dir); }

// Good - debounced batches, coalesced, off the UI thread
let mut deb = new_debouncer(Duration::from_millis(500), None, tx)?;
deb.watch(&dir, RecursiveMode::Recursive)?;
// each batch -> one Message::TreeChanged(paths)
```

Watching recursively over a large tree or a network share is expensive; scope the watch to what the UI displays. Watchers also die silently when the watched directory is deleted or a share disconnects - surface that state and offer a re-arm.

### Clipboard, hotkeys, single instance, autostart

- **Clipboard** (`arboard`): on X11 the contents are owned by the process and disappear on exit; on Windows and macOS the OS takes a copy. Copy of a path uses the display form; the app keeps the `PathBuf`
- **Global hotkeys** (`global-hotkey`): registration fails when another app holds the combination - surface the conflict and let the user rebind. On macOS the hotkey requires Accessibility permission, and **the crate does not drive that flow**: detect the denial, explain it, and deep-link to System Settings. Never register a bare common key
- **Single instance** (`single-instance`): the crate answers only "am I first". Forwarding the second instance's argv to the first (named pipe on Windows, Unix socket on macOS) and raising the existing window is application code. Without it, double-clicking a file does nothing visible
- **Launch on login**: `HKCU\...\Run` on Windows, a `LaunchAgents` plist on macOS. Per-user, opt-in, revocable from the app's own settings, and the setting is read back from the OS rather than from the app's config - the user may have turned it off elsewhere

### Credential storage

`keyring` 4.1.6 fronts Windows Credential Manager and the macOS Keychain. Two traps:

- **macOS returns `-34018` (`errSecMissingEntitlement`) for unsigned binaries.** This is a signing prerequisite, not a code bug
- **Ad-hoc signing changes identity on every rebuild**, so during development each build looks like a different app and cannot read what the previous one stored. Expect the miss; do not treat it as data loss

Store the smallest possible thing - a token, never a re-derivable secret - and treat the store as unavailable-by-default with a working degraded path (`desktop-security-patterns`).

## Output Format

Two modes, chosen by whether the request supplies code to judge or asks for code to be produced.

**Authoring mode** - the request is to write or design something. Emit the code or design, one `Degraded:` line per integration stating what the app does when the OS refuses or the capability is absent, then any `Deferred:` lines. No finding blocks, no severity, no status line.

**Review mode** - source, a diff, or a symptom report was supplied. Emit one block per finding.

```
### [Severity] {file:line | symbol, when source was supplied without paths | symptom, when no source was supplied}

- Category: {Dialog | DragDrop | Tray | Notification | FileWatch | Clipboard | Hotkey | SingleInstance | Autostart | CredentialStore | ThreadAffinity}
- Platform: {Windows | macOS | both}
- Evidence: {source | inferred (state what was not seen)}
- Code: {one-line citation, or `not supplied` when the finding is inferred}
- Prerequisite: {packaging | signing | notarization | OS permission | none}
- Symptom: {what the user observes - "no toast appears and no error is logged"}
- Fix: {the concrete change, and the degraded path when the capability is absent}
```

`Severity: {Critical | High | Medium | Low}` - Critical = a capability the platform cannot provide is depended on with no alternative, or an AppKit object is touched off the main thread. High = the feature silently no-ops in a shipped build (missing AUMID or bundle, unsigned keychain access) or an OS callback runs work on the delivering thread. Medium = a missing degraded path for a denied permission or a conflicting hotkey, or an unbounded recursive watch. Low = a discoverability or persistence nit.

Severity that does not fit a listed band: assign the nearest lower band and state why in `Symptom`. `Category` takes exactly one value - where a defect fits two, pick the one the `Fix` addresses and name the other in `Symptom`.

`Evidence: inferred` is required whenever the source was not read. It bounds the header at High and never raises a block.

A defect owned by a sibling named in the ownership blockquote is written after the findings as `Deferred: {defect} -> {owning skill}`, one per line. Omit when there are none.

In review mode, close with exactly one status line, after any `Deferred:` lines:

| Condition | Line |
| --- | --- |
| One or more findings emitted | none - the findings are the output |
| Source or a symptom report was available, nothing found | `No platform integration findings.` |
| No source, diff, or symptom supplied | `Platform integration check not run: no source supplied.` |

## Avoid

- Designing a drag-out-to-Explorer flow; winit has no drag source
- Treating `Notification::show()` returning `Ok` as proof the user saw it
- Shipping toasts with no Start Menu shortcut carrying the AUMID
- Testing notifications or keychain access from a bare `target/debug` binary on macOS
- Constructing or mutating tray icons and menus off the macOS main thread
- Running application work on the thread a tray, hotkey, or watcher callback arrives on
- Blocking `rfd` pickers on the UI thread
- Treating a cancelled dialog as an error
- Acting on raw `notify` events without a debouncer
- Watching an entire drive or a network share recursively for a single visible folder
- Registering a global hotkey with no conflict handling or rebind path
- Assuming macOS Accessibility permission exists without detecting the denial
- Using `single-instance` without forwarding argv and raising the existing window
- Reading launch-on-login state from app config instead of from the OS
- Reporting macOS keychain `-34018` as a code defect rather than a signing prerequisite
- Converting an OS-supplied path to `String` before use
