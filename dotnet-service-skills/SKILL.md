---
name: dotnet-service-skills
description: Conventions for .NET backend services — DDD layering, aggregate design with records and discriminated unions, pattern matching, Orleans actor concurrency, EF Core persistence and migration safety. Use when structuring a service, modelling a state machine, deciding whether to use grains/reminders/streams, or reviewing service code.
---

# .NET service conventions

Every rule here is followed by the class of defect it prevents. When two rules seem to
conflict, decide by "what breaks if I ignore this one", not by which reads nicer.

One running example throughout: a **`Job`** aggregate holding a FIFO queue of
**`JobStep`**s, each dispatched to an **external executor** that can retry, time out,
and be cancelled.

## Five principles

1. **Make illegal states unrepresentable, rather than detectable.**
   Nullable fields whose validity depends on a status field, with nothing enforcing it
   → make it a discriminated union. The bad combination stops compiling.

2. **Store each fact once.**
   A column derivable from other columns is bookkeeping, not information. Two sources
   of the same truth will drift, and a missed sync raises nothing.

3. **The domain does not know how it is stored or displayed.**
   Names, discriminators and concurrency tokens belong to the conversion layer.

4. **Each site asks only what it needs.**
   General-purpose getters and pass-through properties are back doors around the type.

5. **Comments say why, not what.**
   Skip what the code already says. Write down what it cannot: why a terminal state must
   not be set early, why this call must not be awaited.

> Comment *language* follows the codebase. Only the *content* rule above is prescribed.

## Directory shape

```
Application/
  Domain/<Aggregate>/
    JobTypes.cs             records + unions; data and pure derivations only
    JobImplementation.cs    business: (Aggregate, args) -> Aggregate
    JobRules.cs             validation predicates (optional; split out when numerous)
    JobEvents.cs            domain events (optional)
  Persistence/<Aggregate>/  ports: intent only, scalars only
  UseCases/<Aggregate>/<use case>/
Infrastructure/
  Repositories/<Aggregate>/
    JobRow.cs               storage shape: mutable, trackable, all columns
    RowConversions.cs       domain ⇄ row; the only place that knows column semantics
    EfJobStore.cs           repository
    EfJobQueries.cs         read side: AsNoTracking + projection, never via the aggregate
```

The two naming axes are deliberate: **Application is named after the domain** (a port
should not know whether the backing store is blob storage or a queue);
**Infrastructure is named after the external system**.

## Where to go next

| Task | Read |
|---|---|
| Placing code, drawing layer boundaries | `references/layering.md` |
| Designing an aggregate, a state machine, records and unions | `references/domain-modeling.md` |
| Writing matches; projecting domain → storage / wire | `references/pattern-matching.md` |
| Deciding on actors, concurrency, reminders, timers, streams | `references/orleans.md` |
| EF mapping, repositories over immutable aggregates, migrations | `references/persistence.md` |
| Reviewing service code (yours or someone else's) | `references/review-heuristics.md` |

## Maintaining this skill

This document grows from reviews, but **never edit it without approval.**

When a review turns up something that belongs here, do not write to these files. Instead,
end the review with a short proposal:

- the exact text to add or change, ready to paste
- which file it goes in, and which existing rule it edits or sits beside
- the finding that motivated it, in one sentence

Then stop and wait. Apply it only after the human says so. If they decline or change the
wording, that is the answer — the proposal is not re-raised in a later session.

Rationale: a convention accepted without a human reading it is a convention nobody agreed
to, and it will be applied to every future service by an author who never saw it argued.
The generalisation is the risky part, not the finding — one real defect rarely justifies
a rule as broad as it first looks.

A finding is worth proposing only if it meets one of these:

- **It caught a real defect**, and the defect class will recur. Record the rule, and one
  concrete sentence on what went wrong.
- **It settled a disagreement** about how to structure something, and the reasoning would
  otherwise be re-litigated next time.
- **It is a trap in the tooling** (EF, Orleans, the compiler) that cost time to diagnose.

Not eligible: style preferences, one-off domain facts, anything specific to a single
codebase.

Shape the proposal so it is cheap to judge:

1. **Prefer editing an existing rule over appending a new one.** Most findings sharpen a
   rule that is already here. A file that only grows stops being read.
2. Name the reference file that owns the topic; propose a new file only when a genuinely
   new area appears.
3. Keep the shape: the rule, then the defect it prevents. A rule with no consequence
   attached is a preference, and will not survive the next reader.
4. Strip the origin. Examples use the `Job` / `JobStep` / `StepStage` / `RemoteRun`
   vocabulary — never a real project's type names, service names or infrastructure.
5. If it contradicts an existing rule, say so and propose which one wins. Never leave both.
6. Propose one rule at a time. A batch of five gets waved through as a batch.
