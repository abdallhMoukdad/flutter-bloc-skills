# Review: `flutter-bloc-feature-pattern`

**Reviewer:** Opus (independent)
**Date:** 2026-05-14
**File:** `/home/abdallh/.claude/skills/flutter-bloc-feature-pattern/SKILL.md`
**Design doc:** `/home/abdallh/.claude/skills/_design/2026-05-09-flutter-bloc-skills-family-design.md` §4.2

---

## Verdict: **APPROVE WITH FIXES**

The skill teaches the right pattern (Event/State/Bloc trio, sealed states with exhaustive `switch`, explicit transformers per handler) and the concurrency guidance is technically accurate. However, the JSON-serialization example for `HydratedBloc` is broken as written — a user who copies it gets a compile error. That, plus a frontmatter trigger violation and an elided constructor in the only `HydratedBloc` example, are fixable but worth blocking on before shipping.

---

## Critical issues (would break user's code if followed)

### C1 — `CartState` is missing the `fromJson` factory and `part '*.g.dart'` directive; the persistence example then calls a method that does not exist.

**Where:** lines 39–58 (CartState definition), lines 117 ("come for free"), lines 119–129 (HydratedBloc example).

The state is declared as:

```dart
// lines 41-58
import 'package:freezed_annotation/freezed_annotation.dart';
import 'package:my_app/domain/models/cart_item.dart';
import 'package:my_app/domain/models/money.dart';

part 'cart_state.freezed.dart';   // ← only freezed part; no .g.dart part

@freezed
sealed class CartState with _$CartState {
  const factory CartState.initial() = CartInitial;
  const factory CartState.loading() = CartLoading;
  const factory CartState.loaded({...}) = CartLoaded;
  const factory CartState.failure({required String message}) = CartFailure;
  // ← no `factory CartState.fromJson(...) => _$CartStateFromJson(json);`
}
```

Then line 117 claims:

> "Implement `fromJson` and `toJson` on the state. With `freezed`, both come for free if every state field is JSON-serializable."

This is wrong. Per the official freezed README (verified 2026-05-14, pub.dev/packages/freezed): `fromJson` is **only** generated when the user explicitly writes the factory inside the class body:

```dart
factory CartState.fromJson(Map<String, dynamic> json) => _$CartStateFromJson(json);
```

…and `part 'cart_state.g.dart';` must also be declared so `json_serializable` emits `_$CartStateFromJson`. Without both, line 125 (`CartState.fromJson(json)`) refers to a method that does not exist and the file does not compile.

The skill should:
1. Add `part 'cart_state.g.dart';` next to the freezed part directive.
2. Add the explicit `fromJson` factory inside `CartState`.
3. Soften the "come for free" wording — call out that the factory is required.
4. Mention that `CartItem` and `Money` (referenced in `CartLoaded`) must each be JSON-serializable freezed classes for the union to round-trip.

Also note: for sealed unions, freezed serializes the variant via a discriminator key (defaults to `runtimeType`). If the user wants stable JSON across class renames they should add `@Freezed(unionKey: 'type')`. Worth a one-line callout in the persistence section.

---

## Important issues (would mislead but not break)

### I1 — Description frontmatter leads with a workflow summary, not a "Use when…" trigger.

**Where:** line 3.

```
description: Scaffolds a single feature using the canonical event/state/bloc trio with `freezed` sealed states and `bloc_concurrency` transformers. Use when adding a new feature to a Bloc-based Flutter project (after `flutter-bloc-setup`), structuring an existing feature, or deciding which event handler concurrency strategy fits a use case.
```

The task brief and the family design §3.2 specify "Use when..." triggers only. The leading clause ("Scaffolds a single feature using...") is a workflow summary that should be moved into the body. Suggested rewrite:

```
description: Use when adding a new feature to a Bloc-based Flutter project (after `flutter-bloc-setup`), structuring an existing feature with a canonical event/state/bloc trio, or deciding which `bloc_concurrency` transformer fits an event handler.
```

This matters because skill-selection ranks on description-match, and a workflow-summary lead dilutes the trigger surface.

### I2 — The only `HydratedBloc` example elides its constructor body, hiding the very pattern the skill is teaching.

**Where:** lines 119–130.

```dart
class CartBloc extends HydratedBloc<CartEvent, CartState> {
  CartBloc({required CartRepository repo}) : super(const CartState.initial()) { /* ... */ }
  ...
}
```

A reader who lands on the persistence section first sees no `on<CartItemAdded>(_onItemAdded, transformer: sequential())` registrations inside a `HydratedBloc`. The earlier `CartBloc` snippet (lines 85–96) does show transformers but extends a non-persisting Bloc visually (line 85 says `HydratedBloc` but the comment about concurrent defaults follows lines 99 ignoring persistence concerns). Two issues:

- The pattern that survives in the reader's head is "either persistence or transformers" — they should appear together.
- The Limitations subsection (lines 132–134) refers to "a Bloc whose handler uses `restartable()` or `concurrent()`" with persistence, but no example shows that combination, so the warning has no anchor.

