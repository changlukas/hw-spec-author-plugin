# Template: signal_interface.md

**Primary audience:** Anyone integrating the BFM at the RTL boundary — testbench author, RTL designer of the DUT facing the BFM, DV engineer auditing the connection.
**Goal:** Every wire between the BFM and the DUT is fully specified — name, direction (from BFM perspective), width, active level, sample edge, reset value linkage, optionality in the protocol, and whether the BFM supports it. Anyone wiring up the BFM should be able to do so without reading the BFM source.

**Protocol applicability:** Protocol-agnostic. Applies to AXI, AXI-Lite, AXI-Stream, APB, AHB, TileLink, CHI, custom protocols. For protocols without channels (e.g., APB), Channel grouping section is omitted. Mini-example uses AXI-Lite.

This file replaces `interfaces.md` for protocol-bfm mode. It is denser and more rigorous: a single missing column here causes silent integration bugs that surface only at simulation time.

## Required structure

```
# Signal Interface

**Protocol:** <name and version, e.g. "AXI4-Lite (IHI0022, ID022613, §B")>
**Role:** <master | slave | passive monitor>
**BFM perspective:** Direction in the table below is from the BFM out to the DUT (for a slave BFM, AWVALID is therefore an input from the master DUT).

## Naming convention

State the suffix / prefix convention used. Example:
> Signal names match the protocol specification verbatim, in uppercase per ARM convention. The protocol spec is the single source of truth for capitalization.

## Wire table

**The Wire table excludes the protocol clock and reset signals** (e.g. ACLK / ARESETn / PCLK / PRESETn). Those are listed in §Protocol clock and reset below. The wire-set parity check (`/spec-lint` LINT-BFM-001) applies only to this Wire table — clock and reset are NOT expected to appear in `pin_level_reset.md`.

| Signal | Direction | Width | Active | Sample edge | Reset value | Optional in protocol | BFM supports | Notes |
|--------|-----------|-------|--------|-------------|-------------|----------------------|--------------|-------|
| <name> | <in/out>  | <N>   | <H/L/—>| <pos clk / neg clk / async> | <see pin_level_reset.md §X> | <yes/no> | <yes/no/configurable> | <free text> |
| ...    | ...       | ...   | ...    | ...         | ...         | ...                  | ...          | ...   |

<For each row:
- Direction is from the BFM's perspective. For a slave BFM driven by a master DUT, AWVALID is "input"; for a master BFM driving a slave DUT, AWVALID is "output".
- Width may reference a parameter (e.g. `ADDR_WIDTH`) declared in the Parameters section.
- Active level applies to single-bit control signals; data buses get "—".
- Sample edge: which clock edge the BFM samples on. "pos clk" = rising edge of the protocol clock. Async means combinational.
- Reset value: cite the row in pin_level_reset.md that defines the during-reset value. Do not duplicate the value here — single source of truth.
- Optional in protocol: yes if the protocol marks the signal as optional (e.g. AXI WSTRB on a write channel where all bytes are always valid). No if mandatory.
- BFM supports: yes/no/configurable. "Configurable" rows must reference the config knob in active_passive_mode.md or transaction_api.md.>

## Protocol clock and reset

| Signal | Direction | Description |
|--------|-----------|-------------|
| <clock name>  | input | <e.g. "Protocol clock; all wires sampled on the rising edge unless noted">  |
| <reset name>  | input | <e.g. "Active-low protocol reset; held for at least N clock cycles by the testbench"> |

State the minimum reset assertion duration the BFM requires. Example: "rst_n must be held low for at least 16 cycles of the protocol clock for the BFM to enter a defined reset state."

## Parameters

| Name | Type | Default | Description |
|------|------|---------|-------------|
| ADDR_WIDTH | int (1..64) | 32 | Address bus width |
| DATA_WIDTH | int (32 or 64) | 32 | Data bus width |
| ID_WIDTH   | int (0..16)    | 0  | ID bus width; 0 = no ID channel |

State every constraint. "DATA_WIDTH must be 32 or 64" is enforceable; "DATA_WIDTH up to 1024" is not (where does that limit come from?).

## Optional features in / out of scope

State which optional protocol features the BFM does and does not support, with a one-line rationale per "no":

- **Supported**: <e.g. "Exclusive access (LOCK signal); narrow transfers via WSTRB">
- **Not supported**: <e.g. "Low-power handshake — out of scope for this BFM revision"; "QoS / REGION channels — DUT under test does not generate them, would only add untested code paths">

## Channel grouping

For protocols that organise wires into named channels (AXI: AW/W/B/AR/R; APB has no channel concept; TileLink-UL: A/D), declare the grouping:

| Channel | Wires |
|---------|-------|
| AW      | AWVALID, AWREADY, AWADDR, AWPROT |
| W       | WVALID, WREADY, WDATA, WSTRB     |
| B       | BVALID, BREADY, BRESP            |
| AR      | ARVALID, ARREADY, ARADDR, ARPROT |
| R       | RVALID, RREADY, RDATA, RRESP     |

The channel names declared here are the canonical strings for protocol-rule IDs in `protocol_rules.md` (`<proto>_<role>_<channel>_<name>`) and for the dependency diagrams in `channel_handshake.md`. Do not invent channel names downstream that don't appear here.
```

