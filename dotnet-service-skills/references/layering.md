# Layering and modularity

## Full shape of one aggregate

```
Domain/Jobs/
  JobTypes.cs            JobId · StepStage · JobStep · Job — data only
  JobImplementation.cs   Create / Dispatch / Complete / Cancel — business
  JobRules.cs            PriorityMustBeInRange / JobIsPending — validation predicates
  JobEvents.cs           JobCompleted / JobCancelled
  JobErrorCodes.cs
Repositories/Jobs/
  RowConversions.cs      ToDomain / ToRow
  EfJobStore.cs
  EfJobQueries.cs
UseCases/Jobs/<use case>/
  Command.cs / CommandHandler.cs / Dtos.cs
```

Split out `JobRules.cs` only once the validations are numerous enough to deserve names.
With a handful, a guard plus a throw inside the transition is enough.

## What belongs in Types

- Strongly-typed ids: `sealed record JobId(string Value)`
- Closed string sets that are *not* state machines:
  `readonly record struct JobChannel(string Value)` with static members
- Unions (state machines, polymorphic forms)
- The aggregate root record
- **Pure derivations only**: `QueueHead => Steps.FirstOrDefault(s => !s.IsQueueCompleted)`, `IsActive`

Never: anything that changes state, anything that knows about storage or presentation.

## What belongs in Implementation

Uniform shape: `(Aggregate, args) -> Aggregate`, returning a **new** aggregate.

The test is **whether there is a rule**, not whether the caller happens to be a grain
or a handler:

| Goes in Implementation | Stays out |
|---|---|
| Legal-origin checks (state transitions) | Plain field assignment, `job with { UpdatedAt = at }` |
| Fields that must change together (settling also dequeues, and clears the failure note) | A single field, assigned directly |
| Max / dedupe / clamp rules | |
| Cross-object consistency (stage and remote binding must land together) | |

**Do not split by caller.** The same transition is usually invoked from several places —
the grain's cancel path, a quota-exhausted path, a background sweeper. Splitting gives
one state machine two homes, and "only one place can change state" is exactly its value:
*illegal transitions throw* only holds while there is a single entry point.

One-line methods still go through Implementation when they carry a rule:

```csharp
public static Job MarkRefunding(Job job) => job with { RefundState = RefundState.Refunding };
```

The cost is one indirection; the return is that every state change is greppable in one file.

## Ports express intent only

```csharp
// Good: the link is a storage concern; the port does not know its entity
void MapRemoteRun(Guid stepId, string jobId, string remoteRunId);

// Bad: leaks a persistence structure into Application
void Add(JobStepRemoteRunRow link);
```

## One conversion file per exit from the domain

A domain object leaves the domain through two or three doors. Give each its own file:

```
Infrastructure/Repositories/Jobs/RowConversions.cs   domain → row
Grains/Jobs/GrainDtoConversions.cs                   domain → grain DTO
```

When both flatten the same union into a name, **the name constants must be shared**:

```csharp
public static class StepStageNames
{
    public const string Pending = "Pending";
    public const string Running = "Running";
    ...
}
```

Read-side queries project the stored string straight to the client, while the grain DTO
goes through the in-memory aggregate. If the two drift, the same object shows a different
status on two endpoints.

## Reads and writes are separate paths

Read models (lists, details, status lookups) **never go through the aggregate**:
`AsNoTracking` plus a direct projection.

```csharp
db.Jobs.AsNoTracking()
    .Where(job => job.OwnerId == ownerId && !job.IsDeleted)
    .Select(job => new JobListItemDto(job.JobId, job.Title, job.Status.ToString(), job.UpdatedAt))
```

Loading an aggregate buys write-side consistency: optimistic concurrency, child
collections, change tracking. A list query needs none of it. Routing reads through the
aggregate pays for aggregate assembly on every page view.

## Two sets of test fixtures

- **Domain fixtures** (`JobFixtures`): build aggregates, assert business rules.
- **Row fixtures** (`RowFixtures`): build rows, assert read-side projections.

What the read side must prove is "the projection still holds for *any* historical
combination in the database". Building those cases through write-side fixtures imposes
"what the write side can construct" — which is precisely the constraint under test.
