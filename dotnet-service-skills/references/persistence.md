# Persistence

Rules here hold for any store. Examples are EF Core over a relational database because the
rest of this skill is .NET; a document store hits the same problems through different APIs,
and the places where it changes the answer are called out.

## The stored shape is not the domain shape

The domain is an immutable record. Every store wants the opposite: mutable, settable
throughout, carrying ids and version tokens. **One class must not be both.**

```csharp
internal sealed class JobStepRow      // internal: business code never sees a stored field
{
    public Guid Id { get; set; }
    public string Stage { get; set; } = StepStageNames.Pending;   // discriminator
    public DateTimeOffset? DispatchRequestedAt { get; set; }      // payload
    public int Attempts { get; set; }
    public string? ResultPath { get; set; }
    public string? Error { get; set; }
    ...
}
```

**Storage does not mirror the union**, even where the store could hold one. Filtering by
state — a sweep query, `Stage IN (...)`, an index on `(Stage, CreatedAt)` — needs one
comparable discriminator field. A document store can nest the payload under the
discriminator instead of flattening it; what does not change is that the discriminator is
a single indexed field.

The nullable payload fields cannot be designed away. They are confined to the conversion
layer and invisible past that door — `pattern-matching.md` covers why the discriminator and
its payload must come out of a single switch.

Name the discriminator with the **domain word** (`Stage`, not `Status`). A mismatch usually
means a deleted concept is still casting a shadow. The outward JSON field name is a separate
question: that is a published contract, and changing it is its own decision.

## The repository takes the aggregate

`Save` **must take the aggregate as an argument.**

With a mutable model, an argument-less `Save()` worked by an implicit contract: *what the
caller mutated is the instance the store is holding*. Once the aggregate is an immutable
record that is simply false — every change produces a new instance and the store cannot know
which one the caller has. The contract was implicit all along, so nothing warned when it
stopped being true.

```csharp
public async Task<int> Save(Job job, CancellationToken ct = default)
```

**The concurrency token stays out of the domain.** An aggregate should not know how many
times it has been stored. When a caller needs it, hand back a tuple (`(Job, string ETag)`)
rather than putting a field on the aggregate.

Advance the version **for changes to the root only**. A frequent, small child write — a
heartbeat recording "last heard from at" — should not push the aggregate's version, or every
reader collides over progress that concerns none of them.

## Pre-generated ids defeat "is this new?"

Stores decide insert-versus-update by asking whether the key looks unset. Generate ids in
the domain (`Guid.CreateVersion7()`, so they also sort by creation time) and that heuristic
inverts: a brand-new child looks like an existing one, and the insert becomes an update that
matches nothing. **Nothing throws** — the row is simply never written.

The authority is the aggregate, not the key:

```csharp
// "present in the aggregate, absent from the stored children" is what makes it new
var added = step.ToRow();
row.Steps.Add(added);
db.Entry(added).State = EntityState.Added;
```

Same shape elsewhere: a document driver will happily upsert by `_id`. Say insert when you
mean insert.

## Query predicates must survive translation

A predicate that works in memory does not necessarily translate to the store's query
language. The failure is either a runtime exception or — worse — silent client-side
evaluation that pulls the whole collection into the process.

The trap is that an overload chosen at compile time decides it:

```csharp
// As an array this binds to MemoryExtensions.Contains(ReadOnlySpan<T>, T), which EF
// cannot translate. Declared as IReadOnlyList<>, it becomes Stage IN ('Pending', ...).
public static readonly IReadOnlyList<string> Active = [Pending, Queued, Running, Cancelling];
```

So: stay inside the vocabulary the provider documents, and **execute each query against the
store in a test**. A unit test over `IEnumerable` proves nothing about translation.

## Never persist an enum by its ordinal

Insert a member in the middle and every value after it silently changes meaning, in every
record already written.

```csharp
entity.Property(e => e.Status).HasConversion<string>();
```

If the discriminator is already a string, nothing is at risk and no conversion is needed.

## Renames are where data disappears

Where the store has a schema, migration tooling generates the *shape difference* — and a
rename looks exactly like "drop this, add that". EF scaffolds it as
`DropTable + CreateTable` / `DropColumn + AddColumn`, leaves one line in the build log, and
blocks nothing. **That loss raises no error**; by the time anyone notices there is nothing
to recover.

- Renames use the tooling's rename operation, never drop-and-create.
- Put a gate somewhere the pipeline cannot bypass that fails on any destructive operation
  not explicitly acknowledged, and make the acknowledgement state why nothing is lost.
- A schemaless store does not escape this — it moves the problem to the reader, which must
  keep understanding documents written under every past shape. That is why the read side
  must be total (`pattern-matching.md`).

Whether a field should exist at all is a modelling question, not a storage one — see
`domain-modeling.md`.

## When the mapping changes but the storage does not

Splitting the domain from the stored shape renames the mapped types while table and field
names stay put. If the tooling records type names in a model snapshot, it will see a
difference and want to generate something.

Generate that migration deliberately, with an empty up and down, and say in it why it is
empty. Otherwise the next person finds the rename inside *their* change and has to work out
whether they caused it.

## Reads do not go through the aggregate

Lists, details and status lookups project straight from the store: no tracking, no children,
no aggregate assembly. Loading an aggregate buys write-side consistency — concurrency token,
change tracking, child collections — and a page of list rows needs none of it.
See `layering.md`.
