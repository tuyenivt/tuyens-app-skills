---
name: flutter-build-release
description: "Ship Flutter builds: flavors, --dart-define injection, Android/iOS signing, obfuscation and debug symbols, size reduction, CI store deploy."
metadata:
  category: mobile
  tags: [flutter, dart, release, flavors, signing, obfuscation, app-size, ci, packaging]
user-invocable: false
---

# Flutter Build and Release

> This skill owns staged rollout and rollout gating. A shipped build stays installed for months, so treat every change as one an older installed version must survive.

A mobile release is not revertible. Once an artifact is on user devices the only fix is a new version the user has to accept, so the properties that matter are decided at build time: what the binary contains, how it is signed, and whether its crashes will be readable.

## When to Use

- Setting up flavors and build variants, or adding an environment
- Configuring Android or iOS signing, locally or in CI
- Reducing app size, enabling obfuscation, or making crash reports legible
- Wiring a pipeline to a store track, or packaging for desktop or web
- Diagnosing "works in debug, broken in release"

## Rules

- A flavor is a build-time identity - application/bundle id, display name, icons, endpoints, third-party keys - and it exists on **both** Android (product flavors) and iOS (schemes plus build configurations), or it is not a flavor
- **`--dart-define` values are compiled into the binary and recoverable from it.** They select configuration. No API secret, signing credential, or private key goes through them, or through assets, or through source
- Signing material never enters the repository. Keystores, `.p12` files, provisioning profiles, populated `key.properties`, and service-account JSON live in CI secrets, are materialized at build time, and are removed after
- Whether or not a build is obfuscated, the debug symbols it produced are archived per version+build and uploaded to the crash reporter. Symbols are build-specific: lose them and every crash report from that build is permanently unreadable
- `version: <name>+<build>` in `pubspec.yaml` is the single source of truth; CI supplies the build number. Stores reject a reused build number, and a hand-edited second copy drifts
- Only release artifacts are shippable and only release artifacts are worth testing. Debug and profile builds differ in compilation, assertions, and tree-shaking, and hide the entire class of release-only failures
- Pin the Flutter SDK version in CI. "Latest stable" makes builds non-reproducible and breaks on Flutter release day
- Roll out staged against a monitored crash-free rate. There is no rollback for a shipped mobile build, only rolling forward on the user's schedule

## Patterns

### Flavor wiring

| Layer | Android | iOS |
| --- | --- | --- |
| Variant definition | `productFlavors` in `android/app/build.gradle[.kts]` | one Xcode scheme + build configuration per flavor |
| Identity | `applicationId` / `applicationIdSuffix` | bundle identifier per configuration |
| Assets and icons | `android/app/src/<flavor>/` | asset catalog per configuration |
| Invocation | `flutter build appbundle --flavor staging` | `flutter build ipa --flavor staging` |

`--flavor` fails when the matching Android product flavor or Xcode scheme is missing, so the two platforms drift into "staging exists on Android only" unless both are added together. The Dart entry point is orthogonal: `--target lib/main_staging.dart` selects per-flavor `main` files, while a single `main.dart` plus `--dart-define-from-file` is the alternative that keeps one entry point.

### Configuration in, secrets out

```bash
# Bad - the key ships inside the artifact and `strings` on the binary finds it
flutter build appbundle --dart-define=STRIPE_SECRET=sk_live_...

# Good - the binary carries only an environment selector
flutter build appbundle --flavor prod --dart-define-from-file=config/prod.json
```

```dart
const apiBase = String.fromEnvironment('API_BASE_URL'); // must be a const context
```

`--dart-define-from-file` reads its values into the binary exactly as individual defines do, so the file being uncommitted does not make its contents secret. Anything genuinely secret is fetched at runtime by an authenticated client, or never leaves the server: a client binary is not a trust boundary you control.

### Android signing

```properties
# android/key.properties - gitignored, written by CI from secrets, deleted after
storeFile=/tmp/upload.jks
storePassword=...
keyAlias=upload
keyPassword=...
```

`build.gradle` reads that file into the release `signingConfig`. The specific thing to check for is the Flutter template's default, which points the **release** build type at the **debug** signing config: it builds successfully and produces an artifact no store will accept. With Play App Signing, the key you hold is the upload key and Google holds the app signing key, which is what makes a lost upload key recoverable.

### iOS signing

Signing needs a distribution certificate (identity) plus a provisioning profile (which app id, which capabilities, which devices); a mismatch between them is the usual archive failure. CI authenticates with an App Store Connect API key rather than an Apple ID, and fastlane `match` keeps certificates and profiles in a private repository so machines converge instead of each minting its own. When exporting, the export method has to match the profile type - an app-store profile does not export an ad-hoc build. Capabilities enabled in Xcode (push, keychain sharing, app groups) must also be on the profile, which is the usual "archives locally, fails in CI".

### Obfuscation and symbols

```bash
flutter build appbundle --obfuscate --split-debug-info=build/symbols/<version>+<build>
```

The two flags are used together. Treat the symbols directory as a release artifact: archive it keyed by version and build number, upload it to Crashlytics or Sentry in the same pipeline run, and retain it as long as that build has installs. `flutter symbolize` can restore a captured Dart stack trace given the debug-info from **that exact build** and nothing else.

