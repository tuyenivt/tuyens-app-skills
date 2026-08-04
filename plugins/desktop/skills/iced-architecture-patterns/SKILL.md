---
name: iced-architecture-patterns
description: Iced 0.14 app structure - Model-Message-Update-View, message granularity, view vs domain state, non-blocking update, subscriptions vs Tasks.
metadata:
  category: desktop
  tags: [rust, iced, elm-architecture, message, update, view, subscription, task, state, components]
user-invocable: false
---

# Iced Architecture Patterns

> **Iced is pre-1.0, and this plugin's projects track latest rather than pinning a minor.** The requirement in `Cargo.toml` is therefore a range, so **`Cargo.lock` is the source of truth for which version is actually built** - read it, not the manifest, and verify every signature against that version's docs before writing or accepting one. Never assert an Iced API from memory. Tracking latest means the resolved version moves under you: a signature that was correct last month may not be, and `cargo update` is a deliberate migration to run and test, not routine housekeeping.
>
> This skill owns **the shape of the Iced application: state, messages, and the update/view split**. Which crate a type lives in belongs to `desktop-core-architecture`; widget construction, layout, and lists to `iced-widget-patterns`; `Task` internals, streaming progress, and cancellation to `iced-async-patterns`; long-job execution to `desktop-concurrency-patterns`.

## When to Use

- Starting an Iced app or adding a screen to one
- Reviewing a diff that touches the `Message` enum, `update`, or the model struct
- `update` grew a blocking call and the window stopped repainting
- Deciding between a subscription and a `Task`, or whether to split a component

## Rules

- **`update` never blocks.** No filesystem walk, no hashing, no `thread::sleep`, no blocking channel receive. Anything measured in more than a frame returns a `Task`
- **`update` is the only place state mutates.** `view` takes `&self` and returns an `Element`; a view function that mutates or performs I/O is a defect
- Domain state lives in the core crate; the model holds it plus view-only fields. Selection, scroll offset, dialog visibility, and text input buffers are view state and never leave the UI crate
- A message names **an event that happened**, not a mutation to perform. `FilesScanned(Vec<Entry>)` and `DeleteRequested(GroupId)`, not `SetFileList` and `RemoveRow`
- Messages are `Clone + Debug` and carry owned data. Iced clones them; borrowed data cannot cross the boundary
- **Subscriptions are for streams the app does not initiate**; `Task` is for work the app started. Time ticks, window events, and keyboard input are subscriptions; a scan the user clicked Start on is a `Task`
- A screen is split out when it owns its own state and message set, not when the file grew long
- The model derives no `Clone` it does not need - Iced does not require the whole state to be cloneable, and adding it invites accidental copies of large result sets. A large result set held by the model is by design, not a finding - the defects are cloning it and rendering it eagerly (`iced-widget-patterns`)

## Patterns

### The four pieces

```rust
struct App { /* state: core domain values + view-only fields */ }

#[derive(Debug, Clone)]
enum Message { /* events that happened */ }

fn update(&mut self, message: Message) -> Task<Message> { /* mutate, return follow-up work */ }
fn view(&self) -> Element<'_, Message> { /* pure render of &self */ }
```

The signatures and the entry point (`iced::application(..)`, its builder methods, and how `Settings` is supplied) have all moved between Iced minors. Confirm the exact shape against the pinned version rather than reproducing this sketch verbatim.

### Message granularity

```rust
// Bad - message explosion: one variant per field, mirroring the widgets
enum Message {
    PatternTextChanged(String), PatternFieldFocused, PatternFieldBlurred,
    ReplaceTextChanged(String), ReplaceFieldFocused, ReplaceFieldBlurred,
    CaseToggled(bool), RegexToggled(bool), RecursiveToggled(bool),
}

// Good - one variant per meaningful event, grouped by the thing that changed
enum Message {
    RuleEdited(RuleField, String),
    RuleFlagToggled(RuleFlag),
    PreviewRequested,
}
```

The bad version grows one variant per widget and produces an `update` that is a flat list of one-line assignments with no logic to read. The good version keeps the enum proportional to the app's behaviour.

The opposite failure costs as much:

```rust
// Bad - one catch-all variant; update becomes a nested match on strings
enum Message { UiEvent(String, String) }
```

The working granularity: a variant per event a user or the system produced, parameterized by which field it applied to. Nest per screen once there are several:

```rust
enum Message {
    Scan(scan::Message),
    Results(results::Message),
    Settings(settings::Message),
}
```

### View state versus domain state

```rust
// Bad - view concerns pushed into the core type, and domain logic pulled into the model
// core crate:
pub struct DuplicateGroup { pub files: Vec<PathBuf>, pub is_expanded: bool, pub row_y: f32 }

// Good - core stays domain-only; the UI holds its own view state keyed by domain id
// core crate:
pub struct DuplicateGroup { pub id: GroupId, pub files: Vec<FileEntry>, pub hash: ContentHash }
// UI crate:
struct App { groups: Vec<DuplicateGroup>, expanded: HashSet<GroupId>, selected: Option<GroupId> }
```

`is_expanded` in the core type means the core crate now has an opinion about a disclosure triangle, and any core test must construct one. Keyed view state also survives the domain data being replaced by a rescan.

