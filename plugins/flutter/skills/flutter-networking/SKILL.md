---
name: flutter-networking
description: "Call HTTP APIs from Flutter with Dio: timeouts, CancelToken, interceptors, typed failure mapping, token refresh, backoff retry, caching."
metadata:
  category: mobile
  tags: [flutter, dart, dio, http, timeout, cancellation, retry, auth-token]
user-invocable: false
---


Single owner for outbound HTTP from the app. The device is a **client**: it consumes a contract it does not own, over a link that is slow, metered, and frequently absent. Typed failure shapes live in `flutter-error-handling`; cert pinning and secret storage in `flutter-security-patterns`. This skill owns client construction, timeouts, cancellation, interceptors, error mapping, and retry.

## When to Use

- Adding or reviewing any call to a remote API from a repository or data source
- Wiring auth headers, token refresh, logging, retry, or caching into the HTTP layer
- Deciding where a network error becomes a UI-visible failure
- Reviewing whether in-flight requests are cancelled when the screen that started them is gone

## Rules

- One `Dio` instance per API host, built once and injected. Never construct a client inside `build()`, a widget method, or per repository call
- Every request carries `connectTimeout` and `receiveTimeout`; requests with a body also carry `sendTimeout`. Dio applies no timeouts by default, and a stalled mobile radio hangs indefinitely without them
- Cross-cutting concerns (auth header, retry, logging, caching) live in interceptors, not at call sites
- Every cancellable request carries a `CancelToken` whose `cancel()` is wired to the disposal of whatever owns the request (`ref.onDispose`, `State.dispose`)
- No `DioException` escapes the data layer. Map it to a typed failure at the repository boundary (see `flutter-error-handling`)
- Token refresh is single-flight and serialized: N concurrent 401s trigger one refresh, not N. The refresh request itself must bypass the auth interceptor
- Retry only idempotent requests (`GET`, `HEAD`, `PUT`, `DELETE`), bounded, with jitter, honouring `Retry-After`. A user-blocking call gets a short budget; longer waits belong to a background retry, not a spinner
- Tokens are read from secure storage, never `shared_preferences` (see `flutter-data-persistence`), and never written to logs or crash reports
- Base URLs and keys come from `--dart-define` / flavor config, not string literals in Dart

## Patterns

### One configured client, timeouts on every call

```dart
// Bad - fresh client per call, no timeouts: a dead radio hangs the screen forever
Future<Order> fetch(String id) => Dio().get('$base/orders/$id').then(...);

// Good - one instance, timeouts in BaseOptions, injected everywhere
final dio = Dio(BaseOptions(
  baseUrl: const String.fromEnvironment('API_BASE_URL'),
  connectTimeout: const Duration(seconds: 10),
  receiveTimeout: const Duration(seconds: 15),
  sendTimeout: const Duration(seconds: 15),
));
```

`connectTimeout` covers establishing the connection, `receiveTimeout` the gap while waiting on response data, `sendTimeout` the gap while uploading a body. A GET with no body is unaffected by `sendTimeout`. Override per call via `Options(receiveTimeout: ...)` for known-slow endpoints (uploads, reports) rather than raising the global ceiling.

### Cancellation tied to disposal

```dart
// Bad - user leaves the screen, response still arrives and writes to dead state
final res = await dio.get('/orders');

// Good - token cancelled when the provider/State goes away
final token = CancelToken();
ref.onDispose(() => token.cancel('disposed'));           // Riverpod; State.dispose() otherwise
final res = await dio.get('/orders', cancelToken: token);
```

Screens on mobile are disposed constantly (back navigation, tab switch, low memory). An uncancelled request keeps the socket, the response bytes, and the closed-over state alive. Cancellation surfaces as `DioExceptionType.cancel`, which must map to a "cancelled" failure the UI ignores rather than an error banner.

### Transport errors to typed domain failures

```dart
// Bad - DioException reaches the widget; the UI switches on strings
} catch (e) { state = AsyncError(e, StackTrace.current); }

// Good - one translation point per repository
Failure mapDio(DioException e) => switch (e.type) {
  DioExceptionType.connectionTimeout ||
  DioExceptionType.sendTimeout ||
  DioExceptionType.receiveTimeout  => const Failure.timeout(),
  DioExceptionType.connectionError => const Failure.offline(),
  DioExceptionType.cancel          => const Failure.cancelled(),
  DioExceptionType.badCertificate  => const Failure.insecure(),
  DioExceptionType.badResponse     => Failure.server(e.response?.statusCode),
  DioExceptionType.unknown         => const Failure.unknown(),
};
```

The distinction the UI needs is retryable-vs-not and offline-vs-server-broken. `connectionError` means "no route to the server" and deserves an offline affordance; `badResponse` with 422 is a form error and must not be retried.

### Auth attach plus single-flight refresh

```dart
Future<String>? _inFlight;

Future<String> _refresh() =>                       // one refresh, shared by all waiters
    _inFlight ??= _doRefresh().whenComplete(() => _inFlight = null);

dio.interceptors.add(QueuedInterceptorsWrapper(     // queued: 401s handled one at a time
  onRequest: (options, handler) async {
    final t = await secureStore.accessToken();
    if (t != null) options.headers['Authorization'] = 'Bearer $t';
    handler.next(options);
  },
  onError: (e, handler) async {
    if (e.response?.statusCode != 401) return handler.next(e);
    try {
      final fresh = await _refresh();
      handler.resolve(await dio.fetch(
        e.requestOptions..headers['Authorization'] = 'Bearer $fresh',
      ));                                           // replay the original request
    } catch (_) {
      handler.next(e);                              // refresh dead -> caller forces logout
    }
  },
));
```

