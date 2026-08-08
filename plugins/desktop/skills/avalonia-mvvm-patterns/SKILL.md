---
name: avalonia-mvvm-patterns
description: Avalonia MVVM with CommunityToolkit.Mvvm - ObservableProperty, RelayCommand, state placement, DI, ViewLocator, dialogs, non-blocking ViewModels.
metadata:
  category: desktop
  tags: [csharp, avalonia, mvvm, communitytoolkit, observable-property, relay-command, viewlocator, dialogs, di, navigation]
user-invocable: false
---

# Avalonia MVVM Patterns

> This skill owns **the shape of the Avalonia application: ViewModels, commands, state placement, navigation, and dialogs**. Which project a domain type lives in belongs to `desktop-core-architecture`; XAML, templates, and virtualization to `avalonia-control-patterns`; async execution, dispatcher marshalling, and progress to `csharp-async-patterns`; long-job orchestration to `desktop-concurrency-patterns`. A project already on ReactiveUI keeps it - the Rules still apply there (a service locator is still a defect), the fixes use ReactiveUI idioms, and migrating to the Toolkit is never proposed.

## When to Use

- Starting an Avalonia app or adding a screen to one
- Reviewing a diff touching a ViewModel, command, service registration, or the ViewLocator
- A ViewModel grew a `Window` or control reference, or the UI freezes during a command
- Deciding where a piece of state lives, or how a dialog is opened

## Rules

- **ViewModels never block the UI thread.** No filesystem walk, no hashing, no `.Result`, no synchronous SQLite query in a command body or property getter. Anything longer than a frame runs through an async command - `csharp-async-patterns` owns the mechanics
- **ViewModels reference no visual type.** No `Window`, `Control`, or `Application.Current`. Platform capabilities the VM needs - file picker, confirmation dialog, clipboard - enter behind interfaces the UI project implements
- Domain state lives in the core library and the ViewModel holds it; view state (selection, expansion, input buffers, progress counters) lives in the ViewModel only. Core types never gain view flags
- **Use the source generators.** `[ObservableProperty]` partial properties and `[RelayCommand]` methods on a class deriving `ObservableObject`. Hand-written `INotifyPropertyChanged` boilerplate is a defect in new code
- A command declares its availability: `[RelayCommand(CanExecute = ...)]` plus `[NotifyCanExecuteChangedFor]` on the properties that gate it. A button that no-ops via a guard clause instead of disabling is a defect
- `ObservableCollection<T>` is for small, user-edited lists. A 100k-row result set loads by swapping a read-only list property - one change notification, not 100k `Add` events
- One composition root: services and ViewModels are registered in one place and dependencies arrive by constructor. No service locator calls inside ViewModels
- **ViewModel-to-View resolution is explicit and AOT-safe.** The ViewLocator maps types through a compile-time registry, never `Type.GetType` on a transformed name string
- Views bind to VM state with compiled bindings (`avalonia-control-patterns` owns the XAML side); ViewModels expose bindable state and never reach into controls

## Patterns

### Source-generated properties and commands

```csharp
// Bad - 8 lines of ceremony per property, and PreviewCommand never re-evaluates
private string _pattern = "";
public string Pattern { get => _pattern; set { _pattern = value; OnPropertyChanged(); } }

// Good - the generator emits the property, equality guard, and notifications
[ObservableProperty]
[NotifyCanExecuteChangedFor(nameof(PreviewCommand))]
public partial string Pattern { get; set; }

[RelayCommand(CanExecute = nameof(CanPreview))]
private async Task PreviewAsync(CancellationToken ct) => Plan = await BuildPlanAsync(ct);
private bool CanPreview => Pattern.Length > 0 && !PreviewCommand.IsRunning;
```

`[RelayCommand]` on an `async Task` method generates an `AsyncRelayCommand` with `IsRunning` for busy state and `IncludeCancelCommand = true` for a paired cancel - use those instead of hand-rolled `IsBusy` flags.

### Where state lives

```csharp
// Bad - a view flag on the core type; every core test now constructs UI state
public sealed record DuplicateGroup(GroupId Id, IReadOnlyList<FileEntry> Files, bool IsExpanded);

// Good - core stays domain-only; the VM keys view state by domain id
public sealed record DuplicateGroup(GroupId Id, IReadOnlyList<FileEntry> Files, ContentHash Hash);
// ViewModel: private readonly HashSet<GroupId> _expanded = [];
```

| State | Lives in |
| --- | --- |
| Scan results, rename plans, hashes, settings schema | Core library, held by the ViewModel |
| Selection, expansion, dialog visibility, text buffers | ViewModel only |
| In-flight job state, progress counters | ViewModel only |
| The rule that turns a pattern into a plan | Core library |

Keyed view state also survives the domain data being replaced by a rescan - an `IsExpanded` flag on the row object dies with the row.

### ObservableCollection at scale

```csharp
// Bad - 100k Add calls, each raising CollectionChanged into the ItemsControl
foreach (var g in report.Groups) Groups.Add(g);

// Good - one notification; virtualization renders only the visible rows
[ObservableProperty]
public partial IReadOnlyList<DuplicateGroup> Groups { get; set; } = [];
// after the scan: Groups = report.Groups;
```

`ObservableCollection<T>` earns its place where the user edits incrementally - removing resolved groups after a delete, reordering rules. Bulk loads swap the reference. Rendering the swapped list without realizing 100k controls is `avalonia-control-patterns`' virtualization contract.

### Dialogs without a Window reference

