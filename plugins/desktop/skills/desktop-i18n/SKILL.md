---
name: desktop-i18n
description: Localize .NET desktop apps - resx satellites, CultureInfo, IStringLocalizer, plurals, runtime switching, NFC/NFD filenames, collated sorting.
metadata:
  category: desktop
  tags: [csharp, dotnet, i18n, localization, resx, cultureinfo, istringlocalizer, unicode-normalization, nfc-nfd, collation, pluralization]
user-invocable: false
---

# Desktop i18n

> Confirm the target platforms before applying the normalization section - the NFC/NFD divergence is a Windows-versus-macOS behaviour and does not reproduce on one platform alone.
>
> This skill owns **user-visible language and locale-dependent text behaviour, including filename text**. Path mechanics belong to `desktop-filesystem-patterns`; duplicate-content comparison and batch-operation safety (dry-run, undo) to `desktop-batch-operations`; layout that must survive expanded strings to `avalonia-control-patterns`; text scaling and contrast to `desktop-accessibility`.

## When to Use

- Adding any user-facing string
- Displaying, sorting, comparing, or persisting a filename
- Formatting a number, byte size, date, time, or duration for display
- Adding a language, or building the locale switcher
- A bug that appears only on macOS, only for non-ASCII names, or only in one language

## Rules

- **No hardcoded user-facing string.** Every string a user reads resolves through a resource key - `.resx` via `IStringLocalizer` or the generated accessor
- **Never build a sentence by concatenating translated fragments.** One parameterized message per sentence; word order is not portable
- Plurals come from CLDR plural categories, never from `if (n == 1)`. `.resx` has no plural engine, so the app ships the category rule and keys per category; a one-category language (Japanese) keeps only the `Other` form with the count as a formatted argument
- Numbers, byte sizes, dates, and durations format through the active `CultureInfo`, never through a hardcoded pattern. Interpolated strings default to `CurrentCulture` - right for labels, a defect in persisted data
- **Filenames are compared and keyed on a normalized form; I/O always uses the exact string the filesystem returned.** A filename that round-trips through a normalizing step and back to disk is a corrupted rename
- **A user-visible file list is sorted with a collator, not ordinal comparison.** Byte order is not alphabetical order in any non-ASCII locale
- Locale is detected on first run, overridable in settings, persisted, and applied without a restart
- Anything written to disk - filename patterns, the undo journal, config, logs - formats with `CultureInfo.InvariantCulture`. A number written as `1,5` under German and parsed under en-US is a data defect
- `<InvariantGlobalization>` stays off. It is a trim and AOT size flag that deletes collation and per-culture formatting app-wide

## Patterns

### resx, satellites, and keys

`Strings.resx` holds the neutral language; `Strings.de.resx` and friends compile into satellite assemblies resolved against `CurrentUICulture` through `IStringLocalizer<T>` or the generated accessor.

```csharp
// Bad - unlocalizable, and the plural is wrong in Russian, Polish, and Welsh
var status = n == 1 ? "1 file renamed" : $"{n} files renamed";

// Good - per-category keys; the category comes from the culture's CLDR rule, not n == 1
// RenameStatus_One: "{0} file renamed"   RenameStatus_Other: "{0} files renamed"
var category = PluralRule.For(culture, n);            // One|Few|Many|Other; the rule table ships with the app
var status = string.Format(culture, localizer[$"RenameStatus_{category}"], n);
```

The BCL has no CLDR plural engine - the rule table is shipped as code or a small library, and the keys carry the category suffix. Keys are stable and semantic (`ConfirmDeleteTitle`), never the English text and never derived from a position. When a message's meaning changes, change the key - a stale translation of a changed string is worse than an untranslated one.

### Runtime culture switching in Avalonia

Setting `CultureInfo.DefaultThreadCurrentUICulture` redirects future resource lookups, but bound strings do not re-resolve on their own. Route bindings through one observable localization service:

