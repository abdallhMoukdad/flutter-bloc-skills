# Flutter-Bloc Skills Family — Review Synthesis

**Date:** 2026-05-14
**Reviewers:** 6 independent Opus agents, one per skill
**Reviews:** `~/.claude/skills/_design/reviews/2026-05-14-flutter-bloc-*-review.md`

## Aggregate verdict

**Family-level: SHIP AFTER FIXES.** Conceptual content is sound across all six skills; PRD references are mostly accurate (some need narrowing); cross-skill consistency is good. The damage is concentrated in the **runnable code examples**: 14 critical issues across the family that would break a user who copy-pastes verbatim.

| Skill | Verdict | Critical | Important | Nits |
|---|---|---:|---:|---:|
| `flutter-bloc-setup` | APPROVE WITH FIXES | 0 | 4 | 7 |
| `flutter-bloc-feature-pattern` | APPROVE WITH FIXES | 1 | 4 | 5 |
| `flutter-bloc-async-api` | APPROVE WITH REQUIRED FIXES | 3 | 5 | 3 |
| `flutter-bloc-stream-tracking` | SHIP WITH ONE FIX | 1 | 3 | 4 |
| `flutter-bloc-testing` | NEEDS REVISION | 5 | 4 | 4 |
| `flutter-bloc-forms` | REQUEST CHANGES | 4 | 4 | 3 |
| **total** | | **14** | **24** | **26** |

## Cross-cutting themes

