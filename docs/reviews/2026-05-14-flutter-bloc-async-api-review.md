# Review — flutter-bloc-async-api

**Reviewer:** Opus (independent)
**Date:** 2026-05-14
**Skill:** `~/.claude/skills/flutter-bloc-async-api/SKILL.md` (376 lines)

## Verdict: APPROVE WITH REQUIRED FIXES

The skill correctly captures the canonical pattern (Repository wraps, Bloc pattern-matches, `restartable()` for refresh) and integrates with the rest of the family (forms consumes `ValidationFailure`, testing mocks `StoreRepository`). However there are Dart correctness bugs and a self-inconsistency between the prose contract (`Result<T, Failure>`) and the actual `freezed` definition (`Result<T>`, one generic) that will burn a beginner. The Laravel-422 cast is unsound at runtime, and the example code never actually demonstrates the 422 parse the skill's headline promises.

**Count: 3 Critical, 5 Important, 3 Nits.**

**Top two fixes:**
1. Reconcile `Result<T>` vs `Result<T, Failure>`. The code defines one generic; the prose advertises two. Pick one and apply it across description, prose, workflow, and signatures.
2. Fix `const Failure.validation({})` (line 278) — `{}` does not satisfy `Map<String, List<String>>` and won't compile. And fix the unsound `as Map<String, List<String>>` cast (line 80) — `jsonDecode` returns `Map<String, dynamic>`, so the cast throws at runtime.

## Critical

### C1. `Result<T>` vs `Result<T, Failure>` — public contract contradicts code

- Line 3 (description): "`Result<T, Failure>`"
- Line 11 (overview): "`Future<Result<T, Failure>>`"
- Line 24 (Repository contract): "`Future<Result<T, Failure>>`"
- Line 36 (code): `sealed class Result<T> with _$Result<T>` — **one generic**
- Line 137 (workflow step 5): "Returns `Result<T>`"
- Line 250 (Repository impl): `Future<Result<List<Store>>>`

`flutter-bloc-testing` (lines 71-73, 252-254) uses `Result<List<Store>>` — one generic. So the code is right and the prose is wrong: `Failure` is hard-coded into the `failure` factory; it's not a parameter. **Fix:** Replace every "Result<T, Failure>" with "Result<T>". Optionally add: "Failure is fixed; the second generic would be unused, so we omit it."

### C2. `Failure.validation({})` (line 278) is not valid Dart

```dart
422 => const Failure.validation({}),
```

`{}` is inferred as `Set<dynamic>` or `Map<dynamic, dynamic>`, neither of which satisfies `Map<String, List<String>>`. Won't compile. **Fix:** `Failure.validation(const <String, List<String>>{})`.

### C3. The 422 cast (line 80) throws at runtime, and the example never demonstrates the parse

```dart
Failure.validation(json['errors'] as Map<String, List<String>>)
```

`jsonDecode` returns `Map<String, dynamic>` with `List<dynamic>` values. A direct cast throws `TypeError`. **Fix:**
```dart
final raw = (json['errors'] as Map<String, dynamic>);
final errors = raw.map((k, v) => MapEntry(k, (v as List).cast<String>()));
return Failure.validation(errors);
```

Compounding: the actual repository code (lines 271-282) returns `Failure.validation(const {})` for 422 with the comment "populated by full implementation" and **never parses the response body**. The skill's headline promise — "backend errors round-trip cleanly into the UI" (line 11) — is not demonstrated end-to-end. A beginner copying the example gets empty validation maps and no idea where the parsing was supposed to happen. Biggest substance gap.

## Important

### I1. `_mapHttpFailure` regex-parses `http.ClientException.message` — fragile

Service throws `http.ClientException('HTTP ${res.statusCode}: ${res.body}', uri)` (line 228); Repository parses it back with `RegExp(r'HTTP (\d+)')` (line 272). This is producer/consumer code in the same project — no reason to encode then re-decode through a string. If anyone tweaks the throw site, the regex silently fails and every error becomes `Failure.unknown`.

Also: `http.ClientException` is the wrong exception type. Its package docs describe it as transport-layer failures, not HTTP status errors. Using it for a 500 muddles the `on SocketException` / `on http.ClientException` separation. **Fix:** Define a local `ApiException` with typed `statusCode` and `body` fields. Or have the Service return a `(int, String)` record. At minimum flag this with "for brevity; use a proper exception class in real code."

