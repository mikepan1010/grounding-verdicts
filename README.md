# Grounding Verdicts

A [Claude Code](https://claude.com/claude-code) skill that stops an agent from shipping a
verdict that is *plausible and wrong*.

**中文版：[README.zh-TW.md](README.zh-TW.md)**

## The failure it targets

An agent re-checks an old report. It queries the database. Every number it gets back is
correct. It writes a confident verdict — and the verdict is wrong, because the numbers
belong to a different criterion than the one it thought it was testing.

A real example, reduced to its bones:

> A status table has a `watermark_event_id` column. The watermark sits at 87. The event
> log's latest id for that month is 141. Conclusion: "this month is 54 events behind, but
> the status still says `done` — the status is lying."
>
> All three numbers are right. The verdict is wrong. The production code that defines
> staleness filters events by `source_type`, and the 54 events in question are all of a
> type that is **explicitly excluded**, with a comment in the source explaining why
> (including them would make the line go stale the instant it was confirmed — an infinite
> loop). Under the real criterion, the month is `done`, and `done` is correct.

The data was never the problem. **The attribution of the criterion was.**

This is a nasty class of error because nothing turns red. The query runs, the numbers add
up, the prose reads well, and the conclusion is confidently false.

## What the skill does

It imposes a delivery contract. Every verdict must fill five cells. A cell you cannot
fill is not a cell you skip — it downgrades the verdict to **UNVERIFIED**.

| Cell | Requirement | Not accepted |
|---|---|---|
| **Verdict** | still holds / overturned / needs revision / **unverified** | omitted |
| **Criterion source** | `file:line` of the production code that *defines* the criterion | a table name, a column name, your own inference |
| **What the criterion excludes** | what it explicitly does not cover | left blank (blank = you never read the definition) |
| **Authority** | which production call site actually uses it (grep result) | "the table's name matches" |
| **Independent second method** | a different route to the same answer | re-running the same query |

Plus three hard rules:

1. **Authority is decided by call sites, not by names.** Before calling anything "the
   source of truth" or "the join bridge", grep for who uses it. Domains routinely hold
   several similarly-named tables doing different jobs — and the wrong one still produces
   a beautiful number.
2. **A claim of absence must exhaust the carriers.** Before writing "there is no
   mechanism / no mapping / no record", check all four: columns, side-car audit tables,
   event streams, code paths. Short of that, you may only write "not found in X" — never
   "does not exist".
3. **Summary inconsistent with body = not delivered.** The summary table, the abstract
   and the commit message are all deliverables. Fix the body and leave a stale headline,
   and the wrong version is the one people read.

The skill also carries a list of red-flag sentences ("the numbers look reasonable now, so
this time it's right" — plausibility is not verification) and a standing rule that dated
conclusions in old reports are *hypotheses*, not facts.

## Does it work?

Measured, not asserted. Same four-question exam, same codebase, same read-only database,
independent runs, grading rules registered **before** the runs and not changed afterwards.

| | Without the skill | With the skill |
|---|---|---|
| Cells wrong | **8 / 40** | **0 / 40** |
| Deliverables with ≥1 error | **5 / 10** | **0 / 10** |
| Per model | Opus 2/5, Fable 3/5 dirty | Opus 0/5, Fable 0/5 |

Fisher exact test at the deliverable level: one-tailed *p* = 0.016, two-tailed *p* = 0.033.

The 8 wrong cells in the control group were not spread randomly. They clustered into
exactly three shapes: wrong authority (3), unread exclusion clause (2), single-probe claim
of absence (3). The one question nobody ever got wrong was "count the rows" — a single
probe answers it.

### Limitations, stated plainly

- The exam is one exam, on one codebase, with *n* = 10 per arm. The effect is
  well-evidenced at the outcome level; the exam is not a benchmark.
- **The mechanism is not isolated.** A placebo arm — same prompt, but pointed at a
  same-shaped spec on an unrelated topic (document formatting) — was run 5 times on one
  model and came in at 1/5. The three arms line up as 2/5 (nothing) → 1/5 (placebo) → 0/5
  (this skill), but *n* = 5 separates nothing: Fisher two-tailed *p* = 1.00 against both
  neighbours. Whether the effect belongs to this checklist's content or merely to *having
  a spec to read* is still open. The one placebo failure happened to land on rule ①
  (authority) — a hint, not evidence.
- One passing deliverable violated rule ③ (its summary and body disagreed on a percentage)
  and still cleared the grading key — evidence that the key checks the "authority" half of
  a verdict more strictly than the "number" half.

## Install

```bash
git clone https://github.com/mikepan1010/grounding-verdicts \
  ~/.claude/skills/grounding-verdicts
```

The skill is then available in every project. Claude loads it automatically when the
`description` matches what you are doing, or you can invoke it directly with
`/grounding-verdicts`.

For a subagent, do not rely on automatic loading — put the path in the prompt:

```
Read this work spec first and follow it: ~/.claude/skills/grounding-verdicts/SKILL.md
```

## When to reach for it

Use it when the sentence you are about to write is a **criterion** question:

- Re-checking whether an older report, backlog, or inventory still holds.
- Declaring that something does not exist — no mechanism, no mapping, no record, no caller.
- Naming an authority: "X is the single source of truth / the join bridge / the definition".
- Reading a field that carries an exclusion clause or a threshold (`watermark`, `status`,
  `is_active`, an allowlist).
- Writing a verdict into something durable: a document, a commit message, a report.

Skip it for **data** questions — how many rows, which date, does the test pass. A single
probe answers those, and the checklist only slows you down.

The one-line test: *is the sentence I'm about to write a data question or a criterion
question?* How many, which day, what value — data. Does it count, is it stale, is there
any, who decides — criterion.

## The skill file

Everything above is a description of [`SKILL.md`](SKILL.md), which is the artifact itself.
It is written in Traditional Chinese, is about 60 lines, and is the whole product — there
is no code, no dependency and nothing to build.
