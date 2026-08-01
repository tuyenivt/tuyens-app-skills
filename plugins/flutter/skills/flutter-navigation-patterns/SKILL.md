---
name: flutter-navigation-patterns
description: "Design go_router navigation: route trees, shell routes, auth redirects, typed routes, path and query params, deep links, web URL strategy."
metadata:
  category: mobile
  tags: [flutter, dart, go-router, navigation, deep-links, routing, web]
user-invocable: false
---

# Flutter Navigation Patterns

> go_router is the default. If the project uses `auto_route`, the concepts map (annotated route classes, generated router, `AutoRouteGuard` in place of `redirect`) - keep the guidance, swap the API names. If it uses raw Navigator 2.0 (`RouterDelegate` + `RouteInformationParser`), the route-tree and guard patterns still apply but you own the plumbing; do not propose migrating routers as a side effect of an unrelated change.

## When to Use

- Declaring or restructuring an app's route tree, tabs, or nested navigation
- Adding an auth guard, onboarding gate, or any conditional redirect
- Wiring deep links / universal links, or debugging a link that opens the wrong screen
- Passing data between screens, or fixing a screen that breaks on browser refresh or process death

## Rules

- The `GoRouter` instance is created once at app scope and passed to `MaterialApp.router(routerConfig: ...)`. Rebuilding it inside a widget `build` discards navigation history
- Child `path` values are relative (no leading `/`); only top-level routes start with `/`. A leading slash on a child silently makes it a different absolute route
- `go` replaces the current location and its stack; `push` stacks a route on top and can be popped back off. Tabs and root-level moves use `go`; details and modals opened from the current screen use `push`
- One top-level `redirect` owns cross-cutting gates (auth, onboarding). Per-route `redirect` handles a single route's own condition. Returning `null` means "allow"
- `redirect` must be synchronous or fast, must be idempotent, and must never return the location it was called for - self-redirect loops trip the router's redirect limit and throw
- `redirect` reads auth state from a synchronously readable source. To re-evaluate when that state changes, hand the router a `Listenable` via `refreshListenable`; do not `await` an auth check inside `redirect`
- Path segments carry resource identity (`/orders/:id`); query parameters carry optional, omittable modifiers (filters, pagination, search). Never encode a secret or a token in either
- Every value arriving from a route is untrusted input, whether it came from an in-app tap or an external link. Parse and validate it, and authorize the resource server-side - a route guard controls what renders, not what the user may access
- `state.extra` is not part of the URL. It is `null` after a deep link, a web reload, or state restoration, so no screen may depend on it for correctness - it is an optimization over re-fetching by id
- Always supply an error route (`errorBuilder` / `errorPageBuilder`). An unmatched or malformed link must land on a real screen
- On web, configure the path URL strategy and a server rewrite to `index.html`, or every non-root URL 404s on reload

## Patterns

### Router declaration

```dart
// Bad - a fresh router per rebuild: history and current location reset
Widget build(BuildContext context) =>
    MaterialApp.router(routerConfig: GoRouter(routes: _routes));

// Good - constructed once (top-level final, or a keepAlive provider)
final appRouter = GoRouter(initialLocation: '/', routes: _routes, redirect: _guard);
```

### Route tree and relative paths

```dart
GoRoute(
  path: '/orders',
  builder: (context, state) => const OrdersScreen(),
  routes: [
    GoRoute(                              // resolves to /orders/:id
      path: ':id',                        // Bad: '/:id' would mean a top-level /:id
      builder: (context, state) => OrderScreen(id: state.pathParameters['id']!),
    ),
  ],
)
```

Nesting a route under a parent means the parent participates in the stack, so back from `/orders/42` lands on `/orders`.

### `go` vs `push`

```dart
// Bad - push from a bottom-nav tab: taps accumulate an unbounded stack
onTap: () => context.push('/settings'),

// Good - tab switch replaces, detail pushes
onTap: () => context.go('/settings'),
onOrderTap: (id) => context.push('/orders/$id'),
```

### Shell routes

`ShellRoute` wraps its children in persistent chrome (an app bar, a nav bar) while the child area swaps:

```dart
ShellRoute(
  builder: (context, state, child) => Scaffold(body: child, bottomNavigationBar: const NavBar()),
  routes: [ /* children render into `child` */ ],
)
```

`StatefulShellRoute.indexedStack` is the one to use for bottom-nav tabs that must each keep their own navigation stack and scroll position across switches - it builds from a list of `StatefulShellBranch`, and the builder receives a navigation shell object used to switch branches by index. Plain `ShellRoute` resets the child stack on every switch, which is the usual cause of "my tab forgets where I was".

### Redirect guards

```dart
String? _guard(BuildContext context, GoRouterState state) {
  final loggedIn = authNotifier.isLoggedIn;          // synchronously readable
  final atLogin = state.matchedLocation == '/login';

  if (!loggedIn && !atLogin) return '/login?from=${state.matchedLocation}';
  if (loggedIn && atLogin) return '/';
  return null;                                        // allow
}
```

The `!atLogin` term is what stops the loop: without it, `/login` redirects to `/login` forever. Pass `refreshListenable: authNotifier` so login and logout re-run the guard instead of leaving the user on a stale screen. If auth state lives in a stream, bridge it to a `Listenable` (a `ChangeNotifier` that calls `notifyListeners` on each event) - go_router's docs show this bridge as a small class you own, not a package export.

### Path vs query parameters

