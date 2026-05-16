---
name: flutter-bloc-feature-pattern
description: Use when adding a new feature to a Bloc-based Flutter project (after `flutter-bloc-setup`), structuring an existing feature with a canonical event/state/bloc trio backed by `freezed` sealed states, or deciding which `bloc_concurrency` transformer (`droppable`/`restartable`/`sequential`/`concurrent`) fits an event handler.
metadata:
  model: claude-opus-4-7
  last_modified: Thu, 14 May 2026 00:00:00 GMT
---

# Scaffolding a Bloc Feature

Defines the canonical shape of a Bloc-driven feature: three files (`<feature>_event.dart`, `<feature>_state.dart`, `<feature>_bloc.dart`), sealed states with `freezed`, and an explicit `EventTransformer` per handler. Every other skill in the family (`flutter-bloc-async-api`, `flutter-bloc-stream-tracking`, `flutter-bloc-testing`, `flutter-bloc-forms`) builds on this shape. Prerequisite: `flutter-bloc-setup`.

## Contents
- [The Event/State/Bloc trio](#the-eventstatebloc-trio)
- [Sealed states with freezed](#sealed-states-with-freezed)
- [Event handlers and concurrency](#event-handlers-and-concurrency)
- [Wiring the View](#wiring-the-view)
- [Persistence (optional)](#persistence-optional)
- [Workflow: Scaffold a new feature](#workflow-scaffold-a-new-feature)
- [Applied to Talabat-clone](#applied-to-talabat-clone)
- [Examples](#examples)

## The Event/State/Bloc trio

Every feature has exactly three files under `lib/ui/features/<feature>/bloc/`.

| File | Responsibility | Contains |
|---|---|---|
| `<feature>_event.dart` | User intent and side-effect triggers. Plain data. | Sealed `<Feature>Event` class with one variant per intent. |
| `<feature>_state.dart` | Immutable UI snapshots. Plain data. | Sealed `<Feature>State` class with one variant per render mode. |
| `<feature>_bloc.dart` | Maps events to states. The only place that calls repositories. | `class <Feature>Bloc extends Bloc<<Feature>Event, <Feature>State>` (or `HydratedBloc` if persisting). |

Events and states are **never** mutable. The Bloc constructor registers handlers via `on<Event>(handler, transformer: ...)` and emits new state instances. Views observe state and dispatch events; they never read fields off the Bloc directly.

## Sealed states with freezed

Use `@freezed` with a `sealed` class so a `switch` over state variants is **exhaustive at compile time**. Forgetting to handle a variant becomes a compile error rather than a runtime UI bug.

```dart
// lib/ui/features/cart/bloc/cart_state.dart
import 'package:freezed_annotation/freezed_annotation.dart';
import 'package:my_app/domain/models/cart_item.dart';
import 'package:my_app/domain/models/money.dart';

part 'cart_state.freezed.dart';
part 'cart_state.g.dart'; // required when the state is also `HydratedBloc`-persisted

@Freezed(unionKey: 'type') // stable discriminator key across class renames
sealed class CartState with _$CartState {
  const factory CartState.initial() = CartInitial;
  const factory CartState.loading() = CartLoading;
  const factory CartState.loaded({
    required List<CartItem> items,
    required Money subtotal,
    required Money total,
  }) = CartLoaded;
  const factory CartState.failure({required String message}) = CartFailure;

  // Required for `HydratedBloc` round-trip — `fromJson` is **not** generated
  // automatically; you must declare this factory yourself. `CartItem` and
  // `Money` must each be JSON-serializable (typically also `@freezed` classes).
  factory CartState.fromJson(Map<String, dynamic> json) =>
      _$CartStateFromJson(json);
}
```

The View consumes the sealed state with a Dart 3 pattern `switch`:

```dart
final widget = switch (state) {
  CartInitial()    => const _EmptyCart(),
  CartLoading()    => const Center(child: CircularProgressIndicator()),
  CartLoaded(:final items, :final total) => _CartList(items: items, total: total),
  CartFailure(:final message)            => _ErrorBanner(message: message),
};
```

If a future PR adds a `CartLoaded.suspended` variant, every `switch` in the codebase fails to compile until handled. That is the point.

## Event handlers and concurrency

`bloc_concurrency` provides four `EventTransformer`s. Pick the one that matches the **semantic** of the event, not the one that "feels safe".

| Transformer | Use when | Example event |
|---|---|---|
| `droppable()` | Submit-once. Ignore further taps while one is in flight. | `OrderPlaced`, `PaymentSubmitted` |
| `restartable()` | Only the latest matters. Cancel the in-flight handler when a new event arrives. | `SearchQueryChanged`, `RefreshRequested` |
| `sequential()` | Order matters. Run handlers one at a time, FIFO. | `CartItemAdded`, `CartItemQtyChanged` |
| `concurrent()` (default) | Independent fetches. Run in parallel. | `ProductDetailsLoaded`, `ReviewsLoaded` |

```dart
class CartBloc extends HydratedBloc<CartEvent, CartState> {
  CartBloc({required CartRepository repo})
      : _repo = repo,
        super(const CartState.initial()) {
    on<CartItemAdded>(_onItemAdded, transformer: sequential());
    on<CartItemQtyChanged>(_onQtyChanged, transformer: sequential());
    on<CartCleared>(_onCleared, transformer: droppable());
  }

  final CartRepository _repo;
  // ...handlers
}
```

Sequential is **not** the default. The default `concurrent()` will reorder cart mutations — adding then removing the same item could leave the item in the cart if the remove handler finishes first. Always state the transformer explicitly for any handler that mutates shared state.

## Wiring the View

Four widgets bridge a Bloc to the widget tree. Pick the narrowest one that does the job.

| Widget | Purpose | Rebuilds? |
|---|---|---|
| `BlocProvider` | Inject a Bloc into the subtree. Place at the route, not at app root. | n/a |
| `BlocBuilder<B, S>` | Rebuild on every state change. The default. | yes, on every state |
| `BlocSelector<B, S, T>` | Rebuild only when a derived value `T` changes. Use for fine-grained updates. | yes, only when selector output changes |
| `BlocListener<B, S>` | Fire-and-forget side effects: snackbar, navigation, dialog. | **no** — child is unaffected |
| `BlocConsumer` | Rare combination of builder + listener. | yes |

**Common mistake:** wrapping the entire screen in a `BlocBuilder` that rebuilds 30 widgets when only the cart total changed. Use `BlocSelector` for the total and let the rest of the tree stay stable.

## Persistence (optional)

For state that must survive a cold start (cart, auth tokens, recently-viewed list), extend `HydratedBloc<E, S>` instead of `Bloc<E, S>` — or add `HydratedMixin` to an existing `Cubit`/`Bloc` and call `hydrate()` in its constructor when changing the parent class isn't an option. Override `fromJson` and `toJson` on the Bloc. With `freezed`, the serialization helpers are generated — but **only when the state declares an explicit `factory State.fromJson(...)` factory and the file declares `part 'state.g.dart';`**. See the `CartState` snippet above; missing either of these makes `_$CartStateFromJson` undefined and the file won't compile.

**Always override `storagePrefix`** to a stable string literal:

```dart
@override
String get storagePrefix => 'CartBloc';
```

The default storage key is `runtimeType.toString()`, which gets mangled by Dart's minifier in `flutter build apk --release` / `--obfuscate`. A user upgrading the app would find their persisted cart unreadable because the storage key changed names between builds. Pin the prefix in source so the key survives minification.

```dart
import 'package:bloc_concurrency/bloc_concurrency.dart';
import 'package:hydrated_bloc/hydrated_bloc.dart';

class CartBloc extends HydratedBloc<CartEvent, CartState> {
  CartBloc({required CartRepository repo})
      : _repo = repo,
        super(const CartState.initial()) {
    // Persistence and transformers coexist — the example is deliberately
    // explicit about both so readers see them together.
    on<CartHydrated>(_onHydrated, transformer: droppable());
    on<CartItemAdded>(_onItemAdded, transformer: sequential());
    on<CartItemQtyChanged>(_onQtyChanged, transformer: sequential());
    on<CartCleared>(_onCleared, transformer: droppable());
  }

  final CartRepository _repo;

  // ...handlers (omitted; each emits state.copyWith(...) per business rule)

  // Pin the storage key so release builds (which may minify class names)
  // continue reading the same persisted state across upgrades.
  @override
  String get storagePrefix => 'CartBloc';

  @override
  CartState? fromJson(Map<String, dynamic> json) => CartState.fromJson(json);

  @override
  Map<String, dynamic>? toJson(CartState state) =>
      (state as dynamic).toJson() as Map<String, dynamic>?;
}
```

`toJson` is cast through `dynamic` because freezed's generated `toJson` lives on each variant (e.g. `CartLoaded.toJson()`), not on the sealed parent's static analysis surface. The runtime resolution picks the right variant; the cast just gets past static type checking.

### Limitations

Persisting a Bloc whose handler uses `restartable()` or `concurrent()` has two distinct ordering subtleties — don't conflate them:

1. **`restartable()` and in-flight emits.** When event B arrives during event A's handler, `restartable()` marks A's emitter as done. Any `emit(...)` A tries after that is silently dropped (good). The persisted state reflects whatever was emitted *before* B took over — so persistence sees the last successful emit, not necessarily the last dispatched event.
2. **Process death, separately.** If the app is killed while A is mid-`await`, A never gets the chance to emit at all — there's no "cancellation" because there's no process. The persisted state is whatever was emitted *before A started*.

Either way: treat persisted state as a *starting point*. When the screen mounts after a cold start, re-fetch from the API rather than assuming the persisted state reflects an in-flight request's eventual result.

## Workflow: Scaffold a new feature

### Task Progress
- [ ] **Step 1 — Pick a feature name.** Lowercase, singular noun (e.g., `cart`, `auth`, `restaurant_list`).
- [ ] **Step 2 — Create the directory.** `mkdir -p lib/ui/features/<feature>/{bloc,view}`.
- [ ] **Step 3 — Write `<feature>_event.dart`.** Sealed `<Feature>Event` class with one factory per user intent. No logic.
- [ ] **Step 4 — Write `<feature>_state.dart`.** Sealed `<Feature>State` with `Initial`, `Loading`, `Loaded(...)`, `Failure(message)` at minimum. Add domain-specific variants as needed.
- [ ] **Step 5 — Write `<feature>_bloc.dart`.** Extend `Bloc<E, S>` (or `HydratedBloc<E, S>` if persisting). Inject the repository via constructor.
- [ ] **Step 6 — Register handlers with explicit transformers.** Use `sequential()` for mutations, `restartable()` for queries, `droppable()` for submits, `concurrent()` only for independent fetches.
- [ ] **Step 7 — If persisting.** Override `fromJson` / `toJson` on the Bloc using `freezed`-generated helpers.
- [ ] **Step 8 — Generate code.** `dart run build_runner build --delete-conflicting-outputs` (or rely on the `watch -d` side terminal from `flutter-bloc-setup`).
- [ ] **Step 9 — Wire the View.** Wrap the route in `BlocProvider(create: (_) => <Feature>Bloc(repo: context.read<<Feature>Repository>()))`. Use `BlocBuilder` for rendering, `BlocListener` for side effects, `BlocSelector` for fine-grained rebuilds.
- [ ] **Step 10 — Feedback loop.** Run `flutter analyze` → review compile errors → if a `switch` over state is non-exhaustive, handle the missing variant → re-run.

## Applied to Talabat-clone

The cart in this app has the most rules-heavy state in the system: one-cart / one-store / one-city, persistence across cold starts, and a "switch store" guard (`PRD_catalog.md §15.5`, `PRD_states.md §7`).

```text
// pseudocode sketch — see the Counter example below for runnable form
// CartEvent
sealed class CartEvent
  CartHydrated()                                              // app startup, deserialize from disk
  CartItemAdded(Product product, int qty)                     // user taps "+" on a product card
  CartItemQtyChanged(String itemId, int qty)                  // bottom-sheet qty stepper
  CartItemRemoved(String itemId)                              // swipe-to-delete
  CartCleared()                                               // checkout success or explicit clear
  CartStoreSwitchRequested(Product fromOtherStore)            // user added from a different store
  CartStoreSwitchConfirmed()                                  // user confirmed "discard current cart"
  CartStoreSwitchCancelled()                                  // user backed out

// CartState
sealed class CartState
  CartInitial()
  CartLoaded(items, store, subtotal, discount, total)
  CartStoreSwitchPending(currentLoaded, pendingProduct)        // PRD_catalog.md §15.5
  CartFailure(message)
```

`CartBloc` extends `HydratedBloc` because the cart must survive cold start. `CartItemAdded` uses `sequential()` so two rapid taps on "+" never interleave with `CartItemQtyChanged`. `CartStoreSwitchRequested` is a pure state transition — no API call — that puts the Bloc in `CartStoreSwitchPending` until the user confirms or cancels. `CartCleared` happens at checkout success when the API confirms `Cart: active → converted` (`PRD_states.md §7`).

`StoreListBloc` does **not** extend `HydratedBloc` — store availability depends on operational gating (`PRD_states.md §4`) and must be re-fetched on launch.

## Examples

A complete Counter feature illustrates the trio with no async dependencies. It uses `sequential()` because increment-then-decrement order matters.

### `lib/ui/features/counter/bloc/counter_event.dart`

```dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'counter_event.freezed.dart';

@freezed
sealed class CounterEvent with _$CounterEvent {
  const factory CounterEvent.incremented() = CounterIncremented;
  const factory CounterEvent.decremented() = CounterDecremented;
  const factory CounterEvent.reset() = CounterReset;
}
```

### `lib/ui/features/counter/bloc/counter_state.dart`

```dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'counter_state.freezed.dart';

// We use `sealed` uniformly across the codebase, even for one-variant states.
// `abstract` would also compile here but the family standardizes on `sealed`
// so future variants don't trigger a class-shape change.
@freezed
sealed class CounterState with _$CounterState {
  const factory CounterState({required int value}) = _CounterState;
}
```

### `lib/ui/features/counter/bloc/counter_bloc.dart`

```dart
import 'package:bloc_concurrency/bloc_concurrency.dart';
import 'package:flutter_bloc/flutter_bloc.dart';

import 'counter_event.dart';
import 'counter_state.dart';

class CounterBloc extends Bloc<CounterEvent, CounterState> {
  CounterBloc() : super(const CounterState(value: 0)) {
    on<CounterIncremented>(
      (e, emit) => emit(CounterState(value: state.value + 1)),
      transformer: sequential(),
    );
    on<CounterDecremented>(
      (e, emit) => emit(CounterState(value: state.value - 1)),
      transformer: sequential(),
    );
    on<CounterReset>(
      (e, emit) => emit(const CounterState(value: 0)),
      transformer: droppable(),
    );
  }
}
```

### `lib/ui/features/counter/view/counter_view.dart`

```dart
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';

import '../bloc/counter_bloc.dart';
import '../bloc/counter_event.dart';
import '../bloc/counter_state.dart';

class CounterView extends StatelessWidget {
  const CounterView({super.key});

  @override
  Widget build(BuildContext context) {
    return BlocProvider(
      create: (_) => CounterBloc(),
      child: const _CounterScaffold(),
    );
  }
}

class _CounterScaffold extends StatelessWidget {
  const _CounterScaffold();

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Counter')),
      body: Center(
        child: BlocSelector<CounterBloc, CounterState, int>(
          selector: (state) => state.value,
          builder: (context, value) => Text(
            '$value',
            style: Theme.of(context).textTheme.displayLarge,
          ),
        ),
      ),
      floatingActionButton: Column(
        mainAxisAlignment: MainAxisAlignment.end,
        children: [
          FloatingActionButton(
            heroTag: 'inc',
            onPressed: () =>
                context.read<CounterBloc>().add(const CounterEvent.incremented()),
            child: const Icon(Icons.add),
          ),
          const SizedBox(height: 8),
          FloatingActionButton(
            heroTag: 'dec',
            onPressed: () =>
                context.read<CounterBloc>().add(const CounterEvent.decremented()),
            child: const Icon(Icons.remove),
          ),
        ],
      ),
    );
  }
}
```

`BlocSelector<CounterBloc, CounterState, int>` rebuilds only when `state.value` changes, not when the Bloc emits a new `CounterState` instance with the same value. For a single-field state this is overkill, but the pattern matters for screens with many fields where one cell shouldn't trigger a full rebuild.
