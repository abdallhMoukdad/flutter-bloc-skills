# Review: flutter-bloc-testing SKILL.md

**Reviewer:** Opus (independent)
**Date:** 2026-05-14
**File reviewed:** `/home/abdallh/.claude/skills/flutter-bloc-testing/SKILL.md`
**Family doc:** `/home/abdallh/.claude/skills/_design/2026-05-09-flutter-bloc-skills-family-design.md` §5.2
**Prereq referenced:** `/home/abdallh/.claude/skills/flutter-bloc-async-api/SKILL.md`

---

## Verdict

**Needs revision before shipping.** The conceptual scaffolding (sections, hooks table, mocktail intro, HydratedBloc rationale, PRD anchoring) is solid and matches the family conventions. But the `restartable()` example in §Examples is broken in multiple ways — most prominently the dead `calls.isNotEmpty ? null : null;` expression flagged in the review brief — and the test would not actually demonstrate what the prose claims. The conceptual `Completer`-based snippet earlier in the file is closer to correct but still won't compile against the rest of the skill's own type definitions. There are also two real Dart correctness bugs (a missing `Storage.close()` override, and an inconsistent `Location` constructor signature) that would cause the examples to fail `flutter analyze` on a clean run.

**Issue count:** 5 Critical / 4 Important / 4 Nits.

---

## Critical

### C1. The "restartable" test in §Examples does not test `restartable()` — and may not compile

File: SKILL.md lines 251–282 (the `test('refresh cancels the in-flight initial load', ...)` block).

Three independent defects compound:

1. **The dead `?:` expression is a no-op.** Lines 269–273:
   ```dart
   calls.isNotEmpty
       ? null
       : null; // calls list is empty; both completers are referenced above.
   // ignore: unused_element
   // (The two completers were `removeAt(0)`'d into the answer closures.)
   ```
   Both branches evaluate to `null`; the expression has no effect. The comment hand-waves about "completers were `removeAt(0)`'d into the answer closures" — true, but that means **the completers are now anonymous and unreachable**: the local `calls` list is empty, and there are no other names bound to them. Nothing in this test can ever call `.complete(...)` on them.

2. **No completer is ever resolved, so the bloc cannot reach `Loaded`.** Both handlers (initial and refresh) `await calls.removeAt(0).future`. After two `removeAt(0)` calls, both futures sit in `await` for the lifetime of the test. After `bloc.close()`, the bloc's state will still be `RestaurantListLoading` (from the second handler's pre-await `emit`). The assertion's `or(isA<RestaurantListLoaded>())` branch is therefore unreachable in practice.

3. **`isA<T>().or(...)` is not part of the `matcher` package.** Line 278:
   ```dart
   expect(bloc.state, isA<RestaurantListLoading>().or(isA<RestaurantListLoaded>()));
   ```
   `Matcher` has no `.or()` instance method. The correct API is `anyOf(isA<RestaurantListLoading>(), isA<RestaurantListLoaded>())`. This will fail at compile / analyze time on `bloc_test ^10` with current `flutter_test`. Equivalent to `Matcher` documentation: combinators live as top-level functions (`anyOf`, `allOf`), not as fluent methods on `TypeMatcher`.

The combined effect: the test proves nothing about `restartable()` (which is the whole point of the example), and one of its three bugs will fail the build outright. The opening prose section's snippet (lines 70–93) at least has named completers and calls `.complete(...)` on them — that version expresses the intended invariant, but it still has its own problems (see C2). The §Examples version reads like a refactor of the prose version that got abandoned mid-edit.

**What the test was supposed to do** (and what the §Examples version should be rewritten to do):

- Stub `repo.fetchNearbyStores` to return `firstCompleter.future` then `secondCompleter.future` (two named completers).
- Dispatch load, yield, dispatch refresh, yield.
- Complete the **first** completer **after** the second handler has started (this is the moment `restartable()` cancellation matters).
- Complete the second completer.
- Assert the recorded `emitted` list never contains `Loaded([_storeA])` and ends with `Loaded([_storeB])`.

The current §Examples version drops the named completers, drops the `.complete()` calls, drops `await bloc.close()` being meaningful, and replaces the assertion with `.or()` — i.e. all four of those load-bearing pieces are gone.

