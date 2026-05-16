# Flutter Bloc Skills Family — Implementation Plan (Wave 1)

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Author three SKILL.md files (`flutter-bloc-setup`, `flutter-bloc-feature-pattern`, `flutter-bloc-async-api`) under `~/.claude/skills/` per the family design at `~/.claude/skills/_design/2026-05-09-flutter-bloc-skills-family-design.md`.

**Architecture:** Each skill is a single markdown file with YAML frontmatter, conceptual sections, a `Workflow` checklist, an `Applied to Talabat-clone` anchor, and runnable Dart `Examples`. The skills cross-reference each other (setup → feature-pattern → async-api). The format mirrors the existing `flutter-*` skills exactly.

**Tech stack referenced inside the skills:** Dart 3.x, Flutter 3.x, `flutter_bloc ^9`, `freezed ^3`, `bloc_concurrency ^0.3`, `hydrated_bloc ^10`, `json_serializable ^6.8`, `build_runner ^2.4`. Mobile only (Android + iOS).

**Output locations:**
- `~/.claude/skills/flutter-bloc-setup/SKILL.md`
- `~/.claude/skills/flutter-bloc-feature-pattern/SKILL.md`
- `~/.claude/skills/flutter-bloc-async-api/SKILL.md`

**Out of repo:** Nothing here is committed to `~/Desktop/software/fadi/`.

---

## Pre-flight (one-shot)

### Task 0: Create skill directories

**Files:**
- Create: `~/.claude/skills/flutter-bloc-setup/`
- Create: `~/.claude/skills/flutter-bloc-feature-pattern/`
- Create: `~/.claude/skills/flutter-bloc-async-api/`

**Step 1: Create the three directories**

Run: `mkdir -p ~/.claude/skills/flutter-bloc-setup ~/.claude/skills/flutter-bloc-feature-pattern ~/.claude/skills/flutter-bloc-async-api`

**Step 2: Verify**

Run: `ls -d ~/.claude/skills/flutter-bloc-{setup,feature-pattern,async-api}/`
Expected: three directory paths printed, no errors.

---

## Phase 1 — `flutter-bloc-setup`

### Task 1.1: Frontmatter + Contents + intro paragraph

**File:** `~/.claude/skills/flutter-bloc-setup/SKILL.md`

**Step 1: Write the frontmatter**

```yaml
---
name: flutter-bloc-setup
description: Configure a Flutter project for Bloc state management with `flutter_bloc`, `freezed`, `bloc_concurrency`, and `hydrated_bloc`. Use when initializing Bloc in a new project or migrating from `setState`/`Provider`/`ChangeNotifier`.
metadata:
  model: claude-opus-4-7
  last_modified: Sat, 09 May 2026 00:00:00 GMT
---
```

**Step 2: Write the title, scope sentence, and Contents anchor list**

Title: `# Bootstrapping Bloc in a Flutter Project`
Scope sentence: one paragraph noting Android + iOS only, web/desktop out of scope.
Contents list: Dependencies & generators, App-level providers, Project structure, Workflow, Applied to Talabat-clone, Examples.

**Step 3: Verify frontmatter parses**

Run: `head -10 ~/.claude/skills/flutter-bloc-setup/SKILL.md`
Expected: clean YAML between `---` markers, name matches directory.

---

### Task 1.2: Section "Dependencies & generators"

**File:** `~/.claude/skills/flutter-bloc-setup/SKILL.md` (append)

**Step 1: Write the section**
- Two `flutter pub add` blocks (deps + dev deps).
- Note that `freezed` and `json_serializable` are dev deps.
- Note `build_runner watch -d` runs in a side terminal.
- Reference the existing `flutter-use-http-package` skill for HTTP setup; this skill does not duplicate it.

**Step 2: Verify**

Run: `grep -c "flutter pub add" ~/.claude/skills/flutter-bloc-setup/SKILL.md`
Expected: `>= 2`.

---

### Task 1.3: Section "App-level providers"

**File:** `~/.claude/skills/flutter-bloc-setup/SKILL.md` (append)