```csharp
// Bad - the VM constructs and shows a Window; tests now need a windowing system
var dlg = new ConfirmDeleteWindow();
var ok = await dlg.ShowDialog<bool>(App.MainWindow);

// Good - the VM depends on an interface; the UI project implements it over ShowDialog
public interface IConfirmationService
{ Task<bool> ConfirmAsync(string title, string body, string confirmLabel); }

if (!await _confirm.ConfirmAsync($"Delete {selected.Count} files?", summary, "Delete")) return;
```

The same shape carries file and folder pickers (wrapping `IStorageProvider`) and the clipboard. The interface lives beside the ViewModels, the implementation in the UI project, and the test stub is the second implementer that justifies the interface - this is the exception to `csharp-language-patterns`' one-implementer rule, earned by the dependency direction.

### ViewLocator and navigation

```csharp
// Bad - name-string reflection; trimmed under NativeAOT, every screen renders "not found"
var type = Type.GetType(vm.GetType().FullName!.Replace("ViewModel", "View"));
return (Control)Activator.CreateInstance(type!)!;

// Good - an explicit registry; a missing mapping is visible in one file
public Control Build(object? vm) => vm switch
{
    ScanViewModel => new ScanView(),
    ResultsViewModel => new ResultsView(),
    SettingsViewModel => new SettingsView(),
    _ => new TextBlock { Text = $"No view for {vm?.GetType().Name}" },
};
```

Navigation is a ViewModel concern: the shell VM exposes `CurrentViewModel`, a `ContentControl` bound to it renders through the ViewLocator, and switching screens is assigning the property. A back stack, when needed, is a list of ViewModels on the shell - never control juggling in code-behind.

### DI and the composition root

```csharp
// Bad - a service locator inside the VM; the dependency is invisible and unstubbable
var scanner = ((App)Application.Current!).Services.GetRequiredService<IScanService>();

// Good - constructor injection from the single composition root
public ResultsViewModel(IScanService scanner, IConfirmationService confirm) { ... }
// App startup: services.AddSingleton<IScanService, ScanService>();
```

`Microsoft.Extensions.DependencyInjection` is sufficient - core services as singletons, ViewModels per their screen lifetime. Prefer direct references between VMs that share an owner; the Toolkit's `WeakReferenceMessenger` earns its place only when two ViewModels with no common owner must communicate.

### Never block the UI thread

```csharp
// Bad - a synchronous scan in the command body freezes every visible control
[RelayCommand]
private void Scan() => Results = _scanner.ScanTree(Root);   // 40 s on a large tree

// Good - async command; the window stays responsive and cancel stays possible
[RelayCommand(IncludeCancelCommand = true)]
private async Task ScanAsync(CancellationToken ct)
    => Results = await Task.Run(() => _scanner.ScanTree(Root, ct), ct);
```

The same rule covers property getters: a getter that computes over the result set runs on every binding read, on the UI thread. Compute once into a set property. `Task.Run` placement, progress streaming, and dispatcher rules are `csharp-async-patterns`' contract.

## Output Format

When this skill produces a finding, emit one block per finding, `[Must]` first:

```
[Must|Recommend] {file:line | file, when the line is unknown | symbol, when the source carries no path}
Category: <blocking-ui-thread | view-reference-in-vm | state-placement | inpc-boilerplate | canexecute-missing | collection-scale | service-locator | viewlocator-aot | dialog-coupling | messenger-overuse>
Issue: <the defect, named>
Consequence: <what the user sees or what drifts - "window frozen for the scan's duration", "AOT build renders no views">
Fix: <the concrete change>
```

`[Must]` for blocking-ui-thread, view-reference-in-vm, viewlocator-aot, and collection-scale findings at user-data-driven sizes - freezes, untestable ViewModels, and a broken AOT build are structural. `[Recommend]` otherwise. `Category` takes exactly one value - where a defect fits two, the `[Must]`-listed one wins; between two of equal label, pick the one the `Fix` addresses and name the other in `Consequence`. Repeated instances of the same defect in one file merge into one block with every location named.

A defect - or, in design mode, out-of-scope work the deliverable depends on - owned by a sibling named in the ownership blockquote is written after the findings or the design form as `Deferred: {item} -> {owning skill}`, one per line. Omit when there are none.

When designing a screen rather than reviewing:

```
ViewModel: <name, and the screen it backs>
State: <core-owned fields, then view-only fields>
Commands: <each, with its CanExecute gate and sync | async>
Services: <constructor dependencies, and which project implements each>
Dialogs: <the interfaces used | none>
Navigation: <how the screen is reached and left | single screen>
Blocking work: <each long operation and the async command carrying it | none>
```

A review that produces no finding closes with exactly `No MVVM findings.` - a correct `ObservableCollection` on a small edited list is not written up as a defect.

## Avoid

- Hand-written `INotifyPropertyChanged` where the generators exist (outside ReactiveUI projects)
- `Window`, `Control`, or `Application.Current` reachable from a ViewModel
- An enabled button whose command body starts with a guard-clause no-op
- Loading a bulk result through per-item `Add` on an `ObservableCollection`
- View flags on core domain types
- Service locator calls inside ViewModels
- A ViewLocator built on `Type.GetType` and naming conventions
- `ShowDialog` called from a ViewModel
- Proposing a ReactiveUI-to-Toolkit migration
- Synchronous I/O or queries in a command body or property getter
- `WeakReferenceMessenger` where a constructor reference or plain event reaches
- Code-behind event handlers doing work a bound command should own
