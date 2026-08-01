---
name: flutter-platform-channels
description: "Bridge Dart to native with MethodChannel, EventChannel, Pigeon, and FFI: plugin packages, permissions, threading, argument validation."
metadata:
  category: mobile
  tags: [flutter, dart, platform-channel, pigeon, ffi, plugin, permissions, native]
user-invocable: false
---

# Flutter Platform Channels

> Platform-tier sensitive: mobile is the default target, desktop support varies per plugin, and **web has no platform channels at all**.

The channel is a trust and failure boundary, not a function call. Everything crossing it is asynchronous, weakly typed, and untrusted in both directions.

## When to Use

- Calling a native API Flutter does not expose, or consuming a stream of native events
- Choosing between an existing plugin, `MethodChannel`, Pigeon, and FFI
- Building or reviewing a plugin package: federated structure, platform registration, test doubles
- Requesting and handling runtime permissions
- Diagnosing `MissingPluginException`, `PlatformException`, a call that never returns, or UI freezing while native work runs

## Rules

- Search pub.dev before writing a channel. A maintained plugin already carries the per-platform implementations, the permission plumbing, and the edge cases you have not hit yet
- Pick the mechanism from the shape of the call (table below), not from familiarity
- Prefer Pigeon over hand-written `MethodChannel` beyond a couple of methods: string method names and untyped maps are exactly the failure mode it removes
- **Arguments crossing the boundary are untrusted in both directions.** Dart -> native arguments frequently originate in deep links, WebViews, or server responses; native -> Dart return values are dynamic and cast at runtime. Validate on the receiving side, on both sides
- Only codec-supported types cross: null, bool, int, double, String, byte and typed-number lists, List, Map. Encode anything else explicitly - there is no object graph transfer
- Every call can fail three ways: `PlatformException` (native raised), `MissingPluginException` (no implementation on this platform), and never returning. Bound it with a timeout and handle all three
- Channel calls are wrapped by a repository or service; widgets and notifiers never hold a channel
- Native handlers run on the platform's main/UI thread. Long work moves to a background thread and replies on the main thread - a blocking handler freezes the native UI and stalls the Dart future
- Calling a channel from a background isolate requires initializing the messenger with the root isolate token; the default binary messenger is only wired on the root isolate
- Permissions are **declared** (Android manifest entry, iOS `Info.plist` usage-description string) **and requested at runtime**. A missing iOS usage string crashes on first access and fails store review

## Patterns

### Mechanism selection

| Need | Mechanism | Why |
| --- | --- | --- |
| A few native calls, request/response | `MethodChannel` | lowest setup cost |
| More than a couple of methods, or a signature that will evolve | **Pigeon** | generates typed Dart + Kotlin/Swift; a signature change breaks the build instead of a user's device |
| Native pushes events over time (sensors, connectivity, playback position) | `EventChannel` | maps directly to a Dart `Stream` |
| Existing C/C++ library, or per-call overhead matters | **FFI** (`dart:ffi` + `ffigen`) | direct call, no serialization, no message hop |
| Capability reused across apps, or needing third-party platform impls | plugin package (federated) | per-platform packages behind one interface |
| Web | JS interop (`dart:js_interop`, `package:web`) | there is no native host to receive a channel message |

### Three failure modes, not one

```dart
// Bad - the cast, a native error, and an unimplemented platform all reach the widget as raw crashes
final level = await _channel.invokeMethod('getBatteryLevel') as int;

// Good - typed call, bounded, mapped to a domain failure
final level = await _channel
    .invokeMethod<int>('getBatteryLevel')
    .timeout(const Duration(seconds: 5));
```

Wrap it: `on PlatformException` maps `e.code` to a typed failure, `on MissingPluginException` selects the fallback for platforms without an implementation, and `TimeoutException` covers a native side that never calls `result`. A dropped `result` callback is a permanently pending Dart future, which looks like a hung screen rather than an error.

### Untrusted in both directions

```kotlin
// Bad - a path that arrived from a deep link reaches the filesystem unchecked
val path = call.argument<String>("path")!!
result.success(File(path).readText())
```

```kotlin
// Good - presence, type, and domain checked before the argument reaches a sink
val path = call.argument<String>("path")
    ?: return result.error("BAD_ARG", "path required", null)
if (!isInsideAppSandbox(path)) return result.error("BAD_ARG", "path not allowed", null)
```

The native side is the last check before a real sink (filesystem, intent, SQL, shell, WebView, keychain), and Dart-side validation does not survive a repackaged app. In the other direction, treat a return value as unvalidated input: null-check and range-check it rather than `as T` on whatever came back, since a native-side type change surfaces only as a runtime cast error in release.

### Pigeon

```dart
// pigeons/messages.dart - the source of truth, not compiled into the app
@HostApi()
abstract class BatteryApi {
  int getLevel();
}
```

Running pigeon generates the Dart client plus a Kotlin/Swift protocol to implement natively; `@FlutterApi()` generates the reverse direction. Renaming a method now fails the native build. Treat generated output like all other codegen in the project - committed or CI-generated, consistently, never hand-edited.

### EventChannel lifecycle

```dart
// Bad - the subscription outlives the widget; native keeps the sensor running
_channel.receiveBroadcastStream().listen(_onEvent);

// Good - held, error-handled, cancelled in dispose
_sub = _channel.receiveBroadcastStream().listen(_onEvent, onError: _onError);
```