Recommendation: collapse the two `CartBloc` snippets into one example that extends `HydratedBloc`, registers a `sequential()` handler and a `droppable()` handler, AND overrides `fromJson`/`toJson`. Then the Limitations subsection has something to point at.

### I3 — Counter example uses `sealed class` for a single-variant state without justification.

**Where:** lines 204–208.

```dart
@freezed
sealed class CounterState with _$CounterState {
  const factory CounterState({required int value}) = _CounterState;
}
```

freezed 3.x docs are mixed: the pub.dev/packages/freezed README shows `sealed` for single-factory classes; multiple third-party guides (e.g., zenn.dev/dj_kusuha 2025-04-23) prescribe `abstract` for single-factory and `sealed` for unions. Both compile. Using `sealed` here is technically valid but inconsistent with the rest of the skill, which uses `sealed` to motivate exhaustive `switch` over multiple variants (lines 37, 60–69). For a one-variant state there's no exhaustive `switch` benefit — `abstract` would model intent better, or a comment should explain "we use `sealed` uniformly across the codebase even for single-variant states." Pick one and be explicit.

### I4 — Limitations subsection claim about `restartable()`/`concurrent()` + persistence is half right.

**Where:** lines 132–134.

> "The persisted state reflects the last **emitted** state, not the last **dispatched** event. If a `restartable` request is mid-flight when the app is killed, that request is cancelled — the persisted state is whatever was emitted before."

The first sentence is accurate (HydratedBloc persists on every `emit`, not on event dispatch — see hydrated_bloc source `onChange` override). The second sentence misuses "cancelled" — when the app process is killed, the handler isn't "cancelled" by `restartable`, it's just lost with the process. The subtle ordering problem `restartable` actually causes for persistence is something else: if event A starts a request, then event B fires and `restartable` cancels A's emit, the persisted state is B's emission — but if the app is killed while A is still in-flight (before B), the persisted state is whatever was emitted before A even started. The current wording conflates "request cancellation" with "process death." Worth a one-line tightening.

---

## Nits

### N1 — `last_modified` frontmatter is `Sat, 09 May 2026 00:00:00 GMT` (line 6); today is 2026-05-14. Bump when fixing the issues above.

### N2 — Line 99: "Sequential is **not** the default" — good warning, but it reads as if the author corrected an earlier draft. Reorder so the default-is-concurrent fact appears once, near the table (lines 77–82), and the warning flows naturally rather than as a postscript.

### N3 — Table at lines 105–111 has a `BlocConsumer` row labeled "Rare" — fine, but the prose never explains when "rare" applies. Either drop the row or add one line.

### N4 — Line 144 references `flutter-bloc-setup`'s `watch -d` side terminal; that file uses `watch -d` too — verify they stay in sync.

### N5 — "Applied to Talabat-clone" sketch (lines 154–172) is in pseudo-Dart (no semicolons, no braces). Fine as a sketch, but a comment header `// pseudocode — see Examples for runnable form` would prevent a reader from copying it.

---

## Done well

1. **Concurrency transformer semantics are correct, including the "reorders" claim.** Lines 75–99: per the official `concurrent()` docstring on pub.dev/documentation/bloc_concurrency, "states may be emitted in an order that does not match the order in which the corresponding events were added." The skill's framing — "adding then removing the same item could leave the item in the cart if the remove handler finishes first" — is a clean, project-anchored articulation of that subtlety. Many tutorials get this wrong.

2. **PRD anchors are real and correct.** Lines 152, 170, 174, 176 reference `PRD_states.md §7` and `PRD_catalog.md §15.5`. Both sections exist; `PRD_states.md §7` covers Cart states (`active → converted` / `active → abandoned`) and explicitly references the one-cart/one-store/one-city rule in `PRD_catalog.md §15.5`. The `CartStoreSwitchPending` variant in the sketch maps cleanly to the §15.5 confirm dialog flow.

3. **Widget-selection table (lines 103–113) hits the right level of detail.** The "Common mistake" callout naming `BlocBuilder` over-rebuild is the kind of project-shaped advice that justifies a skill over a docs page.

---

## Family consistency notes (informational, not blockers)

- The skill's sealed `CartState` with `Loaded`/`Failure` variants lines up with what `flutter-bloc-async-api` §4.3 calls a "feature Bloc" (§4.3 specifies `Loaded`, `LoadFailure` — the naming differs slightly: `CartFailure` here vs `LoadFailure` in async-api). Worth aligning naming across the family in a follow-up pass; not a defect in this skill alone.
- `bloc_concurrency` transformer use here matches the `restartable` refresh pattern that `flutter-bloc-async-api` will assume. No conflict.

---

## Suggested fix order

1. **C1** — fix `CartState` JSON wiring (highest impact; user copy-paste fails).
2. **I2** — give the persistence section a single, complete `CartBloc` example that includes transformers + `fromJson`/`toJson`.
3. **I1** — rewrite description to lead with "Use when…".
4. **I3, I4** — tighten.
5. Nits as cleanup.