## Writing rules for this file

1. **Direction is always from the BFM perspective.** If the BFM is a slave, AWVALID is `input`. If the BFM is a master, AWVALID is `output`. Stating direction "from the protocol perspective" or "from the master perspective" is ambiguous and a frequent source of integration bugs — fix the perspective once at the top of the file and stay there.
2. **Every wire is one row.** No "address bus and data bus" lumped together. The wire table is the authoritative inventory; `pin_level_reset.md` cross-references it row-by-row.
3. **Reset values live in `pin_level_reset.md`.** This file states *which* row defines the reset value (cross-reference); it does not duplicate the value. Duplicates drift.
4. **Optional protocol features are explicit.** "Supported" and "Not supported" are both listed. Silence on an optional feature is ambiguous — does the BFM support it or not?
5. **Channel grouping is canonical.** The strings in the Channel column flow into protocol-rule IDs, handshake diagrams, and channel-API task names. Pick them once, stay consistent.
6. **Match the protocol spec for capitalization.** ARM AMBA uses uppercase (`AWVALID`); some protocols use lowercase (`a_valid` in TileLink). Match the source spec verbatim — that is what testbench authors will grep for.

## Anti-patterns

- **Anti-pattern:** "All AXI4-Lite signals." The reader cannot wire up "all signals" — they need names, widths, and directions in a table.
- **Anti-pattern:** Conflating channel grouping with sub-sections. Channels are an organisational concept (used by protocol-rule IDs and handshake dependencies); they do not become sub-sections of the wire table. Keep one flat table; declare channels separately.
- **Anti-pattern:** Listing "BFM supports: configurable" without naming the config knob. The reader will spend an hour grepping for the knob. Cite the knob name in the Notes column or cross-reference `active_passive_mode.md`.
- **Anti-pattern:** "Direction depends on configuration." Pick a direction; make the configurable behavior explicit in `active_passive_mode.md`. A wire whose direction changes based on knob settings is a design smell — usually the BFM is conflating two distinct ports.
- **Anti-pattern:** Stating widths as "32 typically" or "up to DATA_WIDTH". Widths are exact. If parameterized, write `Width: DATA_WIDTH` and state the constraint on `DATA_WIDTH` in the parameter table.

## Mini-example: AXI-Lite slave BFM