The native `onListen`/`onCancel` pair must actually start and stop the underlying resource. An `onCancel` that does not release the sensor, camera, or location updates is a battery drain no Dart-side fix can reach.

### FFI

```dart
// Bad - a long synchronous native call on the UI isolate; the app freezes, no frames
final result = _bindings.compressImage(ptr, len);

// Good - the blocking call runs off the UI isolate
final result = await Isolate.run(() => _bindings.compressImage(ptr, len));
```

FFI has no exception boundary: a native crash takes the process down, unsymbolicated. Memory is yours - pointers allocated through `package:ffi` are not garbage collected and must be freed on every path including error paths. Bundling the native library is per-platform build configuration (podspec, Gradle/CMake, a library beside the desktop executable), so it is part of the change, not an afterthought.

### Plugin packages

- The `flutter: plugin: platforms:` block in `pubspec.yaml` declares which platforms have an implementation and the native entry class for each. A platform absent there throws `MissingPluginException` at runtime, not compile time
- Federated split (app-facing package -> platform interface -> per-platform implementations) is worth it when platforms ship on separate cadences or third parties add implementations. A single-team, single-package plugin is simpler and legitimate
- Tests substitute the platform interface, not the channel. The `example/` app is the real integration harness

### Permissions

| Step | Android | iOS |
| --- | --- | --- |
| Declare | `<uses-permission>` in `AndroidManifest.xml` | usage-description key in `Info.plist` |
| Request | runtime request for dangerous permissions | prompted at first access |
| Refused for good | deep link to app settings | deep link to app settings |

Every permission path has three outcomes - granted, denied, permanently denied - and a flow that only handles the first two strands the user with no way forward. `permission_handler` unifies the request API; it does not remove the platform declaration step. Request at the point of need rather than at launch.

### Platform tiers

- **Mobile:** full support; the default assumption
- **Desktop:** channels and FFI work, but a given pub.dev package may declare no `windows`/`macos`/`linux` implementation. Check the plugin's declared platforms before designing around it; a missing one is a runtime `MissingPluginException`
- **Web:** no `MethodChannel` host and no `dart:ffi`. Web support means a Dart implementation of the platform interface written against JS interop. Guard entry points with `kIsWeb` and provide a fallback or an explicitly disabled state rather than letting the exception surface

## Output Format

When invoked from an implementation workflow:

```
Capability: {<native capability>}
Mechanism: {MethodChannel | EventChannel | Pigeon | FFI | existing plugin: <name>}
Channel Name: {<reverse-dns>/<feature> | N/A}
Direction: {Dart->native | native->Dart | bidirectional}
Payload Types: {<codec-supported types>}
Argument Validation: native {file:line | MISSING} / Dart {file:line | MISSING}
Threading: {handler offloads to background | main-thread only (justified: <why>) | FFI on <ui|helper> isolate}
Failure Handling: {PlatformException -> <mapping>; MissingPluginException -> <fallback>; timeout <duration>} | UNHANDLED: <which>
Lifecycle: {subscription cancelled at file:line | native onCancel releases <resource> | N/A}
Platforms: {android | ios | macos | windows | linux | web} -> {implemented | unsupported: <observed behavior>}
Permissions: {<permission> declared at <file>, requested at file:line | none}
```

When invoked from a review workflow or directly to diagnose a channel symptom (`MissingPluginException`, hung call, frozen UI), emit one finding block per issue:

```
### [Severity] file:line

- Rule: {mechanism-choice | argument-validation | codec-type | failure-handling | threading | subscription-lifecycle | ffi-memory | platform-coverage | permission-declaration | plugin-reinvention}
- Code: {one-line citation}
- Problem: {what breaks at runtime, on which platform}
- Recommendation: {concrete edit}
```

`Severity: {Critical | High | Medium | Low}`. Critical = an unvalidated argument reaching a native sink, a native path that can crash the process, or a leaked subscription holding the camera, microphone, or location. High = a hang or crash under realistic use (no timeout, unhandled MissingPluginException on a shipped platform, blocking main-thread work, unchecked casts). Medium = fragile but working (hand-rolled channel where Pigeon or a plugin fits, missing permission outcome handling). Low = naming, channel-string placement.

If there are no findings, emit exactly `No platform-channel findings.` so the workflow knows the check ran.

## Avoid

- Hand-rolling a channel for something a maintained plugin already does
- Trusting arguments in either direction, or validating only on the Dart side
- `as T` on a channel return value instead of a typed `invokeMethod<T>` plus a null and range check
- Calls with no timeout - a native side that never calls `result` produces a permanently pending future
- Catching only `PlatformException` and letting `MissingPluginException` crash the platform that lacks an implementation
- Passing arbitrary objects and expecting the codec to carry them
- Long-running work on the platform main thread, or a blocking FFI call on the UI isolate
- `receiveBroadcastStream().listen(...)` with no cancellation, or a native `onCancel` that does not release the resource
- Allocating native memory without freeing it on every path, error paths included
- Declaring platform support the plugin does not implement
- A manifest or `Info.plist` declaration without a runtime request, or a runtime request with no declaration
- Treating a permanently-denied permission as a dead end with no route to app settings
- Assuming channels exist on web, or that a mobile plugin has desktop implementations
