---
name: desktop-core-architecture
description: Enforce the UI-free core in C# desktop apps - no Avalonia in the core csproj, one-way project references, plan-and-apply, injection seams.
metadata:
  category: desktop
  tags: [csharp, dotnet, avalonia, architecture, project-boundary, testability, dependency-injection, solution-layout]
user-invocable: false
---

# Desktop Core Architecture

> Confirm the solution layout from the `.sln` and `.csproj` files first - a single-project app with no core split is a different starting point from one that already has the boundary.
>
> This skill owns **where code lives and which way references point**. Avalonia's MVVM shape belongs to `avalonia-mvvm-patterns`; the semantics of preview and undo to `desktop-batch-operations`; whether a layer earns its keep to `desktop-overengineering-review`.

## When to Use

- Planning a new desktop feature's project layout
- Reviewing a diff that adds a class, a project, or a PackageReference
- Deciding whether a type belongs in the core or the UI project

## Rules

- **The core project has no Avalonia PackageReference.** This is enforced by its `.csproj` and a CI dependency-tree check, not by convention. A core that can write `using Avalonia;` will eventually do so
- References point one way: UI references core, never the reverse. A core class that knows a ViewModel or a view exists is a violation regardless of how it is spelled
- An operation resolves to an **inspectable plan** before anything is applied. Preview renders the plan; apply consumes the same plan
- Ambient capabilities - the clock, the filesystem root, randomness - arrive as parameters or interfaces. A core method calling `DateTime.Now` is untestable by construction
- The core owns domain types. A ViewModel re-declaring a parallel copy duplicates a definition that will drift
- **An interface is introduced for a seam that exists**, not for symmetry. One implementer plus one test double is a seam; one implementer alone is not

## Patterns

### The boundary, enforced by the project file

```xml
<!-- Bad - MyApp.Core.csproj; nothing stops a view type leaking in -->
<ItemGroup>
  <PackageReference Include="Avalonia" Version="12.1.0" />
  <PackageReference Include="System.IO.Hashing" Version="10.0.0" />
</ItemGroup>

<!-- Good - MyApp.Core.csproj; the boundary is mechanical -->
<ItemGroup>
  <PackageReference Include="System.IO.Hashing" Version="10.0.0" />
</ItemGroup>
```

The UI project references core, one-way:

```xml
<!-- MyApp.csproj -->
<ItemGroup>
  <PackageReference Include="Avalonia.Desktop" Version="12.1.0" />
  <ProjectReference Include="..\MyApp.Core\MyApp.Core.csproj" />
</ItemGroup>
```

Transitive leaks hide behind innocent-looking packages, so CI checks the whole tree, not just the direct list:

```bash
dotnet list src/MyApp.Core/MyApp.Core.csproj package --include-transitive > core-deps.txt
! grep -qi avalonia core-deps.txt   # fails the build if Avalonia appears anywhere
```

### What goes where

| Concern | Project | Why |
| --- | --- | --- |
| Traversal, filtering, grouping | core | Pure computation over paths |
| Hashing, comparison, dedup grouping | core | No UI involvement |
| Rename planning, collision resolution | core | The rules that must be exhaustively tested |
| Apply and undo | core | Destructive; needs test coverage a GUI cannot give |
| Progress *events* | core | The operation reports; it does not know who listens |
| Progress *rendering* | UI | ViewModel and control state |
| Selection state, scroll position, dialog visibility | UI | View state, meaningless to the core |
| ViewModels, commands, XAML views | UI | Avalonia's shape |
| Settings *schema and migration* | core | Testable, versioned |
| Settings *editing views* | UI | View state |

The recurring mistake is putting "progress" wholly in the UI. The core emits progress as values; the UI decides they become a progress bar.

### Plan and apply share one computation

```csharp
// Bad - preview and apply are two implementations of the same rules; they will drift
public static IReadOnlyList<string> PreviewRenames(IReadOnlyList<string> files);
public static void ApplyRenames(IReadOnlyList<string> files);  // computes names AGAIN, then renames

// Good - one computation, inspected or executed
public sealed record RenameStep(string From, string To);
public sealed record RenamePlan(IReadOnlyList<RenameStep> Steps);

public static RenamePlan PlanRenames(IReadOnlyList<string> files, RenameRules rules);
public static IReadOnlyList<StepOutcome> Apply(RenamePlan plan);
```