**Step 1: Write the section**
- `MultiRepositoryProvider` at app root, initially empty list.
- Custom `AppBlocObserver` extending `BlocObserver`, overriding `onChange`/`onError` for dev logging.
- `Bloc.observer = AppBlocObserver()` in dev builds; gated by `kDebugMode`.

**Step 2: Verify**

Run: `grep -c "MultiRepositoryProvider\|BlocObserver" ~/.claude/skills/flutter-bloc-setup/SKILL.md`
Expected: `>= 2`.

---

### Task 1.4: Section "Project structure"

**File:** `~/.claude/skills/flutter-bloc-setup/SKILL.md` (append)

**Step 1: Write the section**
- ASCII tree showing `lib/ui/features/<feature>/bloc/<feature>_{bloc,event,state}.dart`.
- One sentence noting this replaces the `view_models/` directory from `flutter-apply-architecture-best-practices`.
- One sentence noting `data/repositories/` and `data/services/` from that skill stay unchanged.

**Step 2: Verify**

Run: `grep -c "lib/ui/features" ~/.claude/skills/flutter-bloc-setup/SKILL.md`
Expected: `>= 1`.

---

### Task 1.5: Section "Workflow: Bootstrap a Bloc-ready project"

**File:** `~/.claude/skills/flutter-bloc-setup/SKILL.md` (append)

**Step 1: Write the 8-step `Task Progress` checklist**

Per the design doc §4.1. Each step is one concrete action. End with the feedback loop: "Run validator → review errors → fix → re-run."

**Step 2: Verify**

Run: `grep -c "^- \[ \]" ~/.claude/skills/flutter-bloc-setup/SKILL.md`
Expected: `>= 8`.

---

### Task 1.6: "Applied to Talabat-clone" + "Examples"

**File:** `~/.claude/skills/flutter-bloc-setup/SKILL.md` (append)

**Step 1: Applied section**
- Reference `PRD_states.md` and list which entities will get a dedicated Bloc (Order, Cart, Auth, Store-list, Restaurant detail, Order tracking).
- Note which Blocs persist (`HydratedBloc`): `CartBloc`, `AuthBloc`. Others do not.

**Step 2: Examples section**
- Example 1: minimal `main.dart` initializing `HydratedBloc.storage` via `path_provider`, then `runApp`.
- Example 2: `lib/core/bloc/app_bloc_observer.dart` full body.
- Example 3: root `MyApp` with `MultiRepositoryProvider` + `MaterialApp`.

**Step 3: Verify**

Run: `wc -l ~/.claude/skills/flutter-bloc-setup/SKILL.md`
Expected: `120 - 180` lines.

Run: `grep -c "PRD_states.md\|PRD_catalog.md\|prd.md" ~/.claude/skills/flutter-bloc-setup/SKILL.md`
Expected: `>= 1`.

---

## Phase 2 — `flutter-bloc-feature-pattern`

### Task 2.1: Frontmatter + Contents + intro

**File:** `~/.claude/skills/flutter-bloc-feature-pattern/SKILL.md`

**Step 1: Write the frontmatter** (description per design §4.2; model and last_modified same format as Task 1.1).

**Step 2: Title and Contents** — Title: `# Scaffolding a Bloc Feature`. Contents anchors per design §4.2.

**Step 3: Verify**

Run: `head -10 ~/.claude/skills/flutter-bloc-feature-pattern/SKILL.md`
Expected: clean frontmatter, name matches directory.

---

### Task 2.2: Section "The Event/State/Bloc trio"

**File:** `~/.claude/skills/flutter-bloc-feature-pattern/SKILL.md` (append)

**Step 1: Write the section**
- File naming convention table: `<feature>_event.dart`, `<feature>_state.dart`, `<feature>_bloc.dart`.
- Responsibilities of each: Events = user intent / side-effect triggers. States = immutable UI snapshots. Bloc = event handlers that map events → states.
- Note that all three live in `lib/ui/features/<feature>/bloc/`.

---

### Task 2.3: Section "Sealed states with `freezed`"

