# Pattern matching

## Each site asks only what it needs

A union's value is that call sites state which cases they handle. Reaching around it with
a getter puts you back on nullable fields.

```csharp
// Bad: whether some field happens to be readable
if (head.DispatchedAt is not null) { ... }

// Good: what it is actually asking
if (head.RemoteRun is { } dispatched) { ... }

// Bad
if (head.Stage.GetAttempts() >= maxAttempts) { ... }

// Good: the retry budget only means anything while dispatching — origin and value together
if (head.Stage is StepStage.Dispatching spent && spent.Attempts >= maxAttempts)
{
    Stop(spent.LastError ?? $"could not dispatch after {spent.Attempts} attempts");
}
```

Binding the values you need inside the test is where matching beats a getter: with a
getter you still have to remember, unaided, that the value only means something in one
particular state.

## Outbound projections: discriminator and payload from one switch

```csharp
var (stage, requestedAt, attempts, resultPath, completedAt, error) = step.Stage switch
{
    StepStage.Pending => (
        StepStageNames.Pending,
        (DateTimeOffset?)null, 0, (string?)null, (DateTimeOffset?)null, (string?)null),
    StepStage.Dispatching d => (
        StepStageNames.Pending, d.RequestedAt, d.Attempts, null, null, d.LastError),

    StepStage.Queued  => (StepStageNames.Queued,  null, 0, null, null, null),
    StepStage.Running => (StepStageNames.Running, null, 0, null, null, null),

    StepStage.Completed c => (StepStageNames.Completed, null, 0, c.ResultPath, c.At, null),
    StepStage.Failed f    => (StepStageNames.Failed,    null, 0, null, f.At, f.Error),
    StepStage.TimedOut t  => (StepStageNames.TimedOut,  null, 0, null, t.At, t.Note),

    _ => throw new NotSupportedException($"Unmapped stage: {step.Stage.GetType().Name}"),
};

row.Stage = stage;
row.DispatchRequestedAt = requestedAt;
row.Attempts = attempts;
row.ResultPath = resultPath;
row.CompletedAt = completedAt;
row.Error = error;
```

**Why they must come out together.** Split into "one switch for the name, one for the
columns" and the second switch can only fill *its* columns — so you must null everything
first and then fill back in. That is a merge, not a projection: forget one reset and the
previous state's residue reads back. Producing them together assigns every column
unconditionally, which makes `read(write(x)) == x` true by construction rather than by
discipline.

## Switch expressions, not switch statements

```csharp
// Bad: add a case, and it silently does nothing
switch (step.Stage)
{
    case StepStage.Dispatching d: row.Error = d.LastError; break;
    ...
}

// Good: add a case, and it throws the first time it is reached
var (...) = step.Stage switch
{
    ...
    _ => throw new NotSupportedException($"Unmapped stage: {step.Stage.GetType().Name}"),
};
```

C# gives no compile-time exhaustiveness over a sealed hierarchy, so `_ => throw` is the
only backstop. **The read side is the exception** — see below.

## The read side must be total

Domain → storage may throw. Storage → domain may not. The database can hold any
historical combination; throwing there lets one bad row take down the whole aggregate load.

```csharp
Stage = row.Stage switch
{
    StepStageNames.Pending when row.DispatchRequestedAt is { } at =>
        new StepStage.Dispatching(at, row.Attempts, row.Error),
    StepStageNames.Pending => new StepStage.Pending(),
    ...
    _ => new StepStage.Pending(),        // fall back, never throw
};
```

Trust the discriminator, treat the rest as payload, substitute defaults for what is
missing. Test it with **every discriminator value paired with an empty payload**: assert
it does not throw and that writing it back yields the same discriminator.

## Two traps

**A property shadows the cases' positional parameters.** Declare
`public RemoteRun? Run => ...` on the union base and each case's positional `Run` resolves
back to the base property — infinite recursion, stack overflow. An accessor on the base
must be a method — though by this document you should not have one at all.

**Constant patterns need `const`.** `row.Stage switch { StepStageNames.Pending => ... }`
is only a legal constant pattern when `StepStageNames.Pending` is a `const string`;
`static readonly` will not compile.