The stampede: an app resuming from background fires six parallel requests, all 401, all refresh. Without gating you burn six refresh tokens, and a server that rotates refresh tokens invalidates the session. Two guards, both required: a `QueuedInterceptor` so error handling is serialized, and a shared in-flight `Future` so the second caller awaits the first refresh instead of starting its own. Issue the refresh call on a **separate `Dio` instance** (or an explicitly skipped path) so it cannot recurse into this interceptor.

### Bounded retry honouring `Retry-After`

```dart
const retryable = {408, 429, 500, 502, 503, 504};
final wait = _retryAfter(e.response) ??            // server's number wins
    Duration(milliseconds: (200 * (1 << attempt) * (0.5 + rnd.nextDouble())).round());
```

Only retry when the request is idempotent and the status is in the set above; a retried non-idempotent POST double-charges. `Retry-After` is **seconds or an HTTP-date**, not milliseconds; parse both forms (`HttpDate.parse` handles the date form but comes from `dart:io`, so guard it on web). Jitter your own backoff, never the server's value, and never wait less than it asks. If the wait exceeds the budget for a call a user is staring at, fail fast and let the retry happen on next screen entry. `dio_smart_retry` implements this shape if you would rather not hand-roll it; do not stack it on top of a second retry layer.

### Caching

Prefer HTTP validators the server already sends: send back `If-None-Match` / `If-Modified-Since`, treat 304 as "reuse what you have". `dio_cache_interceptor` wires this into Dio with a pluggable store. Distinguish two caches and do not conflate them: an **HTTP cache** exists to save bytes and is disposable, while **offline data the user expects to see** is a local database concern and belongs in `flutter-data-persistence`. Never make correctness depend on a cache that a low-storage device can evict.

### Alternatives to Dio

`package:http` is the minimal choice: no interceptors, no `CancelToken`, timeouts only via `.timeout()` on the future, so auth, retry, and cancellation become hand-rolled per call. Fine for an app with a handful of endpoints. `chopper` generates a typed client over `http` and has its own interceptor model. `retrofit` generates a typed client over Dio and keeps everything in this skill applicable. Pick one and stay on it: two HTTP stacks means two timeout policies and two places auth can be wrong.

### Platform tiers

Mobile is the default assumption. On **web**, `dart:io` is unavailable (no `HttpDate`, no IO adapter, no cert pinning), CORS governs every request, and cookies are managed by the browser. On **desktop**, behaviour matches mobile, but there is no OS-level "app is backgrounded" signal to hang a refresh trigger on.

## Output Format

When invoked from an implementation workflow, emit the client contract:

```
Client: {Dio | http | chopper | retrofit+Dio | mixed - needs consolidation}
Instance: {file:line of the shared instance | planned: <module/provider> | constructed per call (defect)}
Timeouts: {connect=<d> receive=<d> send=<d> | PARTIAL: <missing> | MISSING}
Cancellation: {CancelToken bound to <dispose site> | CancelToken unbound | NONE}
Auth: {interceptor at file:line | per-call header | none}
Refresh: {single-flight + queued | unguarded (stampede) | none | N/A}
Retry: {none | <n> attempts, expo+jitter, Retry-After honoured | <n> attempts, Retry-After ignored}
Error Mapping: {DioExceptionType -> <failure type> at file:line | raw DioException reaches UI}
Caching: {none | HTTP validators (ETag/Last-Modified) | local store}
```

When invoked from a review workflow, emit one block per finding:

```
### [Blocker | Major | Minor] file:line

- Call: {endpoint and one-line citation}
- Risk: {hang | leak | duplicate write | token invalidation | secret exposure | error leak to UI}
- Recommendation: {concrete edit}
```

`Blocker` = the Risk fires under normal use (no timeouts, ungated refresh, retried POST, logged secrets). `Major` = fires under realistic failure or scale (unbound CancelToken, Retry-After ignored, raw errors to UI). `Minor` = cost or hygiene (missing cache validators, two stacks, hardcoded URL in a dev-only path). If there are no findings, emit exactly `No networking findings.` so the workflow knows the check ran.

## Avoid

- `Dio()` inside `build()`, a widget method, or a repository method body
- Any request without `connectTimeout` and `receiveTimeout` - Dio has no defaults
- Awaiting a response after disposal with no `CancelToken`, then writing to disposed state
- Treating `DioExceptionType.cancel` as an error worth showing the user
- Refresh-on-401 without single-flight gating, or a refresh call that routes through its own auth interceptor
- Retrying POST, or retrying 4xx other than 408 / 429
- Reading `Retry-After` as milliseconds, or jittering the server's value downward
- Stacking a retry interceptor on top of a library's built-in retry
- `DioException`, status codes, or response maps reaching widgets
- Tokens in `shared_preferences`, or logging interceptors that print `Authorization` headers and PII bodies
- Hardcoded base URLs or API keys in Dart source
- Two HTTP stacks in one app
