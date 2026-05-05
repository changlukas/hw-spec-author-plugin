# DV Plan

## Verification scope

Verify the axi_lite_slave_bfm against the AXI4-Lite specification (ARM IHI0022 §B) and against the BFM-specific extension rules (configuration knobs, single-outstanding semantics, active/passive mode). The DV strategy combines:

1. **Constrained-random testing** — randomised transactions from a master stimulus generator, exercising every channel in every legal handshake order. Validates `protocol_rules.md` per-channel and cross-channel rules.
2. **Directed tests for configuration knobs** — verify `set_response_delay`, `set_response_fault`, `set_register_value`, `set_bfm_mode` produce the contracted wire-level effect (corresponds to all `AXI4LITE_SLV_CFG_*` rules).
3. **Mode-switch tests** — exercise ACTIVE → PASSIVE and PASSIVE → ACTIVE transitions with and without in-flight transactions.
4. **Reset tests** — exercise ARESETn assertion mid-transaction at every channel state; verify pin_level_reset.md tables are honoured cycle by cycle.
5. **Assertion-based verification (ABV)** — every protocol_rules.md row maps to one SVA assertion in the BFM testbench; assertions run continuously during all tests.

## Testpoints

Each testpoint covers at least one Feature from `README.md` §Features and references the protocol-rules it exercises. Coverage hooks (covergroup names) listed in §Coverage model below.

| ID  | Feature                          | Testpoint description                                                                                          | Protocol rules exercised |
|-----|----------------------------------|----------------------------------------------------------------------------------------------------------------|--------------------------|
| TP1 | AXI-Lite compliance              | Master issues 1000 randomised single writes (random addr, data, WSTRB pattern). Verify B handshake, BRESP=OKAY. | All AW, W, B per-channel rules; XCH_W_AFTER_AW; XCH_B_AFTER_AW_AND_W |
| TP2 | AXI-Lite compliance              | Master issues 1000 randomised single reads. Verify R handshake, RDATA matches BFM-supplied value, RRESP=OKAY.   | All AR, R per-channel rules; XCH_R_AFTER_AR |
| TP3 | Single-outstanding (write)       | Master issues a write, then a second write before the first B is acked. Verify AWREADY back-pressure on second.  | XCH_NO_OUTSTANDING_WRITE |
| TP4 | Single-outstanding (read)        | Same pattern for reads.                                                                                          | XCH_NO_OUTSTANDING_READ  |
| TP5 | WSTRB byte lanes                 | Issue writes with every WSTRB pattern (0001, 0010, ..., 1111). Verify only strobed bytes change in BFM memory.   | W_WSTRB_BYTE_LANES       |
| TP6 | Configurable response delay      | For delays in {0, 1, 2, 5, 10, 100, 1000} cycles, verify BVALID asserts exactly that many cycles after the latest of AW/W handshake. | CFG_RESPONSE_DELAY_BOUND |
| TP7 | Response delay random range      | Configure (min=2, max=10); issue 100 transactions; verify all delays land in [2,10] inclusive and at least 7 distinct values are observed. | CFG_RESPONSE_DELAY_BOUND |
| TP8 | Response fault BRESP             | `set_response_fault(SLVERR)`; issue write; verify BRESP=2'b10; issue second write; verify BRESP=2'b00 (one-shot cleared). | CFG_FAULT_BRESP, B_BRESP_VALUES |
| TP9 | Response fault RRESP             | Same for reads with SLVERR and DECERR.                                                                            | CFG_FAULT_RRESP, R_RRESP_VALUES |
| TP10 | Register pre-load               | `set_register_value(0x100, 0xCAFEBABE)`; master reads 0x100; verify RDATA=0xCAFEBABE.                            | CFG_REGISTER_PRELOAD    |
| TP11 | Register persistence            | Set value, do 5 reads of same address, verify all return same value.                                              | CFG_REGISTER_PRELOAD    |
| TP12 | Register overwrite by write     | Set value, master writes new value, master reads, verify RDATA = new value (not pre-loaded value).                 | CFG_REGISTER_PRELOAD    |
| TP13 | Active mode driving             | Default mode; verify all 8 BFM-driven outputs respond per protocol.                                                | All FAIL-severity rules |
| TP14 | Passive mode floating           | Switch to PASSIVE; verify AWREADY/WREADY/BVALID/BRESP/ARREADY/RVALID/RDATA/RRESP all hold pin_level_reset values. | CFG_MODE_SWITCH         |
| TP15 | Mode switch mid-transaction     | Issue a write; while waiting for B, switch to PASSIVE; verify expect_write returns MODE_SWITCHED_TO_PASSIVE.       | CFG_MODE_SWITCH         |
| TP16 | Reset during AW phase           | Master raises AWVALID; ARESETn asserts before AWREADY. Verify all driver state machines reset; expect_write returns RESET_DURING_TRANSACTION. | RST_OUTPUTS_LOW         |
| TP17 | Reset during W phase            | Same for W phase.                                                                                                  | RST_OUTPUTS_LOW         |
| TP18 | Reset during B phase            | Same for B phase.                                                                                                  | RST_OUTPUTS_LOW         |
| TP19 | Reset duration too short        | Pulse ARESETn for 8 cycles (less than 16); verify BFM logs warning and behavior is not guaranteed.                | RST_DURATION            |
| TP20 | Channel API back-pressure       | Use Channel API to hold AWREADY low for 1024 cycles after AWVALID; verify BFM logs MAX_WAIT observation.          | AW_AWREADY_MAX_WAIT     |

