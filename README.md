# Tuyen's App Skills

Single marketplace repository for Claude Code client application plugins: `flutter` and `unity`. Each is fully standalone - install exactly the one your project needs.

## Recommended: Project-Scoped Installation

**Install at the project (repo) level, not at the user level.**

Each project should only load the skills it actually needs. Installing every plugin globally at user scope bloats each Claude Code session with skills for stacks you're not using, wasting context window space and making the skill picker noisy.

The right pattern: **one marketplace add per machine, then per-project plugin installs.**

### Step 1 - Add the marketplace once (user scope, done once per machine)

```bash
claude plugin marketplace add tuyenwork/tuyens-app-skills
```

### Step 2 - Install the relevant plugin inside each project (project scope)

Run these commands from your project root. Claude Code stores the selection in the project's local settings, so only those skills load when you open that project.

**Flutter / Dart project:**

```bash
claude plugin install flutter@tuyens-app-skills --scope project
```

**Unity 2D game project:**

```bash
claude plugin install unity@tuyens-app-skills --scope project
```

**C# desktop utility project:**

```bash
claude plugin install desktop@tuyens-app-skills --scope project
```

> Each plugin is self-contained and has no dependencies. Install exactly one per project - `flutter`, `unity`, and `desktop` are never installed together.

## How Skills Work

Each plugin contains two types of skills:

- **Workflow skills** (`task-*`, `user-invocable: true`): End-to-end task flows invoked as slash commands (e.g., `/task-flutter-review`). These are the skills you interact with directly.
- **Atomic skills** (`user-invocable: false`): Focused, single-concern patterns hidden from the slash menu. They are composed automatically by workflow skills or triggered by your prompt - you never call them directly.

> Use only workflow skills (`task-*`) as slash commands. Atomic skills run behind the scenes.

## Which Skill Do I Use?

```
Flutter / Dart (plugin: flutter)
  implement a new feature              -> /task-flutter-implement
  staff-level code review              -> /task-flutter-review
  performance review                   -> /task-flutter-review-perf
  security review                      -> /task-flutter-review-security
  test strategy / scaffolds            -> /task-flutter-test

Unity 2D games (plugin: unity)
  implement a new feature              -> /task-unity-implement
  staff-level code review              -> /task-unity-review
  performance review                   -> /task-unity-review-perf
  security review                      -> /task-unity-review-security
  test strategy / scaffolds            -> /task-unity-test

C# desktop utilities (plugin: desktop)
  implement a new feature              -> /task-desktop-implement
  staff-level code review              -> /task-desktop-review
  performance review                   -> /task-desktop-review-perf
  security review                      -> /task-desktop-review-security
  test strategy / scaffolds            -> /task-desktop-test
```

> Neither plugin reviews API contract design - a client consumes API contracts rather than designing them. The concern that does reach clients, where an installed old app version must survive a server contract change, is checked inside the umbrella review and `/task-<stack>-implement`. Accessibility is handled during `/task-<stack>-implement` and checked at baseline depth inside the umbrella review, alongside adaptivity and localization.

**Common decision points:**

- "Flutter desktop vs. the desktop plugin" - both cover desktop, and they are distinguished by app type rather than by platform. Use `flutter` for a UI-heavy or backend-consuming desktop app, especially one sharing a codebase with mobile. Use `desktop` for a native-performance local utility whose cost is filesystem traversal, hashing, and image decode, and which has no backend at all.
- "Umbrella review vs. focused review" - `/task-<stack>-review` is the general review; it auto-escalates into the perf and security lenses when the change set fires their signals. Call a focused review directly when you already know which lens you need.
- "Review vs. implement" - reviews read a change set and report findings. `/task-<stack>-implement` designs and writes the feature, including its tests and any schema migration.
- "Which change set gets reviewed" - reviews read the **working tree** by default, so uncommitted work is the subject. Pass `--staged` to review only staged changes; with a clean tree the review falls back to the last commit.

## Plugin Catalog

| Plugin                     | Focus                                                                                                                                                            |
| -------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [flutter](plugins/flutter) | Flutter / Dart 3.x client apps - Riverpod, go_router, Dio, Drift. Mobile primary, desktop secondary, web tertiary                                                 |
| [unity](plugins/unity)     | Unity 6.3 LTS / C# 2D games - casual and puzzle titles. Engine-free rules core, URP 2D, UI Toolkit, Addressables. Mobile primary, desktop secondary, WebGL tertiary |
| [desktop](plugins/desktop) | .NET 10 / Avalonia 12 desktop utilities - bulk file operations, dedup, local-first tools. UI-free core, CommunityToolkit.Mvvm, SQLite, SkiaSharp. Windows primary, macOS secondary |

## Notes

- Each plugin is self-contained - no shared plugin, no cross-plugin references.
- Each plugin folder has its own README with stack-specific usage and examples.

## License

This project is proprietary. All rights reserved.