Obfuscation renames Dart identifiers, so anything keying off `runtimeType.toString()` or a type name breaks in obfuscated release only - the canonical release-only bug. It raises reverse-engineering effort; it does not protect a secret. Native crash symbols are a separate upload: Android native symbols and iOS dSYMs still have to be shipped for the native half of a stack.

### Size

| Lever | Effect | Cost |
| --- | --- | --- |
| App Bundle instead of a universal APK | store ships per-device slices | none, and required for Play |
| `--split-per-abi` for APK distribution | one APK per ABI rather than one fat APK | several files to distribute |
| `--obfuscate --split-debug-info` | symbols leave the binary | symbols must be archived |
| Asset audit | usually the single biggest win | effort |
| R8 / resource shrinking on the Android side | shrinks Kotlin/Java and resources | keep-rules needed for reflection |
| Deferred loading | smaller initial download | code-split boundaries to maintain |

Measure first: `--analyze-size` on a build produces a breakdown viewable in DevTools' app size tool (it analyzes one target platform at a time). Uncompressed images, multiple font families, and duplicated icon sets outweigh Dart code far more often than teams expect.

### CI and store deploy

The pipeline shape is: checkout -> pinned SDK -> `flutter pub get` -> codegen if the project uses it -> `flutter analyze` and tests -> materialize signing secrets -> build the flavor artifact -> upload symbols -> upload to a store track. Build numbers come from the CI run counter or a dedicated counter so they stay monotonic per store. fastlane `supply` (Play) and `pilot`/`deliver` (App Store) cover the upload; which track and which tester group are release configuration, not a manual step someone remembers.

```yaml
# Bad - unpinned SDK; the build changes under you on Flutter release day
- uses: subosito/flutter-action@v2
  with: { channel: stable }

# Good - reproducible
- uses: subosito/flutter-action@v2
  with: { flutter-version: 3.27.1 }
```

### Desktop and web

| Target | Artifact | Constraint |
| --- | --- | --- |
| Windows | MSIX (the `msix` package) or an installer | a signing certificate is needed for a trusted install; unsigned triggers SmartScreen |
| macOS | `.app` wrapped as DMG or pkg | Gatekeeper blocks unnotarized apps; hardened runtime and entitlements required |
| Linux | AppImage / snap / flatpak / deb | per-distro tooling, outside Flutter's build |
| Web | `build/web/` output | `--base-href` when hosted under a subpath; service-worker caching decides when users actually get the new build |

Web renderer and Wasm flags have changed across Flutter releases; read `flutter build web --help` on the project's pinned SDK rather than copying flags from an older guide. A web deploy is not live the moment it uploads - version the output and set cache headers deliberately, or returning users keep the previous shell.

## Output Format

When invoked from a build or release workflow:

```
Target: {android | ios | web | macos | windows | linux}
Artifact: {appbundle | apk | ipa | web | msix | dmg | appimage | deb}
Flavor: {<name> | none}
Entry Point: {lib/main.dart | lib/main_<flavor>.dart}
Config Injection: {--dart-define-from-file=<path> | --dart-define x<n> | none}
Secrets in Binary: {none | VIOLATION: <key> from <source>}
Signing: Android {key.properties from CI secret | DEBUG-SIGNED (defect) | unsigned}; iOS {<profile type> via <match | manual> | N/A}
Version: {<name>+<build>, build number from <source>}
Obfuscation: {on, symbols at <path> | off}
Symbol Upload: {<reporter> at <pipeline step> | MISSING}
Size: {<n> MB, measured via --analyze-size | unmeasured}
SDK Pin: {<version> at <file> | unpinned (risk)}
Store Track: {internal | alpha | beta | production | TestFlight <group> | N/A}
Rollout: {staged <n>%, gated on <signal> | 100% immediate (risk)}
```

When invoked from a review workflow or directly to diagnose a release-only failure, emit one finding block per issue:

```
### [Severity] file:line

- Rule: {flavor-parity | secret-in-binary | signing-config | symbol-retention | version-numbering | size-regression | sdk-pin | release-gate | platform-packaging}
- Code: {one-line citation}
- Problem: {what ships wrong, and why it cannot be fixed after release}
- Recommendation: {concrete edit}
```

`Severity: {Critical | High | Medium | Low}`. Critical = a secret in a shippable artifact, a debug-signed or debug-mode release, or a release whose crashes can never be symbolicated. High = the release process breaks or misleads under normal operation (flavor missing on one platform, unpinned SDK, reused build numbers, no rollout gate). Medium = cost or drift without immediate breakage (unmeasured size, manual upload steps). Low = naming, artifact paths.

If there are no findings, emit exactly `No build-release findings.` so the workflow knows the check ran.

## Avoid

- Treating `--dart-define` or `--dart-define-from-file` as secret storage
- Committing keystores, `.p12` files, provisioning profiles, service-account JSON, or a populated `key.properties`
- Shipping a release build that still points at the template's debug signing config
- `--obfuscate` without archiving and uploading that build's symbols
- Discarding symbols once a version is released, while its installs remain in the field
- Reusing a build number, or maintaining the version in two places
- Validating a change against the debug build only
- Adding a flavor on one platform and calling it done
- Cutting size before measuring which assets dominate
- An unpinned Flutter SDK in CI
- Shipping to 100% on day one with no crash-rate gate
- Assuming a web deploy reaches users immediately, or copying renderer flags from an older SDK's documentation
