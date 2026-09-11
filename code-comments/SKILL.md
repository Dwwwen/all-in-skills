---
name: code-comments
description: Decide what a comment should say, and trim ones that say too much. Language-neutral rules — the test a comment must pass, which four kinds to keep, which six to delete, and why history belongs in the commit message. Use when writing a comment, reviewing a diff for comment quality, cleaning up a file whose comments have drifted, or deciding whether a comment earns its place. Applies to code, config, Dockerfiles, shell scripts and build files alike.
---

# Comments

## The test

Before writing a comment, ask: **would a competent reader of this code, who does not know
its history, still be missing this?**

If the answer is no, the comment is a second copy of the code — and copies drift. The code
gets fixed; the sentence above it keeps describing last month's behaviour, and it is now
worse than nothing, because it is read as authoritative.

```python
# Bad: the signature already says this
def cancel_step(job, step_id, at):
    """Cancels the step."""

# Good: says what the signature cannot
def cancel_step(job, step_id, at):
    """Never reached the executor, so no callback is coming — settle it now rather
    than waiting for a confirmation that will never arrive."""
```

## Keep

**Why something must *not* be done.** These are invisible in the code by construction —
the code shows what happens, never what was rejected.

```python
# The books stay open until the executor confirms: that run may still be burning tokens.
stage = Cancelling()
```

**A trade-off a reader would second-guess.** When the obvious alternative is wrong, say
which one you took and what it costs.

```python
# Redeliver rather than stop and wait for a human: a duplicate execution is bounded by
# MAX_ATTEMPTS, whereas a stalled queue needs someone to notice.
```

**A construction that looks optimisable but is not.** Anything a later reader would
"simplify" back into the defect.

```dockerfile
# --retry-all-errors is required: curl does not treat a partial file as retryable, and
# without -C - each retry restarts the 140MB download from zero.
```

**An invariant that spans files.** One place cannot show it, so name the other place.

```python
# The SQL-side spelling of StepStage.is_active. Both must agree.
```

## Delete

| Kind | Example |
|---|---|
| Restating the code | `# increment the counter` above `count += 1` |
| Restating the name | `"""Already queued, not yet started."""` on `class Pending` |
| Teaching the language | explaining what a context manager or a switch expression does |
| Narrating history | "this used to be X, then we changed it to Y because…" |
| Rhetorical build-up | stating the conclusion, arguing it, then restating it |
| Markup that renders nowhere | `<b>` or `🔴` inside a plain `#` comment |

History belongs in the commit message, where it is attached to the change that made it
true. In the file it is unfalsifiable: nobody can tell whether it still applies.

The trap: history is often attached to a rule that *is* worth keeping. Keep the rule,
drop the narration — rewrite "this was a coroutine once and we had an outage" as
"must not be a coroutine: <mechanism>".

## Say it once

A rule stated in three places will be edited in one. When the same constraint governs two
sites, write it at the **source of truth** and point at it from the other.

```python
# At the definition
@property
def is_active(self):
    """Still able to produce usage: the books are open and the sweeper must keep looking."""

# At the other site — a pointer, not a second copy
IS_ACTIVE_SQL = "..."  # The SQL-side spelling of StepStage.is_active.
```

## Evidence is not explanation

Measurements that drove a decision are worth one line, not a paragraph. Keep the number
that would change someone's mind if it were different; drop the rest of the lab report.

```python
# Before: "one run triggered 8 compactions, each squeezing ~170k tokens down to ~20k,
#          1.22M tokens discarded, 20 of 39 minutes spent compacting, and afterwards the
#          model re-reads the whole skill to recover context"
# After:
# Auto-compaction costs more than the call itself — afterwards the model re-reads the
# whole skill to recover context. Long collection runs hit it 8 times in one task.
```

## Length is a signal, not a budget

There is no line limit. But when a function needs three paragraphs, that is usually the
function asking to be split or renamed — the comment is doing work the structure should do.

Try the code first:

```python
# Before: a paragraph explaining that the fallback is not really a signal
# since = step.last_reported_at or step.dispatched_at or step.created_at

# After: the signature refuses the input that needed explaining
def has_timed_out(run: RemoteRun, now: datetime) -> bool: ...
```

A comment that survives that attempt has earned its place.

## Reviewing your own comments

Read the diff with the code hidden. Every sentence that still makes sense on its own is
either load-bearing or a restatement — and restatements are obvious once the code is not
there to lend them meaning.

Comment density is a weak signal (a file of one-line properties is structurally high), so
do not chase a number. Judge them one at a time against the test at the top.

## Language

Comments follow the language of the surrounding codebase. Only the rules above are
prescribed, never the language.

## Doc comments

The rules above govern content. The *form* of a doc comment — which tags exist, what
renders, what tooling reads it — is language-specific; follow the host language's skill
where one exists. Two content rules hold everywhere:

- Put the tooling trap where the reader who is **changing** the member will see it, not
  in the one-line summary a caller hovers.
- Skip the doc comment entirely on a member whose name and signature already say it.
  Public-API completeness is a documentation-generator concern, not a code-quality one.
