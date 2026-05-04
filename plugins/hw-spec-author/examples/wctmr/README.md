# Wallclock Timer (wctmr)

## Overview

This document specifies the Wallclock Timer (wctmr) hardware IP. The block provides a free-running 64-bit counter with two independently programmable compare slots that generate interrupts to the host processor. wctmr is a slave on the configuration bus and follows the standard peripheral integration guideline used by the SoC.

## Features

- 64-bit free-running counter, incrementing once per cycle of `clk_i` after an optional power-of-two prescaler.
- Two independent compare slots (`CMP0`, `CMP1`), each generating a maskable interrupt when the counter reaches or exceeds its programmed compare value.
- Counter and compare values are software-readable and software-writable. Writes to the low half of the counter freeze the increment for one cycle to keep the 64-bit write atomic from software's perspective.
- Programmable prescaler from divide-by-1 (no division) to divide-by-32768 (2^15), selectable at runtime.
- Software-issued counter reset clears the counter to zero without affecting compare values or interrupt state.
- Single clock domain, single reset domain. Reset is synchronous, active-low (`rst_ni`).

## Description

wctmr is intended for OS tick generation, software watchdogs, and one-shot timers. The block is a slave on the configuration bus and contains no DMA-capable master interface.

The counter increments unconditionally while the block is enabled, modulo the prescaler. Software programs a compare slot by writing the 64-bit threshold and enabling the slot; when the counter passes the threshold, an interrupt is asserted. The counter is monotonic and rolls over after 2^64 cycles; software is responsible for interpreting wrap-around if it occurs.

The block has a single clock domain. No CDC paths exist within wctmr itself; the bus interface adapter handles synchronization to the host clock when the two clocks differ.

## Compatibility

wctmr defines a custom register interface. It is not compatible with the ARM Generic Timer or the RISC-V CLINT mtime/mtimecmp interface, although software emulation of either is straightforward.

## Further Reading

- [Theory of Operation](./doc/theory_of_operation.md)
- [Programmer's Guide](./doc/programmers_guide.md)
- [Hardware Interfaces](./doc/interfaces.md)
- [Registers](./doc/registers.md)
- [Design Verification Plan](./dv/plan.md)
