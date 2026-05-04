# Design Verification Plan

## Verification scope

In scope:
- Functional correctness of the counter, prescaler, both compare slots, and interrupt generation/clearing.
- APB protocol compliance for the configuration slave.
- Reset behavior, including software-driven counter clear.
- Register read/write semantics including reserved bits and `PSLVERR` cases.
- Coverage of all CTRL field combinations within their valid ranges.

Out of scope:
- Post-place-and-route timing (handled at SoC integration).
- Power-aware simulation (block has no clock-gating or power domains).
- Gate-level scan-shift verification.
- Cross-block scenarios (validated at SoC level).

## Testbench architecture

- Methodology: UVM 1.2.
- Simulator: VCS or Verilator (both targets supported via FuseSoC).
- Top-level: `tb/tb_wctmr.sv`, instantiating `wctmr` DUT.
- Agents:
  - `apb_agent`: drives and monitors the configuration APB port.
  - `irq_monitor`: passive monitor on `intr_cmp0_o` and `intr_cmp1_o`.
- Reference model: `wctmr_ref.svh`, a cycle-accurate SystemVerilog class that mirrors the counter, prescaler, and compare logic.
- Scoreboard: cross-checks every register read against the reference model and every IRQ assertion against the model's compare predictions.

## Testpoints

| ID    | Feature reference                          | Testpoint description                                                                     |
|-------|--------------------------------------------|-------------------------------------------------------------------------------------------|
| TP-01 | "64-bit free-running counter"              | Enable block with prescaler=1; verify counter increments by 1 every cycle for 4096 cycles. |
| TP-02 | "Programmable prescaler 1..32768"          | For each `prescale_log2` in {0, 1, 4, 8, 15}, verify counter increments at the expected rate. |
| TP-03 | "Two compare slots independently program"  | Program CMP0 and CMP1 to different thresholds; verify each fires only when its threshold reached. |
| TP-04 | "Compare interrupt"                        | Program CMP0 to a near-future value, enable; verify `intr_cmp0_o` rises on cycle counter ≥ CMP0. |
| TP-05 | "Interrupt cleared by RW1C"                | After IRQ asserted, write 1 to `INTR_STATE.cmp0_match`; verify bit clears and `intr_cmp0_o` deasserts. |
| TP-06 | "Interrupt re-asserts when condition still true" | Clear `INTR_STATE.cmp0_match` while counter still > CMP0 and slot enabled; verify bit re-sets within 2 cycles. |
| TP-07 | "Software counter write atomicity"         | Write `CNT_LO` then `CNT_HI`; verify final 64-bit value equals what was written, with no off-by-one from increment between writes. |
| TP-08 | "Counter wrap-around"                      | Pre-load counter to `0xFFFF_FFFF_FFFF_FFFE`; run 4 cycles; verify rollover to 0x0..1, no error. |
| TP-09 | "Reset clears counter and disables block"  | Assert `rst_ni` mid-operation; deassert; verify counter=0, CTRL=0, both INTR_STATE bits=0. |
| TP-10 | "APB sub-word write returns PSLVERR"       | Issue byte and half-word writes to CTRL; verify `pslverr_o` asserts and CTRL value unchanged. |
| TP-11 | "APB unmapped offset returns PSLVERR"      | Read and write at offsets 0x2C, 0x30, 0x3C; verify PSLVERR; reads return 0. |
| TP-12 | "Reserved bits read as 0"                  | After arbitrary CTRL write, read CTRL; verify bits [15:6] and [31:18] read as 0 regardless of write data. |
| TP-13 | "Two compares fire same cycle"             | Set CMP0 == CMP1, both enabled; verify both `INTR_STATE` bits set in the same cycle. |
| TP-14 | "INTR_TEST forces INTR_STATE"              | Write `INTR_TEST.cmp0_match = 1`; verify `INTR_STATE.cmp0_match` sets, even with no compare condition met. |
| TP-15 | "Coherent 64-bit counter read idiom"       | Run a directed test exercising the hi/lo/hi loop pattern from Programmer's Guide §Use case D; verify retry triggers around rollover. |
| TP-16 | "Out-of-range prescale_log2 clamping"      | Write `CTRL.prescale_log2 = 16, 17, 31`; verify register reads back the written value, no `PSLVERR` is raised, and the effective prescaler period equals `2^PRESCALE_W`. |
| TP-17 | "Reset asserted mid-APB-transaction"       | Drive an APB write or read; assert `rst_ni` between setup and access phases; verify slave outputs return to reset values within 1 `clk_i` cycle and post-reset operation is normal. |

