# Review heuristics

Run these over service code — someone else's, or your own previous pass. Each is followed
by the class of defect it catches.

## 1. Look for two places that must be kept in sync by hand

One fact in two places will drift, and the missed update **raises nothing**.

Signals:
- An enum and a timestamp expressing the same thing (`QueueStatus` and `QueueStoppedAt`)
- A boolean that negates the meaning of another field (status says "cancelled" while a
  flag says "not confirmed yet")
- `MarkXxx()` / `SetXxx()` methods that only update a derived value and change no fact —
  every call site is bookkeeping, and one omission is an inconsistency

Fix: keep the source of truth, delete the derived one. If both must exist, make one a
projection of the other with exactly one conversion between them.

## 2. Look for combinations that are representable but invalid

Nullable fields whose validity depends on a status field, with nothing enforcing it → union.

Typical defect: a step that was never dispatched gets declared "timed out", because the
silence clock falls back through `??` all the way to `CreatedAt` — so a step that merely sat
behind a long-running one is judged to have gone silent, having never run at all. As a
union, that transition simply does not accept `Pending` as an origin.

## 3. Count how many places maintain the same state

`grep` the assignments to that field. More than two or three, spread across files, means
the state machine has no home.

Typical defect: a retry budget incremented twice — once by the caller, once inside the
transition — so the **effective budget is half the configured value**. Tests miss it
because the test makes the same mistake as the implementation.

## 4. Check whether what preceded an `await` is still valid after it

The world changes during a long wait. Over a multi-minute cold start, the user may have
cancelled and a sweeper may have declared the work dead.

Ask: **when this response arrives, does its object still want it?**

Typical defect: a remote execution is created, but by the time the response lands the queue
has stopped. The handling was "record the id, do not advance the state" — so a later resume
saw "still dispatching, no external side effects" and treated it as a safe retry. **The same
work ran twice, and the first run really had happened.**

## 5. Look for facts the code is pretending to know

Code inventing information it does not have. The usual form is a `??` fallback chain.

```csharp
// Bad: with the first two absent, CreatedAt impersonates "last heard from the executor"
var since = step.LastReportedAt ?? step.DispatchedAt ?? step.CreatedAt;
```

`CreatedAt` is not an execution signal in any sense. The fix is to make the function accept
only inputs that **did** produce a signal:

```csharp
private bool HasSignalTimedOut(RemoteRun run, DateTimeOffset now) =>
    now - (run.LastReportedAt ?? run.DispatchedAt) >= window;
```

Change the signature and the invented fallback disappears on its own.

## 6. Check who writes the sentence the user reads

When one object carries two accounts — one in the state, one in an `Error` column — whoever
reads one cannot see the other.

Rule: **the explanation lands with the state.** `TimedOut(At, Note)` carries its own
sentence, so callers add nothing on the side. And a stop reason taken from the step's own
note beats "terminated with status Failed".

## 7. Naming: avoid unusual words

`Abandon` / `Retire` / `Kick` / `ObserveXxx` all make the reader guess. Prefer `WriteOff` /
`ScheduleDispatch` / `ApplyStatus`.

Name storage discriminators with the domain word. A mismatch usually means a deleted concept
is still casting a shadow.

## 8. Comments: keep only what the code cannot say

Delete: restatements of the code, full histories of past defects, explanations of language
features.

Keep:
- Why something must **not** be done (why a terminal state must not be set early, why this
  call must not be awaited)
- A non-obvious trade-off (why redeliver rather than stop and wait for a human)
- A necessary construction that looks optimisable (why this must materialise as an array)

## 9. Let the compiler be the checklist

Delete the old API outright rather than marking it `[Obsolete]`, then work through the build
errors. Each error is a decision worth re-making — every chance to turn "read a field" into
"ask a question" is in that list.

Follow up with one `grep` to confirm nothing still bypasses the new path.

## 10. Four gates before a change lands

```bash
dotnet build                                        # 0 errors, 0 warnings
dotnet test                                         # green
dotnet ef migrations has-pending-model-changes      # model and migrations agree
dotnet run --project <migrator> -- --validate-only  # no unacknowledged destructive ops
```

The last two are the easiest to skip, and they guard the least recoverable failures.
