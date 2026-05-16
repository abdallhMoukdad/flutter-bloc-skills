---
name: flutter-bloc-setup
description: Configures a Flutter project for Bloc state management with `flutter_bloc`, `freezed`, `bloc_concurrency`, and `hydrated_bloc`. Use when initializing Bloc in a new project, migrating away from `setState`/`Provider`/`ChangeNotifier`, or before authoring any feature with `flutter-bloc-feature-pattern` or `flutter-bloc-async-api`.
metadata:
  model: claude-opus-4-7
  last_modified: Thu, 14 May 2026 00:00:00 GMT
---

# Bootstrapping Bloc in a Flutter Project

Installs and wires the canonical Bloc stack so feature-level skills (`flutter-bloc-feature-pattern`, `flutter-bloc-async-api`, `flutter-bloc-stream-tracking`, `flutter-bloc-testing`, `flutter-bloc-forms`) plug into a project with no further setup. Mobile only — Android and iOS targets. Web and desktop are out of scope.

## Contents
- [Dependencies and generators](#dependencies-and-generators)
- [App-level providers](#app-level-providers)
- [Project structure](#project-structure)
- [Workflow: Bootstrap a Bloc-ready project](#workflow-bootstrap-a-bloc-ready-project)
- [Applied to Talabat-clone](#applied-to-talabat-clone)
- [Examples](#examples)

## Dependencies and generators

Add the runtime dependencies. Pin major versions explicitly so `dart run build_runner` does not break across upgrades.

```bash
flutter pub add flutter_bloc freezed_annotation json_annotation bloc_concurrency hydrated_bloc path_provider
```

Add the generators as `dev_dependencies`. `build_runner` drives both `freezed` and `json_serializable`.

```bash
flutter pub add --dev build_runner freezed json_serializable
```

Run the generator continuously while editing `freezed`-annotated files in a side terminal. This avoids stale `*.freezed.dart` and `*.g.dart` artifacts during development.

```bash
dart run build_runner watch -d
```

This skill does **not** duplicate HTTP setup. Android `<uses-permission android:name="android.permission.INTERNET" />` and the iOS `com.apple.security.network.client` entitlement live in `flutter-use-http-package`. Run that skill once for the project before adding any feature that calls a backend.

## App-level providers

Two app-wide objects belong at the root of the widget tree.

1. **`MultiRepositoryProvider`** — exposes data-layer repositories to every feature below. Start with `const <RepositoryProvider>[]` (typed, not bare `const []`, which trips `strict-inference` lints); `flutter-bloc-async-api` adds repositories to this list as features land. Each repository entry looks like:

   ```dart
   RepositoryProvider<AuthRepository>(
     create: (_) => AuthRepository(api: AuthApiClient(...)),
     lazy: true, // construct only on first read
   ),
   ```

2. **`AppBlocObserver`** — a global `BlocObserver` that logs every Bloc transition and error to the console in debug mode. It dramatically shortens the loop when a state machine misbehaves. Gate it on `kDebugMode` so production builds stay quiet.

`HydratedBloc.storage` must be initialized **before** `runApp()`; otherwise any `HydratedBloc` constructed during the first frame will crash with `HydratedStorageNotFound`. Use `path_provider` to resolve the storage directory on Android and iOS.

Use `getApplicationDocumentsDirectory()` (durable across app updates and OS reboots), **not** `getTemporaryDirectory()` — the latter can be cleared by the OS without warning, which would silently lose the user's cart on a low-storage device. Some upstream examples use the temp directory because it's faster for one-off demos; for a real app with persisted cart/auth state, the documents directory is the right choice.

## Project structure

Bloc replaces the `view_models/` directory from `flutter-apply-architecture-best-practices`. The Repository, Service, and Domain Model layers from that skill stay unchanged.

```
lib/
├── core/
│   ├── bloc/
│   │   └── app_bloc_observer.dart
│   └── result/
│       ├── failure.dart        # added by flutter-bloc-async-api
│       └── result.dart         # added by flutter-bloc-async-api
├── data/                        # unchanged from flutter-apply-architecture-best-practices
│   ├── models/                  # API DTOs
│   ├── repositories/            # consumed by Blocs via RepositoryProvider
│   └── services/
├── domain/                      # unchanged
│   └── models/
└── ui/
    └── features/
        └── <feature>/
            ├── bloc/
            │   ├── <feature>_bloc.dart
            │   ├── <feature>_event.dart
            │   └── <feature>_state.dart
            └── view/
                └── <feature>_view.dart
```

One folder per feature, three files per Bloc. Tests for each Bloc mirror the path under `test/ui/features/<feature>/bloc/<feature>_bloc_test.dart`.

## Workflow: Bootstrap a Bloc-ready project

### Task Progress
- [ ] **Step 1 — Install runtime deps.** `flutter pub add flutter_bloc freezed_annotation json_annotation bloc_concurrency hydrated_bloc path_provider`.
- [ ] **Step 2 — Install generators.** `flutter pub add --dev build_runner freezed json_serializable`.
- [ ] **Step 3 — Create the observer.** Write `lib/core/bloc/app_bloc_observer.dart` extending `BlocObserver`. Override `onChange` and `onError`.
- [ ] **Step 4 — Initialize storage.** In `main()`, call `WidgetsFlutterBinding.ensureInitialized()` then `HydratedBloc.storage = await HydratedStorage.build(storageDirectory: HydratedStorageDirectory((await getApplicationDocumentsDirectory()).path))` **before** `runApp()`.
- [ ] **Step 5 — Wire the observer.** `if (kDebugMode) Bloc.observer = const AppBlocObserver();` — only in debug builds, so production stays quiet.
- [ ] **Step 6 — Wrap the app root.** Replace your top-level widget with `MultiRepositoryProvider(providers: const <RepositoryProvider>[], child: MaterialApp(...))`. The `providers` list grows as features land.
- [ ] **Step 7 — Create the feature directory.** `mkdir -p lib/ui/features lib/core/bloc`.
- [ ] **Step 8 — Run the generator.** Open a side terminal: `dart run build_runner watch -d`. Leave it running while you work.

**Troubleshooting.** Run `flutter analyze` → if `dart run build_runner` errors, check that every freezed file imports `package:freezed_annotation/freezed_annotation.dart` and declares `part '<file>.freezed.dart';` → re-run. If `HydratedStorageNotFound` is thrown on first launch, your `HydratedBloc.storage` initialization happens after `runApp()` — move it before in `main()`.

## Applied to Talabat-clone

For a multi-store delivery app, the Blocs the family will eventually scaffold are (cross-referenced against `PRD_states.md`):

| Bloc | Persists? | Why |
|---|---|---|
| `AuthBloc` | yes (`HydratedBloc`) | Auth token + the user's `User: active` lifecycle state are both worth persisting across cold start; see `PRD_states.md §14` for the user-account state machine and `prd.md` for the auth/session flow. |
| `CartBloc` | yes (`HydratedBloc`) | One-cart / one-store / one-city rules; cart survives restart (`PRD_states.md §7`, `PRD_catalog.md §15.5`). |
| `StoreListBloc` | no | Re-fetch on launch; depends on location and operational gating (`PRD_states.md §4`). |
| `StoreDetailBloc` | no | Per-route, ephemeral. |
| `OrderBloc` | no | Authoritative source is the API. |
| `OrderTrackingBloc` | no | Stream-driven, scoped to an Order's `picked_up → delivered` window (deferred to `flutter-bloc-stream-tracking`). |
| `PromotionBloc` | no | Validity is computed from `start_date` / `end_date` / `is_paused` (`PRD_states.md §9` Promotion/Offer; siblings §8 Coupon and §10 Advertisement share the shape with minor variations). |

The root `MultiRepositoryProvider` will hold `AuthRepository`, `StoreRepository`, `OrderRepository`, `CartRepository` (cart is local-only; the repository wraps `hydrated_bloc` storage), and `PromotionRepository`.

## Examples

### Minimal `main.dart`

```dart
import 'package:flutter/foundation.dart';
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:hydrated_bloc/hydrated_bloc.dart';
import 'package:path_provider/path_provider.dart';

import 'core/bloc/app_bloc_observer.dart';

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();

  HydratedBloc.storage = await HydratedStorage.build(
    storageDirectory: HydratedStorageDirectory(
      (await getApplicationDocumentsDirectory()).path,
    ),
  );

  if (kDebugMode) {
    Bloc.observer = const AppBlocObserver();
  }

  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MultiRepositoryProvider(
      // Typed empty list — `const []` bare would fail `strict-inference`.
      providers: const <RepositoryProvider>[
        // Repositories are added here by flutter-bloc-async-api as features land.
        // Example:
        //   RepositoryProvider<AuthRepository>(
        //     create: (_) => AuthRepository(...),
        //     lazy: true,
        //   ),
      ],
      child: MaterialApp(
        title: 'Multi-Store Delivery',
        home: Scaffold(
          appBar: AppBar(title: const Text('Smoke test')),
          body: const Center(child: Text('Bloc setup OK')),
        ),
      ),
    );
  }
}
```

### `lib/core/bloc/app_bloc_observer.dart`

```dart
import 'package:flutter/foundation.dart';
import 'package:flutter_bloc/flutter_bloc.dart';

/// Logs every Bloc transition and error during development.
/// Wired in `main()` only when `kDebugMode` is true.
class AppBlocObserver extends BlocObserver {
  const AppBlocObserver();

  // Override `onTransition` (not `onChange`) so Bloc logs show the
  // `event + currentState → nextState` triple. `onChange` only sees the
  // state diff, which loses the event-causation half. Cubits (no events)
  // fall through to `onChange` automatically.
  @override
  void onTransition(
    Bloc<dynamic, dynamic> bloc,
    Transition<dynamic, dynamic> transition,
  ) {
    super.onTransition(bloc, transition);
    debugPrint('[${bloc.runtimeType}] $transition');
  }

  @override
  void onError(BlocBase<dynamic> bloc, Object error, StackTrace stackTrace) {
    debugPrint('[${bloc.runtimeType}] ERROR: $error');
    super.onError(bloc, error, stackTrace);
  }
}
```

### `pubspec.yaml` excerpt

```yaml
dependencies:
  flutter:
    sdk: flutter
  flutter_bloc: ^9.0.0
  freezed_annotation: ^3.0.0
  json_annotation: ^4.9.0
  bloc_concurrency: ^0.3.0
  hydrated_bloc: ^10.0.0
  path_provider: ^2.1.0
  http: ^1.2.0

dev_dependencies:
  flutter_test:
    sdk: flutter
  build_runner: ^2.4.0
  freezed: ^3.0.0
  json_serializable: ^6.8.0
```
