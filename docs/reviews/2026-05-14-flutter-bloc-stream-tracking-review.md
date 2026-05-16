# Review — `flutter-bloc-stream-tracking`

**File:** `~/.claude/skills/flutter-bloc-stream-tracking/SKILL.md`
**Reviewer:** Opus (independent)
**Date:** 2026-05-14
**Scope:** Dart/Bloc correctness, state design, internal consistency with the family, PRD alignment, frontmatter hygiene.

---

## Verdict

**Ship with one fix.** The skill teaches the right pattern (`emit.forEach` + `restartable()` + Repository-owned reconnection + distinct stream states), and its largest technical claims (`emit.forEach` API shape, automatic subscription cancellation on Bloc close, `emit.isDone` semantics) are verified against the bloc package source. PRD references are accurate to the letter.

But the `OrderTrackingBloc` example in §Examples contains a **subtle correctness bug** in `_onDisconnect`, paired with an explanatory comment that asserts behaviour the bloc library does not provide. A reader who copies this example will get a Bloc that *appears* to disconnect but resumes emitting `Connected` on the next stream frame. This single fix is required before the skill is honest about what it claims to teach.

Counts: 1 Critical, 3 Important, 4 Nits, 5 Done-well.

---

## Critical

### C1. `_onDisconnect` does not actually disconnect — bad example + false claim (lines 197–204)

The example:

```dart
Future<void> _onDisconnect(
  TrackingDisconnectRequested event,
  Emitter<OrderTrackingState> emit,
) async {
  emit(const OrderTrackingState.closed());
  // The in-flight `_onConnect` will exit on its next frame because the
  // emitter is now done; `restartable()` would also cancel a fresh attempt.
}
```

This is wrong on two counts.

**(a) Each event handler gets its own `Emitter` instance.** Per `packages/bloc/lib/src/bloc.dart`:

```dart
final emitter = _Emitter(onEmit);
// ...
await handler(event as E, emitter);
```

and per `packages/bloc/lib/src/emitter.dart`, `_disposables` (which holds the `subscription.cancel` thunks created by `emit.forEach`) lives on each `_Emitter` instance, not on the Bloc. The `_onDisconnect` emitter's `isDone` is local to that handler call; setting it has no effect on the `_Emitter` used by `_onConnect`. The `await emit.forEach(...)` inside `_onConnect` keeps its subscription alive.

**(b) The reference to `restartable()` doesn't apply here.** `restartable()` is registered on `TrackingConnectRequested`, so it cancels the prior `_onConnect` only when a *new* `TrackingConnectRequested` arrives. `TrackingDisconnectRequested` flows through a different handler with no transformer — it cannot cancel the connect handler.

**Observable consequence.** After `add(TrackingDisconnectRequested())`:
1. `_onDisconnect` emits `Closed`.
2. `_onConnect`'s `emit.forEach` is still subscribed.
3. The next `DriverLocation` frame fires `onData`, which calls `emit(Connected(...))` on `_onConnect`'s still-live emitter.
4. Final state: `Connected`, not `Closed`. The view's `BlocConsumer.listenWhen: (_, n) => n is TrackingClosed` fires briefly, pops the route — and then the parent (if still alive) is bombarded with `Connected` frames against a dead orderId.

**Fixes (pick one, the skill should pick one and recommend it):**

1. **Repository-driven termination.** Have the Repository expose a `disconnect()`/cancellation handle. `_onDisconnect` calls `_repo.disconnect(event.orderId)` (or similar), which causes the upstream `transport.driverLocationStream(...)` to close. `emit.forEach` completes, the `if (!emit.isDone) emit(Closed)` line at the end of `_onConnect` fires Closed. Clean.

2. **Unified event under one `restartable()` handler.** Use a single `TrackingChanged(targetState)` event with `restartable()`; dispatching `TrackingChanged.disconnected` cancels the previous handler. Loses the "two events for two intents" clarity but is correct.

3. **Internal `StreamController` as a kill-switch.** `_onConnect` listens to `_repo.subscribeToDriver(...).takeUntil(_killSwitch.stream)`. `_onDisconnect` does `_killSwitch.add(null)` and emits `Closed`. Adds the kind of bookkeeping the skill is trying to avoid (`_killSwitch` is morally identical to a `StreamSubscription` field), so probably the wrong fix for this skill's voice.

The §Workflow checklist Step 6 — "The handler should add the event itself when the upstream business rule says to stop … Re-emit `Closed` and let the `forEach` exit" — has the same defect. "Re-emit Closed and let the `forEach` exit" assumes the wrong causality direction: `forEach` exits because the *stream* ends, not because state was emitted elsewhere.

