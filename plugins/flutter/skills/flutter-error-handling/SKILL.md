---
name: flutter-error-handling
description: "Model typed Dart failures: sealed hierarchies, exhaustive switch, Result vs throw, error mapping to UI states, Flutter error boundaries."
metadata:
  category: mobile
  tags: [flutter, dart, error-handling, sealed-classes, result, freezed, riverpod]
user-invocable: false
---

# Flutter Error Handling

> This skill owns failure typing and propagation. Transport specifics live in `flutter-networking`.

## When to Use

- Designing the failure type for a new feature, repository, or data source
- Reviewing code that catches, converts, or renders errors
- Deciding `Result` vs `throw` at a layer boundary
- Wiring app-level error boundaries and the uncaught-error handler

## Rules

- A caught error is handled, converted, or rethrown. `catch (_) {}` and `catch (e) { return null; }` are defects, not style.
- The domain layer owns a `sealed` failure type. Foreign exceptions (`DioException`, `SocketException`, `FormatException`, storage errors) are converted at the repository boundary and never escape above it.
- `switch` over failures is exhaustive with no `default` - adding a variant must break the build at every decision site.
- Catch by type with `on`. Bare `catch (e, st)` only at a boundary that must not crash, and it logs the stack trace.
- Preserve stack traces: `rethrow`, or `Error.throwWithStackTrace(failure, st)` when rethrowing a converted error. Never `throw e` inside a `catch`.
- Exceptions model expected failures; `Error` subclasses model programmer bugs. Never catch an `Error` for control flow.
- Inside a Riverpod async provider, throw the typed failure - `AsyncValue` is already the result type. Wrapping in `Result` as well is duplication.
- Reach for `Result` where nothing else forces the caller to consider failure: Dart has no checked exceptions, so a thrown error is invisible in the signature.
- Every failure carries a stable variant, whether retry is meaningful, and the original cause plus stack for reporting.
- User-facing text is derived from the failure variant in the presentation layer and localized. `e.toString()` is never shown to a user.
- Cancellation is not a failure. Map it to a dedicated `Cancelled` variant at the repository boundary; every presentation-layer switch renders nothing for it (no banner, no retry) and it is never reported as an error.

## Patterns

### Sealed failure hierarchy

```dart
sealed class AppFailure {
  const AppFailure({this.cause, this.stackTrace});
  final Object? cause;
  final StackTrace? stackTrace;

  bool get isRetryable =>
      switch (this) { NetworkFailure() || TimeoutFailure() => true, _ => false };
}

final class NetworkFailure extends AppFailure { const NetworkFailure({super.cause, super.stackTrace}); }
final class TimeoutFailure extends AppFailure { const TimeoutFailure({super.cause, super.stackTrace}); }
final class UnauthorizedFailure extends AppFailure { const UnauthorizedFailure({super.cause, super.stackTrace}); }
final class ParseFailure extends AppFailure { const ParseFailure({super.cause, super.stackTrace}); }
final class ServerFailure extends AppFailure {
  const ServerFailure(this.status, {super.cause, super.stackTrace});
  final int status;
}
```

`sealed` restricts subtyping to this library, which is what makes downstream `switch` exhaustive. `final` on the variants stops feature code from adding a subclass the mapper has never seen.

freezed unions generate the same shape plus `copyWith` and value equality. Prefer them when failures carry several fields. The class-modifier syntax freezed expects differs across its major versions - follow the version pinned in `pubspec.yaml` rather than a remembered snippet.

One shared failure type for cross-cutting variants every feature meets (network, timeout, auth, parse); a feature declares its own sealed type only when its variants are meaningless outside it (validation rules, domain states), and may embed the shared type as a variant's field rather than duplicating transport variants.

### Exhaustive switch, no `default`

```dart
// Bad - default absorbs every future variant silently
String messageFor(AppFailure f) {
  switch (f) {
    case NetworkFailure(): return 'You are offline';
    default: return 'Something went wrong';
  }
}

// Good - no default; adding RateLimitFailure fails compilation here
String messageFor(AppFailure f, AppLocalizations l10n) => switch (f) {
      NetworkFailure() => l10n.offline,
      TimeoutFailure() => l10n.slowConnection,
      UnauthorizedFailure() => l10n.sessionExpired,
      ParseFailure() => l10n.somethingWentWrong,
      ServerFailure(:final status) => l10n.serverError(status),
    };
```

