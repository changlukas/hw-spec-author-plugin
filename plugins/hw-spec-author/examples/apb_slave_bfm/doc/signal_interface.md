# Signal Interface

**Protocol:** APB3 / APB4 (ARM AMBA APB Protocol Specification, IHI 0024).
**Role:** Completer (slave; responds to a Requester / master DUT).
**BFM perspective:** Direction in the table below is from the BFM out to the DUT.

## Naming convention

ARM APB convention: uppercase signal names verbatim from the APB specification. The `P` prefix marks all APB signals.

## Wire table

The wire table excludes PCLK and PRESETn — those are listed in §Protocol clock and reset below.

| Signal   | Direction | Width        | Active | Sample edge | Reset value (see pin_level_reset.md) | Optional in protocol | BFM supports | Notes |
|----------|-----------|--------------|--------|-------------|--------------------------------------|----------------------|--------------|-------|
| PSEL     | input     | 1            | H      | pos clk     | row 1                                | no                   | yes          | High when this peripheral is selected by the master. |
| PENABLE  | input     | 1            | H      | pos clk     | row 2                                | no                   | yes          | High during ACCESS phase. |
| PADDR    | input     | ADDR_WIDTH   | —      | pos clk     | row 3                                | no                   | yes          | Address; valid when PSEL is high. |
| PWRITE   | input     | 1            | H      | pos clk     | row 4                                | no                   | yes          | 1 = write transaction; 0 = read. |
| PWDATA   | input     | DATA_WIDTH   | —      | pos clk     | row 5                                | no                   | yes          | Write data; valid when PSEL && PWRITE. |
| PREADY   | output    | 1            | H      | pos clk     | row 6                                | no                   | yes          | High when slave can complete the ACCESS phase. |
| PRDATA   | output    | DATA_WIDTH   | —      | pos clk     | row 7                                | no                   | yes          | Read data; valid when PSEL && !PWRITE && PREADY. |
| PSLVERR  | output    | 1            | H      | pos clk     | row 8                                | yes (APB3 onward)    | configurable | High to signal error response; see set_response_fault knob. |
| PPROT    | input     | 3            | —      | pos clk     | row 9                                | yes (APB4 only)      | yes          | APB4: protection attributes. Sampled but not enforced; see protocol_rules.md `APB_SLV_PPROT_IGNORED`. |
| PSTRB    | input     | DATA_WIDTH/8 | —      | pos clk     | row 10                               | yes (APB4 only)      | yes          | APB4: byte strobes for write transactions. Honoured when present. |

## Protocol clock and reset

| Signal   | Direction | Description |
|----------|-----------|-------------|
| PCLK     | input     | Protocol clock; all signals sampled on the rising edge. |
| PRESETn  | input     | Active-low protocol reset. Must be held low for at least 4 PCLK cycles before the BFM enters its defined reset state (see protocol_rules.md `APB_SLV_RST_DURATION`). |

## Parameters

| Name       | Type           | Default | Description                                                                |
|------------|----------------|---------|----------------------------------------------------------------------------|
| ADDR_WIDTH | int (1..32)    | 32      | Address bus width.                                                         |
| DATA_WIDTH | int (8/16/32)  | 32      | Data bus width. APB allows 8, 16, or 32; this BFM's primary target is 32. |
| HAS_APB4   | bit            | 1       | If 1, accept and honour PPROT + PSTRB. If 0, those wires are tied 0.       |

## Optional features in / out of scope

- **Supported**: APB3 base set (PSEL/PENABLE/PADDR/PWRITE/PWDATA/PREADY/PRDATA/PSLVERR); APB4 PPROT (sampled-only) and PSTRB (driving byte writes); configurable PSLVERR injection; configurable wait-state count.
- **Not supported**: APB5 user signals — out of scope for this BFM revision. Wake-up signal (CSYSREQ-style) — APB does not define one.

## Channel grouping

APB is a single-channel protocol — one set of signals shared across SETUP and ACCESS phases. There is no per-channel grouping; all signals listed in the Wire table belong to the same logical channel. Protocol-rule IDs therefore use the format `<PROTO>_<ROLE>_<NAME>` (no `<CHANNEL>` field), per the convention defined in `references/templates/bfm/02b_protocol_rules.md` for protocols without channels.
