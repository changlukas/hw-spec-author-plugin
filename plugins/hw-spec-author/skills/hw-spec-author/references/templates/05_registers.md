# Template: registers.md

**Primary audience:** Software engineer, DV engineer.
**Goal:** A driver author and a UVM RAL author can each generate code from this table without ambiguity.

## Required structure

```
# Registers

## Register map

| Offset | Name          | Access | Reset      | Description                  |
|--------|---------------|--------|------------|------------------------------|
| 0x00   | CTRL          | RW     | 0x00000000 | Top-level control            |
| 0x04   | STATUS        | RO     | 0x00000001 | Operational status           |
| 0x08   | CMD           | WO     | —          | Command issue                |
| 0x0C   | INTR_STATE    | RW1C   | 0x00000000 | Interrupt status (write 1 to clear) |
| 0x10   | INTR_ENABLE   | RW     | 0x00000000 | Interrupt enable             |
| 0x14   | INTR_TEST     | WO     | —          | Force-set interrupts (DV only) |
| 0x20   | DATA_OUT      | RW     | 0x00000000 | Output data register         |

## Register details

For each register, provide:
- Offset, width, access type, reset value
- Field-by-field decomposition with bit ranges
- Per-field access type (a register may be RW overall but contain RO and WO bits)
- Behavior on reset, on write, on read
- Cross-reference to the theory_of_operation.md section that explains the *use* of the field

### CTRL

- Offset: 0x00
- Width: 32
- Access: RW
- Reset: 0x00000000

| Bits  | Field        | Access | Reset | Description                        |
|-------|--------------|--------|-------|------------------------------------|
| [0]   | enable       | RW     | 0     | Block-level enable. See Theory of Operation §2.1. |
| [1]   | reset_req    | WO     | 0     | Self-clearing soft-reset request. See Theory of Operation §3.4. |
| [7:4] | mode         | RW     | 0x0   | Operating mode. 0=normal, 1=loopback, 2=debug, others reserved. |
| [31:8]| —            | RO     | 0     | Reserved. Reads as 0; writes ignored. |

### INTR_STATE

- Offset: 0x0C
- Width: 32
- Access: RW1C (write-1-to-clear)
- Reset: 0x00000000

| Bits  | Field   | Access | Reset | Description                                        |
|-------|---------|--------|-------|----------------------------------------------------|
| [0]   | done    | RW1C   | 0     | Set by hardware when an operation completes. Software writes 1 to clear. |
| [1]   | err     | RW1C   | 0     | Set by hardware on any error. Software writes 1 to clear. See Theory of Operation §5 for error sources. |
| [31:2]| —       | RO     | 0     | Reserved.                                          |

(... one sub-section per register ...)
```

## Access type taxonomy

Use these standard tokens. Don't invent new ones.

| Token   | Meaning                                                               |
|---------|-----------------------------------------------------------------------|
| RW      | Software reads and writes; HW does not modify after reset             |
| RO      | Software reads; writes are ignored or PSLVERR                         |
| WO      | Software writes; reads return 0 or undefined (state which)            |
| RW1C    | Software writes 1 to clear; writing 0 has no effect                   |
| RW1S    | Software writes 1 to set; writing 0 has no effect                     |
| RC      | Read-to-clear; the act of reading clears the field                    |
| HW-only | Hardware updates; software cannot modify (rare; usually a STATUS bit) |
| HWRO    | Hardware updates, software reads only                                 |

If a field has a non-standard behavior (e.g., write 0xDEADBEEF unlocks a guard register), document it explicitly in the description.

## Writing rules for this file

1. **One register per sub-section.** Don't merge "the four mode registers" into one table. RAL generators and driver headers want one register at a time.
2. **Reset values are exact.** `0x00000000`, not "all zero" or "see below." For fields whose reset depends on a strap, state both: `Reset: strap-dependent (see Theory of Operation §6)`.
3. **Reserved bits are explicit rows.** Software needs to know whether reserved bits read as 0, read as the last written value, or trigger PSLVERR on write.
4. **Cross-reference, don't restate.** A register field's *behavior* lives in `theory_of_operation.md`; this file gives the *interface* (offset, width, access, reset) and a one-line summary plus a §-reference.
5. **Match the RTL field name exactly, including case.** `INTR_STATE.done`, `CTRL.mode` — not `INTR_STATE.DONE` or `Ctrl.Mode` unless that is what the RTL uses.

## On reserved bits

State the policy once, at the top:

> Reserved fields read as zero. Writes to reserved fields are ignored. Software should write reserved fields as zero to remain forward-compatible with future revisions.

Or, if the policy is stricter:

> Reserved fields read as zero. Writes to reserved fields with non-zero values cause PSLVERR.

Pick one and apply uniformly.

## On INTR_STATE / INTR_ENABLE / INTR_TEST triplet

Many specs converge on this pattern. If your block has interrupts, use this convention:

- `INTR_STATE` — RW1C. Bit `i` reads 1 when interrupt source `i` has fired and not yet been cleared. Software writes 1 to clear.
- `INTR_ENABLE` — RW. Bit `i` enables the corresponding `INTR_STATE` bit to propagate to the `intr_*_o` output.
- `INTR_TEST` — WO. DV/diagnostic only. Writing 1 sets the corresponding `INTR_STATE` bit, allowing testing of the interrupt path.

The `intr_*_o` output is `INTR_STATE & INTR_ENABLE`, ORed across sources.

State this convention once in the file and apply it. Don't re-derive it per IP.

## Anti-patterns

- **Anti-pattern:** Tables with "depends on mode" in the access column. Split into two tables (one per mode) or document the dependency explicitly.
- **Anti-pattern:** "Default: 0" without specifying width. A 32-bit register reset to 0 and a 4-bit field reset to 0 are written as `0x00000000` and `0x0` respectively. Match width.
- **Anti-pattern:** Naming registers with marketing-friendly names (`AwesomeMode_Cfg`). Use plain functional names. RTL uppercase, fields lowercase, snake_case throughout.
- **Anti-pattern:** Documenting register behavior twice (here and in theory_of_operation.md) and having them drift. This file gives offset/width/reset/access; the *use* lives elsewhere. One source of truth.