The compile error is the whole point: a new failure variant should force a decision at every place a user sees an error, not silently inherit the generic message.

### Mapping data-layer errors into domain failures

```dart
// Bad - DioException escapes; now the widget layer imports Dio to read statusCode
Future<User> fetch(String id) async {
  final res = await _dio.get<Map<String, dynamic>>('/users/$id');
  return User.fromJson(res.data!);
}

// Good - one conversion point per data source
Future<User> fetch(String id) async {
  try {
    final res = await _dio.get<Map<String, dynamic>>('/users/$id');
    return User.fromJson(res.data!);
  } on DioException catch (e, st) {
    Error.throwWithStackTrace(_toFailure(e, st), st);
  } on FormatException catch (e, st) {
    Error.throwWithStackTrace(ParseFailure(cause: e, stackTrace: st), st);
  } on TypeError catch (e, st) {
    // generated fromJson surfaces a bad payload as a cast error, not FormatException
    Error.throwWithStackTrace(ParseFailure(cause: e, stackTrace: st), st);
  }
}

AppFailure _toFailure(DioException e, StackTrace st) => switch (e.type) {
      DioExceptionType.connectionTimeout ||
      DioExceptionType.sendTimeout ||
      DioExceptionType.receiveTimeout =>
        TimeoutFailure(cause: e, stackTrace: st),
      DioExceptionType.connectionError => NetworkFailure(cause: e, stackTrace: st),
      DioExceptionType.badResponse when e.response?.statusCode == 401 =>
        UnauthorizedFailure(cause: e, stackTrace: st),
      DioExceptionType.badResponse =>
        ServerFailure(e.response?.statusCode ?? 0, cause: e, stackTrace: st),
      _ => NetworkFailure(cause: e, stackTrace: st),
    };
```

A parse boundary that catches only `FormatException` still crashes on a malformed payload, because `json_serializable`'s generated casts throw `TypeError`. This is the one place catching an `Error` subtype is correct: untrusted server data is input, not a local bug.

`DioExceptionType` member names are Dio 5's; Dio 4 exposes `DioError`/`DioErrorType` with a different member set.

### `Result` vs `throw`

```dart
sealed class Result<T> {
  const Result();
}
final class Ok<T> extends Result<T> { const Ok(this.value); final T value; }
final class Err<T> extends Result<T> { const Err(this.failure); final AppFailure failure; }
```

| Boundary | Choice | Why |
|----------|--------|-----|
| Repository consumed by a Riverpod `AsyncNotifier` / `FutureProvider` | `throw` | `AsyncValue` already captures the error; `Result` would nest two error models |
| Synchronous validation and parsing | `Result` | failure is an ordinary outcome, and the signature forces the caller to handle it |
| A loop that must finish despite per-item failures | `Result` | `try`/`catch` per iteration reads worse and loses which items failed |

Pick one per layer and stay consistent. A codebase where half the repositories throw and half return `Result` forces every call site to check which convention applies.

### Failure to UI state

```dart
final userProvider = FutureProvider.family<User, String>(
  (ref, id) => ref.watch(userRepoProvider).fetch(id),
);

// Widget - the only place a failure becomes words
ref.watch(userProvider(id)).when(
      loading: () => const LoadingView(),
      data: (user) => UserView(user),
      error: (e, st) => switch (e) {
        final AppFailure f => ErrorView(
            message: messageFor(f, l10n),
            onRetry: f.isRetryable ? () => ref.invalidate(userProvider(id)) : null,
          ),
        _ => ErrorView(message: l10n.somethingWentWrong),
      },
    );
```

The `_` arm is required because `AsyncValue`'s error is typed `Object` - anything can be thrown. Treat a hit on that arm as a mapping bug worth reporting, not a normal path.

### App-level error boundaries

```dart
void main() {
  WidgetsFlutterBinding.ensureInitialized();

  FlutterError.onError = (details) {
    Report.error(details.exception, details.stack); // framework: build, layout, paint
  };
  PlatformDispatcher.instance.onError = (error, stack) {
    Report.error(error, stack); // async errors outside the framework
    return true; // handled; returning false lets it reach the platform
  };
  ErrorWidget.builder = (details) => const CrashPlaceholder();

  runApp(const ProviderScope(child: App()));
}
```