This is the single highest-leverage fix in the skill. Either swap the example for fix (1) above, or keep the two-event shape but make `_onDisconnect` actually call into the Repository to terminate the stream.

---

## Important

### I1. `subscribeToDriver`'s reconnection loop has no terminal failure path (lines 143–157)

```dart
Stream<DriverLocation> subscribeToDriver(String orderId) async* {
  var attempt = 0;
  while (true) {
    try {
      yield* transport.driverLocationStream(orderId);
      return; // upstream completed normally
    } on TransientTransportException {
      attempt++;
      final backoff = Duration(milliseconds: 500 * (1 << attempt.clamp(0, 5)));
      await Future<void>.delayed(backoff);
    }
  }
}
```

Two issues:

- `attempt` is never reset on a successful reconnect. If the stream reconnects, runs for an hour, then fails transiently again, the backoff calculation uses the stale (already-clamped-to-32s) attempt count rather than starting fresh. Move `attempt = 0` to after a successful yield batch — but that's hard with `yield*`. Simpler: hoist the reconnect logic to a wrapper that resets on each `onData`.
- **There is no maximum-attempt cap.** A persistent transport outage means the Repository busy-loops forever, capped only at the 32-second backoff ceiling. This contradicts §Workflow Step 8 which talks about distinguishing "fatal" from "transient" errors — there's no mechanism here for a transient error to ever *become* fatal. Add an `if (attempt >= maxAttempts) throw FatalTransportException(...)` after the increment, and document the threshold.

Also: the skill body (line 71) says "Surface a `Reconnecting` state only when the Repository emits a reconnecting sentinel". The example Repository emits no such sentinel — it just silently sleeps. So the `Reconnecting` state defined in `OrderTrackingState` (line 85) and rendered in the View (line 235) is never reachable from this example. Either show the sentinel-emission pattern (probably via `Stream<Either<DriverLocation, ReconnectingTick>>` or two-arm sealed event type) or drop `Reconnecting` from the state. Don't define a state variant the example cannot produce.

### I2. `restartable()` for a connect event is defensible but the rationale is incomplete (lines 73, 100)

The skill justifies `transformer: restartable()` on `TrackingConnectRequested` with "a second `ConnectRequested` cancels the previous handler cleanly". That's correct. But `restartable()` on a long-lived stream subscription has a subtlety the skill should call out: when the new event arrives, `restartable()` cancels the prior emitter, which closes its `_disposables` (the stream subscription), which causes the prior `forEach`'s Future to complete with `isDone == true`. The `if (!emit.isDone) emit(Closed)` epilogue (line 194) is exactly what prevents a spurious `Closed` from leaking out during a restart. Worth mentioning explicitly — this is the kind of detail that distinguishes "I copied the example" from "I understand the pattern". One sentence near line 194 ("`!emit.isDone` skips the Closed emit during a `restartable()` cancellation, where the emitter is already done but the stream did not end naturally") would do it.

### I3. View example: `BlocConsumer.listenWhen` + closed-state navigation has a stale-context risk (lines 227–229)

```dart
listenWhen: (_, n) => n is TrackingClosed,
listener: (ctx, _) => Navigator.of(ctx).pop(),
```

When `_onDisconnect` (post-fix from C1) emits `Closed`, the listener pops the route. The route's `BlocProvider.create` callback (line 225) is responsible for the Bloc, so the Bloc will be closed by the framework when the route disposes — fine. But until C1 is fixed, the pop triggers and *then* the still-running `_onConnect` emits `Connected`, which the BlocBuilder tries to process against a route that's mid-pop. Whether this throws depends on whether `BlocConsumer`'s widget is disposed before the next emit lands. This is a downstream symptom of C1, not a separate bug, but reinforces why C1 must be fixed.

Also: `_DriverMap`, `_Banner`, `_ErrorView` are referenced but undefined. Fine for a skill example, but Step 7's checklist doesn't tell the reader they're skeletons. A one-line note "underscore-prefixed widgets are stubs you supply" would help readers who copy literally.

---

## Nits

### N1. Inconsistent event-naming convention with `flutter-bloc-feature-pattern`

`flutter-bloc-feature-pattern` uses past-tense events (`CartCleared`, `CartItemAdded`, `CartItemQtyChanged`). This skill uses `*Requested` suffix (`TrackingConnectRequested`, `TrackingDisconnectRequested`, `RetryRequested`). Both forms are present in the feature-pattern skill's text (it mentions `Submit` events that are submission *requests*), so the family doesn't have a single mandate — but readers stepping between skills will notice the wobble. Pick one and apply it uniformly across the family, or add a one-liner in `feature-pattern` saying "request-shaped events end in `Requested`, fact-shaped events use past tense".

### N2. The event class is referenced but never shown (lines 162–177, 226)