1. **`Result<T>` vs `Result<T, Failure>` mismatch.** `async-api` defines one generic in code but advertises two in prose at multiple sites. Fix in `async-api`; verify `forms` and `testing` consume the correct shape.
2. **Incomplete `freezed` JSON wiring.** `feature-pattern` and `forms` both promise `fromJson`/`toJson` come "for free" but omit the `part 'x.g.dart';` and the explicit `factory X.fromJson(...) => _$XFromJson(json);` declaration that `freezed` actually requires.
3. **Example rigor degrades with skill length.** The longest skills (`testing` 287, `async-api` 376, `forms` 434) have the most compile errors. Shorter skills (`setup` 202) scored cleanest.
4. **PRD citation precision.** Multiple skills cite PRD section ranges (`§8-§10`, `§14`, `§2.1`) that are wider, narrower, or slightly off from what they claim. Tighten citations; this is the easiest class of fix.
5. **Aspirational vs runnable code.** Several examples mix pseudo-API with real code (e.g., forms' ghost `withServerError(...)` method that's referenced but never defined; async-api's `// populated by full implementation` placeholder for the 422 parse). Decide per-example: runnable, or clearly labelled pseudocode.

## Prioritized fix list (P0 / P1 / P2)

### P0 — won't compile or won't run as written

These will burn a user immediately.

| # | Skill | Issue | Source |
|---|---|---|---|
| 1 | `async-api` | `const Failure.validation({})` at line 278 — empty literal doesn't satisfy `Map<String, List<String>>`. Fix: `Failure.validation(const <String, List<String>>{})`. | C2 |
| 2 | `async-api` | 422 cast `json['errors'] as Map<String, List<String>>` throws at runtime — `jsonDecode` returns `Map<String, dynamic>`. Use `.map((k, v) => MapEntry(k, (v as List).cast<String>()))`. | C3 |
| 3 | `async-api` | `Result<T>` (code) vs `Result<T, Failure>` (prose) contradiction across 6+ lines. Unify on `Result<T>`. | C1 |
| 4 | `feature-pattern` | `CartState` missing `part 'cart_state.g.dart';` and `factory CartState.fromJson(...) => _$CartStateFromJson(json);`. The "JSON comes for free" claim is wrong without these. | C1 |
| 5 | `testing` | The `restartable()` test example has 3 bugs at once: dead `calls.isNotEmpty ? null : null` code, completers never get `.complete()` called, fake `isA<>().or(...)` matcher API (should be `anyOf(...)`). Test won't compile and proves nothing. Full rewrite required. | C1 |
| 6 | `testing` | `_InMemoryStorage` missing `close()` method that `hydrated_bloc ^10`'s `Storage` interface requires (5 members, skill implements 4). | C2 |
| 7 | `testing` | `Location` constructed two contradictory ways in same file: `Location(0,0)` positional vs `Location(lat:, lng:)` named. Pick one. | C3 |
| 8 | `stream-tracking` | `_onDisconnect` doesn't actually disconnect. Each event handler gets its own `_Emitter` — emitting `Closed` in one handler cannot mark another handler's emitter as done. Comment claiming otherwise is false. Route disconnection through the Repository (or unify both events under one `restartable()` handler). | C1 |
| 9 | `forms` | `EmailInput.pure()` / `PasswordInput.pure()` don't initialize the new `final String? serverError` field — hard compile error against formz 0.7. | C1 |
| 10 | `forms` | `withServerError(...)` method referenced at 3 sites but defined nowhere. Skill presents 3 contradictory mechanisms (vestigial extension, ghost method, real constructor param). Delete the first two. | C2 |

### P1 — misleading or substantively wrong, but won't break compile

| # | Skill | Issue |
|---|---|---|
| 11 | `async-api` | `_mapHttpFailure` regex-parses `http.ClientException.message` — fragile producer/consumer coupling. Define a proper `ApiException` with typed `statusCode`/`body`. |
| 12 | `async-api` | Example never demonstrates 422 body parsing — returns `const {}` with `// populated by full implementation` comment. Headline promise of the skill ("backend errors round-trip cleanly") not demonstrated end-to-end. |
| 13 | `async-api` | `(body['data'] as List)` assumes Laravel's `JsonResource::collection()` wrapping — flag this assumption explicitly. |
| 14 | `feature-pattern` | The only `HydratedBloc` example elides the constructor body as `/* ... */`, so reader never sees transformer registrations + persistence together (the exact combination the Limitations subsection warns about). |
| 15 | `stream-tracking` | `subscribeToDriver` busy-loops forever on persistent transient errors — no max-attempt cap, `attempt` never resets after success. |
| 16 | `stream-tracking` | Skill defines `Reconnecting` state but the example Repository never produces it (sleeps silently instead of yielding a reconnecting sentinel). |
| 17 | `forms` | `BlocSelector<LoginFormBloc, LoginFormState, EmailInput>` may skip rebuilds when only `serverError` changes — FormzInput's `==` doesn't include the extra field. Either custom `==`/`hashCode` or select on a tuple. |
| 18 | `setup` | `PRD_states.md §8-§10` reference conflates Coupon/Promotion/Advertisement (three distinct entities). Narrow to §9 for Promotion. |
| 19 | `setup` | `AuthBloc ↔ §14` is a stretch — §14 is user account lifecycle, not token persistence. |
| 20 | `forms` | `prd.md §2.1` doesn't actually describe address-entry fields (those are §27); "building" isn't in the PRD at all. |

### P2 — nits / style

26 nits across the family. Examples: duplicate `_messageFor` in async-api (~10 lines), `const AppBlocObserver();` in setup, prefer `onTransition` over `onChange` in BlocObserver, "feedback loop" workflow step being a tree not an action, etc. Worth a pass if doing a polish round; not blocking.

## Cross-family consistency check (✓ = consistent)

| Item | Status |
|---|---|
| `Result<T>` code signature across skills | ✓ (testing matches; only async-api prose drifts) |
| `Failure.validation(Map<String, List<String>>)` shape | ✓ (forms consumes exactly this shape) |
| `StoreRepository` name | ✓ (across setup, async-api, testing) |
| `MultiRepositoryProvider` at root | ✓ (setup defines, async-api consumes) |
| Event/state/bloc trio file naming | ✓ |
| `BlocBuilder`/`Selector`/`Listener` distinctions | ✓ |
| Pattern destructuring against `@freezed sealed class` | ✓ (valid Dart 3 + freezed 3.x) |
| `restartable()` for queries/refresh | ✓ |

The family is internally coherent. The bugs are in the *individual* examples, not in the contracts between skills.

## Recommended action

1. **Do not use the example code as-is in real Flutter work.** Fix P0 items 1-10 first. Each is a small targeted edit.
2. **Then optionally tackle P1.** These won't break a user's compile but will mislead them about idiomatic patterns.
3. **P2 is polish.** Defer unless doing a dedicated cleanup pass.
4. **The conceptual content (workflows, decision tables, prose explanations) needs no changes.** The skills as guidance are accurate and useful; it's the runnable examples that need rigor.

## What worked

- **Family architecture is sound.** Cross-references resolve, naming is consistent, the design doc's vision is faithfully implemented.
- **`emit.forEach` semantics, `bloc_concurrency` transformer rationale, `restartable()` for refresh, `HydratedBloc.storage` init ordering, freezed sealed switch destructuring** — all the high-level API claims hold up against the actual library docs.
- **PRD anchoring** — the Talabat-clone references are *substantively* right (rules, behaviors), even where section numbers need narrowing.
- **The skill format itself works.** Reviewers had no trouble parsing frontmatter, sections, workflows, or examples.

## Files

All six per-skill reviews and this synthesis live in `~/.claude/skills/_design/reviews/`. They are read-only artifacts; if you choose to apply fixes, the fixes go to the live SKILL.md files in `~/.claude/skills/flutter-bloc-*/`.
