---
name: avalonia-control-patterns
description: Avalonia XAML - virtualized lists for 100k rows, DataTemplates, control themes, theme variants, compiled bindings, keyboard focus, error states.
metadata:
  category: desktop
  tags: [csharp, avalonia, xaml, virtualization, datatemplate, control-theme, theme-variant, compiled-bindings, keyboard, focus]
user-invocable: false
---

# Avalonia Control Patterns

> This skill owns **what is on screen and how it is built**: XAML structure, templates, styling, virtualization, and focus. ViewModel shape, commands, and state placement belong to `avalonia-mvvm-patterns`; async progress and dispatcher marshalling to `csharp-async-patterns`; measured render cost and profiling to `desktop-performance`; keyboard, contrast, and screen-reader policy to `desktop-accessibility`; translated strings to `desktop-i18n`; thumbnail decode to `desktop-image-processing`.

## When to Use

- Building or reviewing an `.axaml` view
- A result set is large enough that rendering every row is in question
- Adding a template, style, theme resource, or custom control
- Rendering the states around the happy path: scanning, empty, error, partial

## Rules

- **A list whose length is driven by user data is virtualized.** A dedup run returns 100k rows; realizing a control per row locks the UI and exhausts memory. `ListBox` virtualizes via its default `VirtualizingStackPanel`; a plain `ItemsControl` or a `StackPanel` items panel does not
- **Never wrap a virtualized list in an outer `ScrollViewer`.** Measured at infinite height, the list realizes every row - virtualization silently gone. The list scrolls itself
- **Compiled bindings everywhere:** `x:DataType` on the view or template, `x:CompileBindings="True"` (or `AvaloniaUseCompiledBindingsByDefault` in the csproj). Reflection bindings fail silently at runtime and break under NativeAOT; compiled bindings fail at build, which is where you want it
- Row state binds to the item, never lives on the control - containers are recycled during scroll, and control-held state surfaces on the wrong row
- Colors, brushes, and spacing come from resources with theme variants; a hardcoded hex color is why a view is unreadable in dark mode
- A `Style` adjusts instances by selector; a `ControlTheme` defines what a control is. App-wide restyling belongs in a `ControlTheme`; a one-off variant is a style class
- Every interactive control is keyboard-reachable in visual order, and custom interactive elements set `Focusable="True"` with a visible focus adorner (`desktop-accessibility` owns the policy depth)
- A view renders its states explicitly: idle, scanning, empty, error, populated. A view handling only the populated case is incomplete
- Compose before creating: a `UserControl` for a reusable composed piece, a `ControlTheme` to reskin, a `TemplatedControl` only for new behavior with its own properties

## Patterns

### Virtualization at 100k rows

```xml
<!-- Bad - ItemsControl does not virtualize; 100k rows realize 100k controls -->
<ScrollViewer>
  <ItemsControl ItemsSource="{Binding Groups}"/>
</ScrollViewer>

<!-- Good - ListBox virtualizes by default and scrolls itself -->
<ListBox ItemsSource="{Binding Groups}" SelectionMode="Multiple,Toggle">
  <ListBox.ItemTemplate>
    <DataTemplate x:DataType="vm:DuplicateGroupViewModel">
      <views:GroupRow/>
    </DataTemplate>
  </ListBox.ItemTemplate>
</ListBox>
```

What keeps it working: row templates stay flat and cheap, because rows are realized and recycled during scroll and a heavy template turns scrolling into stutter; thumbnails arrive asynchronously into a bound property with a placeholder, so a recycled row never decodes on the render path. Choose the container by interaction: `ListBox` for selectable rows, `DataGrid` (the `Avalonia.Controls.DataGrid` package) for sortable columns, `ItemsRepeater` (its own package) for custom layouts - and confirm the layout you give it virtualizes.

```xml
<!-- Bad - control-held state; after recycling, another row shows this row's check -->
<CheckBox IsChecked="True"/>
<!-- Good - state travels with the item -->
<CheckBox IsChecked="{Binding IsKept}"/>
```

### DataTemplates

