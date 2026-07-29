---
name: flutter
description: Flutter/Dart-specific expertise — `build()` as a pure, cheap, frequently-called function (no side effects, no Future creation, no controller construction), `const` constructors and narrow rebuild scope for performance, `StatefulWidget` lifecycle and disposing controllers/streams/subscriptions, the `mounted` check after every `await`, matching the project's existing state management instead of introducing a second one, the constraints-down/sizes-up layout model and the unbounded-constraint errors it produces, `ListView.builder` for long or infinite lists, async UI (`FutureBuilder`/`StreamBuilder` done correctly), navigation/context pitfalls, and platform channels, assets, and permissions declared per platform. Use when `pubspec.yaml` exists, when the work is in `lib/**.dart` or touches `MaterialApp`/widgets, or when asked directly (Thai or English) to สร้างหรือแก้ไขแอป Flutter / เขียนโค้ด Dart. Not a replacement for frontend-dev's web-framework UI work — Flutter is a separate rendering model, so react/html-css/tailwind rules do not carry over; ui-ux-researcher's interaction/layout recommendations still do.
---

# Flutter

The mobile/desktop UI lane. Flutter does not use the DOM or CSS, so
`skills/html-css/SKILL.md`, `skills/react/SKILL.md`, and
`skills/tailwind/SKILL.md` do **not** apply — none of their mechanisms
exist here. What does carry over is `ui-ux-researcher`'s interaction and
information-architecture work, the API contracts `backend-dev` states, and
the verification discipline in `skills/dev-testing/SKILL.md` (a widget that
renders while throwing in the console is still a failure).

## Step 1 — Confirm Flutter and read the project's conventions

Check for `pubspec.yaml`, `lib/main.dart`, `android/`+`ios/` folders, or
`.dart` sources. Then read, before writing anything: the Dart SDK and
Flutter constraints in `pubspec.yaml`, whether null safety is on (it is, on
any modern SDK), the **state management already in use**
(`setState` only, Provider, Riverpod, Bloc/Cubit, GetX), the navigation
approach (`Navigator` 1.0, `go_router`), and the lint set in
`analysis_options.yaml`. Introducing a second state-management or routing
library into a project that already has one is a structural change, not an
implementation detail — say so instead of doing it silently.

## Step 2 — `build()` is pure, cheap, and called often

- `build()` can run on every frame. It may only read state and return
  widgets: no network calls, no `setState`, no file I/O, no analytics
  events, no `SharedPreferences` writes.
- Never create a `Future` inside `build()` and hand it to a `FutureBuilder`
  — a rebuild creates a new Future and refetches, which is the classic
  infinite-loading/flicker bug. Create it in `initState` (or in the state
  layer) and store it.
- Never construct controllers (`TextEditingController`, `AnimationController`,
  `ScrollController`) in `build()` — they'd be recreated and leaked. They
  belong in `initState`, disposed in `dispose()`.
- Widgets are immutable descriptions, not retained objects; treat
  constructing them as free and the *work inside* them as the cost.

## Step 3 — Widget structure and rebuild scope

- Use `const` constructors everywhere they're possible — a `const` widget
  subtree is skipped entirely on rebuild. The linter flags most of these;
  don't suppress it.
- Prefer composition into small widgets over one 400-line `build()` with
  helper methods returning widgets: a separate widget class can be `const`
  and rebuilds independently, while `_buildHeader()` always rebuilds with
  its parent.
- `setState` rebuilds the whole `State` subtree — push it down to the
  smallest widget that owns the changing value rather than calling it at the
  top of a screen.
