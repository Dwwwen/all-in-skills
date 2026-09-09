# Persistence: EF Core over immutable aggregates

## Rows are separate from the domain

The domain is an immutable record; EF wants a mutable, trackable, all-columns class.
**One class must not be both.**

```csharp
internal sealed class JobStepRow      // internal: business code never sees a column
{
    public Guid Id { get; set; }
    public string Stage { get; set; } = StepStageNames.Pending;   // discriminator
    public DateTimeOffset? DispatchRequestedAt { get; set; }      // payload columns
    public int Attempts { get; set; }
    public string? ResultPath { get; set; }
    public string? Error { get; set; }
    ...
}
```

**Storage does not mirror the union.** A `(Stage, CreatedAt)` index and `Stage IN (...)`
filters both need a single comparable discriminator column. The nullable columns cannot be
designed away — but they are confined to `RowConversions`, and invisible past that door.

Name the discriminator with the **domain word** (`Stage`, not `Status`). A mismatch here
usually means a deleted concept is still casting a shadow. The outward JSON field name is
a separate question: that is a published contract and changing it is its own decision.

## The repository over an immutable aggregate

`Save` **must take the aggregate**. With a mutable model it relied on the implicit
contract "what the caller mutated is the tracked instance"; once the aggregate is a record
that no longer holds — and it was implicit all along, so anyone could have missed it.

```csharp
public async Task<int> Save(Job job, CancellationToken ct = default)
{
    var row = db.ChangeTracker.Entries<JobRow>()
        .Select(e => e.Entity)
        .FirstOrDefault(r => r.JobId == job.JobId);

    if (row is null)                       // never tracked = new aggregate
    {
        var inserted = job.ToRow();
        inserted.Version = 1;
        inserted.Steps.AddRange(job.Steps.Select(s => s.ToRow()));
        db.Jobs.Add(inserted);
        return await db.SaveChangesAsync(ct);
    }

    job.WriteTo(row);
    SyncSteps(job, row);

    // The version follows the root only: frequent small child writes should not bump it
    db.ChangeTracker.DetectChanges();
    if (db.Entry(row).State == EntityState.Modified) row.Version++;

    return await db.SaveChangesAsync(ct);
}
```

**The concurrency token stays out of the domain.** An aggregate should not know how many
times it has been stored; when a caller needs it, hand back a tuple (`(Job, string ETag)`)
rather than a field on the aggregate.

## Pre-generated keys must be marked Added explicitly

`Guid.CreateVersion7()` produces a non-default primary key. When EF discovers such an
entity only through a tracked navigation it treats it as `Modified` — so the INSERT becomes
an UPDATE that matches no rows, and the data is silently lost.

```csharp
var added = step.ToRow();
row.Steps.Add(added);
db.Entry(added).State = EntityState.Added;   // "in the aggregate, not in the rows" is the authority
```

## EF translation traps

**`.Contains` on an array does not translate.** It binds to
`MemoryExtensions.Contains(ReadOnlySpan<T>, T)`, which EF does not recognise. Declare it
as `IReadOnlyList<>`:

```csharp
public static readonly IReadOnlyList<string> Active = [Pending, Queued, Running, Cancelling];
// EF translates this to Stage IN ('Pending', 'Queued', ...)
```

**Map enum columns with `HasConversion<string>()`**, never the ordinal — inserting an enum
member renumbers everything after it. If the discriminator is already a `string`, no
conversion is needed and the column type is unchanged (`varchar(16)`).

## A migration safety gate

By default EF scaffolds renames as `DropTable + CreateTable` / `DropColumn + AddColumn`,
leaving one line in the build log and blocking nothing. That kind of loss **raises no
error** — by the time anyone notices, there is nothing to recover.

Put the gate **inside the program that runs migrations** (not in a test), so local, CI and
production all pass through it:

```csharp
// Inspect MigrationOperations; destructive ones must be on the allow-list
"DropRedundantQueueStatus:DropColumn:job.QueueStatus",
```

Rules:
- Renames always use `RenameTable` / `RenameColumn` / `RenameIndex`
- To actually drop something, add the key to the allow-list **with a comment stating why
  nothing is lost**
- Stale allow-list entries matching no migration must be reported, or the gate quietly rots
- Support `--validate-only` (for CI) and `--allow-data-loss` (explicit override)

## Deciding whether a column should exist

Ask: **does it carry independent information?**

```
QueueStatus  ⟺ Stopped: QueueStoppedAt is not null / Idle: QueueHead is null / otherwise
SilentSince  ⟺ LastReportedAt + silence window (off by one sweep interval)
```

If it reduces, it is bookkeeping, not information. Its cost is that every state change must
remember to update two places, and a missed update raises nothing. Confirm every read site
before dropping it, and leave that derivation in the allow-list comment afterwards.

## The migration that renames types but not tables

When the domain and rows are split, the mapped CLR type names change while table and column
names do not. EF's model snapshot records entity type names, so **generate a migration with
empty Up/Down** to refresh the snapshot.

Without it, the next person running `migrations add` finds this rename appearing inside
their own change and has to work out whether they caused it. Say in the migration file why
it is empty.
