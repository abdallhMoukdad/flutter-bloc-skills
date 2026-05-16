# Flutter Bloc Skills Family — Design

**Date:** 2026-05-09
**Author:** Claude (brainstormed with abdallh)
**Status:** Approved, awaiting implementation
**Implementation plan:** TBD via `/superpowers:writing-plans`
**Target consumer:** a Talabat-clone Flutter app paired with a Laravel 12 + PostgreSQL backend; mobile only (Android + iOS).

---

## 1. Summary

A family of six narrowly-scoped skills that teach Bloc state management for Flutter, opinionated toward a Talabat-clone domain (food-delivery / multi-store).

| # | Skill | Wave | Status |
|---|---|---|---|
| 1 | `flutter-bloc-setup` | 1 | shipped 2026-05-09 |
| 2 | `flutter-bloc-feature-pattern` | 1 | shipped 2026-05-09 |
| 3 | `flutter-bloc-async-api` | 1 | shipped 2026-05-09 |
| 4 | `flutter-bloc-stream-tracking` | 2 | shipped 2026-05-09 |
| 5 | `flutter-bloc-testing` | 2 | shipped 2026-05-09 |
| 6 | `flutter-bloc-forms` | 3 | shipped 2026-05-09 |

All six skills are written. Family complete. Awaiting user review.

---

## 2. Decisions log

| Decision | Choice | Rationale |
|---|---|---|
| Scope shape | Family of 6 narrow skills | Matches existing flutter-* convention; description-match triggers more accurately on narrow skills. |
| Example style | Hybrid: generic body + "Applied to Talabat-clone" section | Skills stay reusable; a project-anchor section makes them immediately useful here. |
| Canonical stack | `flutter_bloc` + `freezed` + `bloc_concurrency` + `hydrated_bloc` + `bloc_test` + `mocktail` + `formz` | Covers ~95% of Talabat-clone state needs idiomatically; `formz` adds a 6th skill for forms. |
| Wave 1 scope | 3 skills: setup + feature-pattern + async-api | Self-sufficient starter set; nearly every feature is "fetch from Laravel → render → handle error". |
| Architecture coupling | Bloc replaces the `ChangeNotifier` ViewModel layer | The Repository / Service / Domain Model layers from `flutter-apply-architecture-best-practices` stay unchanged. |
| Platforms | Android + iOS only | Web/desktop out of scope. |
| Repo discipline | Skills and design doc live under `~/.claude/skills/`; nothing committed to `fadi/` | The PRD repo is project artifacts; skills are personal tooling. |

---

## 3. Family architecture & conventions

### 3.1 Layout

```
~/.claude/skills/
├── _design/
│   └── 2026-05-09-flutter-bloc-skills-family-design.md   ← this file
├── flutter-bloc-setup/SKILL.md                           ← wave 1
├── flutter-bloc-feature-pattern/SKILL.md                 ← wave 1
├── flutter-bloc-async-api/SKILL.md                       ← wave 1
├── flutter-bloc-stream-tracking/SKILL.md                 ← wave 2
├── flutter-bloc-testing/SKILL.md                         ← wave 2
└── flutter-bloc-forms/SKILL.md                           ← wave 3
```

### 3.2 Skill file format (matches existing `flutter-*` skills)

```yaml
---
name: flutter-bloc-<topic>
description: <single-sentence description optimized for trigger match>
metadata:
  model: claude-opus-4-7
  last_modified: Sat, 09 May 2026 ...
---

# <Title>

## Contents
- [Section A](#section-a)
- [Section B](#section-b)
- [Workflow: ...](#workflow-)
- [Applied to Talabat-clone](#applied-to-talabat-clone)
- [Examples](#examples)

## <conceptual sections>

...

## Workflow: <Action>

### Task Progress
- [ ] Step 1 ...
- [ ] Step 2 ...
- [ ] Run validator → review errors → fix → re-run.

## Applied to Talabat-clone
<reference at least one PRD section by number>

## Examples
<runnable Dart code>
```

### 3.3 Canonical stack assumed across all skills

