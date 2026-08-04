---
name: desktop-i18n
description: Localize Rust desktop apps with fluent and icu4x - runtime locale switching, plurals, formatting, NFC/NFD filename divergence, collated sorting.
metadata:
  category: desktop
  tags: [rust, i18n, fluent, i18n-embed, fluent-templates, icu4x, unicode-normalization, nfc-nfd, collation, pluralization, locale]
user-invocable: false
---

# Desktop i18n

> Confirm the target platforms before applying the normalization section - the NFC/NFD divergence is a Windows-versus-macOS behaviour and does not reproduce on one platform alone.
>
> This skill owns **user-visible language and locale-dependent text behaviour, including filename text**. Path and `OsStr` mechanics belong to `desktop-filesystem-patterns`; duplicate-content comparison and batch-operation safety (dry-run, undo) to `desktop-batch-operations`; layout that must survive expanded strings to `iced-widget-patterns`; text scaling and contrast to `desktop-accessibility`.

## When to Use

- Adding any user-facing string
- Displaying, sorting, comparing, or persisting a filename
- Formatting a number, byte size, date, time, or duration for display
- Adding a language, or building the locale switcher
- A bug that appears only on macOS, only for non-ASCII names, or only in one language

## Rules

- **No hardcoded user-facing string.** Every string a user reads resolves through a Fluent message keyed by a stable id
- **Never build a sentence by concatenating translated fragments.** One parameterized message per sentence; word order is not portable
- Plurals come from Fluent's CLDR plural categories, never from `if n == 1`. Several target languages have three or more categories; a one-category language (Japanese) keeps the selector with only `*[other]` so the count stays a formatted argument
- Numbers, byte sizes, dates, and durations format through `icu4x` against the active locale, never through `format!` with a hardcoded pattern
- **Filenames are compared and keyed on a normalized form; I/O always uses the `OsString` the filesystem returned.** A filename that round-trips through a normalizing step and back to disk is a corrupted rename
- **A user-visible file list is sorted with a collator, not `str::cmp`.** Byte order is not alphabetical order in any non-ASCII locale
- Locale is detected on first run, overridable in settings, persisted, and applied without a restart
- Data written to disk or a filename pattern formats with a fixed, locale-independent representation. A number formatted under a German locale and parsed under a US one is a data defect

## Patterns

### Fluent, embedding, and keys

`fluent` via `i18n-embed` (with `i18n-embed-fl` for the compile-checked `fl!` macro) is the stack: `.ftl` files per locale, compiled into the binary so a single-file distribution keeps every language.

```rust
// Bad - unlocalizable, and the plural is wrong in Russian, Polish, and Welsh
status = format!("{} files renamed", n);

// Good - one message, CLDR plural categories chosen inside the .ftl
// rename-status = { $count ->
//     [one] { $count } file renamed
//    *[other] { $count } files renamed
// }
status = fl!(LOADER, "rename-status", count = n);
```

Keys are stable, semantic, hyphenated Fluent identifiers (`confirm-delete-title` - a dot is not valid in a Fluent id; it denotes an attribute), never the English text and never derived from a position. When a message's meaning changes, change the key - a stale translation of a changed string is worse than an untranslated one.

Runtime switching is cheap in Iced: the locale lives in state, `view` re-runs on every message, so a `Message::LocaleChanged` that swaps the loader and persists the choice re-renders the whole UI translated. Nothing needs a restart, and requiring one is a defect.

### Unicode normalization: the filename trap

**This is the defect most specific to a file utility, and it only appears when the app runs on both platforms.**

macOS filesystems return decomposed forms (NFD): `é` comes back as `e` + U+0301, two code points. Windows stores and returns what it was given, in practice NFC: `é` as U+00E9, one code point. The same file, copied across, has different bytes in its name.

```rust
// Bad - a cache written on Windows never matches on macOS; a dedup pass
// reports two entries for one file; a rename "changes nothing" but rewrites the encoding
if entry.file_name() == stored_name { /* never true across platforms */ }

// Good - normalize for comparison, keep the original for I/O
use unicode_normalization::UnicodeNormalization;
let key: String = entry.file_name().to_string_lossy().nfc().collect();
if key == stored_key { fs::rename(entry.path(), &target)?; }  // original OsString, not key
```

