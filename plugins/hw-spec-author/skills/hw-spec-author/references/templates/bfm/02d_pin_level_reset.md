# Template: pin_level_reset.md

**Primary audience:** BFM implementer, RTL designer of the DUT facing the BFM, DV engineer auditing reset behavior.
**Goal:** Every wire's value during reset and immediately after reset deassertion is specified. No "default", no "don't care", no "implementation-defined" without an explicit rationale.

**Protocol applicability:** Protocol-agnostic. Channel column groups rows per `signal_interface.md` §Channel grouping; for protocols without channels (APB, simple custom), the column is omitted. Mini-example uses AXI-Lite.

This file is the most rigorous reset specification in the spec set. Behavioral-block specs state register reset values; BFM specs must state every *wire's* reset value, because a single undriven wire during reset can crash the DUT under test before any meaningful stimulus runs.

The set of wires must match `signal_interface.md` exactly. `/spec-lint` LINT-BFM-001 enforces this.

## Required structure

```
# Pin-Level Reset Behavior

**Reset signal:** <name from signal_interface.md, e.g. ARESETn>
**Reset assertion (active level):** <low | high>
**Minimum assertion duration:** <e.g. "16 ACLK cycles">
**Synchronicity to clock:** <synchronous | asynchronous-assertion-synchronous-deassertion | fully asynchronous>

## During reset (reset asserted)

State the value driven on every wire while the reset signal is in its active state. A "may take any value" entry is acceptable but must be deliberately commented (rare — reset is the one window where the BFM should be fully deterministic).

| Channel | Signal | Value during reset | Notes |
|---------|--------|--------------------|-------|
| AW      | AWVALID | <0 | 1 | X | Z | high-Z>     | <commentary>           |
| AW      | AWREADY | <value>                       | <commentary>           |
| ...     | ...     | ...                           | ...                    |

Every signal from signal_interface.md appears here. Group rows by the channel grouping declared there for readability; the table itself is one flat structure.

## After reset (first cycle of reset deassertion)

State the value visible on every wire on the first clock cycle after reset deasserts. This is the cycle where the BFM transitions from its reset state to its operating state — testbench code observing the BFM at this cycle expects deterministic values.

| Channel | Signal | Value first cycle post-reset | Notes |
|---------|--------|------------------------------|-------|
| AW      | AWVALID | <value or "may take any value with rationale">  | <commentary>  |
| ...     | ...     | ...                                              | ...           |

For wires where the BFM transitions from reset-driven to test-controlled, state the transition clearly: "AWVALID returns to whatever the testbench API call most recently requested; defaults to 0 if no transaction is in flight."

## Reset entry sequencing

If the BFM has multi-cycle reset entry behavior — e.g. flushing an internal queue, dropping outstanding response promises — describe it as a numbered sequence:

1. Reset asserts. Outputs immediately driven to their during-reset values (above).
2. Cycle 1 of reset: <BFM internal state — drop outstanding transactions, etc.>
3. Cycle 2..N of reset: <hold>
4. Reset deasserts. Outputs transition to their post-reset values (above).
```

## Writing rules for this file

1. **Every wire from `signal_interface.md` appears in both tables.** No exceptions. `/spec-lint` LINT-BFM-001 fails the spec if a wire is missing. The two tables are the same wire set; they differ only in the "Value" column.
2. **State the value, not a description.** "0" is acceptable; "low" is acceptable; "deasserted" is not — it is interpretation, and the reader has to look up which polarity counts as deasserted. Every value is a literal: `0`, `1`, `0x0`, `X`, `Z`, `high-Z`.
3. **"May take any value" is rare and must justify itself.** The during-reset table should almost never have such entries. If a wire genuinely cannot be driven (e.g. a tri-state bus with no enable during reset), state that as `Z` or `high-Z`, not "may take any value." After-reset entries may legitimately be "may take any value" for outputs that the testbench will immediately reconfigure.
4. **`X` is acceptable as a value for outputs the BFM intentionally drives `X` during reset** (e.g. data buses where the protocol specifies the receiver must not sample data unless a valid handshake is in progress). Mark it `X` in the value column and add a note explaining why X-driving is the deliberate behavior.
5. **The post-reset row for an input is "as driven by the DUT"**, not "may take any value." From the BFM's perspective, inputs during the post-reset cycle are externally controlled; the BFM's job is to be ready to sample whatever appears.

## Anti-patterns

