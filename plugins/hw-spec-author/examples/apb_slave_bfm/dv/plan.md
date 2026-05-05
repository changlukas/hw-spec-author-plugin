# DV Plan

## Verification scope

Verify the apb_slave_bfm against APB3/APB4 (ARM IHI 0024) and against the BFM-specific extension rules (configuration knobs, single-transaction-at-a-time semantics, active/passive mode).

DV strategy:
1. **Constrained-random testing** — randomised transactions from a master stimulus generator. Validates protocol_rules.md per-phase, response, and configuration rules.
2. **Directed tests for configuration knobs** — verify `set_wait_states`, `set_response_fault`, `set_register_value`, `set_bfm_mode` produce the contracted wire-level effect.
3. **Mode-switch tests** — exercise ACTIVE / PASSIVE transitions with and without in-flight transactions.
4. **Reset tests** — exercise PRESETn assertion mid-transaction at every phase.
5. **ABV** — every protocol_rules.md row maps to one SVA assertion; assertions run continuously.

## Testpoints

| ID  | Feature                          | Testpoint description                                                                                          | Protocol rules exercised |
|-----|----------------------------------|----------------------------------------------------------------------------------------------------------------|--------------------------|
| TP1 | APB3 base compliance             | Master issues 1000 randomised single writes (random addr, data). Verify SETUP+ACCESS handshake, PSLVERR=0.    | All phase + response rules |
| TP2 | APB3 base compliance             | Master issues 1000 randomised single reads. Verify SETUP+ACCESS, PRDATA matches BFM-supplied value.            | All phase + response rules |
| TP3 | APB4 PSTRB byte lanes            | Issue writes with every PSTRB pattern. Verify only strobed bytes change in BFM memory.                          | PSTRB_BYTE_LANES         |
| TP4 | Configurable wait states (fixed) | For wait counts in {0, 1, 2, 5, 10, 100}, verify PREADY rises exactly that many cycles after PENABLE.           | CFG_WAIT_STATES_BOUND    |
| TP5 | Configurable wait states (random)| Configure (min=2, max=10); 100 transactions; verify all wait counts in [2,10] inclusive, ≥7 distinct observed.  | CFG_WAIT_STATES_BOUND    |
| TP6 | One-shot PSLVERR injection       | `set_response_fault()`; issue write; verify PSLVERR=1; issue second write; verify PSLVERR=0 (one-shot cleared). | CFG_FAULT_PSLVERR, PSLVERR_VALID_WITH_PREADY |
| TP7 | Register pre-load                | `set_register_value(0x100, 0xCAFEBABE)`; master reads 0x100; verify PRDATA=0xCAFEBABE.                          | CFG_REGISTER_PRELOAD     |
| TP8 | Register persistence             | Set value, do 5 reads of same address, verify all return same value.                                            | CFG_REGISTER_PRELOAD     |
| TP9 | Register overwrite by write      | Set value, master writes new value, master reads, verify PRDATA = new value.                                     | CFG_REGISTER_PRELOAD     |
| TP10 | Active mode driving             | Default mode; verify PREADY/PRDATA/PSLVERR respond per protocol.                                                | All FAIL-severity rules  |
| TP11 | Passive mode floating           | Switch to PASSIVE; verify PREADY/PRDATA/PSLVERR hold pin_level_reset values.                                    | CFG_MODE_SWITCH          |
| TP12 | Mode switch mid-transaction     | Issue a write; while in ACCESS phase, switch to PASSIVE; verify expect_write returns MODE_SWITCHED_TO_PASSIVE.   | CFG_MODE_SWITCH          |
| TP13 | Reset during SETUP phase        | Master asserts PSEL; PRESETn asserts before PENABLE rises. Verify driver resets; expect_write returns RESET_DURING_TRANSACTION. | RST_OUTPUTS_LOW         |
| TP14 | Reset during ACCESS phase       | Same with PRESETn asserting during ACCESS.                                                                       | RST_OUTPUTS_LOW         |
| TP15 | Reset duration too short        | Pulse PRESETn for 2 cycles; verify BFM logs warning.                                                              | RST_DURATION            |
| TP16 | Channel API back-pressure       | Use Channel API to hold PREADY low for 1024 cycles after PENABLE; verify BFM logs MAX_WAIT.                      | PREADY_TIMEOUT          |

## Coverage model

- **cg_phase_transitions**: bins across (SETUP→ACCESS, ACCESS→IDLE, IDLE→SETUP back-to-back).
- **cg_wait_states**: bins across (0, 1-9, 10-99, 100+ wait cycles).
- **cg_response**: bins across (PSLVERR=0, PSLVERR=1) × (PWRITE=0, PWRITE=1).
- **cg_pstrb**: bins across all PSTRB patterns observed (sampled, not exhaustive).
- **cg_mode_switch**: bins across (ACTIVE→PASSIVE, PASSIVE→ACTIVE), (mid-SETUP, mid-ACCESS, idle).
- **cg_protocol_rule_hits**: one cover-property per rule ID in protocol_rules.md (15 rules: 2 RST + 6 phase + 3 wait-state + 4 response + 4 CFG = 19 rules; cg_protocol_rule_hits has 19 cover bins).

D3 coverage closure goal: 100% bin hit on every covergroup.

## ABV / FPV strategy

**ABV:** Every `FAIL`-severity row in `protocol_rules.md` corresponds to one SVA `assert property (...)`. Total: 16 assertions + 3 cover properties (for RECOMMEND-severity rules) at D1.

**FPV:** Not planned. Same reasoning as the AXI-Lite slave example.

## Out of scope

- APB5 user signals.
- Wake-up handshake (APB does not define one).
- Multi-peripheral routing — the BFM models a single peripheral; testbench-level address decoding is out of scope.
