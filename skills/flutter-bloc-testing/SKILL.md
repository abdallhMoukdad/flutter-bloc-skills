---
name: flutter-bloc-testing
description: Writes deterministic unit tests for Blocs using `bloc_test` and `mocktail`, including tests for `bloc_concurrency` transformers and `HydratedBloc` persistence. Use when adding tests for a Bloc's event handlers, verifying its emitted state sequence, mocking a Repository dependency, or asserting that `restartable()` cancels in-flight requests. Prerequisite: `flutter-bloc-setup`, `flutter-bloc-feature-pattern`, and `flutter-bloc-async-api`.
metadata:
  model: claude-opus-4-7
  last_modified: Thu, 14 May 2026 00:00:00 GMT
---

# Testing Blocs

Drives Blocs from outside with deterministic inputs and asserts the exact state sequence they emit. `bloc_test` wraps the lifecycle (build a fresh Bloc, dispatch events, await stream, dispose) into a single declarative call. `mocktail` provides null-safe Repository mocks without code generation. Concurrency tests use a `Completer` to control timing so an assertion can run *between* a stale and a fresh request. Builds on `flutter-bloc-async-api`; the example suite tests `RestaurantListBloc` from that skill.

## Contents
- [The `blocTest` helper](#the-bloctest-helper)
- [Mocking repositories with mocktail](#mocking-repositories-with-mocktail)
- [Testing concurrency transformers](#testing-concurrency-transformers)
- [Testing HydratedBloc persistence](#testing-hydratedbloc-persistence)
- [Workflow: Test a Bloc](#workflow-test-a-bloc)
- [Applied to Talabat-clone](#applied-to-talabat-clone)
- [Examples](#examples)

## The `blocTest` helper

`blocTest<B, S>(description, ...)` registers a Dart test that builds a fresh Bloc, runs an action, and asserts the emitted states. It is a structured wrapper around `expectLater(bloc.stream, emitsInOrder([...]))` with five named hooks:

| Hook | Purpose |
|---|---|
| `build` | Construct a fresh Bloc. Called once per test. |
| `setUp` | Configure mocks before `build`. Use to wire `when(() => repo.fetch()).thenAnswer(...)`. |
| `seed` | Inject an initial state without going through the constructor. Useful for "given the Bloc is in state X, when event Y arrives..." |
| `act` | Dispatch events. The body is `(bloc) => bloc.add(...)`. |
| `wait` | Optional `Duration` to wait after `act` before asserting. Needed only when handlers use timers. |
| `expect` | List of states the Bloc must emit, in order. Either literal values or matchers (`isA<Loading>()`). |
| `verify` | Optional final hook for assertions on mocks: `verify(() => repo.fetch(loc)).called(1)`. |

A passing `blocTest` is proof that **for these exact inputs, the Bloc emits exactly this sequence**. The Bloc's initial state is implicit and not asserted — `expect:` describes the deltas.

## Mocking repositories with mocktail

mocktail generates mocks at runtime via `class MockX extends Mock implements X {}`. No `build_runner`, no generated files. Two patterns matter:

**Stubbing a return value:**

```dart
when(() => repo.fetchNearbyStores(any()))
    .thenAnswer((_) async => const Result.success(<Store>[]));
```

`any()` matches any positional argument. For named arguments use `any(named: 'foo')`. For matching specific values pass the value directly.

**Fallback values for unhandled types:**

If a method takes a custom object as a parameter, register a fallback once in `setUpAll`:

```dart
setUpAll(() {
  registerFallbackValue(const Location(lat: 0, lng: 0));
});
```

Without the fallback, `any()` against a `Location` parameter throws a `StateError` at first use.

## Testing concurrency transformers

`restartable()` says "cancel the in-flight handler when a new event arrives". A naive test that dispatches two events back-to-back and asserts the second result will pass even without `restartable()` — because the events run synchronously fast enough that the first never overlaps with the second.

The reliable test uses a `Completer` to pin the first handler in flight while the second arrives:

```dart
test('refresh cancels the in-flight initial load', () async {
  final initialCompleter = Completer<Result<List<Store>>>();
  final refreshCompleter = Completer<Result<List<Store>>>();
  final calls = <Completer<Result<List<Store>>>>[initialCompleter, refreshCompleter];

  when(() => repo.fetchNearbyStores(any()))
      .thenAnswer((_) => calls.removeAt(0).future);

  final bloc = RestaurantListBloc(repo: repo);
  bloc.add(const RestaurantListLoadRequested(_loc));
  await Future<void>.delayed(Duration.zero); // let _onLoad start
  bloc.add(const RestaurantListRefreshRequested(_loc));
  await Future<void>.delayed(Duration.zero); // let restartable() cancel

  // `_storeA`/`_storeB` are runtime finals (not `const`), so `Result.success([...])`
  // cannot be `const` either.
  initialCompleter.complete(Result.success([_storeA])); // dropped — restartable cancelled this handler's emitter
  refreshCompleter.complete(Result.success([_storeB]));
  await bloc.close();

  // The first emit was Loading, then Loading again from the refresh,
  // then Loaded with the second result. The first Result.success([_storeA])
  // is dropped because the handler that would emit it was cancelled.
  expect(bloc.state, isA<RestaurantListLoaded>());
  expect((bloc.state as RestaurantListLoaded).stores, [_storeB]);
});
```

The `Completer`-based timing is what proves `restartable()` is actually doing its job. Without it the test only proves "events serialize", which they would anyway under `sequential()` or even `concurrent()` for this simple case.

## Testing HydratedBloc persistence

`HydratedBloc` reads and writes a global `HydratedBloc.storage` singleton. Test files that touch HydratedBlocs must register an in-memory `Storage` fake **once per test process**, or tests will pollute each other (and the device, if you forget to swap to a fake).

```dart
class _InMemoryStorage implements Storage {
  final _map = <String, dynamic>{};

  // `dynamic` matches the upstream `Storage` interface — a real fake can tighten it.
  @override
  dynamic read(String key) => _map[key];
  @override
  Future<void> write(String key, dynamic value) async => _map[key] = value;
  @override
  Future<void> delete(String key) async => _map.remove(key);
  @override
  Future<void> clear() async => _map.clear();
  // The official `Storage` interface declares four required members
  // (read/write/delete/clear). `close()` isn't in the contract today,
  // but a no-op makes the fake forward-compatible if a future version
  // adds it.
  @override
  Future<void> close() async {}
}

setUpAll(() {
  HydratedBloc.storage = _InMemoryStorage();
});

setUp(() async {
  // `clear()` returns a Future. With the in-memory fake the mutation lands
  // synchronously before the first await, but always `await` it so the pattern
  // survives a swap to an async-yielding fake (e.g. a SharedPreferences-backed one).
  await HydratedBloc.storage.clear();
});
```

Asserting rehydration is then a two-Bloc dance: build, mutate, close, build a second instance, check state. The `firstWhere` call subscribes to the stream before `add` enqueues the handler (microtask order), so the persisted write happens before `close()`.

```dart
test('CartBloc rehydrates after restart', () async {
  final a = CartBloc(repo: repo);
  a.add(CartItemAdded(_product, 2));
  await a.stream.firstWhere((s) => s is CartLoaded); // subscribes before handler runs
  await a.close();

  // Second instance reads the persisted state synchronously in its constructor.
  final b = CartBloc(repo: repo);
  expect(b.state, isA<CartLoaded>());
  expect((b.state as CartLoaded).items.length, 1);
});
```

## Workflow: Test a Bloc

### Task Progress
- [ ] **Step 1 — Mirror the bloc path.** Test file lives at `test/ui/features/<feature>/bloc/<feature>_bloc_test.dart`.
- [ ] **Step 2 — Add dev deps.** `flutter pub add --dev bloc_test mocktail`.
- [ ] **Step 3 — Declare mock classes.** `class MockStoreRepository extends Mock implements StoreRepository {}` at the top of the file.
- [ ] **Step 4 — `setUpAll` for fallbacks.** Register `registerFallbackValue(<custom-type>(...))` for every non-primitive type used in `any()`.
- [ ] **Step 5 — `setUp` per test.** Instantiate the mock, instantiate the Bloc.
- [ ] **Step 6 — Write `blocTest` cases.** One per "given X event(s), expect Y state sequence". Always cover: happy path, failure path, and any branch in the handler.
- [ ] **Step 7 — Cover transformer behavior.** For each `restartable()` / `droppable()` / `sequential()` handler, write one explicit test that proves the transformer's contract (use `Completer` for timing).
- [ ] **Step 8 — If `HydratedBloc`.** Add the `_InMemoryStorage` fake in `setUpAll`, clear it in `setUp`, and write a rehydration round-trip test.
- [ ] **Step 9 — Feedback loop.** `flutter test test/ui/features/<feature>/bloc/<feature>_bloc_test.dart` → if a test hangs, the Bloc is probably awaiting a future that never completes (check `Completer` usage) → if a test reports an unexpected extra state, the previous test's mock leaked (check `setUp` scoping).

## Applied to Talabat-clone

Two tests from the cart suite illustrate the patterns.

**`CartBloc` rehydrates after cold start (`PRD_states.md §7`).**

The cart is `Cart: active → converted | abandoned`. On cold start, a returning user must see their cart exactly as they left it. The test seeds `_InMemoryStorage`, builds a fresh `CartBloc`, and asserts the initial emitted state already has the persisted items — no `Loading` flicker.

**`CartBloc` rejects items from a second store (`PRD_catalog.md §15.5`).**

The one-cart / one-store rule says adding a product from a different store should transition to `CartStoreSwitchPending(pendingProduct)` rather than blend the carts. The test seeds a `CartLoaded(items: [productFromStoreA])`, dispatches `CartItemAdded(productFromStoreB, 1)`, and asserts the next state is `CartStoreSwitchPending`, not `CartLoaded` with both items. A follow-up test dispatches `CartStoreSwitchConfirmed` and asserts the cart is replaced.

These tests are the **only safety net** between the PRD rules and a future refactor that silently drops one of them.

## Examples

A test suite for `RestaurantListBloc` from `flutter-bloc-async-api`. Covers success, failure, and the restartable refresh transformer.

### `test/ui/features/restaurant_list/bloc/restaurant_list_bloc_test.dart`

```dart
import 'dart:async';

import 'package:bloc_test/bloc_test.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:mocktail/mocktail.dart';

import 'package:my_app/core/result/failure.dart';
import 'package:my_app/core/result/result.dart';
import 'package:my_app/data/repositories/store_repository.dart';
import 'package:my_app/domain/models/location.dart';
import 'package:my_app/domain/models/store.dart';
import 'package:my_app/ui/features/restaurant_list/bloc/restaurant_list_bloc.dart';
import 'package:my_app/ui/features/restaurant_list/bloc/restaurant_list_event.dart';
import 'package:my_app/ui/features/restaurant_list/bloc/restaurant_list_state.dart';

class MockStoreRepository extends Mock implements StoreRepository {}

const _loc = Location(lat: 25.276987, lng: 55.296249); // Dubai
final _storeA = Store(id: 'a', name: 'A', deliveryMinutes: 20);
final _storeB = Store(id: 'b', name: 'B', deliveryMinutes: 30);

void main() {
  setUpAll(() {
    // `_loc` is `const` — same canonical instance used by both registration and dispatch.
    registerFallbackValue(_loc);
  });

  group('RestaurantListBloc', () {
    late MockStoreRepository repo;

    setUp(() {
      repo = MockStoreRepository();
    });

    blocTest<RestaurantListBloc, RestaurantListState>(
      'emits [Loading, Loaded] when fetch succeeds',
      setUp: () {
        when(() => repo.fetchNearbyStores(any()))
            .thenAnswer((_) async => Result.success([_storeA, _storeB]));
      },
      build: () => RestaurantListBloc(repo: repo),
      act: (b) => b.add(const RestaurantListLoadRequested(_loc)),
      expect: () => [
        const RestaurantListState.loading(),
        RestaurantListState.loaded([_storeA, _storeB]),
      ],
      verify: (_) {
        verify(() => repo.fetchNearbyStores(_loc)).called(1);
      },
    );

    blocTest<RestaurantListBloc, RestaurantListState>(
      'emits [Loading, Empty] when fetch returns no stores',
      setUp: () {
        when(() => repo.fetchNearbyStores(any()))
            .thenAnswer((_) async => const Result.success(<Store>[]));
      },
      build: () => RestaurantListBloc(repo: repo),
      act: (b) => b.add(const RestaurantListLoadRequested(_loc)),
      expect: () => [
        const RestaurantListState.loading(),
        const RestaurantListState.empty(),
      ],
    );

    blocTest<RestaurantListBloc, RestaurantListState>(
      'emits [Loading, Failure] on network error',
      setUp: () {
        when(() => repo.fetchNearbyStores(any()))
            .thenAnswer((_) async => const Result.failure(Failure.network()));
      },
      build: () => RestaurantListBloc(repo: repo),
      act: (b) => b.add(const RestaurantListLoadRequested(_loc)),
      expect: () => [
        const RestaurantListState.loading(),
        const RestaurantListState.failure('No internet connection'),
      ],
    );

    // This test drops out of `blocTest` and uses raw `test(...)` because
    // `blocTest`'s `wait:` parameter is a fixed `Duration` — it cannot interleave
    // `Completer.complete` calls between event dispatches, which is what proves
    // the `restartable()` contract.
    test('refresh cancels the in-flight initial load', () async {
      // Named completers so we can resolve them in a deliberate order.
      final initialCompleter = Completer<Result<List<Store>>>();
      final refreshCompleter = Completer<Result<List<Store>>>();
      final completers = [initialCompleter, refreshCompleter];

      // First call to `fetchNearbyStores` returns initialCompleter.future,
      // the second returns refreshCompleter.future.
      when(() => repo.fetchNearbyStores(any()))
          .thenAnswer((_) => completers.removeAt(0).future);

      final bloc = RestaurantListBloc(repo: repo);
      final emitted = <RestaurantListState>[];
      final sub = bloc.stream.listen(emitted.add);

      // Dispatch the initial load; let _onLoad start awaiting the first future.
      bloc.add(const RestaurantListLoadRequested(_loc));
      await Future<void>.delayed(Duration.zero);

      // Dispatch refresh; restartable() cancels the initial handler's emitter.
      bloc.add(const RestaurantListRefreshRequested(_loc));
      await Future<void>.delayed(Duration.zero);

      // Now resolve the FIRST completer. Its emit must be dropped because the
      // initial handler's emitter is already done.
      initialCompleter.complete(Result.success([_storeA]));
      refreshCompleter.complete(Result.success([_storeB]));

      // One more turn so the refresh handler emits its Loaded.
      await Future<void>.delayed(Duration.zero);
      await bloc.close();
      await sub.cancel();

      // The load-bearing invariant: we never emit `Loaded([_storeA])`.
      // Only one `Loaded` should appear, and it must carry the refresh's result.
      final loadedStates = emitted.whereType<RestaurantListLoaded>().toList();
      expect(loadedStates, hasLength(1),
          reason: 'restartable() should drop the stale Loaded([_storeA])');
      expect(loadedStates.single.stores, [_storeB]);
      expect(bloc.state, isA<RestaurantListLoaded>());
    });
  });
}
```

For features that extend `HydratedBloc` (e.g., `CartBloc`), add the in-memory storage fake from the "Testing HydratedBloc persistence" section above to the top-level `setUpAll`, and clear it in `setUp` so tests do not see each other's persisted state.