Three consequences to design for explicitly:

- **Byte-wise round-trip breaks.** A rename rule that reads a name, transforms it, and writes it back may change only the encoding on macOS - a "no-op" rename that still touches every file
- **Cross-platform state.** Any persisted key derived from a filename (cache, undo journal, favourites) normalizes on write and on read, or it misses on the other platform
- **Comparison is normalization plus case.** Windows and default macOS are case-insensitive and case-preserving, so equality is `nfc(lowercase-fold(a)) == nfc(lowercase-fold(b))`, and the fold is Unicode case folding, not `to_lowercase` on ASCII

Normalization also changes length: an NFD name can exceed a length limit its NFC form fits under (`desktop-filesystem-patterns`).

### Sorting a file list

```rust
// Bad - byte order; "Zebra" before "Ähnlich", "ä" after "z", digits ordered as text
names.sort();

// Good - locale collation; numeric ordering is off by default and must be enabled
let mut prefs = CollatorPreferences::from(&locale);
prefs.numeric_ordering = Some(CollationNumericOrdering::True);
let collator = Collator::try_new(prefs, CollatorOptions::default())?;
names.sort_by(|a, b| collator.compare(a, b));
```

Three things a user notices when this is wrong: German and Swedish disagree about where `ä` sorts (the same characters, different locales, different correct answers); accented names cluster after `z` under byte order; and `file2` sorts after `file10` unless numeric collation is on. A file manager's list order is a primary surface - byte order there is a visible defect, not a nit.

The sort key is computed from the normalized name; the row still displays and operates on the original.

### Locale-aware formatting

```rust
// Bad - hardcoded separators and a fixed pattern; wrong across most of Europe
label = format!("{:.1} MB - {}", bytes as f64 / 1e6, date.format("%m/%d/%Y"));

// Good - icu4x formatters bound to the active locale
label = fl!(LOADER, "file-summary",
    size = decimal_fmt.format_to_string(&size),
    date = date_fmt.format(&when)?);
```

`icu4x` 2.0 covers decimal, list, datetime, and plural rules with compiled locale data, so only shipped locales cost binary size. Byte sizes need a decision recorded, not assumed: KB/MB (1000) versus KiB/MiB (1024), and the unit names themselves are translated messages.

The inverse rule matters as much: anything written to a filename pattern, a config file, an undo journal, or a log formats locale-independently. A sequence number rendered as `1.234` because the machine is German is a corrupted filename, not a display quirk.

### Text expansion

German and Russian run 20-35% longer than English, and short button labels can double. Verify the longest shipped locale at the largest OS text scale (`desktop-accessibility`) - that combination is the real worst case. Containers grow and wrap; fixed widths sized to English truncate. A pseudo-locale that brackets and expands every message by 40% catches both hardcoded strings and fixed containers before any translation exists.

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

`Severity: {Critical | High | Medium | Low}` - Critical = locale or normalization behaviour corrupts data (a locale-formatted number in a filename or journal, a rename driven by a normalized key, a dedup that treats NFC and NFD names as different files). High = a user-facing string is untranslatable or a shipped locale is wrong (hardcoded string, concatenated sentence, two-form plural in a multi-form language, byte-order sort of a user-visible list). Medium = locale-dependent display formatting, text clipped by a fixed container, or a locale switch that needs a restart. Low = key naming or message organization nit.

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

- User-facing string literals in Rust source or in `view` code
- Keys that are the English text, or derived from position or index
- Sentences assembled from concatenated translated fragments
- `if n == 1` plural branching in Rust
- `format!` with a hardcoded decimal separator, date pattern, or unit suffix
- Locale-formatted numbers written into filenames, journals, config, or logs
- Comparing filenames without normalizing, on any app that runs on both platforms
- Performing I/O with a normalized name instead of the `OsString` from the filesystem
- `to_lowercase` as the case-insensitivity test for filenames
- `sort()` or `str::cmp` on a user-visible file list
- Sorting `file10` before `file9` by omitting numeric collation
- Fixed-width containers sized to the English string
- A locale change that requires an app restart
- Shipping every locale's data when only a few locales are offered