- **Anti-pattern:** Omitting wires that "obviously" reset to 0. There are no obvious resets in BFM specs. The whole point of this file is to make the obvious explicit so the integration mismatches surface in spec review, not in simulation.
- **Anti-pattern:** A single "All other signals: 0" catch-all row. Defeats LINT-BFM-001 and hides the wires the author didn't think about.
- **Anti-pattern:** "Reset clears the state machine" without listing the wire-level effect. State machines are internal; this file is wire-level. Cross-reference `theory_of_operation.md` for the FSM if needed, but the wire values must be enumerated here.
- **Anti-pattern:** Different wire values in the during-reset table vs. the after-reset table without an explanation in the Reset entry sequencing section. If AWREADY is 0 during reset and 1 the first cycle after, *something* drove the change — say what.
- **Anti-pattern:** Using `—` (em dash) as a value. `—` is not a value; pick `0`, `X`, or `Z` and commit.

## Mini-example: AXI-Lite slave BFM

```
# Pin-Level Reset Behavior

**Reset signal:** ARESETn (defined in signal_interface.md §Protocol clock and reset).
**Reset assertion (active level):** low.
**Minimum assertion duration:** 16 ACLK cycles.
**Synchronicity to clock:** asynchronous assertion, synchronous deassertion to ACLK rising edge.

## During reset (ARESETn == 0)

| Channel | Signal   | Value during reset | Notes |
|---------|----------|--------------------|-------|
| AW      | AWVALID  | as driven by DUT   | input from master DUT — BFM samples but does not act |
| AW      | AWREADY  | 0                  |       |
| AW      | AWADDR   | as driven by DUT   |       |
| AW      | AWPROT   | as driven by DUT   |       |
| W       | WVALID   | as driven by DUT   |       |
| W       | WREADY   | 0                  |       |
| W       | WDATA    | as driven by DUT   |       |
| W       | WSTRB    | as driven by DUT   |       |
| B       | BVALID   | 0                  |       |
| B       | BREADY   | as driven by DUT   |       |
| B       | BRESP    | 0x0 (OKAY)         | Driven to OKAY rather than X to keep waveforms readable; receiver ignores BRESP unless BVALID is high. |
| AR      | ARVALID  | as driven by DUT   |       |
| AR      | ARREADY  | 0                  |       |
| AR      | ARADDR   | as driven by DUT   |       |
| AR      | ARPROT   | as driven by DUT   |       |
| R       | RVALID   | 0                  |       |
| R       | RREADY   | as driven by DUT   |       |
| R       | RDATA    | 0x0...0            | Driven to all-zero rather than X for waveform readability. |
| R       | RRESP    | 0x0 (OKAY)         |       |

## After reset (first ACLK rising edge with ARESETn == 1)

| Channel | Signal   | Value first cycle post-reset | Notes |
|---------|----------|------------------------------|-------|
| AW      | AWVALID  | as driven by DUT             |       |
| AW      | AWREADY  | 0                            | Returns to 1 when the BFM is ready to accept the next address; see channel_handshake.md §AW. |
| AW      | AWADDR   | as driven by DUT             |       |
| AW      | AWPROT   | as driven by DUT             |       |
| W       | WVALID   | as driven by DUT             |       |
| W       | WREADY   | 0                            | Returns to 1 only after AW phase completes for the corresponding write; AXI4LITE_SLV_W_AFTER_AW. |
| W       | WDATA    | as driven by DUT             |       |
| W       | WSTRB    | as driven by DUT             |       |
| B       | BVALID   | 0                            | Asserts only after both AW and W phases complete. |
| B       | BREADY   | as driven by DUT             |       |
| B       | BRESP    | 0x0 (OKAY)                   | Holds OKAY until BVALID asserts with the response for an actual transaction. |
| AR      | ARVALID  | as driven by DUT             |       |
| AR      | ARREADY  | 0                            | Returns to 1 when the BFM is ready to accept the next address. |
| AR      | ARADDR   | as driven by DUT             |       |
| AR      | ARPROT   | as driven by DUT             |       |
| R       | RVALID   | 0                            | Asserts only after AR phase completes and read data is available. |
| R       | RREADY   | as driven by DUT             |       |
| R       | RDATA    | 0x0...0                      |       |
| R       | RRESP    | 0x0 (OKAY)                   |       |

## Reset entry sequencing

1. ARESETn asserts (asynchronous). All BFM-driven outputs in the "During reset" table assert their stated values combinationally — no clock cycle of delay.
2. While ARESETn is low: any outstanding-transaction trackers (write-address-not-yet-acked, read-address-not-yet-acked, response-pending queues) are cleared. Testbench API calls during reset are silently dropped with a warning logged; see transaction_api.md §Behavior under reset.
3. ARESETn deasserts on the rising edge of ACLK. On that same rising edge, BFM-driven outputs transition to their "After reset" values; in particular, AWREADY and ARREADY remain 0 until the testbench enables stimulus acceptance via channel_api.md §Per-channel enable, default-on at reset deassertion.
```

That's the entire reset contract. Two flat tables and a four-line sequencing list. A DUT designer can run the BFM-DUT pair through reset and verify every wire matches this file cycle by cycle.
