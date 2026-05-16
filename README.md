# flutter-bloc-skills

A [Claude Code](https://claude.ai/code) plugin providing six focused, proactive skills for building production-quality Flutter apps using the **Bloc** state-management pattern, paired with a **Laravel** backend. Each skill activates automatically on the right phrases.

Opinionated toward `flutter_bloc ^9.0` + `freezed ^3.0` + `bloc_concurrency ^0.3` + `hydrated_bloc ^10.0` + `formz ^0.7` + `bloc_test ^10.0` + `mocktail ^1.0`. Mobile-only (Android + iOS).

---

## What This Plugin Does

| Skill | Trigger Conditions | What It Covers |
|---|---|---|
| **flutter-bloc-setup** | Initializing Bloc in a new Flutter project, migrating away from `setState`/`Provider`/`ChangeNotifier`, or before authoring any feature | Pinned-version `pub add` lines, `HydratedBloc.storage` init in `main()` (with the `getApplicationDocumentsDirectory()` vs `getTemporaryDirectory()` choice spelled out), root `MultiRepositoryProvider`, `AppBlocObserver` gated on `kDebugMode` with `onTransition` logging, project structure that integrates with `flutter-apply-architecture-best-practices` |
| **flutter-bloc-feature-pattern** | Adding a new feature to a Bloc project, structuring an existing feature, deciding which `bloc_concurrency` transformer fits | Event/state/bloc trio file layout, sealed `freezed` states with Dart 3 pattern-switch consumers, transformer-choice table (`droppable` / `restartable` / `sequential` / `concurrent`), `BlocBuilder` / `BlocSelector` / `BlocListener` distinctions, full `HydratedBloc` example with `storagePrefix` override for minification-safe persistence, the `restartable + persistence` ordering subtleties |
| **flutter-bloc-async-api** | Wiring a Bloc-driven feature to a REST API, mapping HTTP errors to typed failures, handling pull-to-refresh and request cancellation | `Result<T>` sealed union with a single generic and a fixed `Failure` taxonomy mirroring Laravel's HTTP shapes (network / timeout / unauthorized / forbidden / notFound / validation(422) / server / unknown), typed `ApiException` with `validationErrors` getter for round-tripping Laravel's 422 envelope, `restartable()` refresh that cancels in-flight requests cleanly, `_messageFor(Failure)` presentation-layer mapper |
| **flutter-bloc-stream-tracking** | Live order tracking, driver location updates, real-time chat, presence indicators, or any feature where the UI must reflect server-pushed events | `emit.forEach<T>` over `stream.listen` (auto-cancellation on Bloc close), explicit `ConnectRequested` / `DisconnectRequested` events with disconnect routed through the Repository (not by emitting `Closed` in a sibling handler — each handler has its own `Emitter`), Repository-owned reconnection with kill-switches per ID and max-retry caps, distinct stream-lifecycle states (Connecting/Connected/Reconnecting/Disconnected/Closed) kept separate from API states |
| **flutter-bloc-testing** | Adding tests for a Bloc's event handlers, verifying its emitted state sequence, mocking a Repository, asserting `restartable()` cancels in-flight requests | `bloc_test<B, S>` hook anatomy (build/setUp/seed/act/wait/expect/verify), `mocktail`'s `class MockX extends Mock implements X` + `registerFallbackValue` for custom types, `Completer`-controlled timing to actually prove `restartable()` (rather than passing tautologically), `HydratedBloc` rehydration tests with an in-memory `Storage` fake registered in `setUpAll` and cleared in `setUp` |
| **flutter-bloc-forms** | Implementing complex forms (login, OTP verification, address entry, profile editing, password reset) with cross-field validation and server-side error surfaces | `FormzInput<TValue, TError>` value objects per field (`EmailInput`, `PasswordInput`, etc.), `pure([value])` supporting pre-populated forms, `displayError` for "don't show errors before user touches the field", form Bloc state with `FormzSubmissionStatus` independent of `isValid`, per-field 422 round-trip (Laravel `errors` map → `FormzInput.dirty(value, serverError)`), custom `==`/`hashCode` so `BlocSelector` rebuilds when only `serverError` changes |

Each skill is a single `SKILL.md` with conceptual sections, a numbered workflow checklist, runnable Dart examples, and an "Applied to Talabat-clone" anchor showing how the pattern maps onto a real multi-store delivery app.

---

## Installation

### Recommended: Install via Marketplace

In Claude Code, run:

```text
/plugin marketplace add abdallhMoukdad/flutter-bloc-skills
/plugin install flutter-bloc-skills@flutter-bloc-skills
```

Restart Claude Code. All six skills are now active and will trigger automatically on relevant phrases (the prefix `flutter-bloc-skills:flutter-bloc-setup` will appear in the available-skills listing).

To update later:

```text
/plugin marketplace update flutter-bloc-skills
```

### Manual Installation (as personal skills, no plugin)

If you'd rather load the skills directly into `~/.claude/skills/` without going through the plugin mechanism:

```bash
git clone https://github.com/abdallhMoukdad/flutter-bloc-skills.git
cd flutter-bloc-skills
for d in skills/flutter-bloc-*; do
  ln -s "$(pwd)/$d" "$HOME/.claude/skills/$(basename "$d")"
done
```

Restart Claude Code. The six skills will appear without the `flutter-bloc-skills:` prefix.

> ⚠️ Don't do both — picking marketplace install **and** symlinking will register every skill twice (once with the plugin prefix and once without).

---

## Requirements

- Claude Code (any recent version that supports the plugin marketplace).
- For applying the skills to a real project: Flutter SDK with Dart 3.x, and a Flutter app targeting Android/iOS.

---

## Stack

The skills assume this dependency set:

```yaml
dependencies:
  flutter_bloc: ^9.0.0
  freezed_annotation: ^3.0.0
  json_annotation: ^4.9.0
  bloc_concurrency: ^0.3.0
  hydrated_bloc: ^10.0.0
  path_provider: ^2.1.0
  formz: ^0.7.0          # only flutter-bloc-forms
  http: ^1.2.0           # see flutter-use-http-package

dev_dependencies:
  build_runner: ^2.4.0
  freezed: ^3.0.0
  json_serializable: ^6.8.0
  bloc_test: ^10.0.0     # only flutter-bloc-testing
  mocktail: ^1.0.0       # only flutter-bloc-testing
```

`flutter-bloc-setup` covers installation in its `Workflow` section.

---

## Relationship to Other Skill Families

- **[`flutter-apply-architecture-best-practices`](https://docs.flutter.dev)** (a generic Flutter skill not in this plugin) lays out the Repository / Service / Domain Model layers. This plugin's skills replace its UI-state layer (which was `ChangeNotifier`-based ViewModels) with Bloc; the data and domain layers stay unchanged.
- **[`flutter-use-http-package`](https://pub.dev/packages/http)** (a generic Flutter skill) covers the HTTP transport. `flutter-bloc-async-api` builds on it — Service classes use `package:http`, the Repository wraps Service calls and maps to `Result<T>`.
- **`laravel-agent-skills`** ([repo](https://github.com/abdallhMoukdad/laravel-agent-skills)) — the backend companion plugin. The `Failure.validation(Map<String, List<String>>)` shape in `flutter-bloc-async-api` is exactly the 422 envelope that the Laravel skills' Form Requests produce.

---

## What's in `docs/`

- `2026-05-09-flutter-bloc-skills-family-design.md` — the design that shaped the six skills
- `2026-05-09-flutter-bloc-skills-family-implementation-plan.md` — the bite-sized authoring plan
- `reviews/` — independent Opus reviewer reports per skill + a synthesis document

The reviews are an audit trail: the family was reviewed by six parallel Opus reviewers that found 14 critical + 24 important + 26 nit-level issues in the first pass; all critical and important issues were addressed in subsequent edits. A separate verification pass against current `formz` / `flutter_bloc` / `bloc_test` / `hydrated_bloc` docs (via context7) caught four more improvements (notably the `storagePrefix` override for minified release builds).

---

## A note on the "Applied to Talabat-clone" sections

Each skill ends with a section showing how the pattern maps onto a real multi-store food-delivery app modelled after Talabat. References like `PRD_states.md §7` or `PRD_catalog.md §15.5` point to that project's PRDs (not in this repo). The PRD *content* is not embedded — only citation strings — but the citations sketch the shape of the domain (Order has 13 states with cross-entity cascades, Cart has one-store/one-city rules, etc.). If you're applying these skills to a different domain, the "Applied" sections are reference patterns; treat them as illustrative rather than prescriptive.

---

## License

MIT — see [`LICENSE`](LICENSE).