```xml
<!-- Typed template: compiled bindings verify Name and SizeDisplay at build -->
<DataTemplate x:DataType="vm:FileRowViewModel">
  <Grid ColumnDefinitions="*,Auto">
    <TextBlock Text="{Binding Name}" TextTrimming="CharacterEllipsis"/>
    <TextBlock Grid.Column="1" Text="{Binding SizeDisplay}"/>
  </Grid>
</DataTemplate>
```

Every `DataTemplate` carries `x:DataType`. Display formatting (`SizeDisplay`, not a three-converter chain per cell) is precomputed in the ViewModel - at 100k rows, per-cell converter chains are measurable work and unreviewable XAML.

### Styles vs control themes

```xml
<!-- Bad - properties repeated inline per instance; drift guaranteed -->
<Button Background="#2D6CDF" CornerRadius="6" Padding="12,6" Content="Rename"/>

<!-- Good - a style class, applied where the variant is wanted -->
<Style Selector="Button.primary">
  <Setter Property="Background" Value="{DynamicResource AccentBrush}"/>
  <Setter Property="CornerRadius" Value="6"/>
</Style>
<Button Classes="primary" Content="Rename 214 files"/>
```

Pseudo-class selectors (`:pointerover`, `:pressed`, `:disabled`, `:focus`) express interactive states declaratively - code-behind toggling visual properties is a defect. Reach for a `ControlTheme` when the change applies to every instance of the control app-wide or replaces its template - key it `{x:Type Button}` in app resources and derive from the base theme (`BasedOn`) rather than restating it.

### Resources and theme variants

```xml
<!-- Bad - literal color; unreadable when the variant flips -->
<TextBlock Foreground="#222222"/>

<!-- Good - one key, a value per variant -->
<ResourceDictionary.ThemeDictionaries>
  <ResourceDictionary x:Key="Light">
    <SolidColorBrush x:Key="RowMutedBrush" Color="#5B6570"/>
  </ResourceDictionary>
  <ResourceDictionary x:Key="Dark">
    <SolidColorBrush x:Key="RowMutedBrush" Color="#9AA4B0"/>
  </ResourceDictionary>
</ResourceDictionary.ThemeDictionaries>
<TextBlock Foreground="{DynamicResource RowMutedBrush}"/>
```

Prefer the built-in Fluent theme resources before minting new keys - they already carry both variants. `DynamicResource` for anything that changes with the variant; `StaticResource` for values that never do.

### Compiled bindings

```xml
<!-- Bad - reflection binding; the typo renders an empty cell at runtime and breaks AOT -->
<TextBlock Text="{Binding Sizee}"/>

<!-- Good - the root types the whole view; the typo is a build error -->
<UserControl x:DataType="vm:ResultsViewModel" x:CompileBindings="True">
  <TextBlock Text="{Binding SizeDisplay}"/>
</UserControl>
```

Set `<AvaloniaUseCompiledBindingsByDefault>true</AvaloniaUseCompiledBindingsByDefault>` in the csproj so the default is safe. The rare binding that genuinely cannot be typed carries an explicit `x:CompileBindings="False"` and a comment saying why.

### Keyboard navigation and focus

```xml
<StackPanel KeyboardNavigation.TabNavigation="Continue">
  <TextBox Watermark="Pattern"/>
  <TextBox Watermark="Replacement"/>
  <Button IsDefault="True" Content="Preview" Command="{Binding PreviewCommand}"/>
  <Button IsCancel="True" Content="Cancel"/>
</StackPanel>
```

Author the XAML in traversal order so the default tab order matches the visual order - `TabIndex` is for the exception, not the rule. `IsDefault` and `IsCancel` wire Enter and Escape on dialogs. `ListBox` gives arrow-key navigation for free; a custom `ItemsRepeater` UI must add its own key handling, which is a cost to weigh when choosing it.

### The states around the happy path

