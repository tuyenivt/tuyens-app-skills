---
name: flutter-i18n
description: "Localize Flutter with gen_l10n and ARB files: no hardcoded strings, plurals, locale date/number/currency formats, RTL layout, text expansion."
metadata:
  category: mobile
  tags: [flutter, dart, i18n, l10n, gen-l10n, arb, intl, rtl, localization, pluralization]
user-invocable: false
---

# Flutter Internationalization

> Platform tiers are defined in `flutter-adaptive-responsive`; localization behaves identically across tiers, so patterns here are `All` unless stated. Semantic labels are user-facing text and are localized under the same rule - see `flutter-accessibility`.

## When to Use

- Adding or reviewing any user-facing string, number, date, or currency
- Setting up `gen_l10n` and ARB files, or adding a locale to an existing setup
- Adding support for an RTL language, or auditing a layout for directional correctness
- A report of untranslated text, a wrong plural form, a date in the wrong format, a mirrored layout defect, or a label overflowing in one language only

## Rules

- No user-facing string is written as a literal in Dart. Every one lives in an ARB file and is read through the generated localizations class
- Sentences are never assembled by concatenating translated fragments. One key holds one complete sentence with placeholders
- Counted quantities use ICU `plural`; conditional phrasing uses ICU `select`. `if (count == 1)` in Dart is a defect - the number of plural categories is a property of the language, not of the code
- Dates, times, numbers, percentages, and currency are formatted through the ARB placeholder `format` (or `intl` directly) with the active locale. Never `toString()`, never `toStringAsFixed`, never manual `dd/MM/yyyy`
- Every ARB key carries an `@key` entry with a `description`. Translators see the string with no surrounding UI; the description is the only context they get
- Directional padding, alignment, and positioning use the `*Directional` variants. `left` and `right` mean physical sides and do not mirror
- Locale resolution has an explicit fallback and the fallback locale is complete. A missing key must not be able to reach the user
- Layout accommodates text expansion: no fixed-width box around translated text, no single-line assumption on a label
- Currency formatting and currency *conversion* are separate concerns. Formatting an amount in the user's locale never changes which currency it is in

## Patterns

### Setup and the no-hardcoding rule

*Tiers: All.*

`l10n.yaml` at the project root, with `generate: true` under `flutter:` in `pubspec.yaml`:

```yaml
arb-dir: lib/l10n
template-arb-file: app_en.arb
output-localization-file: app_localizations.dart
```

```dart
MaterialApp(
  localizationsDelegates: AppLocalizations.localizationsDelegates,
  supportedLocales: AppLocalizations.supportedLocales,
  home: const HomeScreen(),
)
```

```dart
// Bad - untranslatable, and invisible to any search for user-facing text
const Text('Order total')

// Good
Text(AppLocalizations.of(context)!.orderTotal)
```

The generated class is build output: raise findings against the ARB file and the call site, never against `app_localizations.dart`. Setting `nullable-getter: false` in `l10n.yaml` removes the `!` at every call site.

```dart
// Bad - two fragments; word order, gender agreement, and spacing are not translatable
Text('${l10n.deletedPrefix} ${item.name}')

// Good - one key, one placeholder
Text(l10n.deletedItem(item.name))
```

Concatenation bakes English grammar into the code. Languages that reorder the sentence, inflect the noun, or attach a particle cannot be expressed by joining two independently translated halves.

### Plurals and select

*Tiers: All.*

```dart
// Bad - two categories; correct for English, wrong for Polish, Russian, and Arabic
Text(count == 1 ? l10n.oneOrder : l10n.manyOrders(count))
```

```json
{
  "orderCount": "{count, plural, =0{No orders} =1{1 order} other{{count} orders}}",
  "@orderCount": {
    "description": "Number of orders shown in the list header",
    "placeholders": { "count": { "type": "int" } }
  }
}
```

```dart
Text(l10n.orderCount(orders.length))
```

The ICU categories are `zero`, `one`, `two`, `few`, `many`, `other`, plus exact matches like `=0`. The template ARB only needs the forms English uses; translators supply the forms their language needs, which is exactly what the Dart branch makes impossible.

