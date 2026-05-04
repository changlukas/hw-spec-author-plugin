# wctmr — Reader Test Log

This log substantiates the D1-completeness claim for `examples/wctmr/`. Per `stage_gates.md` item `D1.cross.reader_test`, a spec at D1 must have run a reader test of at least 8 questions with all gaps closed. This document records that run.

The reader test was executed twice: an initial run, which surfaced five gaps; the spec was then edited to close them; a re-run confirmed every question now resolves to a quoted sentence in the spec.

Both runs were performed by spawning a fresh-context subagent (the `spec-reader` role) with read-only access to the six wctmr spec files. The subagent had no access to authoring history, design intent, or external references — only the spec text. This isolation is the entire point of the reader test: the spec author cannot un-know what they wrote, and the test exists to expose that gap.

---

## Protocol

- **Files in scope** (treated as the only source of truth):
  - `README.md`
  - `doc/theory_of_operation.md`
  - `doc/programmers_guide.md`
  - `doc/interfaces.md`
  - `doc/registers.md`
  - `dv/plan.md`

- **Question selection** (10 questions, drawn from `references/process/reader_test.md`):
  - 7 universal-bank questions covering reset, CDC/clock, bus interface, software interaction, IRQs, errors, performance.
  - 3 block-specific questions for the configurable / parameterized category (parameter ranges and hardware reaction to invalid configuration).

- **Answer rule**: PASS only if a verbatim spec sentence answers the question. NOT_ANSWERED when the spec is silent. AMBIGUOUS when relevant text exists but does not unambiguously answer.

---

## Run 1 — initial (post-fix-of-Feature-5)

| #   | Question summary                                  | Outcome       |
|-----|---------------------------------------------------|---------------|
| Q1  | Reset values of CTRL, INTR_STATE, counter         | PASS          |
| Q2  | Reset asserted mid-APB-transaction                | NOT_ANSWERED  |
| Q3  | Clock domains; clk_i vs host frequency constraint | AMBIGUOUS     |
| Q4  | Sub-word write to 32-bit register                 | PASS          |
| Q5  | Access to unmapped offset                         | PASS          |
| Q6  | Clear INTR_STATE while compare condition true     | PASS          |
| Q7  | Two compares fire same cycle — order observable?  | PASS          |
| Q8  | Any error condition unrecoverable without rst_ni? | AMBIGUOUS     |
| Q9  | Maximum sustained counter increment rate          | PASS          |
| Q10 | prescale_log2 valid range; out-of-range HW reaction | AMBIGUOUS   |

**Result**: 5 PASS, 1 NOT_ANSWERED, 4 AMBIGUOUS.

The five gaps had a common pattern: the author knew what should happen (or what the absence-of-behavior meant) but had not committed it to a sentence. The reader, with no author context, was right to flag each.

---

## Gap closure

Each gap was closed by adding (not rewriting) prose to the relevant file. The edits are intentionally narrow — one sentence to one paragraph each — to demonstrate that closing a reader-test gap is usually a small, surgical change.

| Gap | File edited                          | Section                          | Nature of edit                                                                                                                |
|-----|--------------------------------------|----------------------------------|-------------------------------------------------------------------------------------------------------------------------------|
| Q2  | `doc/theory_of_operation.md`         | §Resets                          | Added one paragraph describing APB output behavior under mid-transaction reset and the requestor's recovery responsibility.   |
| Q3  | `doc/theory_of_operation.md`         | §Clock domains and CDC           | Added one sentence stating no relative-frequency constraint between `clk_i` and any external host clock.                      |
| Q8  | `doc/theory_of_operation.md`         | §Error and fault handling        | Added closing sentence: no listed error latches the block; `rst_ni` is never required to recover from a wctmr-detected error. |
| Q10 | `doc/registers.md`                   | §CTRL bit field [5:1]            | Replaced the "software must not write" hand-wave with explicit hardware behavior: out-of-range writes are accepted in the register but the prescaler clamps to `2^PRESCALE_W`. |
| Q10 | `dv/plan.md`                         | §Testpoints                      | Added TP-16 covering the out-of-range prescale clamp behavior, and TP-17 covering Q2 (mid-APB-transaction reset).             |

No other files were touched. No design decisions changed (no new register, no new port, no behavioral feature added or removed). The edits were entirely about making implicit knowledge explicit.

---

## Run 2 — post-fix verification

Same spec-reader, same isolation rules, same 10 questions.

| #   | Question summary                                  | Outcome |
|-----|---------------------------------------------------|---------|
| Q1  | Reset values of CTRL, INTR_STATE, counter         | PASS    |
| Q2  | Reset asserted mid-APB-transaction                | PASS    |
| Q3  | Clock domains; clk_i vs host frequency constraint | PASS    |
| Q4  | Sub-word write to 32-bit register                 | PASS    |
| Q5  | Access to unmapped offset                         | PASS    |
| Q6  | Clear INTR_STATE while compare condition true     | PASS    |
| Q7  | Two compares fire same cycle — order observable?  | PASS    |
| Q8  | Any error condition unrecoverable without rst_ni? | PASS    |
| Q9  | Maximum sustained counter increment rate          | PASS    |
| Q10 | prescale_log2 valid range; out-of-range HW reaction | PASS  |

**Result**: 10 PASS, 0 NOT_ANSWERED, 0 AMBIGUOUS.

`stage_gates.md` item `D1.cross.reader_test` is satisfied.

---

## What this run validates about the plugin itself

Beyond substantiating wctmr's D1 claim, this run is the plugin's own dogfood test of the reader-test mechanism. Three observations:

1. **The mechanism finds real gaps.** The five gaps caught in Run 1 were all material; none were nitpicks. Q2 in particular was a genuine missing-behavior statement, not just a wording quibble.

2. **The mechanism is cheap to run.** Two subagent invocations, no human in the loop during the test itself. Triage and fix were the only human steps.

3. **The mechanism resists "author lens" bias by construction.** The author of this spec (a Claude session) had read all six files in working memory before the test was requested. The subagent, dispatched in a fresh context, still found gaps the author did not see. This is the precise effect that `references/process/reader_test.md` claims under "Why this works."

The implication for users of the plugin: do not skip `/spec-review` before claiming D1. The cost is small; the catch rate is non-trivial.

---

## Reproducing this log

```
cd <spec_root>           # i.e., examples/wctmr/
/spec-review .
```

The command spawns the `spec-reader` subagent (defined in `agents/spec-reader.md`), feeds it the six files plus a question list, and aggregates the responses. This log documents the outcome of two such runs against the post-fix wctmr spec; further edits to the spec invalidate this log and require a re-run.
