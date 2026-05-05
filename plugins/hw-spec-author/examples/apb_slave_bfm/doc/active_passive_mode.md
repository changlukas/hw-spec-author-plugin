# Active vs Passive Mode

## Capability table

| Capability                                                                          | Active | Passive |
|-------------------------------------------------------------------------------------|--------|---------|
| Drives outputs to the DUT (PREADY, PRDATA, PSLVERR)                                 | yes    | no      |
| Samples inputs from the DUT (PSEL, PENABLE, PADDR, PWRITE, PWDATA, PPROT, PSTRB)    | yes    | yes     |
| Reconstructs transactions from observed pin activity                                | yes    | yes     |
| Generates response stimulus via transaction_api.md (`set_register_value` pre-load, `set_wait_states`, `set_response_fault`) | yes | no — knobs accepted but no driving effect |
| Reports protocol-rule violations per protocol_rules.md                              | yes    | yes     |
| Contributes to coverage hooks per dv/plan.md                                        | yes    | yes     |
| `expect_write` / `expect_read` work for monitoring                                  | yes    | yes     |
| `expect_read` returns observed PRDATA driven by an external slave                   | n/a    | yes     |

## Mode switch

- **Knob name**: `bfm_mode`
- **Type / values**: enum `{ACTIVE, PASSIVE}`
- **Default**: `ACTIVE`
- **API to switch**: `set_bfm_mode(mode)` per transaction_api.md.
- **Effect of switching from ACTIVE to PASSIVE**: All BFM-driven outputs (PREADY, PRDATA, PSLVERR) immediately float to their during-reset values per `pin_level_reset.md` (PREADY=0, PRDATA=0, PSLVERR=0). In-flight transaction_api.md calls unblock with `MODE_SWITCHED_TO_PASSIVE`. The BFM continues to monitor and log protocol-rule violations. Corresponds to `APB_SLV_CFG_MODE_SWITCH` in protocol_rules.md.
- **Effect of switching from PASSIVE to ACTIVE**: Outputs return to reset-deassertion values; configuration knobs become effective on the first transaction after the switch.
- **Switching mid-transaction**: Permitted, but the BFM logs a warning. Test author should usually call `reset_state()` before switching back to ACTIVE.

## Common testbench setups

- **Single-master testbench**: one active APB slave BFM at an address segment; the master DUT issues transactions; slave responds.
- **DUT integration regression**: passive APB slave BFM attached to an existing slave port that the DUT-internal master drives; passive BFM contributes to coverage and catches violations without altering DUT behavior.
- **Forbidden**: two BFMs in active mode driving PREADY/PRDATA/PSLVERR for the same peripheral select. APB does not allow multiple drivers; the integrator must declare which BFM drives.

## Reset interaction

Mode is preserved across PRESETn. Reset asserts → all BFM-driven outputs follow pin_level_reset.md (regardless of mode) → on deassertion, BFM returns to whatever mode was last set. Default ACTIVE on first reset deassertion after instantiation.