```dart
// Bad - identity in the query, filter in the path, and no validation
'/order?id=$id'
final id = state.pathParameters['status'];

// Good
'/orders/$id?status=open&page=2'
final id = state.pathParameters['id']!;                          // identity
final page = int.tryParse(state.uri.queryParameters['page'] ?? '') ?? 1;  // optional, validated
```

Path parameters come from `state.pathParameters`; query parameters from `state.uri.queryParameters`. Both are strings and both can be attacker-supplied.

### `extra` is not deep-linkable

```dart
// Bad - screen renders blank when reached by link, reload, or restoration
context.push('/orders/42', extra: order);
final order = state.extra! as Order;

// Good - extra is a fast path, id is the source of truth
final cached = state.extra as Order?;
return OrderScreen(id: id, preloaded: cached);   // fetches by id when preloaded == null
```

### Typed routes

`go_router_builder` generates typed navigation from route classes annotated with `@TypedGoRoute<T>`, where each class extends `GoRouteData` and implements the screen build. Navigation then goes through a constructed object (`OrderRoute(id: '42').go(context)`) instead of a hand-built string, so a renamed parameter is a compile error rather than a runtime null. The exact class boilerplate (mixin requirement, generated route-list symbol) changed across `go_router_builder` versions - copy the shape from the project's existing generated routes rather than writing it from memory.

Worth it when there are more than a handful of parameterized routes. For a small app, `name`d routes plus `context.goNamed('order', pathParameters: {'id': id})` gets most of the safety with no codegen.

### Deep links and app links

Registration is platform-side, not Dart-side: Android needs an `intent-filter` with `android:autoVerify="true"` plus a hosted `assetlinks.json`; iOS needs the Associated Domains capability plus a hosted `apple-app-site-association`. Without the hosted file the link opens a browser instead of the app. Once registered, go_router matches the incoming URI against the route tree like any other location.

Treat the arriving URI as hostile:

```dart
// Bad - a link controls what the app opens next
final next = state.uri.queryParameters['next'];
if (next != null) context.go(next);           // open redirect: any external link can drive navigation

// Good - allowlist, then navigate
const allowed = {'orders', 'profile'};
final next = state.uri.queryParameters['next'];
if (next != null && allowed.contains(next)) context.go('/$next');
```

The same applies to ids in the path: a link can name any resource, so the guard is server-side authorization on fetch, not the route match.

### Web URL strategy

```dart
import 'package:flutter_web_plugins/url_strategy.dart';

void main() {
  usePathUrlStrategy();   // /orders/42 instead of /#/orders/42
  runApp(const App());
}
```

This requires the host to rewrite unknown paths to `index.html`; without that rewrite, a direct load of `/orders/42` returns 404 while in-app navigation works, which is why it usually surfaces only after deploy.

## Output Format

When invoked from an implementation workflow, emit the route table:

```
| Route | Path | Params | Guard | Deep link | Notes |
|-------|------|--------|-------|-----------|-------|
| Orders | /orders | - | auth | yes | inside StatefulShellBranch |
| Order detail | /orders/:id | id (path) | auth | yes | validate id, fetch by id |
| Search | /search | q, page (query) | - | yes | page defaults to 1 |
| Login | /login | from (query) | none | no | redirect target |
```

Followed by: `Shell: {none | ShellRoute | StatefulShellRoute}`, `Guard source: {provider | listenable | none}`, `Error route: {declared | missing}`, `Web URL strategy: {path | hash | n/a}`. `Guard source` names what the guard reads: `provider` = DI/provider state (bridged to `refreshListenable`), `listenable` = a raw `Listenable`/`ChangeNotifier`, `none` = no guard.

When invoked from a review workflow, emit one finding block per issue:

```
### [Severity] file:line

- Rule: {router-lifetime | relative-path | go-vs-push | redirect-loop | guard-source | param-placement | untrusted-param | extra-dependency | error-route | web-strategy}
- Code: {one-line citation}
- Problem: {what breaks at runtime, not what looks unusual}
- Recommendation: {concrete edit}
```

`Severity: {Critical | High | Medium | Low}`. Critical = open redirect or unvalidated deep-link parameter, a guard that can be bypassed, or a redirect loop. High = navigation that breaks under realistic use (router rebuilt in `build`, `extra`-dependent screens, missing error route, guard source that never notifies). Medium = works but fragile or misplaced (go/push misuse, param placement, missing web strategy on a web target). Low = naming or route-ordering style.

For each rule in the enum with zero findings, emit exactly `No <rule> findings.` so the workflow knows the check ran; omit that line for rules with findings.

When invoked directly to diagnose a navigation symptom, emit the same finding block for the diagnosed cause, citing the pattern section that explains it.

## Avoid

- Constructing `GoRouter` inside a widget `build`
- A leading `/` on a child route `path`
- `push` for tab switches, `go` for a detail that should be poppable
- `redirect` that awaits a network or storage call, or that can return its own input location
- Auth guards reading a value that never notifies the router - pair the source with `refreshListenable`
- Reading route parameters without parsing or validating them
- Screens whose correctness depends on `state.extra`
- Passing tokens, secrets, or full objects through the URL
- Navigating to a location taken verbatim from a link parameter - allowlist it
- Shipping without an error route, so a bad link produces a blank or crashed screen
- Treating a route guard as authorization - it hides UI, it does not protect data
- Mixing `Navigator.push` with go_router for routes that should be linkable; the raw Navigator route has no URL and no deep-link entry point