**File:** `~/.claude/skills/flutter-bloc-feature-pattern/SKILL.md` (append)

**Step 1: Write the section**
- `@freezed sealed class CartState` with variants `Initial`, `Loading`, `Loaded({required List<CartItem> items, required Money total})`, `Failure({required String message})`.
- Why sealed — exhaustive `switch` makes "did we handle every state?" a compile-time question.
- Show the consumer pattern: `final widget = switch (state) { Initial() => ..., Loading() => ..., Loaded(:final items) => ..., Failure(:final message) => ... };`.

---

### Task 2.4: Section "Event handlers & concurrency"

**File:** `~/.claude/skills/flutter-bloc-feature-pattern/SKILL.md` (append)

**Step 1: Write the section**
- Table of `bloc_concurrency` transformers:
  | Transformer | Use when | Example |
  | --- | --- | --- |
  | `droppable` | Submit-once buttons | Place order |
  | `restartable` | Search-as-you-type | Restaurant search |
  | `sequential` | Ordering-sensitive | Cart mutations |
  | `concurrent` | Independent fetches | Multiple detail panels |
- Code snippet: `on<SearchChanged>(_onSearchChanged, transformer: restartable());`.

---

### Task 2.5: Section "Wiring the View"

**File:** `~/.claude/skills/flutter-bloc-feature-pattern/SKILL.md` (append)

**Step 1: Write the section**
- `BlocProvider(create: (_) => CartBloc(...))` at the route.
- `BlocBuilder` — rebuild on every state change (default).
- `BlocSelector<B, S, T>` — rebuild only when a derived value `T` changes.
- `BlocListener` — fire-and-forget side effects (snackbar, navigation, dialog). Does not rebuild.
- `BlocConsumer` — combination; use sparingly.
- One-paragraph guidance on when each is appropriate.

---

### Task 2.6: Section "Persistence (optional)"

**File:** `~/.claude/skills/flutter-bloc-feature-pattern/SKILL.md` (append)

**Step 1: Write the section**
- When to use `HydratedBloc` instead of `Bloc`: state must survive cold-start (cart, auth tokens, recently-viewed).
- `fromJson` / `toJson` overrides; for sealed states, use freezed's `toJson`/`fromJson`.
- One short subsection: **Limitations** — persisting while a `restartable` request is in-flight has subtle ordering semantics; the in-flight request is cancelled on close, the persisted state reflects the last emitted state, not the in-flight result.

---

### Task 2.7: Section "Workflow: Scaffold a new feature"

**File:** `~/.claude/skills/flutter-bloc-feature-pattern/SKILL.md` (append)

**Step 1: Write the 10-step `Task Progress` checklist** (per design §4.2).

**Step 2: Verify**

Run: `grep -c "^- \[ \]" ~/.claude/skills/flutter-bloc-feature-pattern/SKILL.md`
Expected: `>= 10`.

---

### Task 2.8: "Applied to Talabat-clone" + "Examples"

**File:** `~/.claude/skills/flutter-bloc-feature-pattern/SKILL.md` (append)

**Step 1: Applied section**
- Sketch `CartBloc`:
  - Events: `CartItemAdded(Product, int qty)`, `CartItemQtyChanged(itemId, int qty)`, `CartItemRemoved(itemId)`, `CartCleared`, `CartHydrated`.
  - States: `Initial`, `Loaded(items, store, subtotal, discount, total)`, `StoreSwitchPending(pendingProduct)` (per `PRD_catalog.md §15.5` one-cart/one-store rule), `Failure(message)`.
  - Reference `PRD_states.md §7` (Cart `active → converted | abandoned`).
  - Note `CartBloc` extends `HydratedBloc` for persistence.
  - Note `StoreListBloc` does not persist.

**Step 2: Examples section** — full Counter end-to-end:
- `counter_event.dart` (sealed `CounterEvent` with `Increment`, `Decrement`, `Reset`).
- `counter_state.dart` (sealed `CounterState` with `Value(int n)`).
- `counter_bloc.dart` (handlers, sequential transformer for ordering).
- `counter_view.dart` (BlocProvider + BlocBuilder + buttons).

