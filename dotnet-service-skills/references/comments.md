# Comments

## The test

Before writing a comment, ask: **would a competent reader of this code, who does not know
its history, still be missing this?**

If the answer is no, the comment is a second copy of the code — and copies drift. The code
gets fixed; the sentence above it keeps describing last month's behaviour, and it is now
worse than nothing, because it is read as authoritative.

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

## Keep

**Why something must *not* be done.** These are invisible in the code by construction —
the code shows what happens, never what was rejected.

```csharp
// The books stay open until the executor confirms: that run may still be burning tokens.
Stage = new StepStage.Cancelling(),
```

**A trade-off a reader would second-guess.** When the obvious alternative is wrong, say
which one you took and what it costs.

```csharp
// Redeliver rather than stop and wait for a human: a duplicate execution is bounded by
// MaxAttempts, whereas a stalled queue needs someone to notice.
```

**A construction that looks optimisable but is not.** Anything a later reader would
"simplify" back into the defect.

```csharp
// Must materialise as an array: with a target type of IReadOnlyList<T> the compiler
// synthesises a type the serialiser has no codec for.
Steps = steps.Select(ToDto).ToArray(),
```

**An invariant that spans files.** One place cannot show it, so name the other place.

```csharp
/// <summary>The SQL-side spelling of <see cref="StepStage.IsActive"/>. Both must agree.</summary>
```

## Delete

| Kind | Example |
|---|---|
| Restating the code | `// increment the counter` above `count++` |
| Restating the name | `/// <summary>Already queued, not yet started.</summary>` on `record Pending` |
| Teaching the language | explaining what `with` or a switch expression does |
| Narrating history | "this used to be X, then we changed it to Y because…" |
| Rhetorical build-up | stating the conclusion, arguing it, then restating it |
| Markup that renders nowhere | `<b>` inside a `//` line comment |

History belongs in the commit message, where it is attached to the change that made it
true. In the file it is unfalsifiable: nobody can tell whether it still applies.

## Say it once

A rule stated in three places will be edited in one. When the same constraint governs two
sites, write it at the **source of truth** and point at it from the other.

```csharp
// At the definition
/// <summary>
/// Still able to produce usage: the books are open and the sweeper must keep looking.
/// </summary>
public bool IsActive => ...

// At the other site — a pointer, not a second copy
/// <summary>The SQL-side spelling of <see cref="StepStage.IsActive"/>.</summary>
```

## Length is a signal, not a budget

There is no line limit. But when a function needs three paragraphs, that is usually the
function asking to be split or renamed — the comment is doing work the structure should do.

Try the code first:

```csharp
// Before: a paragraph explaining that the fallback is not really a signal
// var since = step.LastReportedAt ?? step.DispatchedAt ?? step.CreatedAt;

// After: the signature refuses the input that needed explaining
private bool HasTimedOut(RemoteRun run, DateTimeOffset now) => ...
```

A comment that survives that attempt has earned its place.

## Doc comments

- One sentence fits on one line: `/// <summary>…</summary>`. Three lines of ceremony
  around six words is noise at every call site that hovers it.
- `<remarks>` is for the reader who is **changing** this member; `<summary>` is for the
  reader who is **calling** it. Do not put a tooling trap in `<summary>`.
- Skip the doc comment entirely on a member whose name and signature already say it.
  Public API completeness is a documentation-generator concern, not a code-quality one.

## Reviewing your own comments

Read the diff with the code hidden. Every sentence that still makes sense on its own is
either load-bearing or a restatement — and restatements are obvious once the code is not
there to lend them meaning.

Comment density is a weak signal (a file of one-line properties is structurally high), so
do not chase a number. Judge them one at a time against the test at the top.

## Language

Comments follow the language of the surrounding codebase. Only the rules above are
prescribed, never the language.
