# Process: Reader Test

The reader test is the single best technique for catching ambiguity, hidden assumptions, and missing corner cases in a hardware spec. Run it before claiming D1.

## Concept

A reader test simulates handing the spec to a smart engineer who has never seen the design and asking them concrete, mechanical questions. If the spec answers them, it is well-formed. If the reader has to guess, the spec has a gap.

The reader is an LLM (Claude) instructed to answer **only from the provided spec**, not from common sense, not from training-data knowledge of similar IPs.

## Protocol

1. **Prepare the spec.** All six files at D1 completeness, no `TODO(designer):` markers.
2. **Generate questions.** Use the question bank below, plus any block-specific ones the author thinks of.
3. **Run the reader.** Feed the spec and one question at a time. The reader must:
   - Quote the spec sentence(s) that answer the question, OR
   - Report "not answered by the spec" if the spec is silent or ambiguous.
4. **Triage results.** For each "not answered" or ambiguous result:
   - If the missing info is in scope of this spec → add it.
   - If the missing info is genuinely out of scope → add a sentence stating it's out of scope and pointing to where it lives.
5. **Re-run** until all questions resolve cleanly.

## Question bank — universal

These apply to any digital block. Run all of them.

### Reset behavior
1. After `rst_ni` deasserts, what is the value of every software-visible register at cycle 0? Cycle 1?
2. After reset deassertion, what is the FSM state, and is the block willing to accept a bus transaction immediately?
3. If reset asserts mid-transaction, what happens to in-flight bus transactions? Are they completed, dropped, or held?
4. Are there any registers that are *not* reset (e.g., RAM contents, scrambling seed)? If so, what is their post-reset state?

### Clock and CDC
5. How many clock domains exist in the block? Name each.
6. For each signal that crosses a clock domain, what synchronizer is used (2FF, handshake, FIFO, gray-coded)?
7. Is there any constraint on the relative frequency of the clocks?

### Bus interface
8. What happens on a sub-word write to a 32-bit-aligned register? (Returns error / writes only addressed bytes / undefined?)
9. What happens on a write to an unmapped offset within the address window?
10. Can the bus interface back-pressure the host (assert wait/not-ready), and if so, under what conditions?

### Software interaction during operation
11. What happens if software writes a control register while the block is busy executing a command? Is the write queued, dropped, immediately effective, or does it cause an error?
12. What happens if software writes 0 to an enable bit while a transaction is in flight? Is the transaction completed first, aborted, or left in an undefined state?
13. Are status register reads racy with hardware updates? Specifically, can a single software read return a torn value (some bits old, some new)?

### Interrupts
14. For each IRQ source, what register and bit indicates it has fired? How is it cleared?
15. If multiple IRQ sources fire in the same cycle, can software determine the order? Or only that they fired?
16. What happens if software clears INTR_STATE while the underlying condition is still active? Does the bit re-set, stay clear, or is the behavior unspecified?

### Errors
17. List every condition that the spec says causes an error response. For each, what specifically does software see?
18. Is there any error that puts the block into an unrecoverable state requiring full reset? If so, name it and the recovery procedure.

### Performance and saturation
19. What is the maximum sustained throughput? Under what input pattern is it achieved?
20. What happens at the input under sustained over-rate input? Drop, back-pressure, or undefined?

## Question bank — add per block type

For specific block classes, add these:

**For DMA / bus master blocks:**
- What is the maximum outstanding transaction count?
- How are responses ordered relative to requests?
- What happens on a bus error response (SLVERR / DECERR)?

**For FIFOs / buffers:**
- What is the FIFO depth at every level (TX, RX, internal)?
- Is the FIFO drained on reset?
- What happens on simultaneous read-and-write at empty/full boundary?

**For configurable / parameterized blocks:**
- What parameter combinations are valid? Is there a constraint matrix?
- Does any parameter combination disable a feature listed in README.md?

**For security-critical blocks:**
- What information leaks through side channels (timing, power)?
- What is the threat model? (See OpenTitan secure design guidelines for reference.)

## How to run the reader

When running the reader test inside this skill:

1. Open a fresh context (or sub-agent if available).
2. Provide all six spec files as input.
3. Provide the question list.
4. Instruct: "Answer only from the provided spec. If the spec does not unambiguously answer, reply 'not answered.' Quote the supporting sentence(s) for every answer you do give."
5. Receive the responses. For each "not answered," ask the author whether the gap is:
   - **In-scope**, must add → fix the spec.
   - **Out-of-scope**, must reference → add a "see X" sentence.
   - **Genuinely undefined**, intentionally → add a sentence stating it is implementation-defined or undefined and why.

The third bucket should be small. Most "not answered" results are real gaps.

## What "passing" looks like

A spec passes the reader test when:

- Every universal question resolves to a quoted sentence in the spec.
- Every block-specific question resolves to a quoted sentence in the spec.
- No question resolves to "not answered" without an explicit out-of-scope or undefined annotation in the spec.

It is fine to add to the spec a brief "Out of scope" or "Implementation-defined" subsection that consolidates these. What is *not* fine is silent ambiguity.

## Why this works

The reader test catches a class of bugs that author review cannot: **the author cannot un-know things**. The author wrote the RTL or has it in their head, so they read the spec through that lens and fill gaps unconsciously. A reader without that context cannot. Every "not answered" is information the author has and the spec lacks.

This is the same idea as rubber-duck debugging, but adversarial and structured.
