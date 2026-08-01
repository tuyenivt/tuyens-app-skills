---
name: flutter-security-patterns
description: "Harden Flutter apps: secure storage, cert pinning, obfuscation limits, deep-link and platform-channel input, WebView, biometrics, secrets."
metadata:
  category: mobile
  tags: [flutter, dart, security, secure-storage, pinning, deep-links, webview, biometrics, obfuscation, masvs]
user-invocable: false
---

# Flutter Security Patterns

> Confirm the project's target platforms first - they decide which hardening applies. Store selection lives in `flutter-data-persistence`; HTTP client construction in `flutter-networking`; route shape and deep-link wiring in `flutter-navigation-patterns`. This skill owns the security decision inside each.

**Threat model.** The client runs on the attacker's hardware. The device may be rooted or jailbroken, the binary can be extracted and decompiled, the traffic can be proxied, and the process can be instrumented at runtime. Everything below reduces exposure or raises attacker cost; none of it makes the client trusted. Any control that must hold is enforced server-side.

## When to Use

- Deciding where a token, key, or sensitive record lives on device
- Reviewing deep links, platform channels, WebViews, or notification payloads as untrusted input
- Choosing whether to pin, obfuscate, gate on biometrics, or detect root
- Auditing a diff or release configuration for shipped secrets and leaky platform flags

## Rules

- Tokens, credentials, and keys live in platform secure storage (Keychain / Keystore). Never `shared_preferences`, a plain file, or a plain database column
- **A secret compiled into the app is a published secret.** No API keys in source, `pubspec.yaml`, bundled asset files, or committed `--dart-define` files - those values are baked into the binary and readable from it
- Every value crossing an app boundary is untrusted: deep links, platform-channel arguments, WebView messages, notification payloads, clipboard, imported files. Parse, validate against an allowlist, then use
- Authorization is a server decision. The client hides UI; the server decides access
- Obfuscation, root detection, and integrity checks raise cost. Never gate a security-critical branch on their result alone
- A biometric prompt proves user presence on that device at that moment - not identity to your backend. Bind it to a key the OS withholds, not to an `if`
- Pinning is an operational commitment: multiple pins, a rotation plan, and a remote off-switch, or do not pin
- Configure backup and export flags explicitly. Platform defaults leak
- Never log tokens, PII, or full request bodies; keep debug affordances out of release builds

## Patterns

### Token storage

```dart
// Bad - plaintext plist / XML, readable on a rooted device, may land in a backup
await prefs.setString('access_token', token);

// Good - Keychain (iOS) / Keystore-backed store (Android)
await secureStorage.write(key: 'access_token', value: token);
```

- iOS Keychain items **survive app uninstall**. Delete them on logout, and on first run after install, or the next installer inherits a live session
- Set the Keychain accessibility class so items are unavailable before first unlock and never sync to iCloud (the `AfterFirstUnlockThisDeviceOnly` family). The Android counterpart is a Keystore-backed key excluded from backup
- Secure storage is small and slow: it holds keys and tokens, not datasets. Encrypt the dataset at rest and keep only its key there
- No browser store offers secure-storage guarantees - do not persist secrets on web

### A shipped secret is a public secret

```dart
// Bad - compiled in; `strings` on the extracted app finds it
const key = String.fromEnvironment('MAPS_API_KEY');

// Good - the device holds a user token; the backend holds the vendor key and proxies
final res = await api.post('/v1/geocode', data: {'q': q});
```

- If a vendor SDK genuinely requires an on-device key, treat it as public: restrict it at the vendor to your bundle id / package plus signing certificate, scope it to the minimum API surface, and monitor usage. Restriction is the control; secrecy is not on the table
- `--dart-define` and `--dart-define-from-file` are build configuration, not a secret mechanism. Committing the file makes it worse, not different
- Anything already shipped in a public build is compromised. Rotate it - deleting it from `git` does not un-ship it

### Certificate pinning and its rotation risk