```yaml
dependencies:
  flutter_bloc: ^9.0.0
  freezed_annotation: ^3.0.0
  json_annotation: ^4.9.0
  bloc_concurrency: ^0.3.0
  hydrated_bloc: ^10.0.0
  formz: ^0.7.0          # used only by flutter-bloc-forms
  http: ^1.2.0           # already covered by flutter-use-http-package

dev_dependencies:
  build_runner: ^2.4.0
  freezed: ^3.0.0
  json_serializable: ^6.8.0
  bloc_test: ^10.0.0     # used only by flutter-bloc-testing
  mocktail: ^1.0.0       # used only by flutter-bloc-testing
```

The setup skill installs the base set; later skills add their own dev deps as needed.

### 3.4 Cross-skill linkage

- `flutter-bloc-setup` is the foundation. Every other skill's description starts with "Prerequisite: complete `flutter-bloc-setup` first".
- `flutter-bloc-feature-pattern` defines the canonical Event/State/Bloc shape that all other skills build on.
- `flutter-bloc-async-api` references `flutter-use-http-package` (existing) for the HTTP layer; adds Bloc-specific success/failure mapping.
- `flutter-bloc-testing` references the Repository defined in `flutter-bloc-async-api` for its mocking examples.

### 3.5 Architecture stance

- **Bloc replaces** the `ChangeNotifier` ViewModel layer from `flutter-apply-architecture-best-practices`.
- **Repository / Service / Domain Model** layers from that skill stay identical.
- Project structure changes: `lib/ui/features/<x>/view_models/` → `lib/ui/features/<x>/bloc/`, with `<feature>_bloc.dart`, `<feature>_event.dart`, `<feature>_state.dart`.
- DI: `RepositoryProvider` at app root, `BlocProvider` per feature route. No `get_it`.

---

## 4. Wave-1 skill specs (write now)

### 4.1 `flutter-bloc-setup`

**Description:** Configure a Flutter project for Bloc state management with `flutter_bloc`, `freezed`, `bloc_concurrency`, and `hydrated_bloc`. Use when initializing Bloc in a new project or migrating from `setState`/`Provider`/`ChangeNotifier`.

**Sections:**
1. Dependencies & generators (pubspec, `build_runner`, `HydratedBloc.storage` init in `main()`).
2. App-level providers (`MultiRepositoryProvider` at root, `BlocObserver` for global logging).
3. Project structure (`lib/ui/features/<feature>/bloc/` layout).
4. Workflow: Bootstrap a Bloc-ready project (8 steps).
5. Applied to Talabat-clone — references the entity layer (Order, Cart, Auth, Store-list) from `PRD_states.md`.
6. Examples — minimal `main.dart`, root `MultiRepositoryProvider`, sample `AppBlocObserver`.

**Workflow checklist:**
1. `flutter pub add flutter_bloc freezed_annotation json_annotation bloc_concurrency hydrated_bloc`
2. `flutter pub add --dev build_runner freezed json_serializable`
3. Create `lib/core/bloc/app_bloc_observer.dart`.
4. Initialize `HydratedBloc.storage` in `main()` before `runApp()` using `path_provider`.
5. Set `Bloc.observer = AppBlocObserver()` in dev/debug.
6. Wrap `MyApp` in `MultiRepositoryProvider` (empty list initially).
7. Create `lib/ui/features/`.
8. Run `dart run build_runner watch -d` in a side terminal.

**Length target:** ~150 lines.

---

### 4.2 `flutter-bloc-feature-pattern`

**Description:** Scaffold a single feature using the canonical event/state/bloc trio with `freezed` sealed states. Use when adding a new feature to a Bloc-based Flutter project (after `flutter-bloc-setup`).

**Sections:**
1. The Event/State/Bloc trio — file naming, responsibilities.
2. Sealed states with `freezed` — `@freezed sealed class CartState` with variants `Initial`, `Loading`, `Loaded`, `Failure`. Why sealed (exhaustive `switch`).
3. Event handlers & concurrency — `EventTransformer` from `bloc_concurrency`:
   - `droppable` — submit-once buttons
   - `restartable` — search-as-you-type
   - `sequential` — cart updates (default for ordering-sensitive ops)
   - `concurrent` — parallel detail fetches
4. Wiring the View — `BlocProvider`, `BlocBuilder`, `BlocSelector`, `BlocListener`, when to use each.
5. Persistence (optional) — extending `HydratedBloc<E, S>` with `fromJson`/`toJson`. Opt-in per Bloc.
6. Workflow: Scaffold a new feature (10 steps).
7. Applied to Talabat-clone — full event/state sketch for `CartBloc` (matches cart UI flow in `prd.md` and `PRD_states.md §7`); `CartBloc` extends `HydratedBloc` (cart persists), `StoreListBloc` does not.
8. Examples — full Counter end-to-end (event, state, bloc, view) with freezed sealed switch.