`select` handles non-numeric variation, most commonly gender:

```json
{
  "invitedBy": "{gender, select, male{He invited you} female{She invited you} other{They invited you}}",
  "@invitedBy": {
    "description": "Shown on an invitation card",
    "placeholders": { "gender": { "type": "String" } }
  }
}
```

`other` is mandatory in both forms and is the fallback for any unmatched value.

### Dates, numbers, and currency

*Tiers: All.*

```dart
// Bad - month/day order is a locale property; this is US-only and unlocalized
Text('${d.month}/${d.day}/${d.year}')
Text('\$${amount.toStringAsFixed(2)}')
```

```json
{
  "placedOn": "Placed on {date}",
  "@placedOn": {
    "description": "Order detail header",
    "placeholders": { "date": { "type": "DateTime", "format": "yMMMd" } }
  },
  "orderTotal": "{amount}",
  "@orderTotal": {
    "description": "Order total, already in the order's own currency",
    "placeholders": {
      "amount": {
        "type": "double",
        "format": "currency",
        "optionalParameters": { "decimalDigits": 2 }
      }
    }
  }
}
```

The `format` value maps to an `intl` `DateFormat` / `NumberFormat` skeleton, and the generated code passes the active locale automatically. Outside ARB (a chart axis, an export), use `intl` explicitly with the locale: `DateFormat.yMMMd(Localizations.localeOf(context).toString()).format(d)`.

Locale changes the decimal separator, the grouping separator, the digit set, and the symbol position, so a number rendered by string interpolation is wrong in most locales even when the digits are right. Format at the point of display, never store a formatted string.

### RTL and directional layout

*Tiers: All.*

Text direction comes from the resolved locale via the localization delegates; no manual `Directionality` wrapper is needed for the app as a whole.

```dart
// Bad - the icon stays on the physical left in Arabic and Hebrew, colliding with the text
Padding(padding: const EdgeInsets.only(left: 16), child: Row(children: [icon, label]))

// Good - start and end follow the reading direction
Padding(padding: const EdgeInsetsDirectional.only(start: 16), child: Row(children: [icon, label]))
```

| Physical | Directional |
|----------|-------------|
| `EdgeInsets.only(left:, right:)` | `EdgeInsetsDirectional.only(start:, end:)` |
| `Alignment.centerLeft` | `AlignmentDirectional.centerStart` |
| `BorderRadius.only(topLeft:)` | `BorderRadiusDirectional.only(topStart:)` |
| `Positioned(left:, right:)` | `PositionedDirectional(start:, end:)` |

`Row` with `MainAxisAlignment.start` already mirrors; it is the explicit `left` / `right` values that do not. Directional icons (back arrows, next chevrons, send, undo) need mirroring too - either use a mirrored asset per direction or flip on `Directionality.of(context)`; `Image` exposes `matchTextDirection` for the asset case. Icons with intrinsic orientation (a clock, a checkmark, a brand mark) must not be mirrored.

Verify with a real RTL locale (`ar`, `he`) rather than by inspection: `MaterialApp(locale: const Locale('ar'), ...)` in a widget or golden test.

### Locale resolution and fallback

*Tiers: All.*

Default resolution matches language plus country, then language alone, then falls back to `supportedLocales.first`. Two consequences worth stating explicitly:

- The first entry in `supportedLocales` is the fallback for every unmatched device locale. Make it the complete locale, not an alphabetical accident.
- A key present in the template ARB but missing from a translation falls back to the template value, so an untranslated app silently ships English strings. Set `untranslated-messages-file` in `l10n.yaml` so the gap is a build artifact instead of a discovery in production.

Override only when the default is wrong for the product (for example, routing every unsupported Spanish variant to `es` rather than the first supported locale):

```dart
MaterialApp(
  localeListResolutionCallback: (deviceLocales, supported) => /* explicit choice */,
)
```

A user-selected locale is app state that must be persisted and applied through `MaterialApp(locale:)`; reading the device locale alone cannot express "the user chose a different language than their phone".

### Text expansion

*Tiers: All.*

