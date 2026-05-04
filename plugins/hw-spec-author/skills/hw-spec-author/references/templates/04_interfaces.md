# Template: interfaces.md

**Primary audience:** SoC integrator (the engineer wiring this block into a top-level netlist).
**Goal:** The integrator can connect every signal correctly and configure every parameter without consulting the RTL or the designer.

## Required structure

```
# Hardware Interfaces

## Parameters

| Name              | Type / Width   | Default | Description                          |
|-------------------|----------------|---------|--------------------------------------|
| FIFO_DEPTH        | int            | 16      | TX FIFO depth in entries             |
| AW                | int (1..64)    | 32      | AXI address width                    |
| ASYNC_RESET       | bit            | 0       | If 1, reset is asynchronous          |

<For each parameter: state any constraint (range, power-of-two, dependency
on another parameter). Constraints not stated will be violated.>

## Clocks

| Signal       | Direction | Description                              |
|--------------|-----------|------------------------------------------|
| clk_i        | input     | Primary functional clock                 |
| clk_aon_i    | input     | Always-on clock (only present if PWR_DOMAINS=2) |

## Resets

| Signal       | Direction | Active | Sync | Description                |
|--------------|-----------|--------|------|----------------------------|
| rst_ni       | input     | low    | sync | Reset for clk_i domain     |
| rst_aon_ni   | input     | low    | sync | Reset for clk_aon_i domain |

## Bus interfaces

<For each bus interface (host, device, master, slave), give:
- The protocol (AXI4, AXI4-Lite, APB, TileLink-UL, custom)
- The role (host/master vs. device/slave)
- Address and data width
- Any deviations from the standard
Use a sub-section per bus interface.>

### s_apb (configuration slave)

- Protocol: APB3
- Role: completer (slave)
- Data width: 32
- Address width: 12 (4KB region)
- All registers are 32-bit aligned. Sub-word writes return PSLVERR.

## Sideband signals

<Any signal that is not part of a bus, not a clock, not a reset, and not an
interrupt or alert. Example: idle indication, hardware strap inputs,
debug-mode select, hardware-aware interlocks.>

| Signal          | Direction | Width | Description                                     |
|-----------------|-----------|-------|-------------------------------------------------|
| idle_o          | output    | 1     | High when block is fully quiescent (clock-gating hint) |
| strap_en_i      | input     | 1     | One-shot strap sample enable                    |
| lc_dft_en_i     | input     | 4     | DFT enable from life cycle controller (mubi4_t) |

## Interrupts

| Signal      | Type       | Description                                     |
|-------------|------------|-------------------------------------------------|
| intr_done_o | level      | Asserted while INTR_STATE.done == 1             |
| intr_err_o  | level      | Asserted while INTR_STATE.err == 1              |

State the interrupt type (level or edge), and reference the corresponding
register fields. Software-clearable interrupts are level type, deasserted
when software writes 1 to clear the corresponding INTR_STATE bit.

## Alerts (security-critical events, if applicable)

<If the block participates in a security alerting framework, list each alert.
For each: name, fatal vs. recoverable, triggering condition.

If the block has no alert outputs, state that and remove the section heading
or replace with "This block has no alert outputs."

## Inter-IP signals

<Signals that connect this block directly to another specific IP, bypassing
the bus. Examples: entropy bus to the RNG, key bus to the crypto engine,
clock manager idle hint.>

| Signal               | Direction | Connects to    | Description                       |
|----------------------|-----------|----------------|-----------------------------------|
| edn_req_o            | output    | EDN            | Entropy request                   |
| edn_ack_i            | input     | EDN            | Entropy granted                   |
| edn_data_i[31:0]     | input     | EDN            | Entropy data word                 |
```

## Writing rules for this file

1. **Signal names match RTL.** This file is the integration contract. The integrator copies signal names directly from this table to wire up the design. Mismatch is the most common integration bug.
2. **Direction is from the IP's perspective.** `intr_done_o` is an output of this block. Always.
3. **Width is explicit, never implicit.** Even single-bit signals get `Width: 1` in the table.
4. **Every signal has one row.** No "see below" or "depends on parameters." If a signal is parameterized in width, state `Width: AW` and reference the AW parameter in the parameter table.
5. **Bus interfaces get sub-sections, not just rows.** Bus protocols carry too much per-interface detail (address map, data alignment, deviations) for a single row.

## On naming conventions

OpenTitan-style suffixes — `_i` for input, `_o` for output, `_ni` for active-low input — are recommended but not required. What is required: pick a convention, state it once at the top of the file, and apply it uniformly. Mixing `clk_i` and `clk_in` and `clk` in the same spec is a smell.

State the convention used:
> Naming: `_i` and `_o` suffixes denote module direction (input/output). `_n` infix denotes active-low. `_q` and `_d` denote registered and next-state respectively (RTL-internal, not exposed at the interface).

## Anti-patterns

- **Anti-pattern:** Listing only the bus signals and forgetting sideband. Sideband is where integration bugs live.
- **Anti-pattern:** "All standard AXI signals." Insufficient — state the AXI version, address width, data width, and any optional channels (low-power, USER, REGION) that are or are not present.
- **Anti-pattern:** Over-specifying internal signals. This file is the *external* contract. RTL-internal signals belong in the RTL, not here.
- **Anti-pattern:** Tables with merged cells or ambiguous "varies" entries. If something varies, parameterize it and reference the parameter.

## Mini-example: minimal but complete interface spec

For a small block:

```
# Hardware Interfaces

Naming: `_i` = input, `_o` = output, `_n` = active-low.

## Parameters

| Name      | Type | Default | Description                         |
|-----------|------|---------|-------------------------------------|
| WIDTH     | int  | 32      | Counter width (1..64)               |

## Clocks and resets

| Signal | Direction | Active | Description           |
|--------|-----------|--------|-----------------------|
| clk_i  | input     | —      | Primary clock         |
| rst_ni | input     | low    | Synchronous reset     |

## Bus interface

### s_apb (configuration)

- Protocol: APB3, completer.
- Address width: 8 bits. Data width: 32 bits.
- Sub-word access not supported; returns PSLVERR.

## Interrupts

| Signal       | Type  | Description                              |
|--------------|-------|------------------------------------------|
| intr_match_o | level | Asserted while INTR_STATE.match == 1     |
```

That's it. Three paragraphs and four tables. A complete contract.
