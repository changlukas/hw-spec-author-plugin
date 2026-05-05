# Template: theory_of_operation.md (BFM mode)

**Primary audience:** BFM implementer (C model / SystemC / SystemVerilog), and — when an RTL counterpart exists — RTL implementer of the same block.
**Goal:** Document the internal architecture of every implementation of this block. BFM internal architecture is always required. RTL internal architecture is required when the spec covers a block that also has an RTL implementation; absent if the BFM is purely a verification artifact.

**Protocol applicability:** BFM-mode only. For behavioral-block mode, use `references/templates/02_theory_of_operation.md` instead. Mini-example below uses AXI-Lite slave.

This file complements the wire-level contract (`signal_interface.md`, `pin_level_reset.md`, `protocol_rules.md`, `channel_handshake.md`) and the testbench-API surface (`transaction_api.md`, `channel_api.md`, `active_passive_mode.md`) by describing **how the implementations are structured internally** so a re-implementer can produce a behaviorally equivalent BFM or RTL.

## Required structure

```
# Theory of Operation

## Block diagram

(One Mermaid `flowchart` showing the BFM as a black box plus its testbench-facing
surface and its DUT-facing surface. If an RTL counterpart exists, a second diagram
or extension showing the RTL block structure.)

## BFM internal architecture

This section is **always required** for protocol-bfm mode.

### Driver

Describe the BFM's driver block — per-channel or per-phase state machines, output
registration discipline (e.g. "fully registered, never combinational on inbound
DUT signals"), inputs from sequencer and configuration store.

### Monitor

Describe the BFM's monitor block — per-channel observers, transaction
reconstruction, rule-violation logging, behavior in active vs passive mode.

### Sequencer

Describe the testbench-API surface — how transaction_api.md and channel_api.md
methods translate into driver and monitor activity. Include the in-flight
transaction tracker if applicable.

### Configuration store

List every configuration knob (response_delay / response_fault / register_value /
bfm_mode / etc.) with: default value, persistence (across ARESETn / across
reset_state), one-shot vs persistent, mutual exclusion if any.

### Implementation-specific algorithms

Sub-sections for any non-trivial internal logic that an implementer needs to
reproduce. Examples: response delay countdown implementation, fault injection
flag-clear timing, address-indexed memory representation, priority resolution
when multiple knobs would conflict.

### Reset entry sequencing

Numbered sequence of what happens to the BFM internals when the protocol reset
asserts and deasserts. Cite pin_level_reset.md for wire-level effects.

### Performance commitments

Throughput / latency / resource model from the BFM's perspective. State the
minimum-cycle-count for each transaction type with default configuration.

## RTL internal architecture

**Applies when:** the spec covers a block with an RTL counterpart implementation
(stated in the Block diagram above and in MODE.md `has-rtl-counterpart: yes`).
Omit this section entirely if the BFM has no RTL counterpart.

### RTL block structure

Top-level RTL hierarchy: which RTL modules implement which functional area.
For a typical bus-attached slave:
- Address decoder (which PADDR / AWADDR ranges map to which storage)
- Register file or memory array (size, structure, technology — flop-based vs
  SRAM-based)
- Read multiplexer (how PRDATA / RDATA is selected)
- Write enable logic (how PWDATA / WDATA captures occur, including byte-strobe
  handling)
- Internal FSM (if any — many simple slaves are stateless beyond the protocol
  handshake state)

### RTL pipeline / timing

State the RTL's pipeline depth (typically 1 or 2 cycles for an APB / AXI-Lite
slave). State whether the RTL has fixed timing or variable timing (most simple
slaves are fixed — they do not have a runtime-configurable response delay
equivalent to the BFM's `set_response_delay`).

### RTL reset behavior

Document what every RTL register / memory contents takes on reset. This is more
than the wire-level reset already documented in pin_level_reset.md — pin_level_reset
covers what the BFM/RTL drives on its outputs during reset; this sub-section
covers what the RTL stores internally.

### RTL-vs-BFM behavioral equivalence

For each major behavioral feature the BFM exposes (response delay, fault injection,
register pre-load, active/passive mode), state the RTL counterpart:
- "RTL has fixed N-cycle response time; the BFM's `set_response_delay` is a
  test-only convenience and has no RTL equivalent."
- "RTL has no fault injection knob; the BFM's `set_response_fault` is test-only.
  The RTL only generates real protocol errors (e.g., unmapped address access)."
- "RTL initializes register file to all-zero on reset; the BFM's `set_register_value`
  is test-only pre-load with no RTL equivalent."
- "RTL has no active/passive mode — it is always 'active' in the BFM's sense.
  The BFM's `bfm_mode` is test-infrastructure-only."

This sub-section is what makes the spec a proper contract for **both** implementations:
it states the abstraction gap between BFM (configurable) and RTL (fixed),
explicitly so neither implementer has to guess.

### RTL implementation notes (optional)

Any RTL-specific design decisions worth recording: synthesis pragmas, area /
timing trade-offs, clock-gating opportunities, lint exemptions.
```

## Writing rules for this file