| State | Lives in |
| --- | --- |
| Scan results, plans, hashes, settings schema | core crate, held by the model |
| Selection, expansion, scroll offset, dialog open | model, UI-crate types only |
| Text input buffers, validation display | model, UI-crate types only |
| In-flight job id, progress counters | model, UI-crate types only |
| The rule that turns input into a plan | core crate |

### `update` must not block

```rust
// Bad - the window freezes for the length of the scan; no repaint, no cancel
Message::StartScan => {
    self.results = core::scan(&self.root);   // 40 seconds on a large tree
    Task::none()
}

// Good - the work is handed off; update returns immediately
Message::StartScan => {
    self.scanning = true;
    Task::perform(core::scan_async(self.root.clone()), Message::ScanFinished)
}
Message::ScanFinished(results) => { self.scanning = false; self.results = results; Task::none() }
```

`update` runs on the UI thread between frames. Anything it does is time the window is not repainting and not accepting input. The blocking call is the single most common Iced defect and the one users report as "the app hangs". When the same job can be re-triggered before the last one finishes - a live preview recomputed per keystroke - the completion can arrive after its input changed; the generation-id discipline that drops stale results, like streaming progress and cancellation, belongs to `iced-async-patterns`.

### Subscription or Task

| Source of the work | Mechanism |
| --- | --- |
| User pressed a button, a job must run | `Task` returned from `update` |
| Continuous external stream (time, window resize, global keyboard) | `Subscription` |
| A long job that reports progress while running | `Task` producing a stream of messages |
| Watching a directory for changes while the app is open | `Subscription` |
| One follow-up action after a message | `Task` |

The distinguishing question is whether the app decided the work should start. A subscription is declared from the model each time it is re-evaluated and is keyed by identity - returning `Subscription::none()` from a later evaluation stops it, which is how a watcher is turned off.

### Splitting into components

Iced has no component object with its own hidden state. A "component" here is a module exposing a state struct, a message enum, and free `update`/`view` functions, with the parent owning the state and forwarding messages.

```rust
// results.rs
pub struct State { /* view state for this screen */ }
pub enum Message { RowToggled(GroupId), DeleteRequested(GroupId) }
pub fn update(state: &mut State, msg: Message) -> Action;
pub fn view<'a>(state: &'a State, groups: &'a [DuplicateGroup]) -> Element<'a, Message>;
```

Returning an `Action` type rather than a `Task` lets the child report what the parent must do (delete these files, switch screen) without the child knowing the parent's message enum. Split when a screen has its own state and message set. Do not split a view function that only got long - extract a plain helper function returning `Element` instead, which costs nothing.

## Output Format

When this skill produces a finding:

```
[Must|Recommend] {file:line | symbol, when source was supplied without paths}
Category: <blocking-update | state-placement | message-granularity | view-purity | subscription-vs-task | component-split | version-unverified>
Issue: <the defect, named>
Consequence: <what the user sees or what drifts - "window frozen for the scan's duration", "core type carries a disclosure flag">
Fix: <the concrete change>
```

`[Must]` for blocking-update and view-purity findings, and for a message payload that cannot compile as stated (borrowed or non-`Clone`). `[Recommend]` otherwise.

A defect the scope note assigns to another skill (crate placement, widget construction and lists, `Task` internals, long-job execution) is not filed under a category. It is returned as one line instead, so the observation survives:

```
Handoff: <desktop-core-architecture | iced-widget-patterns | iced-async-patterns | desktop-concurrency-patterns> - <the observation, one line>
```

When designing an app or screen rather than reviewing:

```
Iced version: <the resolved version from Cargo.lock | UNRESOLVED - why the lock could not be read>
Model: <domain fields from core, then view-only fields>
Messages: <the variants, grouped by event source>
Blocking work: <each long operation, and the Task or Subscription that carries it | none>
Subscriptions: <each, and what stops it | none>
Split: <screens with their own state and message set | single screen>
```

`UNRESOLVED` is the branch for a `Cargo.lock` that is absent or unreadable (no project yet, or no file access). The architecture shape - model split, message design, the update/view discipline - is version-stable and is given anyway; every stated signature then carries the `UNVERIFIED` tag, and `Cargo.toml`'s range is never used as a substitute, because a range cannot answer whether an API exists.

Any signature stated for an Iced API carries `verified against <version>` or `UNVERIFIED - confirm against the pinned version`. No Iced signature is asserted without one of the two.

## Avoid

- Filesystem, hashing, decoding, `thread::sleep`, or a blocking receive inside `update`
- `block_on` anywhere in the UI crate
- Mutating state or performing I/O inside `view`
- One message variant per widget or per field assignment
- A single catch-all message variant carrying strings the `update` re-parses
- View state (`is_expanded`, scroll offset, selection) stored on a core domain type
- Domain rules implemented inside `update` instead of called from core
- A message carrying a borrowed value or a non-`Clone` type
- Deriving `Clone` on a model holding large result sets
- A `Subscription` used for work the user explicitly started
- Splitting a screen into a module because the file was long rather than because it owns state
- Reproducing an Iced signature from memory instead of the pinned version's docs