```csharp
public sealed class Loc : INotifyPropertyChanged {
    public string this[string key] => Strings.ResourceManager.GetString(key, Culture) ?? key;
    public void SetCulture(CultureInfo c) {
        CultureInfo.DefaultThreadCurrentUICulture = c;
        CultureInfo.DefaultThreadCurrentCulture = c;
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs("Item[]"));  // every binding re-reads
    }
}
```

Views bind to the indexer (`{Binding [ConfirmDeleteTitle], Source={x:Static loc:Loc.Instance}}`), so one `PropertyChanged` re-renders the whole UI translated. The choice is persisted and applied immediately - requiring a restart is a defect.

### Unicode normalization: the filename trap

**This is the defect most specific to a file utility, and it appears when files cross platforms.**

The accurate platform statement: Windows filesystems store and return the name as given - in practice NFC, because that is what keyboards and software produce. On macOS, HFS+ normalizes names to a decomposed (NFD-style) form and returns that; APFS, the default since 2017, is normalization-preserving but case-insensitive by default, so decomposed names written by HFS+-era tools, archives, or other software persist and round-trip as-is. The same logical name can therefore arrive as `é` (U+00E9, one code point) or `e` + U+0301 (two), and a byte comparison reports one file as two.

```csharp
// Bad - a cache key written on Windows never matches the same file copied from a Mac
if (entry.Name == storedName) { /* never true across platforms */ }

// Good - String.Normalize for comparison and keys; I/O uses the filesystem's string
var key = entry.Name.Normalize(NormalizationForm.FormC);
if (key == storedKey) File.Move(entry.FullName, targetPath);   // original name, not key
```

Three consequences to design for explicitly:

- **Byte-wise round-trip breaks.** A rename rule that reads a name, transforms it, and writes it back may change only the encoding - a "no-op" rename that still touches every file
- **Cross-platform state.** Any persisted key derived from a filename (cache, undo journal, favourites) normalizes on write and on read, or it misses on the other platform
- **Comparison is normalization plus case.** Windows and default APFS are case-insensitive and case-preserving, so filename equality is NFC plus `OrdinalIgnoreCase`. Never `ToLower()` - culture-sensitive casing (the Turkish dotless i) makes it locale-dependent; use `OrdinalIgnoreCase` or `ToUpperInvariant` for filename keys

Normalization also changes length: an NFD name can exceed a length limit its NFC form fits under (`desktop-filesystem-patterns`).

### Sorting a file list

```csharp
// Bad - ordinal; "Zebra" sorts before "Ähnlich", and file10 before file9
names.Sort(StringComparer.Ordinal);

// Good - the active culture's collation for the display order
names.Sort(StringComparer.Create(culture, CompareOptions.IgnoreCase));
```

Three things a user notices when this is wrong: accented names cluster after `z` under byte order; German and Swedish disagree about where `ä` sorts - same characters, different locales, both correct; and `file10` sorts before `file9` unless numeric ordering is implemented. The collator does not provide numeric order - on Windows, P/Invoke `StrCmpLogicalW` (the Explorer order users expect), or ship an explicit comparer that compares digit runs numerically on both platforms. The sort key is computed from the normalized name; the row still displays and operates on the original. `Ordinal` keeps its place in persisted and internal keys, where stability across cultures is the point - just never in the visible list.

### Locale-aware formatting, and the invariant inverse

```csharp
// Bad - hardcoded separators and pattern; wrong across most of Europe
var label = $"{bytes / 1e6:F1} MB - {when:MM/dd/yyyy}";

// Good - the active culture formats what the user sees
var label = string.Format(culture, "{0:N1} {1} - {2:d}", size, localizer["UnitMB"], when);

// Good - anything persisted formats invariant, regardless of UI language
var journalLine = FormattableString.Invariant($"{seq}:{size:F1}:{when:O}");
```

Interpolated strings format with `CurrentCulture` by default - right for on-screen labels, a silent defect in filenames, journals, and logs; `FormattableString.Invariant` is the explicit escape. Byte sizes need a decision recorded, not assumed: KB/MB (1000) versus KiB/MiB (1024), and the unit names themselves are translated resources. Timestamps persist in the round-trip format (`"O"`), never a localized date pattern.