**Step 3: Verify**

Run: `wc -l ~/.claude/skills/flutter-bloc-feature-pattern/SKILL.md`
Expected: `220 - 280` lines.

Run: `grep -c "PRD_" ~/.claude/skills/flutter-bloc-feature-pattern/SKILL.md`
Expected: `>= 2`.

---

## Phase 3 — `flutter-bloc-async-api`

### Task 3.1: Frontmatter + Contents + intro

**File:** `~/.claude/skills/flutter-bloc-async-api/SKILL.md`

**Step 1: Write the frontmatter** (description per design §4.3).

**Step 2: Title and Contents** — Title: `# Async API Calls in Bloc`. Contents per design §4.3.

**Step 3: Verify**

Run: `head -10 ~/.claude/skills/flutter-bloc-async-api/SKILL.md`
Expected: clean frontmatter, name matches directory.

---

### Task 3.2: Section "Repository contract"

**File:** `~/.claude/skills/flutter-bloc-async-api/SKILL.md` (append)

**Step 1: Write the section**
- All repository methods return `Future<Result<T, Failure>>`. Never throw.
- `@freezed sealed class Result<T, F>` with `Success(T data)` / `FailureR(F failure)`.
- Why: keeps error handling in the type system; Bloc never has a try/catch around `await repo.fetch()`.

---

### Task 3.3: Section "Failure taxonomy"

**File:** `~/.claude/skills/flutter-bloc-async-api/SKILL.md` (append)

**Step 1: Write the section**
- `@freezed sealed class Failure` with variants:
  - `Failure.network()` — connection refused, DNS, timeout.
  - `Failure.timeout()` — server reachable but slow.
  - `Failure.unauthorized()` — 401; trigger logout flow upstream.
  - `Failure.forbidden()` — 403.
  - `Failure.notFound()` — 404.
  - `Failure.validation(Map<String, List<String>> errors)` — 422 (Laravel shape).
  - `Failure.server(int statusCode)` — 5xx.
  - `Failure.unknown(Object error)` — fallback; log + report.
- One-paragraph note on the Laravel 422 shape: `{"message": "...", "errors": {"field": ["msg1", "msg2"]}}`. Map to `Failure.validation`.

---

### Task 3.4: Section "Event handler shape"

**File:** `~/.claude/skills/flutter-bloc-async-api/SKILL.md` (append)

**Step 1: Write the canonical async handler**

```dart
Future<void> _onLoadRequested(
  LoadRequested event,
  Emitter<RestaurantListState> emit,
) async {
  emit(const RestaurantListState.loading());
  final result = await _repo.fetchNearbyStores(event.location);
  emit(result.when(
    success: (stores) => RestaurantListState.loaded(stores),
    failure: (f) => RestaurantListState.failure(_messageFor(f)),
  ));
}
```

- Note: `_messageFor(Failure)` is a presentation-layer mapper — keep failure → human-readable string conversion in the Bloc, not in the View.

---

### Task 3.5: Section "Refresh / pull-to-refresh"

**File:** `~/.claude/skills/flutter-bloc-async-api/SKILL.md` (append)

**Step 1: Write the section**
- `RefreshRequested` event with `transformer: restartable()`.
- Why `restartable` and not `droppable`: pull-to-refresh during an in-flight initial load should cancel the initial load, not be ignored.
- Snippet: `on<RefreshRequested>(_onRefresh, transformer: restartable());`.

---

### Task 3.6: Section "Workflow: Wire a feature to the API"

**File:** `~/.claude/skills/flutter-bloc-async-api/SKILL.md` (append)

**Step 1: Write the 8-step `Task Progress` checklist** (per design §4.3).

**Step 2: Verify**

Run: `grep -c "^- \[ \]" ~/.claude/skills/flutter-bloc-async-api/SKILL.md`
Expected: `>= 8`.

---

### Task 3.7: "Applied to Talabat-clone" + "Examples"

**File:** `~/.claude/skills/flutter-bloc-async-api/SKILL.md` (append)