`PlatformDispatcher.instance.onError` covers what `runZonedGuarded` used to be needed for. Use one or the other, not both, or every uncaught error is reported twice.

`ErrorWidget.builder` replaces the red error screen for a failed subtree build. It must never itself throw, and it is a last resort - it means a widget escaped local handling.

### A caught error is handled or rethrown

```dart
// Bad - failure vanishes; the UI shows an empty list forever with no way to retry
Future<List<Order>> load() async {
  try {
    return await repo.fetchOrders();
  } catch (_) {
    return [];
  }
}

// Good - a deliberate fallback is stated, and the failure is still reported
Future<List<Order>> load() async {
  try {
    return await repo.fetchOrders();
  } on NetworkFailure catch (e, st) {
    Report.warn(e, st);
    return _cache.orders(); // documented degraded path, not a silent empty state
  }
}
```

Empty-on-error is the most common Flutter error bug: it renders identically to "no data", so the user sees a legitimate-looking empty screen and support cannot tell the two apart.

## Output Format

When invoked from an implementation workflow, emit one row per boundary:

```
| Boundary | Failure Type | Propagation | Rationale |
|----------|--------------|-------------|-----------|
| UserRemoteDataSource | AppFailure (sealed) | Throw | consumed by FutureProvider; AsyncValue carries it |
| CheckoutFormValidator | ValidationFailure | Result | sync, caller must branch |
| ImportBatchService | AppFailure per item | Result | partial success must survive one bad row |
```

`Propagation: {Throw | Result | Degrade}` - `Degrade` means a documented fallback value plus a report call.

When invoked from a review workflow, emit one finding block per defect:

```
### [Blocker | High | Medium | Low] lib/path/file.dart:LINE

- Code: {one-line citation}
- Defect: {Swallowed | Untyped | Unmapped | Leaked | Stack-Lost | Raw-Message}
- Impact: {what the user sees, or what is lost from reporting}
- Fix: {concrete edit}
```

| Defect | Meaning |
|--------|---------|
| `Swallowed` | caught and discarded, or replaced by an empty/null value with no report |
| `Untyped` | failures modelled as `Exception`/`String` instead of a sealed variant |
| `Unmapped` | `switch` over failures uses `default`, so new variants fall through |
| `Leaked` | a data-layer exception type is visible above the repository boundary |
| `Stack-Lost` | `throw e` inside `catch`, or a conversion that drops the original stack |
| `Raw-Message` | `e.toString()` or an untranslated string rendered to the user |
| `Boundary-Config` | app-level handlers misconfigured: double reporting (`runZonedGuarded` + `onError`), missing `FlutterError.onError`/`PlatformDispatcher.onError`, an `ErrorWidget.builder` that can throw |

`Blocker` = failure invisible or app unable to report (Swallowed, Boundary-Config with no reporting path). `High` = wrong handling on a user-facing path (Leaked, Raw-Message, Unmapped). `Medium` = degraded diagnostics or fragile typing (Stack-Lost, Untyped). `Low` = style. If there are no findings, emit exactly `No error-handling findings.` so the workflow knows the check ran.

## Avoid

- `catch (_) {}`, or returning `null`/`[]`/`false` from a `catch` without a report
- `default` or `_` in a `switch` over the sealed failure type outside the presentation `Object` guard
- `throw e` inside a `catch` block - use `rethrow` or `Error.throwWithStackTrace`
- `DioException`, `SocketException`, `SqliteException` or `PlatformException` in a widget, notifier, or use case
- `Result<T>` returned from a repository that a Riverpod async provider consumes - `AsyncValue` is the result
- Catching `Error` subtypes for control flow, except deserialization of untrusted payloads
- A single `AppException` with a `String code` field - that is a stringly-typed enum with no exhaustiveness
- Showing `e.toString()`, a status code, or a stack trace to the user
- Surfacing `CancelToken` cancellation or post-dispose errors as user-visible failures
- `runZonedGuarded` alongside `PlatformDispatcher.instance.onError` - double reporting
- Error-mapping logic duplicated across data sources instead of one mapper per source
