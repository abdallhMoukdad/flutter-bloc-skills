---
name: flutter-bloc-async-api
description: Implements the canonical async API call pattern in Bloc — dispatch event, emit loading, call Repository returning `Result<T>` with a typed `Failure`, emit success or typed failure. Use when wiring a Bloc-driven feature to a REST API (typically Laravel), mapping HTTP errors to typed failures, or handling pull-to-refresh and request cancellation. Prerequisite: `flutter-bloc-setup` and `flutter-bloc-feature-pattern`.
metadata:
  model: claude-opus-4-7
  last_modified: Thu, 14 May 2026 00:00:00 GMT
---

# Async API Calls in Bloc

Defines the contract between a Bloc and the data layer: every Repository method returns `Future<Result<T>>`, never throws. The `Result<T>` union carries either the successful `T` or a `Failure` — `Failure` is fixed (not a second generic parameter on `Result`), so the union is parameterised by the success type alone. The Bloc handler emits `Loading`, awaits the repository, and emits `Loaded` or `Failure` by pattern-matching on the `Result`. Failure shapes mirror Laravel's HTTP response taxonomy (especially the 422 validation envelope), so backend errors round-trip cleanly into the UI. Builds on `flutter-bloc-feature-pattern`; the HTTP transport itself is covered by `flutter-use-http-package`.