- Pin the **public key** (SPKI hash), not the certificate: the key survives renewal, the certificate does not
- Ship at least one backup pin for a key you are not yet serving. Without it, renewal bricks every installed app - and a bricked client cannot be fixed by a server deploy
- Pair pinning with a server-driven off-switch and a forced-update path so a mis-rotation is recoverable
- Pinning defeats a network attacker with a user-installed CA. It does **not** defeat an attacker who owns the device; hooking frameworks patch the check. Scope the claim accordingly
- Android already distrusts user-added CAs for apps targeting modern API levels, so much of the casual-proxy threat is covered without pinning. Weigh the operational cost against that baseline
- Implement it once in the client factory (the underlying `HttpClient` certificate callback for a Dart-side check, or the platform's network security config / ATS), never per call site

### Obfuscation and its limits

```
flutter build appbundle --obfuscate --split-debug-info=build/symbols
```

- Renames Dart symbols. It does not encrypt strings, hide assets, protect Kotlin/Swift code, or stop runtime instrumentation
- Keep the symbol directory for every shipped build - crash reports are unreadable without the matching one, and a mismatched directory symbolizes to garbage
- It raises reverse-engineering cost. It is never a reason to move a secret or a check into the client

### Deep links are attacker-controlled input

```dart
// Bad - any app or web page can drive navigation into an authenticated surface
router.go(uri.queryParameters['next']!);

// Good - allowlist known routes, reject the rest
final next = uri.queryParameters['next'];
if (next != null && allowedRoutes.contains(next)) router.go(next);
```

- Custom URL schemes are **not exclusive**: another app can register `myapp://` and receive links meant for you. Only verified App Links / Universal Links (`assetlinks.json`, `apple-app-site-association`) bind to a domain you own
- A link navigates; it must not perform a state-changing or privileged action on arrival. The action still requires the session and, when sensitive, an explicit confirmation
- Ids and tokens in parameters are attacker-chosen. Access is decided by server-side authorization on the follow-up call, not by the parameter being present
- Same treatment for notification payloads and inbound intent extras

### Platform channels

```dart
// Bad - native side trusts the Dart caller
//   val path = call.argument<String>("path")!!
//   File(path).readText()

// Good - validate on the receiving side: type, range, allowlisted base directory
```

- The channel is not a boundary you control: anything in the process, including injected code, can invoke a method channel by name. Validate arguments natively even though "only our Dart calls it"
- The return direction is `dynamic`. Check types before casting values coming back into Dart
- Expose the specific operation the feature needs, never a general primitive (arbitrary file read, arbitrary URL open, shell execution)

### WebView safety

| Setting | Posture | Why |
| --- | --- | --- |
| JavaScript | off unless the content requires it | unrestricted JS is the precondition for everything below |
| JavaScript channels | none, or one narrow channel | a channel is a callable bridge for whatever script the page loads, including injected third-party script |
| Navigation | allowlist origins in the navigation delegate | otherwise a redirect walks the WebView to hostile content that still holds your channels |
| Loaded URL | never raw user- or link-supplied | deep link into a WebView is a standard chain |
| File access / file URLs | off | web content reading local files reads app-private data |
| Mixed content | off | silently downgrades the transport for subresources |

A JavaScript channel handler validates its input and must not proxy a privileged operation (token read, storage write, native call) to page script. If the handler would be dangerous when invoked by an arbitrary website, it is dangerous.

### Biometrics

```dart
// Bad - a bool decides access; patch the branch or the return value and it is gone
if (await auth.authenticate(localizedReason: 'Unlock')) showSecrets();

// Good - the OS releases the material only after a successful check; no bool to patch
// key stored with user-authentication required (Keystore) /
// biometry access control (Keychain); read fails without a passing prompt
final token = await secureStorage.read(key: 'refresh_token');
```

- What it proves: a credential enrolled on this device was presented just now. Nothing about which human, nothing about device integrity, nothing to your server. A backend must never accept "the client says biometrics passed"
- The strong form binds a stored key to biometric authentication at the platform layer so the value is unreadable without it. Whether the Flutter package in use exposes that binding varies - verify against the package before designing on it, and fall back to gating a locally encrypted value
- Enrolment changes matter: a newly added fingerprint or face should invalidate a biometric-bound key (both platforms offer a current-set binding for this)
- Always keep a non-biometric path (device passcode or re-login). Biometric-only locks users out on sensor failure
- `local_auth` provides the prompt; the security comes entirely from what a passing prompt is wired to

### Platform flags that leak by default

| Flag / mechanism | Risk at default | Posture |
| --- | --- | --- |
| Android `android:allowBackup` | app data copied off device via backup or device transfer | disable, or exclude sensitive paths via backup / data-extraction rules |
| Android `android:exported` | another app invokes your activity, receiver, or provider | `false` unless deliberately public; a deep-link intent filter forces `true`, so validate its input |
| iOS file backup | app files sync into iCloud / local backups | mark sensitive files excluded from backup; keep secrets in the Keychain with a `ThisDeviceOnly` class |
| Cleartext HTTP (`usesCleartextTraffic`, ATS exceptions) | silent plaintext downgrade | off; no per-domain exception without a written reason |
| Debug logging, debuggable builds, test endpoints | release ships with debug affordances | gate on `kReleaseMode` and verify in the built artifact, not the source |

Sensitive screens: Android `FLAG_SECURE` blocks screenshots and the task-switcher thumbnail; iOS has no equivalent flag, so cover the window when the app resigns active.

### Root and jailbreak detection: defense-in-depth only

- Bypassable by construction - the check runs inside the environment it is assessing. Root-hiding and hooking tools defeat it
- Useful as a **signal**: report the result to the server as one input to risk scoring, where the response can change without an app release
- Useless as a **gate**: refusing to launch on a rooted device blocks hobbyists, stops no real attacker, and breaks legitimate users
- When the decision matters, use platform attestation verified server-side (Play Integrity, App Attest / DeviceCheck) - the server evaluates the verdict, not the app

## Output Format

When invoked from a review workflow, emit one block per finding:

```
### [Severity] file:line

- Category: {Storage | Secrets | Transport | UntrustedInput | WebView | AuthN | PlatformConfig | Integrity}
- MASVS Area: {STORAGE | CRYPTO | AUTH | NETWORK | PLATFORM | CODE | RESILIENCE}
- Code: {one-line citation}
- Attack scenario: {who, with what access - rooted device, network MITM, malicious app, hostile page}
- Fix: {concrete change; name the store, flag, or validation}
- Server-side dependency: {none | what the backend must also enforce}
```

`Severity: {Critical | High | Medium | Low}` - Critical = credentials or user data recoverable with realistic access, or a control that exists only on the client. High = a concrete path to sensitive data or action given one plausible precondition. Medium = exposure requiring an unlikely precondition. Low = hardening with no exploitable path in this app.

**Unknown platform targets:** emit the finding with `Platform: unconfirmed` and name the target it depends on. Do not drop it.

If there are no findings, emit exactly `No security findings.` so the workflow knows the check ran.

When invoked from an implementation workflow, emit a decision table:

```
| Concern | Decision | Rationale |
|---------|----------|-----------|
| Refresh token | secure storage, deleted on logout and first run | Keychain survives uninstall |
| Vendor map key | server proxy | shipped binary is extractable |
| Pinning | not pinned | no rotation owner; platform CA baseline accepted |
| Deep-link `next` param | route allowlist | attacker-supplied navigation target |
```

## Avoid

- Tokens, credentials, or keys in `shared_preferences`, a file, a plain column, or any web store
- API keys in source, `pubspec.yaml`, assets, or a committed `--dart-define` file
- Treating `--obfuscate` as protection for an embedded secret, or shipping without keeping the symbol directory
- Pinning without backup pins, a rotation owner, and a remote off-switch
- Deep links or notification payloads used unvalidated, or performing a privileged action on arrival
- Trusting a custom URL scheme as if it were exclusive to your app
- Platform-channel arguments consumed natively without validation, or a channel exposing a general-purpose primitive
- WebViews with unrestricted JavaScript, unrestricted navigation, file access, or a channel that proxies privileged work to page script
- Loading a user- or link-supplied URL into a WebView
- Branching on a biometric bool, or reporting biometric success to a server as authentication
- Biometric-only access with no passcode or re-login fallback
- Leaving `allowBackup`, `exported`, or cleartext-traffic settings at their defaults
- Root detection as a gate, or any client-side check standing in for server-side authorization
- Logging tokens, PII, or full request bodies; `print` / verbose logging surviving into release
