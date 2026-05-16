---
name: flutter-bloc-forms
description: Models form fields as `formz` value objects and drives form state with a Bloc, including round-tripping Laravel 422 validation errors back to per-field UI errors. Use when implementing complex forms like login, OTP verification, address entry, profile editing, password reset, or any form with cross-field validation rules and server-side error surfaces. Prerequisite: `flutter-bloc-setup`, `flutter-bloc-feature-pattern`, and `flutter-bloc-async-api`.
metadata:
  model: claude-opus-4-7
  last_modified: Thu, 14 May 2026 00:00:00 GMT
---

# Forms with Bloc and Formz

Pushes per-field validation into typed value objects (`FormzInput<TValue, TError>`) so the Bloc only orchestrates: hold the fields, derive overall validity, submit, and route server-side validation errors back to the right field. Form fields become reusable types (`EmailInput`, `PhoneInput`) shared across screens. Submission status (`FormzSubmissionStatus.inProgress` / `success` / `failure`) is decoupled from field validity so a valid form can still fail to submit, and vice versa. Builds on `flutter-bloc-async-api` for the underlying request/response flow.

## Contents
- [Why formz](#why-formz)
- [FormzInput per field](#formzinput-per-field)
- [Form Bloc state shape](#form-bloc-state-shape)
- [Mapping Laravel 422 errors to fields](#mapping-laravel-422-errors-to-fields)
- [Workflow: Build a form with Bloc and formz](#workflow-build-a-form-with-bloc-and-formz)
- [Applied to Talabat-clone](#applied-to-talabat-clone)
- [Examples](#examples)

## Why formz

A naive form Bloc holds raw `String` fields and runs validation in the submit handler. That has three failure modes: (1) validation rules duplicate across forms that share a field (email validator written three times); (2) the View has to know whether to display an error (don't show "required" before the user touches the field); (3) server-side and client-side errors are tracked separately and drift apart.

`formz` solves all three by making each field a value object:

- A `FormzInput<TValue, TError>` knows its current value, its validity, and whether it has been touched.
- `displayError` returns the error only if the field is `dirty` — the View can naively render it without an "if pure return null" check.
- Validation rules live on the input class, not the Bloc. Reuse `EmailInput` in login, sign-up, profile-edit, and forgot-password.

## FormzInput per field

A field is a sealed enum of failure modes plus a `FormzInput` subclass.

```dart
import 'package:formz/formz.dart';

enum EmailValidationError { empty, malformed }

class EmailInput extends FormzInput<String, EmailValidationError> {
  const EmailInput.pure() : super.pure('');
  const EmailInput.dirty([super.value = '']) : super.dirty();

  static final _emailRegex = RegExp(r'^[^@\s]+@[^@\s]+\.[^@\s]+$');

  @override
  EmailValidationError? validator(String value) {
    if (value.isEmpty) return EmailValidationError.empty;
    if (!_emailRegex.hasMatch(value)) return EmailValidationError.malformed;
    return null;
  }
}
```

`.pure(value)` is for the initial state — field is untouched, no error displayed.
`.dirty(value)` is for every subsequent update from user input — the validator runs and `displayError` will surface its result.

For one-off validators (a field used only in one form), inline the class in the same file as the Bloc. For reusable validators (`EmailInput`, `PhoneInput`, `PasswordInput`), put them in `lib/core/forms/` so every form Bloc can import them.

## Form Bloc state shape

The state holds **all fields as inputs**, an overall submission status, and a slot for server-level failures that aren't per-field.

```dart
@freezed
class LoginFormState with _$LoginFormState {
  const factory LoginFormState({
    @Default(EmailInput.pure()) EmailInput email,
    @Default(PasswordInput.pure()) PasswordInput password,
    @Default(FormzSubmissionStatus.initial) FormzSubmissionStatus status,
    String? serverError, // for 401, 500, network — never for 422
  }) = _LoginFormState;

  const LoginFormState._();

  bool get isValid => Formz.validate([email, password]);
}
```

A getter `isValid` derives from `Formz.validate([...])` rather than being stored. The View consults `state.isValid` to decide whether the submit button is enabled — no duplicated truth to keep in sync.

The initial state with all-`pure()` inputs is **not** valid — pure inputs whose validator rejects empty strings are invalid-but-not-displayed. Submit is correctly disabled at start; `displayError` returns `null` until the field is dirty, so the user doesn't see "Email required" before typing.

**Submission status is independent of validity.** A form can be `FormzSubmissionStatus.inProgress` while `isValid` is true. After a `422`, `status` flips to `failure` but `isValid` may still be true on the client (the server disagreed). Treat them as orthogonal axes.

`formz` exposes a `FormzSubmissionStatusX` extension with convenience getters: `status.isInitial`, `status.isInProgress`, `status.isSuccess`, `status.isFailure`, `status.isCanceled`, and the load-bearing `status.isInProgressOrSuccess` for "keep the submit button disabled while in flight *and* after success until navigation completes". Use them instead of raw `status == FormzSubmissionStatus.inProgress` comparisons.

## Mapping Laravel 422 errors to fields

When `flutter-bloc-async-api`'s repository returns `Result.failure(Failure.validation(errors))` — the `Failure` taxonomy defined in `flutter-bloc-async-api` — the form Bloc splatters those errors back onto the corresponding inputs.

Laravel response:
```json
{
  "message": "The given data was invalid.",
  "errors": {
    "email": ["The email has already been taken."],
    "phone": ["The phone must be at least 10 digits."]
  }
}
```

Each `FormzInput` subclass declares an extra `final String? serverError` field initialized by both `.pure()` and `.dirty()` constructors. The View prefers `serverError` over the client-side `displayError`. There is no extension, no `withServerError(...)` method, no freezed wrapper — just a constructor parameter.

The mapping in the Bloc's submission handler:

```dart
Future<void> _onSubmitted(
  LoginFormSubmitted event,
  Emitter<LoginFormState> emit,
) async {
  if (!state.isValid) return;
  emit(state.copyWith(status: FormzSubmissionStatus.inProgress));

  final result = await _repo.login(
    email: state.email.value,
    password: state.password.value,
  );

  emit(switch (result) {
    ResultSuccess() =>
        state.copyWith(status: FormzSubmissionStatus.success),
    ResultFailure(failure: ValidationFailure(:final errors)) =>
        state.copyWith(
          status: FormzSubmissionStatus.failure,
          email: errors.containsKey('email')
              ? EmailInput.dirty(state.email.value, errors['email']!.first)
              : state.email,
          password: errors.containsKey('password')
              ? PasswordInput.dirty(state.password.value, errors['password']!.first)
              : state.password,
        ),
    ResultFailure(:final failure) =>
        state.copyWith(
          status: FormzSubmissionStatus.failure,
          serverError: _messageFor(failure), // canonical mapper lives in flutter-bloc-async-api
        ),
  });
}
```

**Stale-server-error invariant.** The `_onEmailChanged` keystroke handler emits `EmailInput.dirty(e.value)` with no second argument, so `serverError` defaults to `null` and stale 422 messages clear automatically as the user types. The form re-renders without the obsolete error before the next submit even fires.

## Workflow: Build a form with Bloc and formz

### Task Progress
- [ ] **Step 1 — Add the dep.** `flutter pub add formz`.
- [ ] **Step 2 — Decide which inputs are reusable.** Inputs used in 2+ forms go in `lib/core/forms/`. One-offs stay in the feature folder.
- [ ] **Step 3 — Write the `FormzInput` subclasses.** Each has a `pure()` ctor, `dirty()` ctor, validation enum, and `validator()` override.
- [ ] **Step 4 — Define the form state.** `@freezed` class with one input per field, `FormzSubmissionStatus`, optional `serverError`, and a derived `isValid` getter using `Formz.validate([...])`.
- [ ] **Step 5 — Define the events.** One `<Field>Changed(value)` per field, plus `Submitted` and (optional) `Reset`.
- [ ] **Step 6 — Implement field handlers.** Each handler emits `state.copyWith(<field>: <Input>.dirty(newValue))`. Use `transformer: sequential()` for keystroke-driven fields (avoid `concurrent()` reordering).
- [ ] **Step 7 — Implement the submit handler.** Use `transformer: droppable()` so double-taps on submit are ignored. Guard with `if (!state.isValid) return;`. Emit `inProgress`, call the repository, pattern-match the `Result`.
- [ ] **Step 8 — Map 422 to fields.** In the failure branch, distribute `ValidationFailure.errors` map keys onto the matching inputs via `<Input>.dirty(value)` + a server-error attachment.
- [ ] **Step 9 — Wire the View.** `TextField` per input — `decoration.errorText` reads from `state.<field>.displayError` (mapped through a `_message()` helper). Submit button uses `state.isValid && state.status != FormzSubmissionStatus.inProgress`.
- [ ] **Step 10 — Feedback loop.** Run on device → submit with invalid data → assert per-field errors render → submit with data that triggers a server 422 → assert those errors render on the right fields. If a 422 error message shows in a snackbar instead of next to the field, the Bloc swallowed `ValidationFailure` into `serverError` instead of splatting it.

## Applied to Talabat-clone

Two forms from the PRDs map directly.

### `AddressFormBloc` (`prd.md §2.1` selection, `§27` fields)

Address *selection* (saved-addresses list, "use my current location", far-address warning) is `prd.md §2.1`. Address *entry* fields are `prd.md §27`: apartment number (`رقم الشقة`), floor (`الطابق`), landmark (`علامة مميزة`), plus special instructions (`تعليمات خاصة بالعنوان`). The form Bloc holds an `ApartmentInput`, `FloorInput`, `LandmarkInput`, `SpecialInstructionsInput`, and a `CityInput` populated automatically from a reverse-geocode lookup (the city field is shown read-only and is "dirty" on initial load — pure inputs whose validator rejects empty strings would block submit).

Submitting POSTs to `/api/users/me/addresses`. The Laravel side validates that `apartment` is non-empty and `city_id` exists in the cities table. A 422 with `{"apartment": ["The apartment field is required."]}` maps to `state.copyWith(apartment: ApartmentInput.dirty(state.apartment.value, errors['apartment']!.first))` — the user sees the error inline.

### `OtpVerifyBloc` (`PRD_states.md §14`)

The user lifecycle is `pending_verification → active` via phone/email OTP. The form has a single `OtpCodeInput` with a fixed-length validator (6 digits) and a submit. Server-side errors include 422 (`{"code": ["Invalid code"]}`); expired codes come back as `Failure.unknown` or a custom code your repository maps to `Failure.unknown(...)`, surfaced as `state.serverError` rather than a per-field error. A `Resend` event triggers a separate repository call gated by a cooldown timer — `Stream.periodic(Duration(seconds: 1))` consumed via a `Ticked` event handler is the lightest implementation; no real-time transport, so `flutter-bloc-stream-tracking` does not apply.

## Examples

A `LoginFormBloc` complete with field inputs, state, events, handler, and view. The form has email and password; the API is `POST /api/login` returning either a 200 with a token or a 422 with field errors.

### `lib/core/forms/email_input.dart`

```dart
import 'package:formz/formz.dart';

enum EmailValidationError { empty, malformed }

class EmailInput extends FormzInput<String, EmailValidationError> {
  // `pure` accepts an optional seed value so an edit-profile screen can
  // pre-populate the field without making it dirty. `serverError` is always
  // null for pure inputs (you only get a server error after submitting).
  // Dart requires every final field to be set by every constructor.
  const EmailInput.pure([String value = ''])
      : serverError = null,
        super.pure(value);
  const EmailInput.dirty([super.value = '', this.serverError]) : super.dirty();

  final String? serverError;

  static final _re = RegExp(r'^[^@\s]+@[^@\s]+\.[^@\s]+$');

  @override
  EmailValidationError? validator(String value) {
    if (value.isEmpty) return EmailValidationError.empty;
    if (!_re.hasMatch(value)) return EmailValidationError.malformed;
    return null;
  }

  // FormzInput's default `==` covers `isPure` and `value` only — we add
  // `serverError` so a `BlocSelector<EmailInput>` rebuilds when only the
  // server message changes (same value, same purity, new error string).
  @override
  bool operator ==(Object other) =>
      identical(this, other) ||
      (other is EmailInput &&
          other.value == value &&
          other.isPure == isPure &&
          other.serverError == serverError);

  @override
  int get hashCode => Object.hash(value, isPure, serverError);
}
```

### `lib/core/forms/password_input.dart`

```dart
import 'package:formz/formz.dart';

enum PasswordValidationError { empty, tooShort }

class PasswordInput extends FormzInput<String, PasswordValidationError> {
  const PasswordInput.pure([String value = ''])
      : serverError = null,
        super.pure(value);
  const PasswordInput.dirty([super.value = '', this.serverError]) : super.dirty();

  final String? serverError;

  @override
  PasswordValidationError? validator(String value) {
    if (value.isEmpty) return PasswordValidationError.empty;
    if (value.length < 8) return PasswordValidationError.tooShort;
    return null;
  }

  @override
  bool operator ==(Object other) =>
      identical(this, other) ||
      (other is PasswordInput &&
          other.value == value &&
          other.isPure == isPure &&
          other.serverError == serverError);

  @override
  int get hashCode => Object.hash(value, isPure, serverError);
}
```

### `lib/ui/features/login/bloc/login_form_state.dart`

```dart
import 'package:formz/formz.dart';
import 'package:freezed_annotation/freezed_annotation.dart';

import '../../../../core/forms/email_input.dart';
import '../../../../core/forms/password_input.dart';

part 'login_form_state.freezed.dart';

@freezed
class LoginFormState with _$LoginFormState {
  const factory LoginFormState({
    @Default(EmailInput.pure()) EmailInput email,
    @Default(PasswordInput.pure()) PasswordInput password,
    @Default(FormzSubmissionStatus.initial) FormzSubmissionStatus status,
    String? serverError,
  }) = _LoginFormState;

  const LoginFormState._();

  bool get isValid => Formz.validate([email, password]);
}
```

### `lib/ui/features/login/bloc/login_form_bloc.dart`

```dart
import 'package:bloc_concurrency/bloc_concurrency.dart';
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:formz/formz.dart';

import '../../../../core/forms/email_input.dart';
import '../../../../core/forms/password_input.dart';
import '../../../../core/result/failure.dart';
import '../../../../core/result/result.dart';
import '../../../../data/repositories/auth_repository.dart';
import 'login_form_event.dart';
import 'login_form_state.dart';

class LoginFormBloc extends Bloc<LoginFormEvent, LoginFormState> {
  LoginFormBloc({required AuthRepository repo})
      : _repo = repo,
        super(const LoginFormState()) {
    on<LoginEmailChanged>(_onEmailChanged, transformer: sequential());
    on<LoginPasswordChanged>(_onPasswordChanged, transformer: sequential());
    on<LoginSubmitted>(_onSubmitted, transformer: droppable());
  }

  final AuthRepository _repo;

  void _onEmailChanged(LoginEmailChanged e, Emitter<LoginFormState> emit) {
    emit(state.copyWith(email: EmailInput.dirty(e.value)));
  }

  void _onPasswordChanged(LoginPasswordChanged e, Emitter<LoginFormState> emit) {
    emit(state.copyWith(password: PasswordInput.dirty(e.value)));
  }

  Future<void> _onSubmitted(
    LoginSubmitted event,
    Emitter<LoginFormState> emit,
  ) async {
    if (!state.isValid) return;
    emit(state.copyWith(status: FormzSubmissionStatus.inProgress));

    final result = await _repo.login(
      email: state.email.value,
      password: state.password.value,
    );

    emit(switch (result) {
      ResultSuccess() => state.copyWith(status: FormzSubmissionStatus.success),
      ResultFailure(failure: ValidationFailure(:final errors)) =>
          state.copyWith(
            status: FormzSubmissionStatus.failure,
            email: errors.containsKey('email')
                ? EmailInput.dirty(state.email.value, errors['email']!.first)
                : state.email,
            password: errors.containsKey('password')
                ? PasswordInput.dirty(state.password.value, errors['password']!.first)
                : state.password,
          ),
      ResultFailure(failure: UnauthorizedFailure()) =>
          state.copyWith(
            status: FormzSubmissionStatus.failure,
            serverError: 'Email or password is incorrect',
          ),
      ResultFailure(:final failure) =>
          state.copyWith(
            status: FormzSubmissionStatus.failure,
            serverError: _messageFor(failure),
          ),
    });
  }

  String _messageFor(Failure f) => switch (f) {
    NetworkFailure()      => 'No internet connection',
    TimeoutFailure()      => 'Server is taking too long. Try again.',
    ServerFailure(:final statusCode) => 'Server error ($statusCode)',
    _ => 'Something went wrong',
  };
}
```

### `lib/ui/features/login/view/login_view.dart`

```dart
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:formz/formz.dart';

import '../../../../core/forms/email_input.dart';
import '../../../../core/forms/password_input.dart';
import '../bloc/login_form_bloc.dart';
import '../bloc/login_form_event.dart';
import '../bloc/login_form_state.dart';

class LoginView extends StatelessWidget {
  const LoginView({super.key});

  @override
  Widget build(BuildContext context) {
    return BlocProvider(
      create: (ctx) => LoginFormBloc(repo: ctx.read()),
      child: const _LoginScaffold(),
    );
  }
}

class _LoginScaffold extends StatelessWidget {
  const _LoginScaffold();

  @override
  Widget build(BuildContext context) {
    return BlocListener<LoginFormBloc, LoginFormState>(
      listenWhen: (p, n) => p.status != n.status,
      listener: (ctx, state) {
        if (state.status == FormzSubmissionStatus.success) {
          Navigator.of(ctx).pushReplacementNamed('/home');
        } else if (state.serverError != null) {
          ScaffoldMessenger.of(ctx).showSnackBar(
            SnackBar(content: Text(state.serverError!)),
          );
        }
      },
      child: Scaffold(
        appBar: AppBar(title: const Text('Sign in')),
        body: Padding(
          padding: const EdgeInsets.all(16),
          child: Column(
            children: [
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
              const SizedBox(height: 12),
              BlocSelector<LoginFormBloc, LoginFormState, PasswordInput>(
                selector: (s) => s.password,
                builder: (ctx, password) => TextField(
                  obscureText: true,
                  decoration: InputDecoration(
                    labelText: 'Password',
                    errorText: password.serverError ?? _passwordMsg(password.displayError),
                  ),
                  onChanged: (v) => ctx.read<LoginFormBloc>().add(LoginPasswordChanged(v)),
                ),
              ),
              const SizedBox(height: 24),
              BlocBuilder<LoginFormBloc, LoginFormState>(
                builder: (ctx, state) => FilledButton(
                  onPressed: state.isValid && state.status != FormzSubmissionStatus.inProgress
                      ? () => ctx.read<LoginFormBloc>().add(const LoginSubmitted())
                      : null,
                  child: state.status == FormzSubmissionStatus.inProgress
                      ? const CircularProgressIndicator()
                      : const Text('Sign in'),
                ),
              ),
            ],
          ),
        ),
      ),
    );
  }

  String? _emailMsg(EmailValidationError? e) => switch (e) {
    EmailValidationError.empty     => 'Email is required',
    EmailValidationError.malformed => 'Enter a valid email',
    null                           => null,
  };

  String? _passwordMsg(PasswordValidationError? e) => switch (e) {
    PasswordValidationError.empty    => 'Password is required',
    PasswordValidationError.tooShort => 'At least 8 characters',
    null                             => null,
  };
}
```

`errorText: email.serverError ?? _emailMsg(email.displayError)` is the whole 422 round-trip in one line: server error wins when present, otherwise client-side validator output (or null when the field is still pure).