## Functional coverage model

| Covergroup            | Bins / crosses                                                                            |
|-----------------------|-------------------------------------------------------------------------------------------|
| ctrl_cov              | `CTRL.enable` ∈ {0,1}; `prescale_log2` ∈ {0, 1..7, 8..14, 15}; `cmp0_en` ∈ {0,1}; `cmp1_en` ∈ {0,1}; cross of all four. |
| intr_state_cov        | Each `INTR_STATE` bit individually: set, cleared, set-again-after-clear, both-set-same-cycle. |
| counter_value_cov     | Counter at: 0, small values (1..255), mid (~2^31), near-rollover, post-rollover.          |
| compare_match_cov     | Compare value: 0, equal-to-current-counter, far-future, far-past (already exceeded at cmp_en assertion). |
| reset_during_op_cov   | `rst_ni` asserted while: counter incrementing, IRQ pending, mid-APB-write, mid-prescale-period. |
| apb_cov               | Aligned 32-bit access to every register (full read and write); sub-word write to each register; access to unmapped offset. |

## Assertion-based verification (ABV)

| Assertion             | Property                                                                                  |
|-----------------------|-------------------------------------------------------------------------------------------|
| a_apb_handshake       | If `psel_i & ~penable_i` then `psel_i & penable_i & pready_o` next cycle (no stuck setup phase). |
| a_no_x_on_outputs     | `prdata_o` and `intr_*_o` are never X after `rst_ni` deassertion.                         |
| a_intr_propagation    | `INTR_STATE.cmpX_match & INTR_ENABLE.cmpX_match` ↔ `intr_cmpX_o`.                         |
| a_counter_monotonic   | When enabled and prescaler tick, `counter_q[t+1]` == `counter_q[t]+1` (mod 2^64) UNLESS a software write to `CNT_LO`/`CNT_HI` is in progress. |
| a_cnt_lo_freeze       | The cycle following an APB write to `CNT_LO`, the counter does not increment.             |
| a_reset_cleans        | After `rst_ni` deassertion, `counter_q == 0`, `CTRL == 0`, `INTR_STATE == 0`.             |

## Formal property verification (FPV)

FPV is applied to the interrupt logic only:

| Property              | Description                                                                              |
|-----------------------|------------------------------------------------------------------------------------------|
| fpv_intr_consistency  | `intr_cmpX_o` == (`INTR_STATE.cmpX_match` AND `INTR_ENABLE.cmpX_match`) is invariant.    |
| fpv_intr_clear        | After software writes 1 to `INTR_STATE.cmpX_match` and the underlying condition is false, `INTR_STATE.cmpX_match` is 0 within 2 cycles. |

The counter datapath is not under FPV (state space too large; covered by simulation and ABV).

## Security countermeasure testpoints

Not applicable. wctmr has no security countermeasures (see Theory of Operation §Security countermeasures).

## Stages and exit criteria

| Stage | Status      | Exit criterion                                                                          |
|-------|-------------|-----------------------------------------------------------------------------------------|
| V0    | reached     | This DV plan drafted and reviewed.                                                      |
| V1    | not started | All TP-01..TP-15 sanity-passing; coverage model coded; smoke test passes.               |
| V2    | not started | 100% testpoint pass; ≥ 90% functional coverage; ≥ 95% code coverage; all ABV passing.   |
| V3    | not started | 100% functional coverage; ≥ 99% code coverage; FPV properties proven; signoff review.   |
