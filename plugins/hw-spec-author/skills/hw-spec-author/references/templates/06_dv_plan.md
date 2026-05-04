# Template: dv/plan.md

**Primary audience:** DV engineer (testbench author, coverage planner).
**Goal:** A DV engineer can build a verification plan and coverage model from this document, with every feature in the spec mapped to at least one testpoint.

## Required structure

```
# Design Verification Plan

## Verification scope

<One paragraph stating what this testbench covers and what it does not.
Out-of-scope items (e.g., post-place-and-route timing, DFT scan, gate-level
power) should be named explicitly.>

## Testbench architecture

<Brief description: simulator, methodology (UVM, cocotb, plain SV TB),
top-level TB structure, agents, scoreboards. Reference reusable common
verification components if the project has them.>

## Testpoints

<A test*point* is a verification objective, not a test*case*. One testpoint
may map to multiple testcases. Every Feature in README.md maps to at least
one testpoint here.>

| ID    | Feature                                  | Testpoint description                                       |
|-------|------------------------------------------|-------------------------------------------------------------|
| TP-01 | Single-byte transmission                 | Drive a series of single bytes through TX_FIFO; verify each appears on txd_o with correct framing. |
| TP-02 | Back-to-back transmission                | Drive bytes with no software-side gap; verify no underrun. |
| TP-03 | TX FIFO overflow handling                | Write to TX_FIFO when FULL; verify silent drop and no IRQ. |
| TP-04 | Software-issued reset during transmission| Issue CTRL.reset_req mid-transmission; verify FIFO is cleared and block returns to IDLE within 4 cycles. |
| ...   | ...                                      | ...                                                         |

## Functional coverage model

<List each covergroup and its purpose. Cross-product crosses should be named
explicitly.>

| Covergroup            | Bins / Crosses                                            |
|-----------------------|-----------------------------------------------------------|
| cmd_cov               | All values of CTRL.mode (0..2); cross with CTRL.enable    |
| intr_state_cov        | Each INTR_STATE bit individually, set and cleared         |
| fifo_level_cov        | FIFO occupancy at empty, 1, mid, full-1, full             |
| reset_during_op_cov   | Reset asserted in each FSM state                          |

## Assertion-based verification (ABV)

<List the SVA assertions that are part of the design contract — not RTL
implementation assertions, but assertions that capture *spec-level*
behavior.>

| Assertion              | Property                                                         |
|------------------------|------------------------------------------------------------------|
| a_intr_propagation     | INTR_STATE[i] && INTR_ENABLE[i] |-> intr_*_o[i]                  |
| a_no_x_on_bus          | s_axi.* signals are never X after reset deassertion              |
| a_fsm_reachable        | Every state in tx_ctrl_fsm is reachable                          |
| a_fifo_no_overflow     | FIFO write attempt while full does not corrupt occupancy counter |

## Formal property verification (FPV) targets, if applicable

<If the block uses FPV (recommended for FSMs and security-critical paths),
list the properties proven and any constraints.>

## Security countermeasure testpoints (if applicable)

<If the block has security countermeasures listed in theory_of_operation.md
"Security countermeasures", every countermeasure has a corresponding
testpoint here. Often auto-generated from a sec_cm metadata file in mature
projects.>

## Stages and exit criteria

<Cite the stage gates from process/stage_gates.md. State current stage
and what is needed to reach the next.>

| Stage | Status     | Exit criterion                                                       |
|-------|------------|----------------------------------------------------------------------|
| V0    | reached    | DV plan drafted (this document)                                      |
| V1    | in progress| All TP-01..TP-NN sanity-passing on a smoke test; coverage model coded |
| V2    | not started| 100% testpoint pass; >= 90% functional coverage; >= 95% code coverage |
| V3    | not started| 100% functional coverage; >= 99% code coverage; signoff review        |
```

## Writing rules for this file

1. **Every Feature in README.md maps to at least one testpoint here.** This is the audit trail. Maintain it through spec revisions.
2. **A testpoint is what to verify, not how.** "Verify back-to-back transmission" is a testpoint. "Run test `tx_btb_test`" is a testcase. Don't conflate.
3. **Coverage model is concrete from day one.** Don't write "TBD coverage model." If you can't name covergroups, the spec doesn't yet specify enough behavior — go fix the spec.
4. **Assertions cited here are spec-level.** RTL-implementation assertions (e.g., "this counter never wraps in this implementation") are RTL concerns. ABV here covers things the RTL must satisfy regardless of how it's coded.

## On the relationship between Features, Testpoints, Coverage

```
README.md Features       Theory of Operation       DV plan testpoints
──────────────────────   ──────────────────────    ──────────────────────
"Sustains 1 trans/cyc"   "Pipeline depth 3, no     TP: back-to-back stress
 under back-to-back"      stall under full-rate"     test, full-rate input
                                                     for 1024 cycles, observe
                                                     output rate.
```

Each layer adds detail. README says *what*, ToO says *how*, DV plan says *how do we know it works*. The three should never contradict.

## Anti-patterns

- **Anti-pattern:** Listing testpoints that don't map back to any spec feature. Either the testpoint is unnecessary, or the feature was missing from README.
- **Anti-pattern:** "TBD" in the coverage model column. If undefined, the spec is undefined.
- **Anti-pattern:** Confusing code coverage and functional coverage thresholds. State both, separately.
- **Anti-pattern:** Skipping the "verification scope" paragraph. Without it, V2 sign-off arguments about whether something is in or out of scope are unwinnable.
