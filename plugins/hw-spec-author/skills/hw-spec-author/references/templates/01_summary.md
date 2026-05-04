# Template: README.md (Summary Index)

**Primary audience:** Everyone (first read).
**Goal:** A reader new to this IP knows in 60 seconds what the block does, what its main features are, and where to read more.

## Required sections, in order

```
# <IP Name>

## Overview

This document specifies <IP Name> hardware IP functionality.

<One-paragraph description: what the block is, where it sits in the SoC,
and what its host-side interface is. Match the level of OpenTitan IP READMEs:
factual, no marketing tone.>

## Features

- <Feature 1: a concrete capability, not a property.>
- <Feature 2>
- ...

## Description

<2–4 paragraphs of plain-prose description. This is the place to convey the
*shape* of the design: how data flows in, what the block does to it, how it
flows out, and what software's role is. No register names yet, no port names
yet — those go in later sections. This is the elevator pitch in technical form.>

## Compatibility

<If this IP is compatible with an existing register interface or industry
standard (e.g., "16550-compatible UART", "ARM PL011-compatible"), state it
here with the deltas. If not compatible with anything, write
"This IP defines a custom register interface." and remove the section
heading-only confusion.>

## Further Reading

- [Theory of Operation](./doc/theory_of_operation.md)
- [Programmer's Guide](./doc/programmers_guide.md)
- [Hardware Interfaces](./doc/interfaces.md)
- [Registers](./doc/registers.md)
- [Design Verification Plan](./dv/plan.md)
```

## Writing rules for this file

1. **Features = capabilities, not adjectives.** Bad: "High-performance design." Good: "Sustains 1 transaction per cycle on the AXI write channel under back-to-back bursts."
2. **Each feature should be testable.** If a feature is in the list, the DV plan must have a testpoint for it. If you can't write the testpoint, the feature description is too vague.
3. **The Description paragraph(s) name no registers and no ports.** Those are introduced in later sections. The summary stays at the *behavioral* level.
4. **Compatibility section is short or omitted.** If there's nothing to say beyond "no compatibility constraint", say that and move on. Don't pad.

## Length guidance

| Block complexity | Target README length |
|---|---|
| Simple peripheral (UART, GPIO, SPI device) | 30–50 lines |
| Mid-complexity (DMA, timer with multiple modes, simple crypto) | 50–100 lines |
| Complex (CPU, NoC router, full crypto engine, memory controller) | 100–200 lines |

If your README is over 200 lines, it stops being a summary. Move material into `theory_of_operation.md` or split sub-features into a separate doc.

## Anti-patterns

- **Anti-pattern:** Repeating the feature list inside the Description paragraph. Pick one.
- **Anti-pattern:** Listing every register in the Features bullets. Features are *capabilities*; registers go in `registers.md`.
- **Anti-pattern:** Putting block diagrams in the README. The block diagram lives in `theory_of_operation.md`. The README is text-only.
- **Anti-pattern:** Marketing tone ("blazing fast", "industry-leading", "highly configurable"). Replace with measurable claims or delete.

## Worked mini-example (a fictional 32-bit timer block)

```
# Wallclock Timer (wctmr)

## Overview

This document specifies the Wallclock Timer (wctmr) hardware IP. The block
provides a free-running 64-bit counter with two independently programmable
compare values that generate interrupts to the host processor. wctmr conforms
to the standard peripheral integration guideline used by the SoC.

## Features

- 64-bit free-running counter, incrementing once per cycle of `clk_timer_i`.
- Two independent compare slots, each generating a maskable interrupt when the
  counter equals or exceeds the programmed compare value.
- Counter and compare values are software-readable and software-writable.
  Writes are atomic on the bus side; the counter freezes for one cycle on a
  software write to its low half.
- Optional prescaler (1, 2, 4, ..., 32768) selectable by software at runtime.
- One reset domain, synchronous to `clk_timer_i`, active-low (`rst_timer_ni`).

## Description

wctmr is intended for OS tick generation and one-shot software timers. The
block is a slave on the configuration bus and contains no DMA-capable master.
Software programs a compare value, optionally enables the corresponding
interrupt, and is signaled when the counter reaches the threshold. The counter
is monotonic and rolls over after 2^64 cycles; software is responsible for
interpreting wrap-around if it occurs.

The block has a single clock domain. No CDC paths exist within wctmr itself;
the bus interface adapter handles synchronization to the host clock when the
two clocks differ.

## Compatibility

wctmr defines a custom register interface. It is not compatible with the ARM
Generic Timer or the RISC-V CLINT mtime/mtimecmp interface, although software
emulation of either is straightforward.

## Further Reading

- [Theory of Operation](./doc/theory_of_operation.md)
- [Programmer's Guide](./doc/programmers_guide.md)
- [Hardware Interfaces](./doc/interfaces.md)
- [Registers](./doc/registers.md)
- [Design Verification Plan](./dv/plan.md)
```

Notice in the example: every Feature is concrete and testable; the Description names no registers or signals; Compatibility is one sentence + one clarification; nothing is hyped.