### Text expansion

German and Russian run 20-35% longer than English, and short button labels can double. Verify the longest shipped locale at the largest OS text scale (`desktop-accessibility`) - that combination is the real worst case. Containers grow and wrap; fixed widths sized to English truncate (`avalonia-control-patterns`). A pseudo-locale that brackets and expands every message by 40% catches both hardcoded strings and fixed containers before any translation exists.

## Output Format

Two modes, chosen by whether the request supplies code to judge or asks for code to be produced.

**Authoring mode** - the request is to write or design something. Emit the code or design, then any `Deferred:` lines. No finding blocks, no severity, no status line.

**Review mode** - source, a diff, or a symptom report was supplied. Emit one block per finding, ordered by severity, Critical first.

```
### [Severity] {file:line | symbol, when source was supplied without paths | symptom, when no source was supplied}

- Category: {HardcodedString | KeyStability | Concatenation | Pluralization | Formatting | InvariantOutput | Normalization | CaseFolding | Collation | TextExpansion | LocaleSwitching}
- Platform: {Windows | macOS | both}
- Evidence: {source | inferred (state what was not seen)}
- Code: {one-line citation, or `not supplied` when the finding is inferred}
- Impact: {what the user sees or loses - "the same file is listed twice after copying from a Mac"}
- Fix: {the concrete change}
```

`Severity: {Critical | High | Medium | Low}` - Critical = locale or normalization behaviour corrupts data (a locale-formatted number in a filename or journal, a rename driven by a normalized key, a dedup that treats NFC and NFD names as different files). High = a user-facing string is untranslatable or a shipped or planned locale is wrong (hardcoded string, concatenated sentence, two-form plural in a multi-form language, byte-order sort of a user-visible list). Medium = locale-dependent display formatting, text clipped by a fixed container, or a locale switch that needs a restart. Low = key naming or message organization nit.

Severity that does not fit a listed band: assign the nearest lower band and state why in `Impact`. `Category` takes exactly one value - where a defect fits two, pick the one the `Fix` addresses and name the other in `Impact`.

`Evidence: inferred` is required whenever the source was not read. It bounds the header at High and never raises a block; when the cap lowers a would-be Critical, say so in `Impact`.

When the app ships a single locale and has no localization stack installed, emit exactly `Single-locale project - i18n review limited to normalization, collation, and formatting findings.` before any findings, and report only the `Normalization`, `CaseFolding`, `Collation`, `Formatting`, and `InvariantOutput` categories. Normalization and collation defects are filename-correctness bugs and are still in scope in a monolingual app.

A defect - or, in authoring mode, out-of-scope work the deliverable depends on - owned by a sibling named in the ownership blockquote is written after the findings or the authored output as `Deferred: {item} -> {owning skill}`, one per line. Omit when there are none.

In review mode, after any `Deferred:` lines, close per this table - when findings were emitted there is no separate closing line:

| Condition | Line |
| --- | --- |
| One or more findings emitted | none - the findings are the output |
| Source or a symptom report was available, nothing found | `No i18n findings.` |
| No source, diff, or symptom supplied | `I18n check not run: no source supplied.` |

## Avoid

- User-facing string literals in C# or XAML
- Keys that are the English text, or derived from position or index
- Sentences assembled from concatenated translated fragments
- `if (n == 1)` plural branching
- Hardcoded decimal separators, date patterns, or unit suffixes in display strings
- `CurrentCulture` formatting written into filenames, journals, config, or logs
- Comparing filenames without normalizing, in any app that runs on both platforms
- Performing I/O with a normalized name instead of the string the filesystem returned
- `ToLower()` as the case-insensitivity test for filenames
- `Sort()` with the default or `Ordinal` comparer on a user-visible file list
- `file10` sorting before `file9` because numeric ordering was never implemented
- `<InvariantGlobalization>true</InvariantGlobalization>` in a localized build
- Fixed-width containers sized to the English string
- A locale change that requires an app restart