1. **BFM internal architecture is always required.** Even for the simplest BFM, document driver / monitor / sequencer split (or the equivalent in your chosen language framework). Without it, a re-implementer in a different language has nothing to align to.
2. **RTL internal architecture is conditional.** Include it iff the spec covers a block with an RTL implementation. State the condition by:
   - `has-rtl-counterpart: yes` in `MODE.md` (recommended), and
   - Presence of the `## RTL internal architecture` section here.
3. **Do not duplicate wire-level content.** This file is about internal architecture; wire values are in `pin_level_reset.md`, protocol rules are in `protocol_rules.md`. Cross-reference, don't restate.
4. **The RTL-vs-BFM behavioral equivalence sub-section is mandatory when RTL counterpart exists.** This is the load-bearing contract — it states the abstraction gap explicitly so each implementer knows what to ignore from the other side.
5. **No RTL implementation rationale.** Architectural facts (decoder structure, FSM state list, register file size) belong here. Why those choices were made (synthesis trade-off rationale, area-vs-power discussion) belongs in commit messages or design history docs, not in spec.

## Anti-patterns

- **Anti-pattern:** Copying behavioral-block ToO structure verbatim. Behavioral-block ToO has datapath / FSM / errors / performance sections geared to RTL-only blocks. BFM mode ToO splits this by implementation flavor (BFM internal + optional RTL internal).
- **Anti-pattern:** Conflating BFM and RTL internal architecture in one section. They are different implementations with different internal structures. Mixing them produces a spec that fits neither implementer's mental model.
- **Anti-pattern:** Stating "RTL counterpart is obvious from protocol_rules.md" instead of writing the RTL section. Some bus-attached blocks are obvious (a transparent register file slave); many are not (a slave with internal arbitration, a slave that gates response on a side input). Write the section even when it feels obvious — silent assumptions cost more than redundant documentation.
- **Anti-pattern:** Listing every RTL register's reset value here. That belongs in `registers.md` (if the spec has one) or in the RTL's own header comments. This file states the *structural* reset behavior (what gets cleared, what gets randomised, what survives), not the per-register values.
- **Anti-pattern:** RTL implementation notes that drift from RTL source over time (synthesis pragmas, lint exemptions). If those are load-bearing, link to the RTL source comment instead of restating.

## Mini-example: AXI-Lite slave BFM (skeleton — full content in `examples/axi_lite_slave_bfm/doc/theory_of_operation.md`)

```
# Theory of Operation

## Block diagram

[Mermaid diagram showing testbench → BFM (driver/monitor/sequencer/config) → DUT (master),
 plus a second sub-diagram showing RTL block: address decoder → register file → read mux.]

## BFM internal architecture

### Driver
The driver owns five per-channel state machines (AW, W, B, AR, R) implementing
the IDLE → WAITING_VALID → WAITING_READY → ENDED → IDLE lifecycle. Fully registered.
[...]

### Monitor
[...]

### Sequencer
[...]

### Configuration store
- response_delay: (min_cycles, max_cycles), default (0, 0). Persistent across ARESETn.
- response_fault: enum {NONE, SLVERR, DECERR}, default NONE. One-shot.
- register_value: map<addr, value>. Persistent until reset_state().
- bfm_mode: enum {ACTIVE, PASSIVE}, default ACTIVE.

### Implementation-specific algorithms
[Response delay countdown; fault injection clear; etc.]

### Reset entry sequencing
[1. ARESETn asserts → driver state machines reset; etc.]

### Performance commitments
Throughput: single-outstanding per direction. Latency: 1 + max(1, delay) cycles.

## RTL internal architecture

### RTL block structure
Top-level RTL: address decoder → 1024-entry × 32-bit register file (flop-based) →
read multiplexer driving RDATA → write enable per-byte (PSTRB) driving register
file write port. No internal FSM beyond the AXI handshake registered outputs.

### RTL pipeline / timing
1-cycle pipeline: AR handshake on cycle N → RDATA available on cycle N+1.
AW + W handshake on cycles N1, N2 → BVALID on cycle max(N1, N2)+1.
Fixed timing; no runtime delay configuration in RTL.

### RTL reset behavior
On ARESETn assertion: register file is NOT cleared (preserves prior content).
On power-on / first instantiation: register file initialised via Verilog
$readmemh from a file specified at synthesis-time parameter; default empty.

### RTL-vs-BFM behavioral equivalence
- BFM `set_response_delay`: test-only. RTL has fixed 1-cycle response.
- BFM `set_response_fault`: test-only. RTL only generates SLVERR on access to
  unmapped addresses; never DECERR (this slave does not propagate DECERR).
- BFM `set_register_value`: test-only convenience. RTL initial values come from
  $readmemh at instantiation; runtime equivalent is a normal AXI write.
- BFM `bfm_mode`: test-only. RTL is always active.
- BFM `expect_*`: test-only assertions; RTL has no equivalent (RTL is the DUT
  in some tests, the responder in others).

### RTL implementation notes
Synthesis target: ASIC 7nm; register file inferred as flops (no SRAM macro for
this size). Lint exemption: `WIDTH_TRUNC` on PADDR[31:10] zero-check (intentional;
upper bits not used).
```

The full `theory_of_operation.md` for a real BFM/RTL pair will be 150–250 lines. The two-section structure scales by implementation complexity; simple register-file slaves get a short RTL section; pipelined / arbitrated / cached slaves get a long one.