- Give stateful list items stable `Key`s (a `ValueKey` from the model's id)
  when the list can reorder, filter, or splice — otherwise element state
  attaches to the wrong row, the same failure mode as index keys in React.
- `StatelessWidget` unless there is genuinely local mutable state; local UI
  state (a toggle, a text field) belongs in the widget, shared/business
  state belongs in the project's state layer.

## Step 4 — Lifecycle, disposal, and async safety

- Everything created in `initState` is released in `dispose()`: controllers,
  `StreamSubscription`s, timers, focus nodes, platform listeners. An
  undisposed subscription keeps firing after the screen is gone and throws
  on `setState` after unmount.
- After **every** `await` in a `State` method, check `if (!mounted) return;`
  before calling `setState` or using `context`. The user can navigate away
  mid-request, and "setState called after dispose" is exactly this.
- Don't hold a `BuildContext` across an async gap; capture what you need
  (a `ScaffoldMessenger`, a `Navigator`) before the `await` instead — the
  `use_build_context_synchronously` lint exists for this.
- Handle errors on every `Future` shown to a user: an unhandled exception
  becomes a red error widget in debug and a silent broken screen in release.
  Set `FlutterError.onError`/`PlatformDispatcher.instance.onError` for
  reporting.
- `didChangeDependencies` (not `initState`) for work depending on inherited
  widgets; `didUpdateWidget` when a parent passes new configuration.

## Step 5 — Layout: constraints go down, sizes go up

- A parent passes constraints down; a child picks its size within them; the
  parent positions it. Most layout errors are one of:
  - **"Unbounded height/width"** — a `Column` inside a `SingleChildScroll
    View`/another `Column` giving a `ListView` infinite height. Fix with
    `Expanded`/`Flexible`, `shrinkWrap: true` (only for small lists), or a
    `SizedBox` with a real height.
  - **"RenderFlex overflowed by N pixels"** — content wider/taller than the
    constraint; fix with `Expanded`, `Flexible`, `Wrap`, or scrolling, not
    by hardcoding a smaller size.
- `Expanded` vs `Flexible`: `Expanded` forces the child to fill the free
  space; `Flexible` lets it be smaller.
- Use `MediaQuery`/`LayoutBuilder` for responsive breakpoints and
  `SafeArea` for notches and system bars; don't hardcode device pixel
  values.
- Long or unbounded lists use `ListView.builder`/`GridView.builder`/
  `CustomScrollView` with slivers — a `ListView(children: [...])` builds
  every child up front.
- Respect the platform's text scaling and don't fix text container heights
  in pixels; an accessibility font size setting will overflow them.

## Step 6 — State management and data

- Match the existing solution. If there is none and the app is beyond
  trivial, recommend one and say why, rather than mixing `setState`,
  `InheritedWidget`, and a package across screens.
- Keep business logic and API access out of widgets — a repository/service
  layer keeps `build()` free of network code and makes the logic testable
  without a widget tree.
- Model async UI states explicitly (loading / data / empty / error) as a
  sealed class or the state library's equivalent; a nullable field plus a
  bool leaves impossible combinations representable — same reasoning as
  `skills/typescript/SKILL.md`'s discriminated unions.
- Dispose of/close whatever the state layer opens (Blocs, Riverpod
  providers with side effects, streams).
- Local persistence: `shared_preferences` for small key-values, a real
  database (Drift/sqflite → `skills/sqlite/SKILL.md`) for structured data.
  Never store tokens or secrets in `shared_preferences` — use
  `flutter_secure_storage`, and keep API secrets on a server; a shipped APK
  or IPA is unpackable.

## Step 7 — Platform, assets, and release

- Every asset must be declared in `pubspec.yaml`; a missing declaration
  fails at runtime, not at build.
- Permissions are declared per platform (`AndroidManifest.xml`,
  `Info.plist`) *and* requested at runtime — adding a plugin is not enough,
  and a missing iOS usage-description string is an App Store rejection.
- Platform channels (`MethodChannel`) need an implementation on each
  targeted platform, plus a fallback when a platform isn't supported. Check
  `Platform.isX`/`kIsWeb` before calling platform-only APIs.
- `flutter analyze` clean and `dart format` applied before done; fix lints
  rather than adding ignores.
- Test on a real profile/release build for performance claims — debug builds
  are slow by construction, so "it janks in debug" is not a measurement. Use
  DevTools' timeline and the repaint rainbow to find real jank; check for
  expensive work in `build`, unnecessary `Opacity`/`ClipRRect`/shadow
  layers, and full-size images decoded at full resolution
  (`cacheWidth`/`cacheHeight`, `ListView` item images).
- `flutter run` on both a phone-sized and a tablet/desktop-sized target if
  the app claims to support both.

## Step 8 — Self-check before reporting done

Confirm no `build()` creates a Future, a controller, or a side effect, and
that `const` is used wherever the analyzer allows. Confirm every controller,
subscription, and timer created in `initState` is disposed, and every
`await` in a `State` is followed by a `mounted` check before touching
`setState` or `context`. Confirm `setState` is scoped to the smallest widget
that owns the change, list items have stable keys, and long lists use a
builder. Confirm layout has no overflow or unbounded-constraint error in the
console at any supported size, assets and permissions are declared per
platform, and `flutter analyze` is clean. Then run the app and drive one
real user task per `skills/dev-testing/SKILL.md` — treat any console
exception or red error widget as a failure even if the screen looks right.

## What not to do

- Don't do side effects, network calls, or `Future` creation inside
  `build()`, and don't build controllers there.
- Don't skip `const`, or replace small widget classes with
  `_buildSomething()` helpers that always rebuild.
- Don't call `setState` at the top of a screen for a change that affects one
  small widget.
- Don't leave a controller, stream subscription, or timer undisposed.
- Don't use `context` or call `setState` after an `await` without checking
  `mounted`.
- Don't fix an overflow by hardcoding sizes, or a `ListView` in a `Column`
  by reaching for `shrinkWrap` on a long list.
- Don't build a long list with `ListView(children: [...])`.
- Don't introduce a second state-management or routing library alongside the
  project's existing one without saying so.
- Don't store secrets or tokens in `shared_preferences`, and don't embed API
  secrets in the app at all.
- Don't judge performance from a debug build.
- Don't carry web assumptions over — there is no DOM, no CSS, and
  `skills/react/SKILL.md`/`skills/html-css/SKILL.md`/
  `skills/tailwind/SKILL.md` don't apply here.