### I2. The `data` envelope assumption is project-specific (lines 222-223)

`(body['data'] as List)` assumes Laravel's `JsonResource::collection(...)` wrapping. A hand-rolled `return response()->json($stores)` returns an unwrapped array. Beginner copying this against the wrong controller gets a confusing runtime error. Add one sentence: "This assumes a Laravel `AnonymousResourceCollection` response shape."

### I3. The view references state classes that aren't defined (lines 360-372)

`RestaurantListInitial/Loading/Empty/Loaded/Failure` are switched on in the view, but the `RestaurantListState` sealed union is never shown. A reader who landed here from search (without reading `flutter-bloc-feature-pattern` first) can't reproduce this. Add a stub or a `// see flutter-bloc-feature-pattern` cross-reference.

### I4. `_messageFor` is duplicated (lines 103-113 and 332-341)

Same function, slightly different strings ("Nothing found for this location" vs "No restaurants near this location"). Reads as two near-identical 10-line blocks without explanation. Drop one or frame the second as "the Talabat-clone copy."

### I5. `UnknownFailure(:final message)` is rendered verbatim to the user (lines 64, 111, 340)

The catch-all fallback surfaces `e.toString()` (e.g. a `FormatException`) as a snackbar — contradicts the "typed failures" promise. Add a sentence: "`UnknownFailure.message` is a log line, not user-facing copy. Render a generic fallback in production."

## Nits

### N1. Design-doc drift: `RestaurantRepository` (design §4.3) vs `StoreRepository` (skill)

Family is internally consistent (`flutter-bloc-testing` line 188 and `flutter-bloc-setup` line 110 also say `StoreRepository`); the design doc is stale.

### N2. `prd.md §2.1` reference (line 148) — verified

§2.1 describes saved-address selection. The "far address" guard is actually §2.2 (`التحقق من بُعد العنوان`), but the skill paraphrases rather than asserting a section number for the guard.

### N3. `PRD_states.md §4` reference (line 150) — fully verified

§4 defines exactly `paused_by_merchant = false`, `suspension_reason IS NULL`, within working hours. Skill's three filter predicates match verbatim. Skill also correctly notes working hours are "computed at read time, not a stored state" — matches the PRD's load-bearing convention (line 13).

## Done well

- **`restartable()` rationale (line 128).** Correct. `droppable()` would discard the new event, keeping the in-flight handler — so a pull-to-refresh on top of a slow initial load would be silently dropped.
- **"Repository never throws" stance.** Sound and well-motivated. The example **does** uphold it — every code path in `fetchNearbyStores` (lines 251-269) returns `Result.success` or `Result.failure`; four `catch` blocks cover `SocketException`, `TimeoutException`, `http.ClientException`, and a final `catch (e)`.
- **Laravel 422 envelope shape (lines 70-78).** Matches the actual Laravel response; map-of-list-of-string is conceptually correct (Laravel wraps even single-message errors in arrays).
- **Cross-family handoff to `flutter-bloc-forms`** (line 80, 171). Forms skill (lines 118, 301) destructures `ResultFailure(failure: ValidationFailure(:final errors))` exactly as advertised.
- **401 handling via `BlocListener<AuthBloc>` higher in the tree (line 115).** Right pattern.
- **Pattern destructuring** (lines 93-96, 323-329). Valid Dart 3 syntax against `@freezed sealed class` — freezed 3.x emits the named constructor classes (`ResultSuccess<T>`, `ResultFailure<T>`), so the destructure works.
- **Frontmatter and trigger surface.** YAML parses, name matches directory, description is triggers-only.

## Cross-family consistency

| Item | This skill | Family | Status |
|---|---|---|---|
| `Result<T>` signature (code) | 1 generic | testing uses `Result<List<Store>>` | Consistent |
| Repository name | `StoreRepository` | setup line 110, testing line 188 | Consistent |
| `ValidationFailure.errors` shape | `Map<String, List<String>>` | forms destructures `:final errors` | Consistent |
| `RepositoryProvider` registration | "root `MultiRepositoryProvider`" | setup line 47 defines that root | Consistent |
| Description says `Result<T, Failure>` | Yes | Code says `Result<T>` | **Inconsistent (C1)** |

## Recommended action

Block on **C1**, **C2**, **C3** — self-contradicting public contract, won't-compile literal, headline promise not demonstrated. Importants are follow-up. The skill is otherwise structurally sound and well-grounded in the project's PRDs.