**Workflow checklist:**
1. Pick feature name (e.g., `cart`).
2. Create `cart_event.dart` with sealed `CartEvent` variants.
3. Create `cart_state.dart` with sealed `CartState` variants.
4. Create `cart_bloc.dart` extending `Bloc<CartEvent, CartState>` (or `HydratedBloc` if persisting).
5. Register event handlers with appropriate `transformer:`.
6. If persisting, implement `fromJson` / `toJson` on the state.
7. `dart run build_runner build --delete-conflicting-outputs`.
8. Wrap the route in `BlocProvider(create: (_) => CartBloc(repo: ...))`.
9. Use `BlocBuilder` (rebuild), `BlocListener` (side-effects: snackbar/nav), `BlocSelector` (fine-grained).
10. Run validator → review compile errors → fix → re-run.

**Length target:** ~250 lines.

---

### 4.3 `flutter-bloc-async-api`

**Description:** Implement the canonical async API call pattern in Bloc: dispatch event → emit loading → call Repository → emit success or typed failure. Use when wiring a feature to a REST API (typically a Laravel backend).

**Sections:**
1. Repository contract — `Future<Result<T, Failure>>`. Repositories never throw; they wrap.
2. Failure taxonomy — `Failure.network`, `Failure.validation(Map<String,List<String>>)` (Laravel 422 shape), `Failure.unauthorized`, `Failure.server(int code)`.
3. Event handler shape — emit `Loading`, await `repo.fetch()`, pattern-match `Result.success` / `Result.failure`, emit `Loaded` or `LoadFailure`.
4. Refresh / pull-to-refresh — `Refresh` event with `restartable` transformer.
5. Workflow: Wire a feature to the API (8 steps).
6. Applied to Talabat-clone — `RestaurantListBloc.fetchNearbyStores()`: `LocationDto` input → `GET /api/stores/nearby` → list of `StoreDto` → maps to domain `Store`. Notes how 422 surfaces in forms (deferred to `flutter-bloc-forms`).
7. Examples — `Result`/`Failure` sealed types, `RestaurantRepository`, `RestaurantListBloc`, View consuming with `BlocBuilder`.

**Workflow checklist:**
1. Define DTO with `freezed` + `json_serializable`.
2. Define domain model.
3. Define `Result<T, Failure>` and `Failure` sealed types in `lib/core/result/`.
4. Implement Service (HTTP via `flutter-use-http-package`).
5. Implement Repository — wraps Service, transforms DTO → domain, returns `Result`.
6. Implement Bloc event handler.
7. Register Repository in root `MultiRepositoryProvider`.
8. Wire View with `BlocBuilder` rendering loading/error/data branches.

**Length target:** ~250 lines.

---

## 5. Wave-2 / Wave-3 skill specs (design-only)

### 5.1 `flutter-bloc-stream-tracking` (Wave 2)

**Description:** Subscribe to streams (Pusher/WebSocket/Firestore) inside a Bloc and emit state on each event. Use when implementing real-time features like order tracking, driver location, or live chat.

**Sections:**
1. Stream subscription pattern — `await emit.forEach(stream, onData: ...)` (preferred over `stream.listen`).
2. Connection lifecycle — explicit `Connect` / `Disconnect` events; reconnection with backoff in Repository.
3. State design for streams — `Connecting`, `Connected(data)`, `Disconnected(reason)`, `Reconnecting`. Don't conflate with API loading states.
4. Workflow: Wire a real-time stream into a Bloc (7 steps).
5. Applied to Talabat-clone — `OrderTrackingBloc` subscribes from `picked_up`, unsubscribes at `delivered` / `failed_delivery` (`PRD_states.md §1`). Chat-thread time-window check (`PRD_states.md §15`) opens/closes the chat stream.
6. Examples — `OrderTrackingBloc` consuming `Stream<DriverLocation>` from `OrderRepository.subscribeToDriver(orderId)`.

---

### 5.2 `flutter-bloc-testing` (Wave 2)