The preview is `PlanRenames` rendered. The apply is `PlanRenames` executed. There is no second place for the naming rules to live, so preview cannot lie about what apply will do - which for a destructive operation is the difference between a safe tool and an unsafe one.

### Injection seams

```csharp
// Bad - reaches the real world; a test must create real files at real times
public static List<string> StaleEntries(string dir) {
    var cutoff = DateTime.Now.AddDays(-1);
    // ...
}

// Good - the capability arrives; the test supplies a fixed instant
public static List<string> StaleEntries(string dir, DateTimeOffset now) {
    var cutoff = now.AddDays(-1);
    // ...
}
```

Prefer a plain parameter over an interface; `TimeProvider` is the framework seam when many call sites need the clock. A filesystem interface is worth it when tests must simulate permission errors; it is overhead when a temp directory under `Path.GetTempPath()` already gives a real one cheaply.

### Domain types cross the boundary; view types do not

```csharp
// Bad - the ViewModel re-declares what core defines; two definitions to keep in sync
public partial class DuplicateRowViewModel : ObservableObject {
    [ObservableProperty] private string _path;
    [ObservableProperty] private long _size;
    [ObservableProperty] private string _hash;
}

// Good - the ViewModel wraps the core type; view state stays beside it
public partial class DuplicateRowViewModel(DuplicateGroup group) : ObservableObject {
    public DuplicateGroup Group { get; } = group;
    [ObservableProperty] private bool _isSelected;
}
```

The core type is an immutable record so the UI can bind to it without mutating it. That is the core accommodating a general constraint, not a UI dependency.

### Brownfield: the boundary starts as a namespace

In a single-project app with logic inline in ViewModels, scope the fix to the feature at hand: extract its rules into a folder and namespace with no Avalonia `using`, called from the ViewModel rather than written in it. The namespace gets the plan/apply shape and injected capabilities like any core code; only the project-file enforcement is missing, so `grep -r "using Avalonia" src/MyApp/Rename/` returning nothing stands in for the dependency-tree check. Promote the namespace to a real class library when a second feature joins it - as its own change, not this one.

## Output Format

When this skill produces a finding, each carries the block below; findings are ordered `[Must]` first:

```
[Must|Recommend] {file:line | file - symbol, when the line is unknown or the excerpt's numbering is unreliable | symbol, when source was supplied without paths}
Boundary: <core -> UI | UI -> core | within core | within UI>
Issue: <the violation, named>
Why it matters: <what becomes untestable, or what will drift>
Fix: <the concrete move or signature change>
```

`core -> UI` covers any dependency from core toward the UI side, including an Avalonia package in the core project file. A UI-side re-declaration of a core type is `UI -> core`: the dependency that should exist is being dodged.

`[Must]` when the boundary is breached mechanically (an Avalonia package in core's dependency tree, core importing a UI type) or when preview and apply compute the plan in two places. `[Recommend]` for seams, type placement, and structure.

A defect owned by a sibling named in the ownership blockquote is written after the findings or the assessment form as `Deferred: {defect} -> {owning skill}`, one per line. Omit when there are none.

When designing a new layout or assessing an existing one, rather than reviewing a diff, produce instead the form below. A diff or concrete change set takes finding blocks; any other request - greenfield design, brownfield advice, "where should this live" - takes the form. For a greenfield design the form describes the target state.

```
Projects: <projects and their reference edges>
Core purity: <clean | Avalonia present in core | no core project>
Seams: <injected capabilities, or `none - core reaches ambient state at <file:line | symbol, when lines are unknown>`>
Misplaced: <each type in the wrong project, with its destination | none>
Next: <the scoped move this change makes and the trigger that promotes the boundary | boundary is a class library from day one - no promotion applies | none>
```

## Avoid

- An Avalonia package anywhere in the core project's dependency tree
- A core class importing a view type, a ViewModel, or `Dispatcher`
- Preview and apply computing the same plan in two places
- A core method calling `DateTime.Now`, `Random.Shared`, or reading an environment variable directly
- Re-declaring a core type in the UI project to avoid a reference
- An interface with one implementer and no test double, introduced for symmetry
- Putting progress reporting entirely in the UI so the core cannot be tested for it
- A whole-solution re-layering proposed as a fix for one feature's misplacement
- Making every core member `public` so the UI can reach internals the boundary should hide
