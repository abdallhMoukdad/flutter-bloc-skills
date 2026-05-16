# Review — flutter-bloc-forms

**Reviewer:** Opus (independent)
**Date:** 2026-05-14
**Skill:** `~/.claude/skills/flutter-bloc-forms/SKILL.md` (434 lines)

## Verdict: REQUEST CHANGES (won't compile as written)

The skill articulates the right architecture — `FormzInput` per field, derived `isValid`, server-error round-trip — and the underlying *idea* of carrying an optional server message on the input is sound. But the runnable code in the Examples section does not compile, the mid-skill prose pseudocode references a method (`withServerError`) that is never defined and is inconsistent with the final implementation, and the "Applied to Talabat-clone" prose perpetuates that ghost method. A developer following the skill end-to-end will hit a compile error on the very first `EmailInput.pure()` they construct, and a different compile error if they copy the prose 422-mapping snippet verbatim. Beyond those, the skill is also the longest in the family (434 lines vs. ~250 target) and could be trimmed substantially.

**Count: 4 Critical, 4 Important, 3 Nits.**

**Top two fixes:**
1. Add a `serverError` initializer to `EmailInput.pure()` / `PasswordInput.pure()` (lines 189, 213). As written, the `final String? serverError` field is uninitialized in the pure constructor — compile error: "Final field 'serverError' is not initialized by this constructor."
2. Pick one mechanism for attaching a server error and apply it everywhere. The skill currently presents three contradictory mechanisms (the extension at line 142, the `withServerError(...)` method at lines 122/127/171, and the constructor-parameter at line 190/214). The runnable example uses (3); the prose and the Talabat section use (2); the explicit code snippet on (1) is vestigial. Delete (1) and (2); keep (3).

---

## Critical

### C1. The runnable `EmailInput.pure()` doesn't compile (line 189)

```dart
class EmailInput extends FormzInput<String, EmailValidationError> {
  const EmailInput.pure() : super.pure('');
  const EmailInput.dirty([super.value = '', this.serverError]) : super.dirty();
  final String? serverError;
  ...
}
```

`serverError` is declared `final` on the class. The `dirty` constructor initializes it via `this.serverError`. The `pure` constructor does not. Dart rejects this:

> Error: Final field 'serverError' is not initialized by this constructor.

Empirically reproduced against `formz: ^0.7.0` on Dart 3.11 with the skill's exact code. Same bug applies to `PasswordInput.pure()` at line 213.

**Fix:** `const EmailInput.pure() : serverError = null, super.pure('');` — or make the field non-final, or move `serverError` to a wrapper instead of the FormzInput subclass.

### C2. The `withServerError(...)` method is referenced three times and defined zero times

Appears at lines 122-123, 126-127, and 171 in prose:

```dart
EmailInput.dirty(state.email.value).withServerError(errors['email']!.first)
```

Searched the file: no `withServerError` method on `EmailInput`, no extension `on EmailInput`, no extension on `FormzInput` that defines it. The only nearby extension (line 142) declares a *getter* named `serverError` returning a constant null — and notes it's "implemented per-input subclass via @freezed", which is yet another path nothing in the file follows.

This is the highest-risk thing in the skill: the explanation paragraph (lines 100-137) tells the reader to call `.withServerError(...)`, but the only runnable code (lines 287-322) uses the constructor parameter `EmailInput.dirty(value, errors['email']!.first)`. A reader who copies the prose snippet first will get "The method 'withServerError' isn't defined for the type 'EmailInput'" and have to reverse-engineer the intent.

**Fix:** Replace every `.withServerError(x)` in the prose with the second positional constructor argument: `EmailInput.dirty(state.email.value, errors['email']!.first)`. Then delete the misleading extension snippet at lines 141-145 and the "(In practice, define EmailInput as a @freezed class…)" parenthetical at line 147.

### C3. The "implemented per-input subclass via @freezed" extension (lines 141-147) is incoherent

```dart
extension on FormzInput {
  String? get serverError => null; // implemented per-input subclass via @freezed
}
```

Three problems:

1. An extension cannot be "implemented per-input subclass" — extensions are not virtual, do not dispatch dynamically, and are resolved statically by static type. A subclass cannot "override" an extension getter.
2. The parenthetical (line 147) says to "define EmailInput as a @freezed class". `EmailInput` extends `FormzInput<String, EmailValidationError>` — a regular Dart class with a custom validator method. Making it `@freezed` collides with the formz inheritance model (FormzInput's constructors and the `validator` override don't play with freezed's generated machinery).
3. The final runnable example does neither of these things — `EmailInput` is a plain class with a constructor parameter.

This entire block is vestigial. It's a half-thought-through earlier draft that survived alongside the final design.

**Fix:** Delete lines 141-147 entirely. Replace with one sentence: "Each FormzInput subclass declares a final `String? serverError` field initialized by both `pure()` and `dirty()` constructors. The View prefers it over the client-side `displayError`."

### C4. `prd.md §2.1` does not describe the AddressFormBloc fields

The skill (line 167-169) says:

> `AddressFormBloc` (`prd.md §2.1`). Address entry on the home screen has these fields per the PRD: address line, building, floor, apartment, plus an implicit city derived from GPS.

`prd.md §2.1` (Arabic: "اختيار مكان التوصيل" / "choosing delivery location") describes *picking* an address — listing saved addresses, choosing one, using current location, searching manually. It does **not** enumerate address-entry fields.

The fields are actually in `prd.md §27` ("العناوين" / "Addresses"), line 373:

> إضافة تفاصيل العنوان: رقم الشقة، الطابق، علامة مميزة

That gives: apartment number ("رقم الشقة"), floor ("الطابق"), landmark ("علامة مميزة"), plus special instructions ("تعليمات خاصة بالعنوان") on line 374. There is **no** "building" field in the PRD — the skill invented one. There is also no "address line" as a distinct field in §27 (though §2.1's manual-search implies a query string).

**Fix:** Cite `prd.md §27` (or `§2.1 + §27`). Replace `BuildingInput` with `LandmarkInput` (the PRD's "علامة مميزة") and add `SpecialInstructionsInput` if the form covers it. Or rewrite the section to focus on the fields the PRD actually lists.

---

## Important

### I1. The mapping in C2 also has a subtle bug worth flagging

Both the prose (line 122) and the runnable example (line 305) clear the stale server error on the *other* fields by passing `state.email` unchanged when `email` isn't in the new error map. That's fine. But neither version handles the case where a stale server error survives across submissions on a field that *does* have a current client-side error.

Concretely: user submits with `email = "a@b.co"`, server returns 422 `{"email": ["taken"]}`. State becomes `EmailInput.dirty("a@b.co", "taken")`. User then types `"a@"` (invalid client-side) and submits again. The `_onEmailChanged` handler at line 280 emits `EmailInput.dirty(e.value)` — server error cleared, that's correct. But then submit aborts (`!state.isValid`), so the user re-types and we go again. This actually works. False alarm? Trace once more:

Actually re-tracing: line 280 `emit(state.copyWith(email: EmailInput.dirty(e.value)))` — the new EmailInput has `serverError = null` (default). Good, this is correct: every keystroke clears the prior server error. **No bug, but the skill should explicitly call out this behavior** since it's non-obvious and load-bearing for UX (the user shouldn't see "Email taken" after they've started fixing the email).

**Fix:** Add a sentence after line 280-281: "Note: the dirty constructor's default `serverError = null` means typing clears any stale server error on that field."

### I2. `Formz.validate([EmailInput.pure(), PasswordInput.pure()])` returns `false`

The skill states (line 81): "The View consults `state.isValid` to decide whether the submit button is enabled". This is correct, but the implication that `isValid` is `true` for the initial state is wrong. Empirically: `EmailInput.pure()` has value `""`, validator returns `EmailValidationError.empty`, so `isValid == false`. The submit button is correctly disabled at start, which is the desired behavior — but the skill should call it out, because the language "the View consults `state.isValid`" can be misread as "the form starts valid and becomes invalid on bad input".

Also: the AddressFormBloc section says `CityInput` is `dirty` on initial load (line 169). The skill should be explicit that the bloc constructor must seed all server-populated fields as `dirty(value)` not `pure()`, otherwise `isValid` will be `false` for fields the user never sees.

**Fix:** Add one line after the "Submission status is independent of validity" paragraph: "The initial state with all-`pure()` inputs is *not* valid — pure inputs whose validator rejects empty strings are invalid-but-not-displayed. Submit is correctly disabled."

### I3. `Failure.validation(Map<String, List<String>>)` consumption — verify shape matches async-api

The skill consumes `Failure.validation(errors)` at line 118 via `ResultFailure(failure: ValidationFailure(:final errors))`. Cross-checking against `flutter-bloc-async-api/SKILL.md` line 61-62:

```dart
const factory Failure.validation(Map<String, List<String>> errors) = ValidationFailure;
```

Match. The pattern `ValidationFailure(:final errors)` against a freezed factory `Failure.validation(...)` does require that `errors` be a named field on the generated class. Freezed 3.x generates positional factories as positional parameters that surface as a property named per the parameter name — so this pattern works. **No bug.** But the skill should reference `flutter-bloc-async-api` more explicitly at line 87 (it only says "from flutter-bloc-async-api's repository"). One concrete sentence like "as defined in `flutter-bloc-async-api`'s `Failure` taxonomy".

### I4. `BlocSelector` rebuild semantics on `displayError`

Line 380-389:

```dart
BlocSelector<LoginFormBloc, LoginFormState, EmailInput>(
  selector: (s) => s.email,
  builder: (ctx, email) => TextField(
    decoration: InputDecoration(
      labelText: 'Email',
      errorText: email.serverError ?? _emailMsg(email.displayError),
    ),
    onChanged: (v) => ctx.read<LoginFormBloc>().add(LoginEmailChanged(v)),
  ),
),
```

`BlocSelector` rebuilds when the selected value changes by `!=`. `EmailInput`'s equality is whatever formz's `FormzInput` defines — checking the package: `FormzInput.==` compares `isPure` and `value`. So if `serverError` changes but `value` and `isPure` don't, the selector **may not rebuild** because the new `EmailInput` is equal to the old one under formz's equality. Concretely: after a 422, the bloc emits a new state with `EmailInput.dirty(sameValue, newServerError)` — this is `==` to `EmailInput.dirty(sameValue)` under formz's equality, so `BlocSelector` may skip the rebuild and the server error never appears.

This is a real bug that depends on FormzInput's `==` implementation. **Action:** Either (a) override `==` and `hashCode` on `EmailInput` to include `serverError`, (b) replace `BlocSelector` with `BlocBuilder` + `buildWhen: (p, n) => p.email != n.email || p.email.serverError != n.email.serverError` (ugly), or (c) make `EmailInput` a `@freezed` value class that wraps a FormzInput (also ugly). Recommend (a). Add to the example.

### I5. The `Resend` cooldown reference (line 175) cross-skill is mis-aimed

> A `Resend` event triggers a separate repository call with a cooldown timer state — see `flutter-bloc-stream-tracking`-style countdown if cooldown is timer-driven.

`flutter-bloc-stream-tracking` is for *external* streams (WebSocket, Pusher, Firestore). A countdown timer is a `Stream.periodic` — that's covered by Dart core, not by anything in stream-tracking. This cross-reference will mislead the reader to that skill, find nothing about timers there, and come back confused.

**Fix:** Drop the reference. Just say "implement the cooldown with a `Stream.periodic(Duration(seconds: 1))` consumed via `emit.forEach(...)` in a separate handler".

---

## Nits

### N1. Skill is 434 lines — longest in the family

For comparison:
- setup: 202
- feature-pattern: 298
- async-api: 376
- stream-tracking: 247
- testing: 287
- forms: **434**

The design spec (§5.3) had no explicit line target but the family runs ~250-300. Trimmable sections:
- "Why formz" (lines 22-31) — 4 paragraphs to say what one sentence and the validator code already show. Cut to one paragraph.
- "Form Bloc state shape" intro (lines 61-83) — drops to ~10 lines.
- Lines 141-147 (the vestigial extension) — delete per C3.
- The Applied section's PasswordInput/AddressLineInput discussion — the Examples section already has runnable PasswordInput; collapse the Applied prose.

Realistic target after fixes: ~330 lines.

### N2. The `_messageFor(Failure f)` helper appears twice (lines 324-329 here, 332-341 in async-api)

Slightly different (this skill's version omits Unauthorized/NotFound/Forbidden/Validation/Unknown branches; reuses fallback `_ => ...`). Inconsistent with async-api's exhaustive `switch`. If async-api's version is canonical, point to it ("see `flutter-bloc-async-api` for the canonical mapper") and inline only the form-Bloc-specific delta (mapping `UnauthorizedFailure()` to the user-facing "Email or password is incorrect"). Currently both skills define the same helper differently.

### N3. `OtpVerifyBloc` (line 173-175) — the 410 mapping is undefined elsewhere

Says "a custom 410 for expired codes that the Bloc maps to `serverError: 'This code has expired — request a new one'`". But `Failure` (in async-api) has no `410` / `gone` variant. The reader would need to either extend `Failure` (and that ripples through every consumer) or shoehorn it into `Failure.server(410)` — which the skill doesn't explain. Either acknowledge the gap ("requires adding `Failure.gone()` to the taxonomy") or use a status code other than 410 in the example.

---

## Done well

- The architectural shape (FormzInput per field + `FormzSubmissionStatus` independent of `isValid` + per-field 422 routing) is exactly right and well motivated.
- The `errorText: email.serverError ?? _emailMsg(email.displayError)` one-liner (line 385) is genuinely the cleanest expression of the server-wins precedence — keep it as the punchline of the skill (line 434).
- `droppable()` for submit and `sequential()` for keystrokes are the correct transformer choices, and the rationales (double-tap protection vs. keystroke ordering) at lines 157-158 are accurate.
- The `_messageFor` in `LoginFormBloc` correctly routes `UnauthorizedFailure()` to a friendly "Email or password is incorrect" snackbar via `state.serverError` rather than letting it bleed into the email field — the right call.
- The cross-skill linkage in the description ("Prerequisite: `flutter-bloc-setup`, `flutter-bloc-feature-pattern`, and `flutter-bloc-async-api`") is correct and unambiguous.
- `PRD_states.md §14` citation for `pending_verification → active` is accurate.
- Empirical verification: `Formz.validate(...)` is `Formz.validate(List<FormzInput<dynamic, dynamic>>) → bool` in 0.7.0; `FormzSubmissionStatus` values are exactly `{initial, inProgress, success, failure, canceled}`; `displayError` returns `isPure ? null : error`. The skill's API claims match the library.