### C2. `const Result.success([_storeA])` is not a valid `const` expression

In the prose snippet at lines 84–85:
```dart
initialCompleter.complete(const Result.success([_storeA])); // ignored
refreshCompleter.complete(const Result.success([_storeB]));
```

`_storeA` and `_storeB` are declared on lines 191–192 as plain `final` (not `const`):
```dart
final _storeA = Store(id: 'a', name: 'A', deliveryMinutes: 20);
final _storeB = Store(id: 'b', name: 'B', deliveryMinutes: 30);
```

A `const` collection literal can only contain compile-time constants. `_storeA`/`_storeB` are runtime `final`s. The compiler will reject `const Result.success([_storeA])`. Drop the `const` keyword on both lines, or make `_storeA`/`_storeB` themselves `const` (which requires `Store` to have a `const` constructor — not shown in any sibling skill, so the safer fix is to drop `const`).

This same mistake **does not** appear in the §Examples blocTest bodies (lines 210, 216), which correctly use a non-const `Result.success([...])`. The inconsistency is itself a smell — the author knew once but forgot again.

### C3. `_InMemoryStorage implements Storage` is missing `close()`

Lines 103–113:
```dart
class _InMemoryStorage implements Storage {
  final _map = <String, dynamic>{};
  @override
  dynamic read(String key) => _map[key];
  @override
  Future<void> write(String key, dynamic value) async => _map[key] = value;
  @override
  Future<void> delete(String key) async => _map.remove(key);
  @override
  Future<void> clear() async => _map.clear();
}
```

`hydrated_bloc`'s `Storage` interface (since the package split off from `hydrated_bloc` v8, and unchanged at v10) declares **five** members: `read`, `write`, `delete`, `clear`, **and `close()`**. Omitting `close()` makes this `implements Storage` declaration non-exhaustive — `dart analyze` flags it as "Missing concrete implementation of `Storage.close`" and the file won't compile.

Add:
```dart
@override
Future<void> close() async {}
```

### C4. `Location` constructor signature is internally inconsistent

The skill uses two contradictory `Location` constructors in the same file:

- Line 196: `registerFallbackValue(const Location(0, 0));` — **positional**, two args.
- Line 190: `const _loc = Location(lat: 25.276987, lng: 55.296249);` — **named** args.

A Dart class can't satisfy both calls with a single `const` constructor. Either:
- `const Location(this.lat, this.lng)` — then `Location(lat: ..., lng: ...)` is a compile error.
- `const Location({required this.lat, required this.lng})` — then `Location(0, 0)` is a compile error.

`flutter-bloc-async-api` (the prereq) doesn't define `Location` explicitly; it only refers to `loc.lat`, `loc.lng`. So the testing skill is the first to pin a shape — and it pins two incompatible shapes within 6 lines of each other. Pick one (recommend named, to match the `StoreDto` constructor style elsewhere in the family) and use it everywhere.

### C5. The `await a.stream.firstWhere(...)` rehydration test races the persistence write

Lines 127–137:
```dart
test('CartBloc rehydrates after restart', () async {
  final a = CartBloc(repo: repo);
  a.add(CartItemAdded(_product, 2));
  await a.stream.firstWhere((s) => s is CartLoaded);
  await a.close();

  final b = CartBloc(repo: repo);
  expect(b.state, isA<CartLoaded>());
  expect((b.state as CartLoaded).items.length, 1);
});
```

Two related issues:

1. **`HydratedBloc.storage.write` is called from `onChange` and is `Future<void>`-returning, but `close()` does not await it.** With the simple `_InMemoryStorage` shown in C3 (synchronous mutation before the first `await`-yield), the assignment to `_map[key]` happens before the future yields, so in practice this works — but only because of an implementation detail of the fake. Spelling that out in prose (or making `_InMemoryStorage.write` truly synchronous) would make the test honest about why it works. Worse, swapping `_InMemoryStorage` for any storage that yields before persisting (a realistic SharedPreferences-style fake) silently breaks rehydration tests written to this pattern.

