# Layering and modularity

## Full shape of one aggregate

```
Application/Domain/Jobs/
  JobTypes.cs            JobId · StepStage · JobStep · Job — data only
  JobImplementation.cs   Create / Dispatch / Complete / Cancel — business
  JobRules.cs            PriorityMustBeInRange / JobIsPending — validation predicates
  JobEvents.cs           JobCompleted / JobCancelled
  JobErrorCodes.cs
Application/Repositories/Jobs/
  IJobRepository.cs      the port, plus the plain shapes it hands back
Application/UseCases/Jobs/<use case>/
  Command.cs / CommandHandler.cs / Dtos.cs

Infrastructure/Data/
  AppDbContext.cs        the context, and every row it maps
  JobRow.cs
Infrastructure/Repositories/Jobs/
  JobRepository.cs       implements IJobRepository
  RowConversions.cs      ToDomain / ToRow
```

Split out `JobRules.cs` only once the validations are numerous enough to deserve names.
With a handful, a guard plus a throw inside the transition is enough.

**Every persisted type gets this shape, behaviour or not.** A ledger row with no rules
still gets `Domain/Entries/EntryTypes.cs` and its own `EntryImplementation.cs` — the
Implementation may hold nothing but the factory. Placement follows *what the thing is*,
never *how much behaviour it has today*.

The alternative — a second home for "just data" — puts the boundary on an axis that
moves: the day that type grows one rule it changes directory, and every `using` in the
service changes with it. Classification churn costs more than an Implementation file
holding a single `Create`.

## Rows sit with the context, conversions sit with the repository

`JobRow` belongs in `Data/` next to the context that maps it — not under
`Repositories/Jobs/`. Rows are shared: a maintenance check on the job aggregate
legitimately asks whether any *ledger* row is still pending, and the context maps both.
File the row under one repository and every such read starts looking like a boundary
violation, which is how a service ends up with a second row type for the same table.

`RowConversions` is the opposite — it only ever serves one aggregate, so it lives with
that repository.

## One repository per aggregate: writes and reads together

`IJobRepository` holds `Save` / `Find` **and** the list, detail and status projections.
Two ports over one table give handlers two things to inject with no invariant separating
them, and nothing in the type system says which to reach for.

What actually needed separating is the *query style*, and that is per method, not per
type — see below. Split a second port out only when there is a genuinely different
consumer of the same table (an operations view with its own shape), never reflexively.

## Reads project; they never assemble the aggregate

Lists, details and status lookups go straight from row to DTO: no tracking, no children,
no aggregate. Loading the aggregate buys write-side consistency — concurrency token,
change tracking, child collections — and a page of list rows needs none of it.

The subtler reason is staleness. Within one unit of work a tracked query keeps handing
back the first snapshot it materialised; a long-lived loop polling for "is it finished
yet" against a tracked query never sees the writer's commit, because the writer is in
another scope. Untracked projection re-reads every time.

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

**Each aggregate owns its own Implementation.** `Entry` is constructed from a `Job`, so
`Entry.Book(job, ...)` is tempting to park in `JobImplementation` — put it in
`EntryImplementation`, which may take a `Job` as an argument. Otherwise "every state
change to X is greppable in one file" stops being true for X, and the next person
looking for how entries come into being greps `EntryImplementation` and finds nothing.

**Do not split by caller.** The same transition is usually invoked from several places —
the grain's cancel path, a quota-exhausted path, a background sweeper. Splitting gives
one state machine two homes, and "only one place can change state" is exactly its value:
*illegal transitions throw* only holds while there is a single entry point.

One-line methods still go through Implementation when they carry a rule — here,
monotonicity:

```csharp
public static Job AdvanceCursor(Job job, long seq) =>
    seq > job.Cursor ? job with { Cursor = seq } : job;
```

The cost is one indirection; the return is that every state change is greppable in one file.

## One directory per use case

```
UseCases/Jobs/CancelJob/         Command.cs · CommandHandler.cs · CommandValidator.cs · Dtos.cs
UseCases/Jobs/ListJobs/          Query.cs   · QueryHandler.cs   · Dtos.cs
```

The directory is named for the use case (`<Verb><Noun>`), sits under the aggregate it
serves, and **owns its DTOs** — file names stay the same in every one of them, so the
shape is greppable and a new use case has nothing to invent.

Never share a DTO between two use cases. The moment one of them needs a field, the
other's response grows it too, and its callers now depend on a field that use case never
meant to promise. Copy the record instead; they answer different questions.

## Ports express intent only

```csharp
// Good: the link is a storage concern; the port does not know its entity
void MapRemoteRun(Guid stepId, string jobId, string remoteRunId);

// Bad: leaks a persistence structure into Application
void Add(JobStepRemoteRunRow link);
```

## A port returns only types it owns

The aggregate, or a **read model** declared beside the port in `Models.cs` — never a use
case's DTO.

```csharp
// Repositories/Jobs/IJobRepository.cs
Task<PagedResult<JobListItem>> List(...);      // JobListItem lives in Repositories/Jobs/Models.cs

// Bad: the port now depends on one use case's response shape
Task<PagedResult<ListJobsItemDto>> List(...);
```

A DTO is a published contract that moves for presentation reasons; a port moves for
storage reasons. Tie them together and a second use case wanting the same rows in a
different shape has only bad options — duplicate the repository method, or widen the DTO
into the union of both use cases and quietly promise every caller the extra fields. The
DTO also stops being changeable without touching persistence.

**The check is mechanical: nothing under `Repositories/` imports `UseCases`.** Keep
shared containers like `PagedResult<T>` out of `UseCases/` so that stays true.

The mapping hop is not waste — it is the seam where the two vocabularies are allowed to
differ. The read model says `Input` and `Error` because that is what the column holds;
the DTO says `Prompt` and `Message` because that is what was published. Pin the published
names in a test **over the handler**, not over the repository: assert on the repository
and you have pinned the wrong layer's names.

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

Put the constants next to the union. They are the union's outward vocabulary, and keeping
them adjacent is what stops a new case from being added without a name. The *mapping* from
case to name still lives in each conversion — only the strings are shared.

This is a deliberate exception to "the domain does not know how it is displayed": the names
have to be visible to every layer that projects them, so moving the file only lengthens the
reference chain while making case-and-name drift easier.

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
