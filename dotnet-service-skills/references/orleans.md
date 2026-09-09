# Orleans: when to use it, and how

## When an actor is worth it

One test: **is there a consistency boundary that must be serialised, and can one key
enclose it?**

A job queue qualifies: enqueue, dispatch, cancel, status report and settlement for one
job must be serialised or FIFO order and accounting both break — and `jobId` is naturally
that key.

Do not reach for it when:
- The work is stateless request/response (use a plain service)
- The boundary is a *batch*, not one entity (use a transaction or a pessimistic lock)
- You just want a background timer (use a `BackgroundService`)

## Turn-based concurrency: await does not release the activation

**The easiest thing to get wrong.** Grains are non-reentrant by default and handle one
request at a time — but `await` does **not** yield the activation. A grain method that
awaits a 300-second remote call blocks **every** other call to that grain — cancel,
status report, new enqueue — for those 300 seconds.

The fix: **move the long wait into another grain.**

```csharp
// Aggregate grain: record the intent and return immediately
private void ScheduleDispatch() =>
    GrainFactory.GetGrain<IJobDispatcherGrain>(this.GetPrimaryKeyString())
        .ScheduleDispatch().Ignore();
```

Three constraints come with it:

1. **One-way notification, never awaited.** Both grains are non-reentrant and the callee
   calls back into you — awaiting in both directions is a deadlock.
2. **The callee's entry point registers a timer and returns.** The real work happens in
   the timer callback, so the "notification" genuinely returns at once.
3. **User-initiated short calls stay on the original grain.** Move cancel to the
   dispatcher too and it queues behind that 300-second dispatch — which is exactly the
   problem the split was meant to solve.

## Reminders vs timers

| | Reminder | Timer |
|---|---|---|
| Durable | Yes, survives a cluster restart | No, dies with the activation |
| Minimum period | 1 minute | none |
| Use for | Backstop: settlement, sweeps, declaring things dead | Low-latency progress while active |

Use both: **a reminder as a backstop, edge-triggered progress for everything else.**
Queue progress is kicked directly by enqueue, terminal report and backoff expiry; the
reminder only covers "all of those kicks were lost". Do not poll on a reminder — a
one-minute delay is user-visible, and it is a durable write.

A reminder needs a retirement condition, or one entity wakes up every minute forever:

```csharp
if (!await _maintenance.HasPendingWork(jobId))
{
    await RemoveReminder();
    DeactivateOnIdle();
}
```

`HasPendingWork` should be a **read model** (one SQL query). Do not load the aggregate for it.

> "When it may stop" matters as much as "when it must run". If one step's terminal state
> is never confirmed, `HasPendingWork` stays true forever and the reminder never retires —
> a leak that raises nothing.

## When not to use Streams

Orleans Streams fit "one event, many subscribers, delivery must be durable". They do not fit:

- **Pushing to a single client connection.** Use SSE/WebSocket directly; a stream in the
  middle is one more hop.
- **When the real event source is an external system.** Subscribing may have side effects.
  A common shape: subscribing to a remote event feed also wakes a remote instance that had
  been reclaimed, costing a multi-minute cold start. Pick an explicitly **side-effect-free**
  read/liveness path instead of reusing the subscription that has side effects.
- **Driving state.** State must be advanced by authoritative reports, never by replaying
  history. Archived/replayed events render the past; they must not set terminal states.

## Persistence: do not default to grain persistence

Use your own database (EF plus optimistic concurrency) whenever any of these hold:

- You must query by something other than the key (list pages, sweeps, reconciliation)
- You need cross-aggregate transactions (aggregate state and ledger rows committed together)
- You need database-level guarantees: foreign keys, unique constraints
- The data must be readable directly by BI or reconciliation scripts

Put a `Version` column on the aggregate row as the optimistic-concurrency token, with the
grain as its sole writer. It is a little more code than grain persistence, and it buys a
database that SQL can actually ask questions of.

## Three concrete traps

**Collection expressions plus `IReadOnlyList<T>` have no codec.**

```csharp
// With a target type of IReadOnlyList<T> the compiler synthesises <>z__ReadOnlyList<T>;
// Orleans has no copier for it and throws CodecNotFoundException on same-silo deep copy
Steps = steps.Select(ToGrainDto).ToArray(),     // must materialise as an array
```

**The default `ResponseTimeout` is 30 seconds.** Any grain call that might exceed it needs
either a configuration change or — more likely — to stop being a synchronous call at all
(see the long-wait section).

**`RegisterGrainTimer` throws outside an Orleans context.** A hand-constructed grain in a
unit test cannot reach it. Either move the scheduling out of the method under test, or
have the method return "how long to wait" and let the caller register.

## Testing a grain

Give it one seam rather than an "explicit state" overload on every method worth testing:

```csharp
internal void AttachStateForTests(Job job) => _job = job;
internal Job StateForTests => _job!;      // the aggregate is immutable, so the read side matters too
```

The read side is not optional: every change hands the grain a **different** instance, so a
test holding the one it built sees nothing.