```xml
<!-- Bad - independent booleans; two states can render at once, or none -->
<views:ScanProgress IsVisible="{Binding IsScanning}"/>
<views:ErrorPane IsVisible="{Binding HasError}"/>
<ListBox ItemsSource="{Binding Groups}" IsVisible="{Binding HasResults}"/>

<!-- Good - one state object; the selected template renders exactly one branch -->
<ContentControl Content="{Binding Screen}"/>
<!-- typed DataTemplates for IdleViewModel, ScanningViewModel (with cancel),
     EmptyResultViewModel, FailedViewModel (with retry), ResultsViewModel -->
```

Model the screen as one state the view switches over, so impossible combinations cannot render. An empty result is its own state ("No duplicates found"), distinct from idle ("Choose a folder to scan"). A partial failure is the populated state with a banner - the successes are still shown.

### Custom control, UserControl, or template

| Need | Approach |
| --- | --- |
| A reusable composed piece (progress header, file row) | `UserControl` |
| Restyle every instance of an existing control | `ControlTheme` |
| A one-off variant of a styled control | Style class |
| New behavior with its own bindable properties | `TemplatedControl` + `StyledProperty` |
| A one-off drawn visual (size histogram) | `Render` override on a `Control` |

The first three cover most needs. A `TemplatedControl` costs owning styled properties, a control theme, and keyboard behavior - reach for it last, and only when the behavior is genuinely new.

## Output Format

When this skill produces a finding, emit one block per finding, `[Must]` first:

```
[Must|Recommend] {file:line | file, when the line is unknown | symbol, when source was supplied without paths}
Category: <virtualization | recycling-state | compiled-binding | data-template | style-vs-theme | theme-variant | keyboard-focus | missing-state | control-choice>
Issue: <the defect, named>
Consequence: <what the user sees - "UI locks for seconds on a 100k-row result", "unreadable in dark mode">
Fix: <the concrete change>
```

`[Must]` when the consequence is a freeze or memory exhaustion, a reflection binding in an AOT-shipped view (`[Recommend]` when AOT status is unknown), wrong-row state after recycling, a blank pane on a reachable state, an interactive control with no keyboard path, or text unreadable under a shipped variant. `[Recommend]` otherwise. `Category` takes exactly one value - where a defect fits two, the one whose consequence meets a `[Must]` criterion wins; between two of equal severity, pick the one the `Fix` addresses and name the other in `Consequence`. Repeated instances of the same defect in one file merge into one block with every location named.

A defect, or out-of-scope work a fix or the design depends on, owned by a sibling named in the ownership blockquote is written immediately after the findings or the design form as `Deferred: {item} -> {owning skill}`, one per line, before any file list. Omit when there are none.

When designing a view rather than reviewing:

```
View: <name, and the screen it renders>
Structure: <the control composition, containers named>
Large lists: <the virtualizing container and its panel | N/A - bounded at <n> rows>
Bindings: <compiled via x:DataType | each exception, with why>
States rendered: <idle | scanning | empty | error | partial | populated - every branch the screen can reach>
Keyboard: <traversal order, dialog defaults, list navigation>
Theming: <resources used and variant coverage | gaps named>
```

When asked for guidance with no code supplied, answer in the design form, fill every slot the request allows, and name the `.axaml` files needed to go further - never emit `file:line` findings against code that was not shown. A guidance question asked alongside reviewed code is answered in prose after the findings, grounded in the Rules - the design form is for whole views only.

A review that produces no finding closes with exactly `No control findings.` - a bounded, non-virtualized list of a dozen rows is not a defect.

## Avoid

- `ItemsControl` or a `StackPanel` items panel for a user-data-driven list
- An outer `ScrollViewer` around a virtualized list
- Row state held by the control instead of the bound item
- Reflection bindings in new XAML; `x:CompileBindings="False"` without a comment
- Hardcoded colors instead of variant-aware resources
- Inline property soup repeated per instance where a style class fits
- Code-behind toggling visuals that pseudo-class selectors express
- Per-cell converter chains where a precomputed ViewModel property fits
- Synchronous image decode reachable from a row template
- A view that renders only the populated case
- Independent visibility booleans where one state object makes bad combinations unrepresentable
- A `TemplatedControl` where a `UserControl` or style class suffices
- Fixing tab order with `TabIndex` when reordering the XAML would do
