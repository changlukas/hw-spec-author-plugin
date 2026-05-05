# Active vs Passive Mode

## Capability table

| Capability                                                                          | Active | Passive |
|-------------------------------------------------------------------------------------|--------|---------|
| Drives outputs to the DUT (AWREADY, WREADY, BVALID, BRESP, ARREADY, RVALID, RDATA, RRESP) | yes    | no      |
| Samples inputs from the DUT (AWVALID, AWADDR, AWPROT, WVALID, WDATA, WSTRB, BREADY, ARVALID, ARADDR, ARPROT, RREADY) | yes | yes |
| Reconstructs transactions from observed pin activity (write addr/data pairs, read addr/return-data pairs) | yes | yes |
| Generates response stimulus via transaction_api.md (`set_register_value` pre-load, `set_response_delay`, `set_response_fault`) | yes | no — knobs accepted but have no driving effect |
| Reports protocol-rule violations per protocol_rules.md                              | yes    | yes     |
| Contributes to coverage hooks per dv/plan.md                                        | yes    | yes     |
| `expect_write` / `expect_read` work for monitoring                                  | yes    | yes — blocks until matching write/read seen, but BFM does not drive response in PASSIVE |
| `expect_read` returns observed RDATA driven by an external slave                    | n/a    | yes     |

## Mode switch

- **Knob name**: `bfm_mode`
- **Type / values**: enum `{ACTIVE, PASSIVE}`
- **Default**: `ACTIVE`
- **API to switch**: `set_bfm_mode(mode)` per transaction_api.md.
- **Effect of switching from ACTIVE to PASSIVE**: All BFM-driven outputs (AWREADY, WREADY, BVALID, BRESP, ARREADY, RVALID, RDATA, RRESP) immediately float to their during-reset values per `pin_level_reset.md` (i.e., AWREADY/WREADY/BVALID/ARREADY/RVALID = 0; BRESP/RRESP = 0x0; RDATA = 0x0). In-flight transaction_api.md calls (e.g., a blocked `expect_write`) are unblocked with status `MODE_SWITCHED_TO_PASSIVE`. The BFM continues to monitor inbound transactions and continues to log protocol-rule violations — only the driving stops. Corresponds to protocol_rules.md `AXI4LITE_SLV_CFG_MODE_SWITCH`.
- **Effect of switching from PASSIVE to ACTIVE**: BFM-driven outputs return to their reset-deassertion values per `pin_level_reset.md` (AWREADY/WREADY/BVALID/ARREADY/RVALID = 0 until the BFM accepts the next transaction). Configuration knobs that were set during passive mode become effective on the first transaction after the switch.
- **Switching mid-transaction**: Permitted, but the BFM logs a warning. The test author should usually call `reset_state()` before switching back to ACTIVE to drop any partial-transaction trackers.

## Common testbench setups

- **Single-master testbench**: one active AXI-Lite slave BFM at address segment 0x0000_0000–0x0000_FFFF; the master DUT drives reads and writes; the slave BFM responds and observes. Most common configuration.
- **Multi-master testbench (SoC integration)**: one active slave BFM as the primary target; passive slave BFMs at the same address segment to cross-check that two parallel masters do not both observe the same write (illegal in the test scenario).
- **DUT integration regression**: passive slave BFM attached to an existing slave port that the DUT-internal master already drives; the passive BFM contributes to coverage and catches protocol violations without changing the DUT's behavior.
- **C model in performance loop**: active BFM provides cycle-accurate response timing; testbench varies `set_response_delay` to characterise master DUT throughput sensitivity to slave latency.
- **Forbidden**: two BFMs in active mode driving the same port. The protocol does not allow multiple drivers; even with one in active mode and one in passive, the active one is unique. The integrator must explicitly declare which BFM drives.

## Reset interaction

Mode is preserved across ARESETn. Reset asserts → all BFM-driven outputs follow pin_level_reset.md (regardless of mode) → on deassertion, the BFM returns to whatever mode was last set. Default is ACTIVE on first reset deassertion after instantiation.
