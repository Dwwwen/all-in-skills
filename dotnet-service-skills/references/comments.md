# Comments

The content rules — the test a comment must pass, the four kinds to keep, the six to
delete, "say it once", "length is a signal", and comment language — live in the
**`code-comments`** skill. They are language-neutral; read that first. This file carries
only what is specific to C# and this codebase.

## The test, in C# form

```csharp
// Bad: the signature already says this
/// <summary>Cancels the step.</summary>
public static Job Cancel(Job job, Guid stepId, DateTimeOffset at)

// Good: says what the signature cannot
/// <summary>
/// Never reached the executor, so no callback is coming — settle it now rather than
/// waiting for a confirmation that will never arrive.
/// </summary>
```

## Worked examples of the four "keep" kinds

**Why something must not be done.**

```csharp
// The books stay open until the executor confirms: that run may still be burning tokens.
Stage = new StepStage.Cancelling(),
```

**A trade-off a reader would second-guess.**

```csharp
// Redeliver rather than stop and wait for a human: a duplicate execution is bounded by
// MaxAttempts, whereas a stalled queue needs someone to notice.
```

**A construction that looks optimisable but is not.**

```csharp
// Must materialise as an array: with a target type of IReadOnlyList<T> the compiler
// synthesises a type the serialiser has no codec for.
Steps = steps.Select(ToDto).ToArray(),
```

**An invariant that spans files.**

```csharp
/// <summary>The SQL-side spelling of <see cref="StepStage.IsActive"/>. Both must agree.</summary>
```

## Say it once, with `<see cref>`

C# can point at the source of truth in a way the compiler checks — a renamed member
breaks the `cref`, so the pointer cannot silently rot into a stale copy. Prefer it over
restating the rule.

```csharp
// At the definition
/// <summary>
/// Still able to produce usage: the books are open and the sweeper must keep looking.
/// </summary>
public bool IsActive => ...

// At the other site — a pointer, not a second copy
/// <summary>The SQL-side spelling of <see cref="StepStage.IsActive"/>.</summary>
```

## XML doc comments

- One sentence fits on one line: `/// <summary>…</summary>`. Three lines of ceremony
  around six words is noise at every call site that hovers it.
- `<remarks>` is for the reader who is **changing** this member; `<summary>` is for the
  reader who is **calling** it. Do not put a tooling trap in `<summary>`.
- Markup that renders nowhere is noise: no `<b>` inside a `//` line comment.
