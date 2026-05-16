# flutter-bloc-skills

A family of six [Claude Code](https://claude.com/claude-code) skills for building Flutter apps with the Bloc pattern, paired with a Laravel backend.

Opinionated toward `flutter_bloc ^9.0` + `freezed ^3.0` + `bloc_concurrency ^0.3` + `hydrated_bloc ^10.0` + `formz ^0.7` + `bloc_test ^10.0` + `mocktail ^1.0`. Mobile-only (Android + iOS).

## The family

| Skill | What it covers | Prereqs |
|---|---|---|
| [`flutter-bloc-setup`](flutter-bloc-setup/SKILL.md) | Project bootstrap: deps, `HydratedBloc.storage` init, `MultiRepositoryProvider`, `BlocObserver` | — |
| [`flutter-bloc-feature-pattern`](flutter-bloc-feature-pattern/SKILL.md) | Canonical event/state/bloc trio with `freezed` sealed states; `bloc_concurrency` transformer choice | setup |
| [`flutter-bloc-async-api`](flutter-bloc-async-api/SKILL.md) | `Result<T>` + typed `Failure` taxonomy; Laravel 422 round-trip; `restartable()` refresh; `ApiException` mapping | setup, feature-pattern |
| [`flutter-bloc-stream-tracking`](flutter-bloc-stream-tracking/SKILL.md) | Real-time streams via `emit.forEach`; Repository-owned reconnection with kill-switches | setup, feature-pattern |
| [`flutter-bloc-testing`](flutter-bloc-testing/SKILL.md) | `bloc_test` + `mocktail` patterns; `Completer`-timed concurrency tests; `HydratedBloc` rehydration tests | setup, feature-pattern, async-api |
| [`flutter-bloc-forms`](flutter-bloc-forms/SKILL.md) | `FormzInput` per field; form Bloc state shape; per-field 422 error round-trip | setup, feature-pattern, async-api |

Each skill is a single `SKILL.md` with conceptual sections, a numbered workflow checklist, runnable Dart examples, and an "Applied to Talabat-clone" anchor showing how the pattern maps onto a real multi-store delivery app.

## Install

These are personal-scope Claude Code skills (loaded from `~/.claude/skills/`, not as plugins). Two install options:

**Option A — symlink the cloned repo (recommended).** Source-of-truth is the repo; edits flow both ways.

```bash
git clone https://github.com/abdallhMoukdad/flutter-bloc-skills.git
cd flutter-bloc-skills
for d in flutter-bloc-*; do
  ln -s "$(pwd)/$d" "$HOME/.claude/skills/$d"
done
```

**Option B — copy the skill directories.** Simpler, but no sync with future updates.

```bash
git clone https://github.com/abdallhMoukdad/flutter-bloc-skills.git
cp -r flutter-bloc-skills/flutter-bloc-* ~/.claude/skills/
```

Restart Claude Code (or start a new session) and the six skills will appear in the available-skills listing. Description-match triggers them — ask "how do I set up bloc in this flutter project?" and `flutter-bloc-setup` should fire.

## What's in `docs/`

- `2026-05-09-flutter-bloc-skills-family-design.md` — the design that shaped the six skills
- `2026-05-09-flutter-bloc-skills-family-implementation-plan.md` — the bite-sized authoring plan
- `reviews/` — independent Opus reviewer reports per skill + a synthesis document

The reviews are an audit trail: the family was reviewed by six parallel Opus reviewers that found 14 critical + 24 important + 26 nit-level issues in the first pass; all critical and important issues were addressed in subsequent edits. The reviews remain as documentation of the rigor pass.

## A note on the "Applied to Talabat-clone" sections

Each skill ends with a section showing how the pattern maps onto a real multi-store food-delivery app modelled after Talabat. References like `PRD_states.md §7` or `PRD_catalog.md §15.5` point to that project's PRDs (not in this repo). The PRD *content* is not embedded — only citation strings — but the citations sketch the shape of the domain (Order has 13 states, Cart has one-store/one-city rules, etc.). If you're applying these skills to a different domain, the "Applied" sections are reference patterns; treat them as illustrative rather than prescriptive.

## License

MIT — see [`LICENSE`](LICENSE).