## Contents
- [Repository contract](#repository-contract)
- [Failure taxonomy](#failure-taxonomy)
- [Event handler shape](#event-handler-shape)
- [Refresh and request cancellation](#refresh-and-request-cancellation)
- [Workflow: Wire a feature to the API](#workflow-wire-a-feature-to-the-api)
- [Applied to Talabat-clone](#applied-to-talabat-clone)
- [Examples](#examples)

## Repository contract

Every Repository method returns `Future<Result<T>>`. **Repositories never throw.** Errors that escape from the HTTP layer (network, JSON, status code) are caught at the Repository boundary and converted into a `Failure` variant. A Bloc handler that calls a Repository never needs `try`/`catch` — the type system already accounts for failure.

`Result` is a sealed `freezed` union with two variants:

```dart
// lib/core/result/result.dart
import 'package:freezed_annotation/freezed_annotation.dart';
import 'failure.dart';

part 'result.freezed.dart';

@freezed
sealed class Result<T> with _$Result<T> {
  const factory Result.success(T data) = ResultSuccess<T>;
  const factory Result.failure(Failure failure) = ResultFailure<T>;
}
```

Why a typed `Result` instead of throwing: in Dart, `throw` is invisible at the call site. The Bloc author has no compile-time signal that `repo.fetch()` might fail. With `Result`, the consumer must pattern-match — an unhandled failure becomes a non-exhaustive `switch` and a compile error.

## Failure taxonomy

`Failure` is a sealed union covering every error category that can come back from a Laravel API. Map each HTTP outcome to exactly one variant.

```dart
// lib/core/result/failure.dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'failure.freezed.dart';

@freezed
sealed class Failure with _$Failure {
  const factory Failure.network() = NetworkFailure;                  // connection refused, DNS, no internet
  const factory Failure.timeout() = TimeoutFailure;                  // server unreachable past deadline
  const factory Failure.unauthorized() = UnauthorizedFailure;        // 401 — trigger logout flow
  const factory Failure.forbidden() = ForbiddenFailure;              // 403
  const factory Failure.notFound() = NotFoundFailure;                // 404
  const factory Failure.validation(Map<String, List<String>> errors) // 422 — Laravel field errors
      = ValidationFailure;
  const factory Failure.server(int statusCode) = ServerFailure;      // 500..599
  const factory Failure.unknown(String message) = UnknownFailure;    // fallback; log + report
}
```

Laravel's 422 response envelope:

```json
{
  "message": "The given data was invalid.",
  "errors": {
    "email": ["The email has already been taken."],
    "phone": ["The phone field is required.", "The phone must be at least 10 digits."]
  }
}
```

A direct cast `json['errors'] as Map<String, List<String>>` throws at runtime — `jsonDecode` returns `Map<String, dynamic>` with `List<dynamic>` values. Map carefully:

```dart
final raw = json['errors'] as Map<String, dynamic>;
final errors = raw.map(
  (k, v) => MapEntry(k, (v as List).cast<String>()),
);
return Failure.validation(errors);
```

Forms surface per-field errors via `flutter-bloc-forms`. For non-form features, render the first message of the first key as a snackbar.

## Event handler shape

The canonical async handler is five lines: emit loading, await the repository, pattern-match the result, emit the next state.

```dart
Future<void> _onLoadRequested(
  RestaurantListLoadRequested event,
  Emitter<RestaurantListState> emit,
) async {
  emit(const RestaurantListState.loading());
  final result = await _repo.fetchNearbyStores(event.location);
  emit(switch (result) {
    ResultSuccess(:final data)    => RestaurantListState.loaded(data),
    ResultFailure(:final failure) => RestaurantListState.failure(_messageFor(failure)),
  });
}
```

`_messageFor(Failure)` is a private presentation-layer mapper — it converts a typed `Failure` into a human-readable string. Keep this mapper in the Bloc, not the View. The View should never know that `Failure.network()` even exists; it just renders `state.message`. The full mapper for every `Failure` variant appears in the Examples section below.

For 401 responses specifically, prefer a `BlocListener<AuthBloc, AuthState>` higher in the tree that triggers a global logout when `Failure.unauthorized()` is observed, rather than rendering an inline error.

`Failure.unknown(message)` is the catch-all branch and `message` is `e.toString()` from the inner exception — useful in logs and crash reports, but *not* user-facing copy. The presentation mapper should map it to a generic fallback like "Something went wrong. Try again." rather than rendering the raw text.

## Refresh and request cancellation

A pull-to-refresh that arrives while an initial load is still in flight should **cancel** the in-flight request, not be silently ignored. Use `bloc_concurrency`'s `restartable()` for refresh handlers.

```dart
on<RestaurantListLoadRequested>(_onLoadRequested, transformer: restartable());
on<RestaurantListRefreshRequested>(_onRefreshRequested, transformer: restartable());
```

`restartable()` makes the second event abort the handler running for the first. The HTTP request itself is not cancelled (unless the underlying client supports `CancellationToken`s) — but the Bloc stops listening, so any state the obsolete handler tries to emit is dropped. This prevents the "stale data flicker" where the cancelled request resolves after the new one and overwrites it.

Do **not** use `droppable()` for refresh — that would silently ignore the user's pull, leaving them stuck staring at stale data while a slow request finishes.

## Workflow: Wire a feature to the API

### Task Progress
- [ ] **Step 1 — Define the DTO.** `freezed` + `json_serializable` data class matching the API response shape exactly. Lives in `lib/data/models/`.
- [ ] **Step 2 — Define the domain model.** Cleaner shape consumed by UI. Lives in `lib/domain/models/`. The Repository transforms DTO → domain.
- [ ] **Step 3 — Add `Result` and `Failure`.** Drop `lib/core/result/result.dart` and `lib/core/result/failure.dart` if they don't already exist (one-time per project).
- [ ] **Step 4 — Implement the Service.** HTTP client wrapper using `package:http`. Returns the raw DTO or throws. Lives in `lib/data/services/`. Setup follows `flutter-use-http-package`.
- [ ] **Step 5 — Implement the Repository.** Wraps the Service in `try`/`catch`. Maps DTO → domain on success. Maps each thrown error to a `Failure` variant. Returns `Result<T>`. Lives in `lib/data/repositories/`.
- [ ] **Step 6 — Implement the Bloc handler.** Emit `Loading`, `await repo.fetch(...)`, pattern-match the `Result`, emit `Loaded` or `Failure`. Use `restartable()` for queries and refreshes.
- [ ] **Step 7 — Register the Repository.** Add it to the root `MultiRepositoryProvider` in `main.dart` (introduced by `flutter-bloc-setup`).
- [ ] **Step 8 — Wire the View.** `BlocBuilder` with a sealed-state `switch` rendering Loading / Loaded / Failure (and Empty if relevant). Run `flutter analyze` → review compile errors → fix non-exhaustive switches → re-run.

## Applied to Talabat-clone

A representative feature: the list of nearby stores on the home screen.

**Endpoint:** `GET /api/stores/nearby?lat=<lat>&lng=<lng>` on the Laravel backend.

**Inputs:** `LocationDto(lat, lng)` derived from device GPS or a saved address. Saved-address selection is `prd.md §2.1`; the "this address is far from your current location" guard is `prd.md §2.2`. The guard check happens in the View before dispatching the load event, not in the Bloc.

**Output:** `List<StoreDto>` on success. The Repository maps DTOs to `Store` domain models and **filters out** stores that fail operational gating (`PRD_states.md §4`):

- `paused_by_merchant == true` → drop.
- `suspension_reason != null` → drop.
- Outside working hours → drop (computed at read time using `working_hours` config; not a stored state).

```dart
// inside StoreRepository.fetchNearbyStores
final dtos = await _api.fetchNearbyStores(loc);
final now = DateTime.now();
final available = dtos
    .where((d) => !d.pausedByMerchant)
    .where((d) => d.suspensionReason == null)
    .where((d) => d.workingHours.contains(now))
    .map((d) => d.toDomain())
    .toList();
return Result.success(available);
```

The home screen calls `RestaurantListBloc.add(RestaurantListLoadRequested(location))` when a saved address is selected. Pull-to-refresh dispatches `RestaurantListRefreshRequested(location)` with the same location. Both handlers use `restartable()`.

When a Laravel form-related call (e.g., placing an order, submitting an address) returns 422, the `Failure.validation` payload is forwarded to the relevant form Bloc per `flutter-bloc-forms` (deferred to wave 3). For non-form 422 cases, render the first message via `_messageFor`.

## Examples

End-to-end implementation of `RestaurantListBloc` consuming `GET /api/stores/nearby`. The Service uses a project-defined `ApiException` that carries the raw status code and parsed body — no regex round-tripping through `http.ClientException.message`.

### `lib/core/api/api_exception.dart`

```dart
/// Thrown by Service implementations on any non-2xx HTTP response.
/// Repositories catch this and map to a `Failure`.
class ApiException implements Exception {
  const ApiException({required this.statusCode, this.body});

  final int statusCode;
  final Map<String, dynamic>? body; // null when body wasn't JSON

  /// Laravel-style validation errors when statusCode == 422, else null.
  Map<String, List<String>>? get validationErrors {
    if (statusCode != 422) return null;
    final raw = body?['errors'];
    if (raw is! Map<String, dynamic>) return null;
    return raw.map(
      (k, v) => MapEntry(k, (v as List).cast<String>()),
    );
  }

  @override
  String toString() => 'ApiException($statusCode)';
}
```

### `lib/data/models/store_dto.dart`

```dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'store_dto.freezed.dart';
part 'store_dto.g.dart';

@freezed
class StoreDto with _$StoreDto {
  const factory StoreDto({
    required String id,
    required String name,
    required String logoUrl,
    required double rating,
    required int ratingsCount,
    required int deliveryMinutes,
    @JsonKey(name: 'paused_by_merchant') required bool pausedByMerchant,
    @JsonKey(name: 'suspension_reason') String? suspensionReason,
  }) = _StoreDto;

  factory StoreDto.fromJson(Map<String, dynamic> json) => _$StoreDtoFromJson(json);
}
```

### `lib/data/services/store_api_client.dart`

```dart
import 'dart:convert';
import 'package:http/http.dart' as http;
import '../../core/api/api_exception.dart';
import '../models/store_dto.dart';

class StoreApiClient {
  StoreApiClient({required http.Client client, required Uri baseUrl})
      : _client = client,
        _baseUrl = baseUrl;

  final http.Client _client;
  final Uri _baseUrl;

  Future<List<StoreDto>> fetchNearbyStores(double lat, double lng) async {
    final uri = _baseUrl.resolve('/api/stores/nearby?lat=$lat&lng=$lng');
    final res = await _client.get(uri, headers: const {'Accept': 'application/json'});

    if (res.statusCode == 200) {
      final body = jsonDecode(res.body) as Map<String, dynamic>;
      // This assumes Laravel's `AnonymousResourceCollection` envelope —
      // `JsonResource::collection($stores)` returns `{"data": [...]}`.
      // A controller that does `return response()->json($stores)` returns
      // a bare array; in that case decode `as List` directly.
      final list = (body['data'] as List).cast<Map<String, dynamic>>();
      return list.map(StoreDto.fromJson).toList();
    }

    // Parse the body if it's JSON (Laravel error responses usually are);
    // fall back to null when the server returned plain text or HTML.
    Map<String, dynamic>? parsedBody;
    try {
      parsedBody = jsonDecode(res.body) as Map<String, dynamic>;
    } catch (_) {/* keep parsedBody == null */}

    throw ApiException(statusCode: res.statusCode, body: parsedBody);
  }
}
```

### `lib/data/repositories/store_repository.dart`

```dart
import 'dart:async';
import 'dart:io';

import '../../core/api/api_exception.dart';
import '../../core/result/failure.dart';
import '../../core/result/result.dart';
import '../../domain/models/location.dart';
import '../../domain/models/store.dart';
import '../services/store_api_client.dart';

class StoreRepository {
  StoreRepository({required StoreApiClient api}) : _api = api;
  final StoreApiClient _api;

  Future<Result<List<Store>>> fetchNearbyStores(Location loc) async {
    try {
      final dtos = await _api.fetchNearbyStores(loc.lat, loc.lng);
      final available = dtos
          .where((d) => !d.pausedByMerchant && d.suspensionReason == null)
          // .where((d) => d.workingHours.contains(now))   // omitted for brevity
          .map(Store.fromDto)
          .toList();
      return Result.success(available);
    } on SocketException {
      return const Result.failure(Failure.network());
    } on TimeoutException {
      return const Result.failure(Failure.timeout());
    } on ApiException catch (e) {
      return Result.failure(_mapApiException(e));
    } catch (e) {
      return Result.failure(Failure.unknown(e.toString()));
    }
  }

  Failure _mapApiException(ApiException e) => switch (e.statusCode) {
    401 => const Failure.unauthorized(),
    403 => const Failure.forbidden(),
    404 => const Failure.notFound(),
    422 => Failure.validation(e.validationErrors ?? const <String, List<String>>{}),
    int s when s >= 500 && s < 600 => Failure.server(s),
    _ => Failure.unknown('HTTP ${e.statusCode}'),
  };
}
```

### `lib/ui/features/restaurant_list/bloc/restaurant_list_state.dart`

The view below switches on this sealed state. See `flutter-bloc-feature-pattern` for the full event/state/bloc scaffolding pattern.

```dart
import 'package:freezed_annotation/freezed_annotation.dart';
import '../../../../domain/models/store.dart';

part 'restaurant_list_state.freezed.dart';

@freezed
sealed class RestaurantListState with _$RestaurantListState {
  const factory RestaurantListState.initial() = RestaurantListInitial;
  const factory RestaurantListState.loading() = RestaurantListLoading;
  const factory RestaurantListState.empty()   = RestaurantListEmpty;
  const factory RestaurantListState.loaded(List<Store> stores) = RestaurantListLoaded;
  const factory RestaurantListState.failure(String message)    = RestaurantListFailure;
}
```

### `lib/ui/features/restaurant_list/bloc/restaurant_list_bloc.dart`

```dart
import 'package:bloc_concurrency/bloc_concurrency.dart';
import 'package:flutter_bloc/flutter_bloc.dart';

import '../../../../core/result/failure.dart';
import '../../../../core/result/result.dart';
import '../../../../data/repositories/store_repository.dart';
import '../../../../domain/models/location.dart';
import 'restaurant_list_event.dart';
import 'restaurant_list_state.dart';

class RestaurantListBloc
    extends Bloc<RestaurantListEvent, RestaurantListState> {
  RestaurantListBloc({required StoreRepository repo})
      : _repo = repo,
        super(const RestaurantListState.initial()) {
    on<RestaurantListLoadRequested>(_onLoad, transformer: restartable());
    on<RestaurantListRefreshRequested>(_onLoad, transformer: restartable());
  }

  final StoreRepository _repo;

  Future<void> _onLoad(
    RestaurantListEvent event,
    Emitter<RestaurantListState> emit,
  ) async {
    final loc = switch (event) {
      RestaurantListLoadRequested(:final location)    => location,
      RestaurantListRefreshRequested(:final location) => location,
      _ => null,
    };
    if (loc == null) return;

    emit(const RestaurantListState.loading());
    final result = await _repo.fetchNearbyStores(loc);
    emit(switch (result) {
      ResultSuccess(:final data) when data.isEmpty =>
          const RestaurantListState.empty(),
      ResultSuccess(:final data) => RestaurantListState.loaded(data),
      ResultFailure(:final failure) =>
          RestaurantListState.failure(_messageFor(failure)),
    });
  }

  String _messageFor(Failure f) => switch (f) {
    NetworkFailure()      => 'No internet connection',
    TimeoutFailure()      => 'Server is taking too long. Try again.',
    UnauthorizedFailure() => 'Please sign in again',
    NotFoundFailure()     => 'No restaurants near this location',
    ValidationFailure(:final errors) => errors.values.first.first,
    ServerFailure(:final statusCode) => 'Server error ($statusCode). Try again.',
    ForbiddenFailure()    => 'Access denied',
    // UnknownFailure.message is from `e.toString()` — useful for logs/crashlytics,
    // not user-facing. Render a generic fallback and report the raw message elsewhere.
    UnknownFailure()      => 'Something went wrong. Try again.',
  };
}
```

### `lib/ui/features/restaurant_list/view/restaurant_list_view.dart`

```dart
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';

import '../bloc/restaurant_list_bloc.dart';
import '../bloc/restaurant_list_state.dart';

class RestaurantListView extends StatelessWidget {
  const RestaurantListView({super.key});

  @override
  Widget build(BuildContext context) {
    return BlocBuilder<RestaurantListBloc, RestaurantListState>(
      builder: (context, state) => switch (state) {
        RestaurantListInitial() => const SizedBox.shrink(),
        RestaurantListLoading() => const Center(child: CircularProgressIndicator()),
        RestaurantListEmpty()   => const Center(child: Text('No restaurants near you')),
        RestaurantListLoaded(:final stores) => ListView.builder(
            itemCount: stores.length,
            itemBuilder: (_, i) => ListTile(
              title: Text(stores[i].name),
              subtitle: Text('${stores[i].deliveryMinutes} min'),
            ),
          ),
        RestaurantListFailure(:final message) => Center(child: Text(message)),
      },
    );
  }
}
```