**Step 1: Applied section**
- `RestaurantListBloc.fetchNearbyStores`:
  - Input: `LocationDto(lat, lng)` derived from device GPS or saved address (`prd.md §2.1`).
  - Endpoint: `GET /api/stores/nearby?lat=...&lng=...`.
  - Maps `List<StoreDto>` to domain `List<Store>` (filters out stores per `PRD_states.md §4` operational gating: `paused_by_merchant` true OR `suspension_reason != null`).
  - Note 422-on-form errors will be detailed in `flutter-bloc-forms` (wave 3).

**Step 2: Examples section** — end-to-end:
- `Result<T, F>` and `Failure` sealed classes.
- `RestaurantApiClient` (Service) using `package:http`.
- `RestaurantRepository` mapping DTO → domain, returning `Result`.
- `RestaurantListBloc` event handler.
- `RestaurantListView` consuming with `BlocBuilder` (loading / error / empty / data branches).

**Step 3: Verify**

Run: `wc -l ~/.claude/skills/flutter-bloc-async-api/SKILL.md`
Expected: `220 - 280` lines.

Run: `grep -c "Laravel\|422\|PRD_" ~/.claude/skills/flutter-bloc-async-api/SKILL.md`
Expected: `>= 3`.

---

## Phase 4 — Validation pass

### Task 4.1: Mechanical verification

**Step 1: Frontmatter validity**

Run: `for f in ~/.claude/skills/flutter-bloc-{setup,feature-pattern,async-api}/SKILL.md; do echo "--- $f ---"; head -10 "$f"; done`
Expected: each opens with `---`, has `name:`, `description:`, `metadata:` keys, name matches its directory.

**Step 2: All three files present and non-trivial**

Run: `wc -l ~/.claude/skills/flutter-bloc-{setup,feature-pattern,async-api}/SKILL.md`
Expected:
- setup: 120-180 lines
- feature-pattern: 220-280 lines
- async-api: 220-280 lines

**Step 3: Cross-reference integrity**

Run: `grep -l "flutter-use-http-package\|flutter-apply-architecture-best-practices" ~/.claude/skills/flutter-bloc-{setup,feature-pattern,async-api}/SKILL.md`
Expected: at least one match in `setup`, at least one in `async-api`.

**Step 4: PRD anchoring**

Run: `grep -c "PRD_\|prd\.md" ~/.claude/skills/flutter-bloc-{setup,feature-pattern,async-api}/SKILL.md`
Expected: each file has `>= 1` reference.

---

### Task 4.2: Functional verification (mental walk-through)

**Step 1: Re-read each SKILL.md against its design spec**

For each of the three skills, open the file alongside the design doc section (§4.1, §4.2, §4.3) and check:
- Every section listed in the design appears in the SKILL.md.
- Every workflow step in the design checklist appears in the SKILL.md.
- The "Applied to Talabat-clone" section references at least one PRD section by number.
- All Dart code examples are syntactically valid (no truncated braces, no obvious typos).

**Step 2: Confirm skill discovery**

In a fresh Claude Code session (or by listing skills):
- The three skills appear in the available-skills list.
- Their descriptions are distinct enough that a query like "set up bloc" matches `flutter-bloc-setup` and not the other two.

---

## Out of scope (deferred to later sessions)

- `flutter-bloc-stream-tracking` (wave 2)
- `flutter-bloc-testing` (wave 2)
- `flutter-bloc-forms` (wave 3)

These are fully designed in `~/.claude/skills/_design/2026-05-09-flutter-bloc-skills-family-design.md` §5 and can be implemented later without re-brainstorming.

---

## Notes on plan adaptations

This plan adapts the standard writing-plans TDD template to a markdown-authoring task:
- "Write the failing test" → "Draft the section against the design spec".
- "Run the test" → `grep`/`wc -l` shape checks against the SKILL.md.
- Git commits are skipped — these files live in `~/.claude/skills/`, not in any tracked repo.
- "Tests pass" → all four Phase 4 verifications pass.