```
# Signal Interface

**Protocol:** AXI4-Lite (ARM IHI0022, ID022613, §B).
**Role:** slave (responds to a master DUT).
**BFM perspective:** Direction below is from the BFM out to the DUT.

## Naming convention

ARM AMBA convention: uppercase signal names verbatim from the AXI4-Lite specification.

## Wire table

| Signal   | Direction | Width      | Active | Sample edge | Reset value (see pin_level_reset.md) | Optional in protocol | BFM supports | Notes |
|----------|-----------|------------|--------|-------------|--------------------------------------|----------------------|--------------|-------|
| AWVALID  | input     | 1          | H      | pos clk     | §AW row 1                            | no                   | yes          |       |
| AWREADY  | output    | 1          | H      | pos clk     | §AW row 2                            | no                   | yes          |       |
| AWADDR   | input     | ADDR_WIDTH | —      | pos clk     | §AW row 3                            | no                   | yes          |       |
| AWPROT   | input     | 3          | —      | pos clk     | §AW row 4                            | no                   | yes          | Sampled but not enforced; see protocol_rules.md AXI4LITE_SLV_AW_AWPROT_IGNORED |
| WVALID   | input     | 1          | H      | pos clk     | §W row 1                             | no                   | yes          |       |
| WREADY   | output    | 1          | H      | pos clk     | §W row 2                             | no                   | yes          |       |
| WDATA    | input     | DATA_WIDTH | —      | pos clk     | §W row 3                             | no                   | yes          |       |
| WSTRB    | input     | DATA_WIDTH/8 | —    | pos clk     | §W row 4                             | yes                  | yes          | Honoured for byte-granular writes |
| BVALID   | output    | 1          | H      | pos clk     | §B row 1                             | no                   | yes          |       |
| BREADY   | input     | 1          | H      | pos clk     | §B row 2                             | no                   | yes          |       |
| BRESP    | output    | 2          | —      | pos clk     | §B row 3                             | no                   | configurable | OKAY by default; SLVERR when configured fault active (see transaction_api.md set_response_fault) |
| ARVALID  | input     | 1          | H      | pos clk     | §AR row 1                            | no                   | yes          |       |
| ARREADY  | output    | 1          | H      | pos clk     | §AR row 2                            | no                   | yes          |       |
| ARADDR   | input     | ADDR_WIDTH | —      | pos clk     | §AR row 3                            | no                   | yes          |       |
| ARPROT   | input     | 3          | —      | pos clk     | §AR row 4                            | no                   | yes          | Sampled but not enforced |
| RVALID   | output    | 1          | H      | pos clk     | §R row 1                             | no                   | yes          |       |
| RREADY   | input     | 1          | H      | pos clk     | §R row 2                             | no                   | yes          |       |
| RDATA    | output    | DATA_WIDTH | —      | pos clk     | §R row 3                             | no                   | yes          |       |
| RRESP    | output    | 2          | —      | pos clk     | §R row 4                             | no                   | configurable | OKAY by default; SLVERR when configured fault active |

## Protocol clock and reset

| Signal  | Direction | Description |
|---------|-----------|-------------|
| ACLK    | input     | Protocol clock; all wires sampled on the rising edge. |
| ARESETn | input     | Active-low protocol reset. Must be held low for ≥ 16 ACLK cycles before the BFM enters its defined reset state. |

## Parameters

| Name       | Type             | Default | Description                            |
|------------|------------------|---------|----------------------------------------|
| ADDR_WIDTH | int (1..32)      | 32      | Address bus width.                     |
| DATA_WIDTH | int (32 or 64)   | 32      | Data bus width. AXI4-Lite restricts this to 32 or 64 — no other values are legal. |

## Optional features in / out of scope

- **Supported**: WSTRB byte-granular writes; configurable BRESP/RRESP fault injection (see transaction_api.md).
- **Not supported**: low-power handshake (CSYSREQ/CSYSACK) — AXI4-Lite makes this optional and the DUT under test does not implement it; out of scope for this BFM revision. USER signals — AXI4-Lite does not define them.

## Channel grouping

| Channel | Wires |
|---------|-------|
| AW      | AWVALID, AWREADY, AWADDR, AWPROT |
| W       | WVALID, WREADY, WDATA, WSTRB     |
| B       | BVALID, BREADY, BRESP            |
| AR      | ARVALID, ARREADY, ARADDR, ARPROT |
| R       | RVALID, RREADY, RDATA, RRESP     |
```

That's the entire wire contract. Twenty rows, four short tables. The integrator can wire up the BFM end-to-end from this file alone.
