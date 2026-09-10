# Domain modelling: records and unions

## Aggregates are immutable

```csharp
public sealed record Job
{
    public required string JobId { get; init; }
    public JobStatus Status { get; init; } = JobStatus.Pending;
    public ImmutableList<JobStep> Steps { get; init; } = [];

    // Pure derivations only
    public JobStep? QueueHead => Steps.FirstOrDefault(step => !step.IsQueueCompleted);
}
```

The cost is that callers must capture the return value. **Bind "write back" and "persist"
to the same call** so that skipping the write-back cannot be expressed:

```csharp
private async Task<Job> Save(Job job, CancellationToken ct = default)
{
    _job = job;                        // write back
    await _store.Save(job, ct);        // persist
    return job;
}

// Caller: every business change ends on this line
job = await Save(JobImplementation.BeginRunning(job, now));
```

## Model state as a union

**When**: a group of nullable fields whose validity depends on some status field, with
nothing enforcing the correspondence.

```csharp
public abstract record StepStage
{
    private StepStage() { }             // seals the union: no cases can be added outside

    public sealed record Pending : StepStage;
    public sealed record Dispatching(DateTimeOffset RequestedAt, int Attempts, string? LastError) : StepStage;
    public sealed record Queued : StepStage;
    public sealed record Running : StepStage;
    public sealed record Cancelling : StepStage;
    public sealed record Completed(string? ResultPath, DateTimeOffset At) : StepStage;
    public sealed record Failed(string Error, DateTimeOffset At) : StepStage;
    public sealed record Cancelled(DateTimeOffset At, string? Note = null) : StepStage;
    public sealed record TimedOut(DateTimeOffset At, string? Note = null) : StepStage;

    public bool IsActive => this is Pending or Dispatching or Queued or Running or Cancelling;
}
```

### The union is state, nothing else

**No behaviour**: transitions live in `JobImplementation`.
**It does not know its own name**: names are produced by the conversions on the way out
(see `pattern-matching.md`).

A name is the union's storage-and-API face, and belongs to the conversion layer. Putting
it on the union makes one type both a domain concept and a presentation concept, and
those two change for different reasons.

### Anti-pattern 1: a general getter on the union

```csharp
// Bad: collapses seven cases into one field access, so callers stop matching
public RemoteRun? GetRun() => this switch { Queued q => q.Run, Running r => r.Run, ... };
```

Once it exists, nothing matches any more. Each site should ask only what it needs.

### Anti-pattern 2: pass-through properties on the aggregate

```csharp
// Bad: a row of pass-throughs — the union, flattened right back out
public string? RemoteRunId => Stage.GetRun()?.Id;
public DateTimeOffset? DispatchedAt => Stage.GetRun()?.DispatchedAt;
public string? Error => Stage.GetNote();
```

These usually appear during a refactor, as a shim for existing call sites — and the shim
is exactly what defeats the union. Keep only **semantic** derivations (`IsActive`, "does
this still count as in flight"), never field access.

### Facts that outlive the state go outside the union

```csharp
public sealed record RemoteRun(
    string Id,
    DateTimeOffset DispatchedAt,
    DateTimeOffset? StartedAt = null,
    DateTimeOffset? LastReportedAt = null,
    string? LastReportedEventId = null);

public sealed record JobStep
{
    public StepStage Stage { get; init; } = new StepStage.Pending();
    public RemoteRun? RemoteRun { get; init; }   // exists from dispatch confirmation onward
}
```

The test: **is it still needed after a terminal state?** The remote run id is — you may
still have to cancel an execution that is somehow alive. The start time is — the UI still
shows it. They are not one step's payload; they are facts about the object.

Forcing them into the cases means four terminal cases each carry a nullable copy of them.
*That* is two sources of truth.

### No status enum alongside the union

The domain uses the stage. The outward name comes from the conversions. Keeping a
`StepStatus` enum means every new case must be added in two places, and missing one
raises nothing.

## What to derive, what to store

Delete anything derivable from existing fields:

| Typical redundant column | What it actually equals |
|---|---|
| A `QueueStatus` enum | `QueueStoppedAt is not null` / `QueueHead is null` / otherwise |
| An `OutcomeConfirmed` flag | whether `Stage` is terminal |
| A `SilentSince` timestamp | `LastReportedAt + silence window` (off by one sweep interval) |
| A retry counter beside `Dispatching.Attempts` | the one the case already carries |

The test: **does it carry independent information, or is it bookkeeping that must be kept
in sync by hand?** Bookkeeping that drifts raises nothing.

Conversely, **not everything related is derivable**. "Left the queue" looks like "reached
a terminal state", but is not: a head that stopped the queue is terminal yet still holds
its slot, and a step being cancelled has yielded its slot without an outcome yet. Store
that one.

## Do not let a signature force you to invent facts

A `??` chain that bottoms out on an unrelated field is code pretending to know something:

```csharp
// Bad: with the first two absent, CreatedAt impersonates "last heard from the executor"
var since = step.LastReportedAt ?? step.DispatchedAt ?? step.CreatedAt;
```

`CreatedAt` is not an execution signal in any sense, and the step that merely sat in a queue
is now judged to have gone silent. The fix is not a better fallback — it is a signature that
only accepts inputs which really did produce a signal:

```csharp
private bool HasSignalTimedOut(RemoteRun run, DateTimeOffset now) =>
    now - (run.LastReportedAt ?? run.DispatchedAt) >= window;
```

Change the signature and the invented fallback disappears on its own. This is the same move
as putting facts outside the union: make the caller pass the thing that exists.

## The explanation lands with the state

When an object carries two accounts of itself — one in the stage, one in a separate `Error`
field — whoever reads one cannot see the other.

```csharp
public sealed record TimedOut(DateTimeOffset At, string? Note = null) : StepStage;
```

The case carries its own sentence, so no caller writes one on the side, and a stop reason
taken from the step's own note beats "terminated with status Failed".

## Transitions live in Implementation

```csharp
public static Job RequestCancel(Job job, Guid stepId, DateTimeOffset at)
{
    var step = Require(job, stepId);

    return Replace(job, step with
    {
        Stage = step.Stage switch
        {
            // Already at the executor: wait for its confirmation, keep the books open
            StepStage.Queued or StepStage.Running => new StepStage.Cancelling(),
            StepStage.Cancelling cancelling => cancelling,
            // Never reached it: no callback is coming, so waiting is pointless
            StepStage.Pending or StepStage.Dispatching => new StepStage.Cancelled(at),
            _ => throw Illegal(step, nameof(RequestCancel)),
        },
    });
}
```

Each transition accepts only its legal origins. The rule catches defects on its own:
when scattered assignments are rewritten as transitions, "this object was moved twice"
surfaces as a throw on the second move.

Replace a child **by position**, never remove-then-add — list order is often business
order (a FIFO queue), and one reordering changes the head:

```csharp
private static Job Replace(Job job, JobStep step)
{
    var index = job.Steps.FindIndex(candidate => candidate.Id == step.Id);
    return job with { Steps = job.Steps.SetItem(index, step) };
}
```