**Description:** Write deterministic tests for Blocs using `bloc_test` and `mocktail`. Use when adding unit tests for a Bloc's event handlers and emitted state sequences.

**Sections:**
1. The `blocTest<B, S>` helper — anatomy: `build`, `seed`, `act`, `expect`, `verify`.
2. Mocking repositories with `mocktail` — `class MockX extends Mock implements X`.
3. Testing concurrency transformers — verifying `restartable` cancels stale requests using `Completer`.
4. Testing `HydratedBloc` — overriding `HydratedBloc.storage` with an in-memory fake.
5. Workflow: Test a Bloc (6 steps).
6. Applied to Talabat-clone — `CartBloc` tests: add item, switch store (abandon-and-restart per `PRD_states.md §7`), persist & rehydrate.
7. Examples — full test suite for `RestaurantListBloc` from §4.3: success, failure, restartable refresh.

---

### 5.3 `flutter-bloc-forms` (Wave 3)

**Description:** Model form fields as value objects with `formz` and drive form state with a Bloc. Use when implementing complex forms like address entry, OTP, profile editing, or login.

**Sections:**
1. Why `formz` — encapsulate validation + dirty/pure tracking per field.
2. `FormzInput` subclass per field — `EmailInput`, `PhoneInput`, `OtpCodeInput`. Validators return enum `Invalid` cases.
3. Form Bloc state shape — fields + `FormzSubmissionStatus` + `Failure?`. Submission ≠ field validity.
4. Mapping Laravel 422 errors to fields — `Failure.validation(Map<String,List<String>>)` splat onto corresponding `FormzInput`s; emit with `pure: false` to surface.
5. Workflow: Build a form with Bloc + formz (8 steps).
6. Applied to Talabat-clone — `AddressFormBloc` (line, building, floor, apartment, city — matches `prd.md §2.1`); `OtpVerifyBloc` from `PRD_states.md §14` (`pending_verification → active`).
7. Examples — `LoginFormBloc` with `EmailInput`, `PasswordInput`, submission, 422-error mapping.

---

## 6. Validation strategy

### 6.1 Mechanical checks (after writing each SKILL.md)

- Valid YAML frontmatter; `name` matches directory.
- All referenced packages exist on pub.dev at the stated versions (`flutter pub deps` against a throwaway project, optional).
- Markdown links resolve.
- Code examples parse (no truncated braces, valid Dart syntax).

### 6.2 Functional checks (mental walk-through)

For each wave-1 skill, walk one Talabat-clone scenario:
- `setup` → bootstrap a clean `flutter create` project, follow the checklist, `flutter run`.
- `feature-pattern` → scaffold `CartBloc` skeleton from the "Applied to Talabat-clone" example.
- `async-api` → wire `RestaurantListBloc` to a stub Laravel endpoint.

We do NOT run a real Flutter project this session unless explicitly asked.

---

## 7. Build sequence

1. **Done:** Brainstorming → design doc.
2. **Next:** `/superpowers:writing-plans` — produce implementation plan for the 3 wave-1 SKILL.md files.
3. **Then:** Implement the 3 SKILL.md files in `~/.claude/skills/`.
4. **Out of scope (this session):** Wave-2 and Wave-3 SKILL.md files.

---

## 8. Risks & open questions

| Risk | Mitigation |
|---|---|
| Bloc 9.x and freezed 3.x are recent. User's other Flutter projects may pin older versions. | Skill examples target the latest stable. If user requests a compat note, add a "Migrating from Bloc 8.x" callout to `flutter-bloc-setup`. |
| `bloc_concurrency` transformers + `HydratedBloc`: persisting while a `restartable` request is in flight has subtle ordering semantics. | Flag in `flutter-bloc-feature-pattern` as a "Limitations" subsection rather than try to solve it. |
| Skills assume Laravel-style 422 validation response shape. Other backends differ. | Hybrid examples: keep generic in skill body, document Laravel shape in "Applied to Talabat-clone" section only. |

---

## 9. Out of scope (intentionally not in this family)

- State managers other than Bloc (Riverpod, Provider, GetX).
- Codegen for non-Bloc concerns (e.g., DTO generation outside the async-api skill).
- Theming, Material 3, RTL Arabic — separate skill families.
- CI/release / `flutter build` automation.
- Web and desktop platform configs.
- Platform channels / native interop.