2. **`firstWhere` returns a `Future` that completes with the matching state, but `a.add(CartItemAdded(...))` is fire-and-forget.** The dispatch returns immediately and the handler is queued. `firstWhere` will work because it subscribes to `stream` before the handler runs (microtask ordering is on our side here). Still, the test is one accidental `await Future.microtask(() {})` away from a deadlock. Prefer the `bloc_test` `blocTest(..., act: ..., expect: ...)` pattern for this — `bloc_test` has a known subscription-before-act ordering and supports HydratedBloc identically. The hand-rolled `test(...)` here is justified only if you genuinely need two `CartBloc` instances in sequence — fine — but call that out.

Neither issue is fatal in the way C1–C4 are, but the prose claims the assertion is straightforward when it actually depends on subtle await ordering that the skill never explains.

---

## Important

### I1. `setUp(() { HydratedBloc.storage.clear(); });` is not awaited

Line 119–121:
```dart
setUp(() {
  HydratedBloc.storage.clear();
});
```

`Storage.clear()` returns `Future<void>`. The `setUp` callback here is sync — the future is created and discarded. With the in-memory fake, the body runs `_map.clear()` synchronously before yielding, so the test sees the cleared map. But:
- This is undocumented brittleness (same root cause as C5).
- The skill's own §Workflow Step 8 says "clear it in `setUp`" — readers will copy this verbatim and be confused when they swap to a real `HiveStorage` or other async-yielding fake.

Use `setUp(() async { await HydratedBloc.storage.clear(); });`.

### I2. The §Examples `restartable` test never uses `bloc_test` — inconsistency with the rest of the suite

The other three tests in §Examples (lines 206–249) are written with `blocTest<B, S>(..., setUp, build, act, expect, verify)`. The fourth test (line 251 onward) drops to raw `test(...)` with manual stream subscription. The drop is justified for fine-grained timing — `blocTest`'s `wait:` is too coarse for `Completer`-based interleaving — but the skill never explains *why* it switches styles. A reader copy-pasting will see two patterns side by side and either pick wrong, or assume `blocTest` doesn't support custom assertions.

Add one sentence at the top of the test: "We drop down to raw `test(...)` here because `blocTest`'s `wait:` parameter is a fixed `Duration` and we need to interleave `Completer.complete` calls between event dispatches."

### I3. `verify(() => repo.fetchNearbyStores(_loc)).called(1)` will fail if `Location` lacks value equality

Line 219:
```dart
verify(() => repo.fetchNearbyStores(_loc)).called(1);
```

`mocktail.verify` uses `==` to match recorded calls. If `Location` is a regular class (not `freezed`, no `==`/`hashCode` override), `_loc == _loc` is true (same reference) — but the reader hasn't been shown `Location`'s implementation. If they implement `Location` as a vanilla class, the `verify` works because it's the same const-canonicalized instance; if they later switch to a non-const `Location`, this breaks subtly.