Translated strings are commonly longer than English: short labels can double or more, and running text typically grows 20-40%. German, Finnish, and Russian are the usual first failures.

```dart
// Bad - fits "Save"; clips "Speichern unter" in German
SizedBox(width: 88, child: ElevatedButton(onPressed: _save, child: Text(l10n.save)))

// Good - the button sizes to its label, with a floor rather than a fixed width
ConstrainedBox(
  constraints: const BoxConstraints(minWidth: 88),
  child: ElevatedButton(onPressed: _save, child: Text(l10n.save)),
)
```

The same defect class as fixed-height text containers in `flutter-accessibility`, on the other axis, and it compounds with text scaling: a long German string at 200% font scale is the worst case both mechanisms produce together. Where truncation is genuinely acceptable, make it explicit and reversible (`maxLines` plus `overflow: TextOverflow.ellipsis` plus a way to see the full text), rather than letting a `Row` overflow.

Test the long case: render key screens in a verbose locale in golden or widget tests instead of only in the template locale.

## Output Format

When invoked from an implementation workflow, emit the localization plan:

```
Setup: {l10n.yaml at <path> | not configured}
Template locale: {code}   Supported: {codes}   RTL in scope: {yes | no}

| String / value | ARB key | Form | Format | Notes |
|----------------|---------|------|--------|-------|
| Order count header | orderCount | plural | - | =0 / =1 / other |
| Order date | placedOn | plain | DateTime yMMMd | placeholder |
| Order total | orderTotal | plain | currency, 2 dp | currency is the order's, not the locale's |
| Delete tooltip | deleteOrder | plain | - | semantic label |
```

When invoked from a review workflow or directly to diagnose a localization defect report, emit one block per finding:

```
### [Blocker | High | Medium | Low] file:line

- Area: I18n
- Check: {Hardcoded-String | Concatenation | Plural-Form | Locale-Format | RTL-Layout | Locale-Resolution | Text-Expansion | Translator-Context}
- Tier: {Mobile | Desktop | Web | All}
- Code: {one-line citation}
- Impact: {what a user in an affected locale reads, or cannot read}
- Fix: {concrete edit}
```

Close with one coverage line: `Checks clean: {comma-separated Check values with zero findings | none}`.

**Severity calibration.** `Blocker` = text is unreadable or wrong in a supported locale (clipped label, mirrored layout collision, a date or amount that reads as a different value). `High` = correct characters, wrong language or wrong grammatical form for a supported locale (hardcoded string, two-branch plural). `Medium` = correct today but structurally fragile (concatenation, fixed-width box that currently fits, missing `description`). `Low` = key naming and ARB organization.

**Label mapping for the umbrella review:** `Blocker`, `High` -> `[Must]`; `Medium`, `Low` -> `[Recommend]`.

If the project has no `l10n.yaml` or ARB directory, emit `Localization not configured` once, cap hardcoded-string findings at `Medium`, and raise the setup itself as a single `High` finding rather than filing one finding per literal. Findings cite the ARB file or the Dart call site, never the generated localizations output.

## Avoid

- User-facing string literals in Dart, including semantic labels, tooltips, error messages, and dialog titles
- Sentences built by concatenating or interpolating translated fragments
- `count == 1` branching in Dart instead of an ICU `plural` key
- `toString()`, `toStringAsFixed`, or a hand-built `dd/MM/yyyy` for anything the user reads
- A hardcoded currency symbol, or reformatting an amount into the locale's currency instead of the order's
- ARB keys with no `@key` description
- `EdgeInsets.only(left:)`, `Alignment.centerLeft`, `Positioned(left:)` in a layout that ships an RTL locale
- Mirroring an icon with intrinsic orientation, or leaving a directional arrow unmirrored
- `supportedLocales.first` chosen by accident - it is the fallback for every unmatched device
- Shipping without `untranslated-messages-file`, so missing translations surface as silent English
- Fixed-width buttons, chips, and labels around translated text
- Verifying RTL or long-text behaviour by inspection instead of rendering an `ar` or `de` locale in a test
- Raising findings against generated localization output instead of the ARB source
