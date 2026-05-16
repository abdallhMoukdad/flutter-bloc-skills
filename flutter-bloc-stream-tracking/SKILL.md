---
name: flutter-bloc-stream-tracking
description: Subscribes to a real-time stream (WebSocket, Pusher, Firestore) inside a Bloc via `emit.forEach` and emits a new state on every frame. Use when implementing live order tracking, driver location updates, real-time chat, presence indicators, or any feature where the UI must reflect server-pushed events. Prerequisite: `flutter-bloc-setup` and `flutter-bloc-feature-pattern`.
metadata:
  model: claude-opus-4-7
  last_modified: Thu, 14 May 2026 00:00:00 GMT
---

# Real-Time Streams in Bloc

Couples a `Stream<T>` from the data layer to a Bloc's state machine. The Bloc consumes the stream inside an event handler using `await emit.forEach(...)`, which ties the subscription's lifetime to the handler's lifetime and the Bloc's `close()` — no manual `StreamSubscription` to track, no leaks. Connection lifecycle (connect, disconnect, reconnect-with-backoff) is explicit events on the Bloc; the transport plumbing lives in the Repository. Builds on `flutter-bloc-feature-pattern`.

## Contents
- [Stream subscription pattern](#stream-subscription-pattern)
- [Connection lifecycle](#connection-lifecycle)
- [State design for streams](#state-design-for-streams)
- [Workflow: Wire a real-time stream into a Bloc](#workflow-wire-a-real-time-stream-into-a-bloc)
- [Applied to Talabat-clone](#applied-to-talabat-clone)
- [Examples](#examples)

## Stream subscription pattern

Inside an event handler, `await emit.forEach(stream, onData: ..., onError: ...)` keeps the handler alive for as long as the stream yields. When the Bloc closes — route pop, app background, hot restart — the subscription is cancelled automatically. There is **no `StreamSubscription` field** on the Bloc, and **no `cancel()` call** in `close()`.

```dart
Future<void> _onConnect(
  TrackingConnectRequested event,
  Emitter<OrderTrackingState> emit,
) async {
  emit(const OrderTrackingState.connecting());
  await emit.forEach<DriverLocation>(
    _repo.subscribeToDriver(event.orderId),
    onData: (loc) => OrderTrackingState.connected(loc),
    onError: (err, _) => OrderTrackingState.disconnected(reason: err.toString()),
  );
}
```

Compare with the older `stream.listen` approach:

```dart
// AVOID — manual lifetime management
StreamSubscription? _sub;

Future<void> _onConnect(/* ... */) async {
  _sub?.cancel();
  _sub = _repo.subscribeToDriver(...).listen(
    (loc) => emit(...),         // emit-after-close hazard
    onError: (e) => emit(...),
  );
}

@override
Future<void> close() {
  _sub?.cancel();               // easy to forget
  return super.close();
}
```

`emit.forEach` eliminates two classes of bug: (1) emit-after-close (where `.listen` calls back after the Bloc is gone) and (2) leaked subscriptions in feature-with-route Blocs.

## Connection lifecycle

A stream-driven Bloc has at least two explicit events:

| Event | Triggered by | Effect |
|---|---|---|
| `ConnectRequested(id)` | View `initState` after `BlocProvider` mounts; or after a `RetryRequested` | Start `emit.forEach` on the stream |
| `DisconnectRequested` | View `dispose`; or business rule (e.g., Order moved past `delivered`) | Calls `_repo.disconnect(id)` which closes the upstream stream. `emit.forEach` in `_onConnect` then completes naturally and emits `Closed` via its own epilogue. |

**Crucial:** `_onDisconnect` **cannot** terminate `_onConnect`'s `emit.forEach` by just emitting `Closed` itself. Each event handler gets its own `Emitter` instance; marking one as done has no effect on the other's stream subscription. The Repository owns the stream's lifetime — the only correct way to stop tracking is to close the stream at the source.

**Reconnection with backoff lives in the Repository**, not the Bloc. The Repository wraps the raw stream so transient transport errors are caught, retried with exponential backoff, and capped at a maximum attempt count. From the Bloc's perspective, the stream stays "open" across transient failures. A `Reconnecting` state is reachable only if the Repository explicitly emits a reconnecting sentinel (e.g., a wrapper type like `Either<Reconnecting, DriverLocation>`); the simplest implementations just sleep silently between retries and never surface a `Reconnecting` state to the Bloc.

Use `transformer: restartable()` on the `ConnectRequested` handler so dispatching `ConnectRequested` again — say, after a navigation pop-then-push — cancels the previous handler cleanly. The `if (!emit.isDone) emit(Closed)` epilogue after `emit.forEach` exists so a `restartable()` cancellation (where the prior emitter is already done) doesn't leak a spurious `Closed` state during a normal restart.

## State design for streams

Do **not** reuse `Loading` / `Loaded` from API-call states for streams. They mean different things: API `Loaded` is terminal, stream `Connected` is ongoing. Use distinct variants.

```dart
@freezed
sealed class OrderTrackingState with _$OrderTrackingState {
  const factory OrderTrackingState.initial() = TrackingInitial;
  const factory OrderTrackingState.connecting() = TrackingConnecting;
  const factory OrderTrackingState.connected(DriverLocation location) = TrackingConnected;
  const factory OrderTrackingState.reconnecting({required String reason}) = TrackingReconnecting;
  const factory OrderTrackingState.disconnected({required String reason}) = TrackingDisconnected;
  const factory OrderTrackingState.closed() = TrackingClosed; // terminal: stream is done
}
```

The View's `switch` over this state is exhaustive — adding a `Reconnecting` variant later forces every consumer to handle it explicitly.

## Workflow: Wire a real-time stream into a Bloc

### Task Progress
- [ ] **Step 1 — Add Repository stream method.** `Stream<T> subscribeTo<...>(args)` returning a hot or cold `Stream`. Lives in `lib/data/repositories/`.
- [ ] **Step 2 — Wrap transient errors.** Inside the Repository, transform the raw transport stream so transient disconnects do not surface — emit a `Reconnecting` sentinel and retry with backoff. Only fatal errors propagate.
- [ ] **Step 3 — Define stream events.** Sealed `<Feature>Event` with `ConnectRequested(id)`, `DisconnectRequested`, and optionally `RetryRequested`.
- [ ] **Step 4 — Define stream states.** Sealed `<Feature>State` with `Initial`, `Connecting`, `Connected(data)`, `Reconnecting(reason)`, `Disconnected(reason)`, `Closed` variants. Do not reuse `Loaded` / `Failure` from API skills.
- [ ] **Step 5 — Implement the handler with `emit.forEach`.** Use `transformer: restartable()` so a second `ConnectRequested` cancels the first. Never store a `StreamSubscription` on the Bloc.
- [ ] **Step 6 — Wire `DisconnectRequested`.** The handler calls `await _repo.disconnect(id)` which closes the upstream stream. `_onConnect`'s `emit.forEach` then completes naturally and emits `Closed` via its own epilogue. Do **not** just emit `Closed` from `_onDisconnect` — each handler has its own `Emitter`, so it can't terminate another handler's stream subscription.
- [ ] **Step 7 — Dispatch from the View lifecycle.** `BlocProvider.create` should add `ConnectRequested(id)` immediately; `BlocListener` for `Closed` can pop the route or show a summary.
- [ ] **Step 8 — Feedback loop.** Run on a device → background the app → foreground it → verify the Bloc resumes (`reconnecting → connected`) rather than wedges in `Disconnected`. If it wedges, the issue is almost always the Repository swallowing a fatal error as transient or vice versa.

## Applied to Talabat-clone

Two real-time features map to this pattern.

### Order tracking (`OrderTrackingBloc`)

Per `PRD_states.md §1`, an Order's driver-tracking window is open only while the Order is in `picked_up` or `in_transit`. Before `picked_up` there is no driver assigned; after `delivered` / `failed_delivery` / `cancelled` the assignment is over.

```
Order state         → OrderTrackingBloc behavior
─────────────────────────────────────────────────
awaiting_driver     → not yet — show "looking for driver"
picked_up           → add ConnectRequested(orderId)
in_transit          → still subscribed; frames flow
delivered           → add DisconnectRequested → emit Closed → show summary
failed_delivery     → add DisconnectRequested → emit Closed → trigger refund flow
cancelled           → add DisconnectRequested → emit Closed
```

The `OrderBloc` (separate, not in this skill) owns the Order state transitions. `OrderTrackingBloc` is a child route's Bloc; the parent screen reacts to `OrderBloc`'s state changes via `BlocListener` and dispatches the tracking lifecycle events.

### Chat thread (`ChatThreadBloc`)

`PRD_states.md §15` makes the chat window **time-computed**, not state-stored: open iff `Order.state ∈ {picked_up, in_transit}` OR `Order.state == delivered AND now() < delivered_at + 24h`. The chat Bloc subscribes when this predicate is true and unsubscribes when it flips. Per-message translation status is the `Message` model's async job lifecycle — out of scope for this Bloc.

## Examples

### `lib/data/repositories/order_repository.dart` (excerpt)

```dart
import 'dart:async';

/// Real transport may be Pusher, WebSocket, or Firestore. Backoff/retry,
/// max-attempts, and per-orderId kill-switches are the Repository's
/// responsibility — the Bloc only consumes the resulting `Stream`.
class OrderRepository {
  OrderRepository({required this.transport, this.maxRetries = 5});
  final RealtimeTransport transport;
  final int maxRetries;

  // One StreamController per active subscription; closing the controller
  // is the kill switch that propagates "stop tracking" to the consumer.
  final Map<String, StreamController<DriverLocation>> _active = {};

  Stream<DriverLocation> subscribeToDriver(String orderId) {
    // If somehow a previous subscription is still around, close it first.
    _active[orderId]?.close();

    final controller = StreamController<DriverLocation>();
    _active[orderId] = controller;
    unawaited(_pumpWithRetry(orderId, controller));
    return controller.stream;
  }

  /// Closes the upstream stream for `orderId`. `_onConnect` in the Bloc
  /// sees `emit.forEach` complete and emits its own `Closed` state.
  Future<void> disconnect(String orderId) async {
    final controller = _active.remove(orderId);
    await controller?.close();
  }

  Future<void> _pumpWithRetry(
    String orderId,
    StreamController<DriverLocation> sink,
  ) async {
    var attempt = 0;
    while (!sink.isClosed) {
      try {
        await for (final loc in transport.driverLocationStream(orderId)) {
          if (sink.isClosed) return;
          sink.add(loc);
          attempt = 0; // reset on any successful frame
        }
        // Upstream ended normally — close the sink so `emit.forEach` exits.
        if (!sink.isClosed) await sink.close();
        return;
      } on TransientTransportException catch (e) {
        attempt++;
        if (attempt > maxRetries) {
          // Promote a transient failure to fatal after N attempts.
          if (!sink.isClosed) sink.addError(FatalTransportException(e.toString()));
          if (!sink.isClosed) await sink.close();
          return;
        }
        final backoff = Duration(milliseconds: 500 * (1 << (attempt - 1).clamp(0, 5)));
        await Future<void>.delayed(backoff);
      }
    }
  }
}
```

### `lib/ui/features/order_tracking/bloc/order_tracking_bloc.dart`

```dart
import 'package:bloc_concurrency/bloc_concurrency.dart';
import 'package:flutter_bloc/flutter_bloc.dart';

import '../../../../data/repositories/order_repository.dart';
import '../../../../domain/models/driver_location.dart';
import 'order_tracking_event.dart';
import 'order_tracking_state.dart';

class OrderTrackingBloc
    extends Bloc<OrderTrackingEvent, OrderTrackingState> {
  OrderTrackingBloc({required OrderRepository repo})
      : _repo = repo,
        super(const OrderTrackingState.initial()) {
    on<TrackingConnectRequested>(_onConnect, transformer: restartable());
    on<TrackingDisconnectRequested>(_onDisconnect);
  }

  final OrderRepository _repo;

  Future<void> _onConnect(
    TrackingConnectRequested event,
    Emitter<OrderTrackingState> emit,
  ) async {
    emit(const OrderTrackingState.connecting());
    await emit.forEach<DriverLocation>(
      _repo.subscribeToDriver(event.orderId),
      onData: (loc) => OrderTrackingState.connected(loc),
      onError: (err, _) =>
          OrderTrackingState.disconnected(reason: err.toString()),
    );
    // Stream completed normally — driver delivered, etc.
    if (!emit.isDone) emit(const OrderTrackingState.closed());
  }

  Future<void> _onDisconnect(
    TrackingDisconnectRequested event,
    Emitter<OrderTrackingState> emit,
  ) async {
    // Route disconnection through the Repository: closing the upstream stream
    // makes `_onConnect`'s `emit.forEach` complete naturally, at which point
    // `_onConnect`'s `if (!emit.isDone) emit(Closed)` epilogue fires.
    // Do NOT just emit `Closed` here — each handler has its own Emitter, so
    // marking this one done has zero effect on the other's subscription.
    await _repo.disconnect(event.orderId);
  }
}
```

### `lib/ui/features/order_tracking/view/order_tracking_view.dart`

```dart
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';

import '../bloc/order_tracking_bloc.dart';
import '../bloc/order_tracking_event.dart';
import '../bloc/order_tracking_state.dart';

class OrderTrackingView extends StatelessWidget {
  const OrderTrackingView({super.key, required this.orderId});
  final String orderId;

  @override
  Widget build(BuildContext context) {
    return BlocProvider(
      create: (ctx) => OrderTrackingBloc(repo: ctx.read())
        ..add(TrackingConnectRequested(orderId)),
      child: BlocConsumer<OrderTrackingBloc, OrderTrackingState>(
        listenWhen: (_, n) => n is TrackingClosed,
        listener: (ctx, _) => Navigator.of(ctx).pop(), // back to order summary
        builder: (context, state) => switch (state) {
          TrackingInitial() || TrackingConnecting() =>
              const Center(child: CircularProgressIndicator()),
          TrackingConnected(:final location) =>
              _DriverMap(location: location),
          TrackingReconnecting(:final reason) =>
              _Banner(text: 'Reconnecting: $reason'),
          TrackingDisconnected(:final reason) =>
              _ErrorView(message: reason),
          TrackingClosed() => const SizedBox.shrink(),
        },
      ),
    );
  }
}
```

`emit.forEach` plus `restartable()` plus a single `Closed` sentinel: that is the whole pattern. Everything else — backoff timing, transport choice, frame format — lives one layer down in the Repository where Bloc-level skills have no business reaching.