The skill should either (a) note that `Location` must have value equality (it's a `freezed` data class — fine, but say so) or (b) use `verify(() => repo.fetchNearbyStores(any())).called(1)` for the example, which doesn't rely on equality semantics that aren't established in the prereq.

### I4. The §Workflow numbering claims 6 steps (per design doc §5.2) but ships with 9

Design doc line 248: `"Workflow: Test a Bloc (6 steps)."` — current skill has 9 (lines 142–150). Not a bug, but if `to-issues` or `feature-dev` cross-references the design doc step count, that drifts. Either update the design doc or trim the workflow.

The current 9 steps are mostly good. Step 9 (the "feedback loop") is more of a debugging hint than a discrete workflow step and could be folded into Step 8's body without losing information.

---

## Nits

### N1. `dynamic` in the in-memory `Storage` fake

Lines 105, 107:
```dart
dynamic read(String key) => _map[key];
Future<void> write(String key, dynamic value) async => _map[key] = value;
```

This matches the `Storage` interface exactly (which uses `dynamic`), so it's correct — but readers in a `strict-casts: true` project will get a lint. Worth a one-line comment: `// dynamic matches Storage's signature; real fakes can tighten it.`

### N2. Comment on line 84 says "ignored" — under-explained

```dart
initialCompleter.complete(const Result.success([_storeA])); // ignored
```

A reader unfamiliar with `restartable()` won't know whether "ignored" means "the future never resolves" (false), "the await throws" (false), or "the emit is a no-op because `emit.isDone` is true" (true). Expand to: `// the second event already cancelled this handler; its emit is now a no-op.`

### N3. The "PRD section" anchors are correct but unverifiable from the skill alone

- `PRD_states.md §7` (Cart) — exists, line 169. The skill's description ("cart abandon on store-switch", "rehydrate after restart") matches the PRD's `active → abandoned` transition and the cart-as-`HydratedBloc` design decision.
- `PRD_catalog.md §15.5` — exists as `### ١٥.٥ قواعد السلّة` (Arabic numbering "15.5"). Confirms one-cart/one-store/one-city. The PRD also specifies the **exact confirmation dialog text** and the `409 Conflict` server response — the skill could mention these as testable boundaries (e.g., "and a server-rejected 422/409 should not move the cart out of `active`"). Not a defect; an opportunity.

PRD alignment passes.

### N4. The frontmatter `description:` is well within trigger-only norms, but the prerequisite clause is borderline

```yaml
description: Writes deterministic unit tests for Blocs using `bloc_test` and `mocktail`, including tests for `bloc_concurrency` transformers and `HydratedBloc` persistence. Use when adding tests for a Bloc's event handlers, verifying its emitted state sequence, mocking a Repository dependency, or asserting that `restartable()` cancels in-flight requests. Prerequisite: `flutter-bloc-setup`, `flutter-bloc-feature-pattern`, and `flutter-bloc-async-api`.
```

Two sentences of triggers, one sentence of prerequisites. The other Wave-2/3 skills do the same. Consistent — leave as-is. Just flagging that the "Prerequisite" sentence is dead weight at trigger-match time (the description-matcher will rank on the trigger sentence anyway). If you ever tighten the family for trigger precision, this is one of the cuts.

---

## Done well

- **Hook table (lines 24–34)** is clear, exhaustive, and matches `bloc_test ^10` exactly. `build`, `setUp`, `seed`, `act`, `wait`, `expect`, `verify` are the seven hooks, named correctly.
- **The "naive test passes anyway" framing in §Testing concurrency transformers (lines 64–68)** is the right pedagogical opener — it tells the reader *why* `restartable()` needs a Completer-based test instead of just two `bloc.add` calls. This is the single most-skipped insight in Bloc tutorials and the skill calls it out clearly.
- **`registerFallbackValue` callout (lines 53–61)** correctly anticipates the `StateError` first-time users hit with mocktail+`any()` on custom types. Including the `setUpAll` pattern instead of `setUp` is correct — `registerFallbackValue` is idempotent but cheaper to do once.
- **Applied-to-Talabat-clone section (lines 152–164)** cites two real PRD anchors and ties them to specific test scenarios. Both anchors verify against the PRD source. The "**only safety net** between the PRD rules and a future refactor" sentence is the kind of motivation that makes a skill stick.
- **Step 9's debugging heuristic** ("if a test hangs, the Bloc is probably awaiting a future that never completes") is exactly the failure mode caused by C1. Ironic, but the heuristic itself is correct and well-placed.
- **Mocktail `class MockX extends Mock implements X {}` pattern (line 188)** is the canonical mocktail boilerplate. No code generation, no fallback for non-nullable returns needed for `Future<Result<...>>` (since `Result` is nullable-free). Right idiom.
- **Step 4's emphasis on `setUpAll` for fallbacks vs `setUp` for per-test mocks** (lines 145–146) prevents the most common mocktail footgun (forgetting `registerFallbackValue`).

---

## Suggested fix order

1. Rewrite the `restartable` test in §Examples to actually test `restartable` (C1). This is the most visible bug and the one a reader will hit first.
2. Add `Future<void> close() async {}` to `_InMemoryStorage` (C3).
3. Pick one `Location` constructor signature and use it consistently (C4).
4. Drop `const` from `Result.success([_storeA])` / `Result.success([_storeB])` in the prose snippet (C2).
5. Make `setUp(...)` in HydratedBloc section `async`/`await` (I1) and add a sentence about why the in-memory write-before-yield matters (C5).
6. Add the one-line justification for dropping out of `blocTest` (I2).
7. Either generalize `verify` to `any()` or note the equality requirement for `Location` (I3).

C1, C3, C4 are blocking. C2, C5, I1–I4, N1–N4 are quality-of-revision items.
