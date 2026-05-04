# Template: programmers_guide.md

**Primary audience:** Software engineer (driver author), DV engineer (test author).
**Goal:** A driver author can write working code from this document alone, without reading RTL.

## Required structure

```
# Programmer's Guide

## Initialization

<The exact sequence software must perform after reset to bring the block into
operational state. Each step references registers by name. Order matters and
the document must say so.>

```c
// Pseudocode is acceptable. C is preferred when the IP has a real DIF.

// 1. Configure clock and prescaler
write(CFG, (PRESCALE_DIV_8 << CFG_PRESCALE_LSB) | CFG_ENABLE);

// 2. Clear any pending interrupts from prior sessions
write(INTR_STATE, 0xFFFFFFFF);

// 3. Enable interrupts of interest
write(INTR_ENABLE, INTR_ENABLE_DONE_MASK);

// 4. Block is now ready for the first command.
```

State which order is *the* tested order. Note that other orders may work but
have not been verified.

## Use case A: <descriptive name, e.g. "Single-shot transmission">

<Walk through the use case: what software writes, what the block does in
response, what software polls or waits on, and what success/failure looks
like. Include a code fragment.>

## Use case B: <e.g. "Chained DMA descriptors">

<As above. Add as many use cases as the block supports.>

## Error handling

<For every error condition listed in theory_of_operation.md "Error and fault
handling", describe what software should do:
- How software detects the error (register read, IRQ)
- What recovery actions are required (clear, re-init, full reset)
- Whether the IP is usable while the error is pending>

## Interrupt handling

<For each IRQ source:
- What event triggers it
- Whether it is edge or level, and how to clear it
- Recommended servicing order if multiple IRQs can be pending
- Any timing constraints (e.g., "must be cleared within N cycles or block stalls")>

## Register accesses during operation

<Subtle and often missed. State explicitly:
- Which registers are safe to write while the block is busy
- Which writes are dropped, queued, or take effect on the next idle
- Whether reads of status registers can race with hardware updates
- Atomicity guarantees (e.g., "writing CMD is single-cycle atomic")>

## Reset handling from software

<If the block has a software-issued reset path:
- How it is triggered
- What happens to in-flight transactions
- How long software must wait before issuing a new command>
```

## Writing rules for this file

1. **Code fragments belong here, not in theory_of_operation.md.** Driver code is the SW reader's primary mental model.
2. **Initialization order is *the* tested order.** Don't write "any of the following sequences works." DV verifies one sequence; that one is the spec.
3. **Every error condition gets a software response paragraph.** If theory_of_operation.md lists an error, programmers_guide.md must say what software does about it. Pair them.
4. **Be explicit about register-write races.** "What happens if I write CMD while the block is busy?" is the most common question SW asks of HW. Answer it preemptively.
5. **No new design behavior is introduced here.** This file describes how to *use* the block; it does not describe new behavior. If you find yourself documenting a behavior here that isn't in theory_of_operation.md, move it.

## Use-case writing style

A use case has three parts: precondition, action, observable result.

**Bad (mixed concern):**
> To send data, write the data to TX_FIFO. The block transmits and raises an interrupt. Note: if the FIFO is full the write is dropped.

**Good (separated, complete):**

> ### Use case A: Single-byte transmission
>
> **Precondition:** Block initialized per the Initialization section. `STATUS.tx_fifo_full == 0`.
>
> **Action:**
> ```c
> if (read(STATUS) & STATUS_TX_FIFO_FULL) { return -EAGAIN; }
> write(TX_FIFO, byte);
> ```
>
> **Observable result:** Within 16 cycles, the byte appears on the `txd_o` pin and `INTR_STATE.tx_done` is set. Software clears the interrupt by writing 1 to `INTR_STATE.tx_done`.
>
> **Failure mode:** If software omits the FULL check and writes anyway, the write is silently dropped (no error indication). The block does not implement back-pressure on the bus.

That last paragraph is the kind of thing that, if missing, becomes a bug found two months into integration.

## Anti-patterns

- **Anti-pattern:** "Refer to RTL for full programming model." The whole point of the spec is that the reader doesn't need to read RTL.
- **Anti-pattern:** Pseudocode that uses non-existent registers. Cross-check every register name against `registers.md` before submitting.
- **Anti-pattern:** "Software should do X." Replace with concrete action: "Software writes 1 to `INTR_STATE.X` to clear the interrupt."
- **Anti-pattern:** Skipping the Register Accesses During Operation section because "it's obvious." It is never obvious. Write it.