The example imports from `order_tracking_event.dart` and uses `TrackingConnectRequested(orderId)` positionally. For a skill whose §Workflow Step 3 says "Sealed `<Feature>Event` with `ConnectRequested(id)`", a `@freezed`-annotated event file would close the loop — same shape as the state file already shown.

### N3. Workflow step count drift (line 95–103)

The design doc (§5.1) said "7 steps". The skill ships 8 steps. Not a problem in itself — the extra "Feedback loop" step is good — but the design doc should be updated or the skill annotated, otherwise the family-level inventory drifts. Likewise the skill's own §Contents lists "Workflow: Wire a real-time stream into a Bloc" but not a step count.

### N4. "When the Bloc closes, the subscription is cancelled automatically" — true, but worth a forward pointer

The claim on line 23 is correct (verified: `Bloc.close()` iterates `_emitters` and calls `cancel()` on each, which fires `_disposables` which cancels the stream subscriptions). Readers may want to know *how* — a parenthetical "(Bloc.close iterates active emitters and cancels their `emit.forEach` subscriptions)" would head off the inevitable "but how does it know?" reading.

### N5. Frontmatter compliance (lines 1–7)

YAML parses, `name` matches directory, description is a single sentence in triggers-only voice ("Use when…"). Length is at the upper edge of "single sentence" — three sub-clauses joined by sentences. Acceptable. The `last_modified` is `Sat, 09 May 2026 00:00:00 GMT` which matches the design doc's ship date; if you ever edit the skill, bump it.

---

## Done well

### D1. The `emit.forEach` API claim is precisely correct

Verified against `packages/bloc/lib/src/emitter.dart`. Signature is `forEach<T>(Stream<T>, {required State Function(T) onData, State Function(Object, StackTrace)? onError})`, returns `Future<void>` which "completes when the event handler is cancelled or when the provided stream has ended". The skill's usage (lines 31–35) matches this exactly. The optional `onError` is correctly provided in the example, which avoids the documented foot-gun where "if `onError` is omitted, any errors on this stream are considered unhandled, and will be thrown by `forEach`". Subtle and right.

### D2. `emit.isDone` is a real, public getter

`bool get isDone => _isCanceled || _isCompleted;` — correctly used at line 194 as a guard against emitting after cancellation. This is exactly the canonical use.

### D3. State design distinguishes stream lifecycle from API lifecycle

The argument in §State design for streams (lines 75–91) — that `Connecting`/`Connected`/`Reconnecting`/`Disconnected`/`Closed` should not be conflated with `Loading`/`Loaded`/`Failure` from API skills — is correct and well-reasoned. A `Loaded` state for a query has "this is the final answer" semantics; `Connected` has "this is the current frame in an ongoing stream" semantics. The `Closed` terminal variant is a nice touch that the View example uses well (modulo C1).

### D4. Repository-owned reconnection is the right architectural call

Pushing exponential backoff and transport-layer retry into the Repository, leaving the Bloc to think in terms of business-level connection lifecycle, is correct and matches what the family's design doc §3.5 implies ("Repository / Service / Domain Model layers stay identical"). The "Bloc has no business reaching one layer down" line at the end of §Examples is a quotable principle.

### D5. PRD references are precise

- §1: `picked_up`, `in_transit`, `delivered`, `failed_delivery`, `cancelled` all exist as documented. The mapping table (lines 113–122) is faithful to the state machine in `PRD_states.md:33-53`. ✓
- §15: "time-computed, not state-stored" plus the `picked_up | in_transit ∪ (delivered AND now() < delivered_at + 24h)` predicate is reproduced exactly from `PRD_states.md:300-307`. ✓
- The note that translation status is the Message model's lifecycle (out of scope here) matches `PRD_states.md:307`. ✓

These references will not bit-rot silently — they cite section numbers and reproduce the predicates verbatim, so a future PRD edit will create a visible mismatch.

---

## Recommended actions, in priority

1. **C1** — Rewrite `_onDisconnect` and the comment that explains it. Either route disconnection through the Repository (preferred), or restructure events under a single `restartable()` handler. Update §Workflow Step 6 to match.
2. **I1** — Add a maximum-attempt cap and reset-on-success to `subscribeToDriver`; either show a `Reconnecting`-sentinel emission or drop the `Reconnecting` state variant.
3. **I2** — One sentence at line 194 explaining why `!emit.isDone` is the right guard for a `restartable()`-cancelled emitter.
4. **N1, N2, N4** — Cosmetic polish; address opportunistically.

After C1, this is a strong skill. The `emit.forEach` + `isDone` story is told accurately, the PRD alignment is precise, and the separation between stream and API state taxonomies is the right teaching. The single bug is in a single example function.
