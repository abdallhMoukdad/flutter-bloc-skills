# Review — flutter-bloc-setup

**Reviewer:** Opus (independent)
**Date:** 2026-05-14
**Skill:** `~/.claude/skills/flutter-bloc-setup/SKILL.md`

## Verdict: APPROVE WITH FIXES

The skill is technically sound and follows the family design. The Dart bootstrap code compiles against the pinned `^9.0 / ^10.0 / ^3.0` stack. Two PRD citations are slightly off, one structural gap will bite consumers of `flutter-bloc-async-api`, and one small `const` issue could trip dart-analyzer in strict-mode projects. None are catastrophic.

## Critical issues

None. A user following the workflow on a clean `flutter create` project will end up with a runnable app whose `HydratedBloc` instances persist correctly.

## Important issues

**I1. `MultiRepositoryProvider(providers: const [])` may break under strict inference (lines 146–149).**
`MultiRepositoryProvider.providers` is typed `List<SingleChildWidget>`. An empty `const []` infers in most setups but raises `inference_failure_on_collection_literal` under `strict-inference: true` / `always_specify_types`. Safer: `providers: const <RepositoryProvider>[],` — which also primes the insertion site for `flutter-bloc-async-api`.

**I2. The skill never shows what a populated `MultiRepositoryProvider` entry looks like (lines 47, 91, 110, 147–149).**
Multiple references promise that `async-api` will append `RepositoryProvider`s here, but no entry shape is shown. A 4-line comment example (`// RepositoryProvider<AuthRepository>(create: _, lazy: true),`) would close the cross-skill hand-off gap that design doc §3.4 calls out.

**I3. `PRD_states.md §8–§10` for `PromotionsBloc` conflates three entities (line 108).**
§8 = Coupon, §9 = Promotion/Offer, §10 = Advertisement — three distinct features. Either narrow the cite to §9 (and the row to `PromotionBloc`), or rename to `OffersBloc` and footnote the siblings.

**I4. `AuthBloc` ↔ `PRD_states.md §14` is a stretch (line 102).**
§14 describes user account lifecycle (pending_verification/active/suspended/banned/deleted), not token persistence. Either soften the cite to "see §14 for user lifecycle" or cite the actual session/token source in `prd.md`.

## Nits

- **N1.** `AppBlocObserver` (lines 167–180) has no fields — add `const AppBlocObserver();` so wiring can say `Bloc.observer = const AppBlocObserver();`.
- **N2.** Overriding `onTransition` instead of `onChange` (lines 168–172) would log `event + currentState → nextState` for Blocs, more useful than `onChange`'s state-diff-only output. Family is Bloc-only (no Cubits), so `onTransition` fits better.
- **N3.** Workflow Step 5 (line 90) says "in dev/debug" — be concrete and say `kDebugMode` directly, to match the example (line 134).
- **N4.** Mobile-only scope appears twice with slight variation (line 11 vs line 41) — cosmetic.
- **N5.** The `flutter pub add` lines duplicate verbatim between "Dependencies and generators" (lines 26, 32) and workflow steps 1–2 (lines 86, 87).
- **N6.** "Feedback loop" (line 94) is listed as a `- [ ]` checkbox under "Task Progress" but reads as a conditional tree, not an action. Move to a Troubleshooting subsection or split into concrete steps.
- **N7.** `home: const SizedBox.shrink()` (line 152) — a `Scaffold(body: Center(child: Text('Hello')))` would make the smoke-test more visible. Cosmetic.

## Done well

1. **`HydratedBloc.storage` init order is explicit and correct** (lines 50, 89, 128–132). The example uses the exact `HydratedStorageDirectory((await getApplicationDocumentsDirectory()).path)` form required by `hydrated_bloc ^10.0` — the API changed from `Directory` in ^9.x to `HydratedStorageDirectory` in ^10.x, easy to get wrong, the skill gets it right.
2. **Cross-skill hand-offs are signalled** (lines 41, 62–63, 107). HTTP entitlements deferred to `flutter-use-http-package`; `failure.dart`/`result.dart` flagged as added by `async-api`; `OrderTrackingBloc` deferred to `stream-tracking`. Right shape for a foundation skill.
3. **Project structure integrates with the prior architecture skill cleanly** (lines 52–79). Explicitly notes `view_models/` is replaced by `bloc/` while Repository/Service/Domain layers stay — prevents two architecture skills fighting in one repo.

## Verification footnotes

- PRD `§7 Cart`: exists, matches claim (one-cart / one-store / one-city, abandon on store/city switch), correctly cross-cites `PRD_catalog.md §15.5` (verified: "قواعد السلّة" / cart rules exists at that section).
- PRD `§4 Store operational gating`: exists, matches claim (working-hours + merchant-pause + admin-suspension three-gate rule).
- PRD `§14 User account`: exists but describes account-status FSM, not token persistence — see I4.
- PRD `§8`/`§9`/`§10`: all exist as separate Coupon/Promotion/Advertisement sections — see I3.
- Dart APIs (`HydratedBloc.storage` setter, `HydratedStorage.build(storageDirectory:)`, `HydratedStorageDirectory(String)`, `BlocObserver.onChange(BlocBase, Change)`, `BlocObserver.onError(BlocBase, Object, StackTrace)`): all match `^9.0 / ^10.0` API surface as pinned at lines 188–193.
- Frontmatter: valid YAML, `name` matches directory, `description` is clear "Use when…" trigger that names the chained family skills.
- Section structure matches design doc §4.1 (Deps → Providers → Structure → Workflow → Applied → Examples).