## Coverage model

Covergroups, each binned across the rules it exercises:

- **cg_aw_handshake**: bins across (AWVALID-before-AWREADY, AWVALID-after-AWREADY, simultaneous), (response_delay = 0, 1-9, 10-99, 100+), (fault SLVERR, DECERR, NONE).
- **cg_w_handshake**: bins across (WVALID-before-WREADY, WVALID-after-WREADY, simultaneous), (WSTRB = each of 16 patterns for 32-bit data; each of 256 patterns for 64-bit data — sampled-only, not exhaustive).
- **cg_b_handshake**: bins across (BREADY-before-BVALID, BREADY-after-BVALID, simultaneous), (BRESP = OKAY, SLVERR, DECERR).
- **cg_ar_handshake**, **cg_r_handshake**: analogous to cg_aw_handshake / cg_b_handshake.
- **cg_xch_write_ordering**: bins across (AW-before-W, W-before-AW, AW-and-W-same-cycle).
- **cg_back_pressure**: bins across (single-outstanding write back-pressured, single-outstanding read back-pressured, both simultaneously).
- **cg_mode_switch**: bins across (ACTIVE→PASSIVE, PASSIVE→ACTIVE), (mid-write, mid-read, idle).
- **cg_protocol_rule_hits**: one cover-property per rule ID in protocol_rules.md (28 rules total: 2 RST + 6 AW + 4 W + 3 B + 6 AR + 3 R + 5 XCH + 5 CFG = 34 rules; cg_protocol_rule_hits has 34 cover bins).

D3 coverage closure goal: 100% bin hit on every covergroup; in particular every protocol-rule cover-property must hit at least once.

## ABV / FPV strategy

**ABV (always-on SVA assertions, paired with regression tests):**
- Every `FAIL`-severity row in `protocol_rules.md` corresponds to one SVA `assert property (...)` in the BFM testbench. ID matches the rule ID (e.g., `assert_AXI4LITE_SLV_AW_AWVALID_STABLE`).
- Every `RECOMMEND`-severity row corresponds to a `cover property (...)` rather than `assert property` — violation does not fail the test, but the cover provides observability.
- Total: 28 assertions + ~6 cover properties at D1.

**FPV (formal property verification):**
- Not planned. The BFM is not on a security-critical path; the protocol rules are well-understood and exhaustive simulation provides adequate confidence.
- If a future revision adds security-critical features (e.g., access-control checks), revisit FPV.

## Out of scope

- Full AXI4 burst semantics (length, size, burst type, address wrap) — not part of AXI-Lite.
- ID-bearing channel verification — AXI-Lite has no ID channels.
- Exclusive access (AXI4 LOCK) — AXI-Lite does not support exclusive access.
- Cache coherency snoop responses — AXI-Lite is not cache-coherent.
- Low-power handshake (CSYSREQ/CSYSACK) — explicitly excluded per signal_interface.md §Optional features.
