# Signal Interface

**Protocol:** AXI4-Lite (ARM IHI0022, ID022613, §B).
**Role:** slave (responds to a master DUT).
**BFM perspective:** Direction in the table below is from the BFM out to the DUT. For a slave BFM driven by a master DUT, AWVALID is therefore an input from the master DUT.

## Naming convention

ARM AMBA convention: uppercase signal names verbatim from the AXI4-Lite specification.

## Wire table

| Signal   | Direction | Width        | Active | Sample edge | Reset value (see pin_level_reset.md) | Optional in protocol | BFM supports | Notes |
|----------|-----------|--------------|--------|-------------|--------------------------------------|----------------------|--------------|-------|
| AWVALID  | input     | 1            | H      | pos clk     | §AW row 1                            | no                   | yes          |       |
| AWREADY  | output    | 1            | H      | pos clk     | §AW row 2                            | no                   | yes          |       |
| AWADDR   | input     | ADDR_WIDTH   | —      | pos clk     | §AW row 3                            | no                   | yes          |       |
| AWPROT   | input     | 3            | —      | pos clk     | §AW row 4                            | no                   | yes          | Sampled but not enforced; see protocol_rules.md `AXI4LITE_SLV_AW_AWPROT_IGNORED`. |
| WVALID   | input     | 1            | H      | pos clk     | §W row 1                             | no                   | yes          |       |
| WREADY   | output    | 1            | H      | pos clk     | §W row 2                             | no                   | yes          |       |
| WDATA    | input     | DATA_WIDTH   | —      | pos clk     | §W row 3                             | no                   | yes          |       |
| WSTRB    | input     | DATA_WIDTH/8 | —      | pos clk     | §W row 4                             | yes                  | yes          | Honoured for byte-granular writes. |
| BVALID   | output    | 1            | H      | pos clk     | §B row 1                             | no                   | yes          |       |
| BREADY   | input     | 1            | H      | pos clk     | §B row 2                             | no                   | yes          |       |
| BRESP    | output    | 2            | —      | pos clk     | §B row 3                             | no                   | configurable | OKAY by default; SLVERR/DECERR when configured fault active (see transaction_api.md `set_response_fault`). |
| ARVALID  | input     | 1            | H      | pos clk     | §AR row 1                            | no                   | yes          |       |
| ARREADY  | output    | 1            | H      | pos clk     | §AR row 2                            | no                   | yes          |       |
| ARADDR   | input     | ADDR_WIDTH   | —      | pos clk     | §AR row 3                            | no                   | yes          |       |
| ARPROT   | input     | 3            | —      | pos clk     | §AR row 4                            | no                   | yes          | Sampled but not enforced. |
| RVALID   | output    | 1            | H      | pos clk     | §R row 1                             | no                   | yes          |       |
| RREADY   | input     | 1            | H      | pos clk     | §R row 2                             | no                   | yes          |       |
| RDATA    | output    | DATA_WIDTH   | —      | pos clk     | §R row 3                             | no                   | yes          |       |
| RRESP    | output    | 2            | —      | pos clk     | §R row 4                             | no                   | configurable | OKAY by default; SLVERR/DECERR when configured fault active. |

## Protocol clock and reset

| Signal  | Direction | Description |
|---------|-----------|-------------|
| ACLK    | input     | Protocol clock; all wires sampled on the rising edge. |
| ARESETn | input     | Active-low protocol reset. Must be held low for ≥ 16 ACLK cycles before the BFM enters its defined reset state (see protocol_rules.md `AXI4LITE_SLV_RST_DURATION`). |

## Parameters

| Name       | Type           | Default | Description                                                                                  |
|------------|----------------|---------|----------------------------------------------------------------------------------------------|
| ADDR_WIDTH | int (1..32)    | 32      | Address bus width. Constraint: 1..32 inclusive.                                              |
| DATA_WIDTH | int (32 or 64) | 32      | Data bus width. AXI4-Lite restricts this to 32 or 64 — no other values are legal.            |

## Optional features in / out of scope

- **Supported**: WSTRB byte-granular writes; configurable BRESP/RRESP fault injection (`set_response_fault` per transaction_api.md).
- **Not supported**: low-power handshake (CSYSREQ/CSYSACK) — AXI4-Lite makes this optional and is out of scope for this BFM revision. USER signals — AXI4-Lite does not define them.

## Channel grouping

| Channel | Wires |
|---------|-------|
| AW      | AWVALID, AWREADY, AWADDR, AWPROT |
| W       | WVALID, WREADY, WDATA, WSTRB     |
| B       | BVALID, BREADY, BRESP            |
| AR      | ARVALID, ARREADY, ARADDR, ARPROT |
| R       | RVALID, RREADY, RDATA, RRESP     |
